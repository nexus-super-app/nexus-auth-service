# Contrato de Eventos Domínio Nexus

---

## 🎯 Princípio: "One Event, One Meaning"

**Regra:** Um evento = uma mudança de estado do domínio  
**Exceção:** Nenhum evento deve depender de state externo (apenas aggregate + outbox)

---

## 📜 Envelope Format (Universal)

```json
{
  "eventId": "uuidv7-timestamp-random",    // Sempre UUIDv7, nunca UUIDv4
  "eventType": "UserRegisteredEvent",       // Nome PascalCase, único
  "occurredAt": "2024-01-15T10:30:00.000Z", // ISO-8601 UTC
  "aggregateId": "uuidv7-user-here",        // Originador do evento (número único)
  "aggregateType": "user-account",          // Nome do bounded context
  "payload": { ... }                        // Dados específicos do evento
}
```

### Regras de Validação

| Campo | Regra |
|-------|-------|
| `eventId` | **Nunca** gerar UUIDv4, sempre UUIDv7 (tempo-ordenado) |
| `eventType` | PascalCase, nome único no domínio (ex: `UserRegisteredEvent`, não `user_created`) |
| `occurredAt` | UTC, sem timezone offset (Z suffix), milliseconds precision |
| `aggregateId` | UUIDv7 do aggregate que originou o evento |
| `aggregateType` | Nome do bounded context (`user-account`, `message-conversation`, etc) |

---

## 📊 Eventos por Bounded Context (Atual: Auth Only)

### `UserRegisteredEvent` (Auth → Messaging, Analytics, etc)

**Quando:** UserAccount agregado criado após registro bem-sucedido  

**Payload:**
```json
{
  "phoneNumber": "+5511999999999",
  "profile": {
    "name": "João Silva",
    "email": "joao@example.com",
    "avatarUrl": null
  },
  "createdAt": "2024-01-15T10:30:00Z",
  "initialContactsCount": 0
}
```

**Consumidores:**
- `messaging-service`: Cria UserAccount local, sync profile
- `analytics-service`: Registra registro de usuário ativo
- `notification-service` (opcional): Push "Bem-vindo ao Nexus!"

**Idempotência:** Por `eventId` (não por `phoneNumber` ou `occurredAt`)  

**Retry Logic:** 3x retry com backoff exponencial (1s, 2s, 4s) após 5min de inatividade do consumer  

---

### `UserBlockedEvent` (Auth → Auth, Analytics, etc)

**Quando:** Usuário A bloqueia Usuário B  

**Payload:**
```json
{
  "blockedUserId": "uuidv7-user-b",    // Quem foi bloqueado
  "blockerId": "uuidv7-user-a",        // Quem bloqueou (aggregateId do originador)
  "blockedAt": "2024-01-15T10:35:00Z"
}
```

**Consumidores:**
- `auth-service`: Remove UserAccount B dos contatos de A em DB (consistência eventual)  
- `analytics-service`: Registra bloqueio, motivo opcional  

**Idempotência:** Por `eventId` + par `(blockerId, blockedUserId)`  

---

### Eventos Futuros (Messaging → Auth, etc)

### `MessageSentEvent`

**Quando:** Mensagem enviada para uma conversation  

**Payload:**
```json
{
  "conversationId": "uuidv7-conversation",
  "senderId": "uuidv7-sender-user",
  "content": {
    "type": "text",
    "value": "Olá mundo!"
  },
  "attachments": []
}
```

**Consumidores:**
- `auth-service`: Opcional, sync última mensagem no perfil  
- `messaging-service` (self): Notifica participantes  

---

### `AttachmentUploadedEvent`

**Quando:** Arquivo subido via S3/MinIO  

**Payload:**
```json
{
  "conversationId": "uuidv7-conversation",
  "senderId": "uuidv7-sender-user",
  "attachment": {
    "filename": "foto.jpg",
    "mimeType": "image/jpeg",
    "sizeBytes": 123456,
    "url": "http://minio.local/bucket/path/foto.jpg"
  }
}
```

**Consumidores:**
- `auth-service`: Opcional  
- `analytics-service`: Registra upload de media  

---

## 🔐 JWT Contract (Auth → Todos)

### Access Token

**Claims:**
```json
{
  "userId": "uuidv7-user",              // Sempre UUIDv7, nunca inventar novo ID
  "phoneNumber": "+5511999999999",
  "iat": 1705315800,                    // Unix timestamp seconds
  "exp": 1705319400,                    // TTL: 1 hora (padrão), rotacionável
  "scope": ["read:profile", "write:contacts"],
  "type": "access"
}
```

**Validação por Consumidor:**
- **Slice 0 (apenas auth):** Skip JWKS, assume JWT válido se assinatura OK  
- **Slice 1+:** Usar JWKS ou shared key para validar assinatura  

**TTL Padrão:**
- Access: 1 hora (rotacionável)
- Refresh: 7 dias (não rotacionável por now, Phase 2)

---

### Refresh Token

**Claims:**
```json
{
  "userId": "uuidv7-user",
  "phoneNumber": "+5511999999999",
  "iat": 1705315800,
  "exp": 1706522400,                    // TTL: 7 dias
  "type": "refresh"
}
```

**Validação:** Sem JWKS inicial, por now  
**Rotacionável?** Não até Phase 2 (quando auth-service gerencia sessão centralizada)  

---

## 📡 Event Publishing (Transactional Outbox Pattern)

### Fluxo no Auth-Service (Java/Kotlin + Postgres)

```kotlin
@Transactional
fun registerUser(phoneNumber: String, profile: Profile): Result<RegisterResponse> {
    val user = userAccountRepository.save(UserAccount(
        id = generateUuidv7(),           // UUIDv7 sempre!
        phoneNumber = phoneNumber,
        profile = profile,
        createdAt = Instant.now()
    ))
    
    val payload = UserRegisteredPayload(
        phoneNumber = phoneNumber,
        profile = profile,
        createdAt = user.createdAt
    )
    
    val eventEnvelope = EventEnvelope(
        eventId = generateUuidv7(),      // Novo UUIDv7 para evento
        eventType = "UserRegisteredEvent",
        occurredAt = Instant.now(),
        aggregateId = user.id,            // ID do agregado que originou
        aggregateType = "user-account",
        payload = payload
    )
    
    outboxEventRepository.insert(          // TX outbox + domain mutation!
        EventOutbox(
            eventId = eventEnvelope.eventId,
            eventType = eventEnvelope.eventType,
            occurredAt = eventEnvelope.occurredAt,
            aggregateType = eventEnvelope.aggregateType,
            aggregateId = eventEnvelope.aggregateId,
            payload = eventEnvelope.payload.toJson(),
            published = false               // Para consumer ler
        )
    )
    
    return Result.success(
        RegisterResponse(
            accessToken = generateJwt(payload),  // JWT sem incluir evento no TX
            refreshToken = generateRefreshToken(user.id)
        )
    )
}
```

### Consumer (Messaging-Service) - Exemplo Java:

```java
@KafkaListener(topics = "auth-events", containerFactory = eventContainerFactory)
public void handle(UserRegisteredEventEnvelope envelope) {
    // 1. Validação idempotência por eventId
    Optional<EventLog> existing = eventLogRepository.findByEventId(envelope.getEventId());
    if (existing.isPresent()) {
        return; // Já processado!
    }
    
    // 2. Processamento de negócio
    UserAccount user = userRepository.upsert(
        envelope.getAggregateId(),           // userId do agregado
        envelope.getPayload().getPhoneNumber()
    );
    
    profileSyncService.syncProfile(user, envelope.getPayload().getProfile());
    
    // 3. Marca evento como consumido (opcional se consumer é confiável)
    eventLogRepository.markConsumed(envelope.getEventId());
}
```

---

## 🧪 Event Testing Strategy

### Unitários do Aggregate + Outbox:
```kotlin
@Test
fun `registerUser emits UserRegisteredEvent in same TX` {
    val result = authService.registerUser("5511999999999", Profile(name = "João"))
    
    verify(outboxEventRepository, times(1)).insert(any(EventOutbox()))
}
```

### Integração com Testcontainers:
```java
@Embedded
class AuthServiceIntegrationTests {
    @TestContainer private static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>();
    
    @Autowired OutboxEventRepository outboxEventRepository;
    
    @Test
    void registerEmitsUserRegisteredEvent() {
        // Arrange + Act
        authService.registerUser("5511999999999", Profile(name = "João"));
        
        // Assert
        EventOutbox event = outboxEventRepository.findPublishedByAggregateId("uuidv7-user");
        assertEquals("UserRegisteredEvent", event.getEventType());
    }
}
```

### Contrato Consumer (WireMock):
```java
@Embedded
class MessagingServiceContractTests {
    @Autowired WireMockServer wireMock;
    
    @Test
    void handleIdempotentSameEventTwice() {
        // Primeira chamada
        wireMock.post("/events/UserRegisteredEvent");
        
        // Segunda chamada (mesmo eventId)
        assertDoesNotThrow(() -> 
            messagingService.receiveEvent(new UserRegisteredEventEnvelope(...))
        );
    }
}
```

---

## 📋 Checklist de Event Compliance

Antes de emitir evento:
- [ ] `eventId` é UUIDv7 (não v4)
- [ ] `aggregateId` é o ID do agregado originador (nunca inventar novo)
- [ ] `eventType` é PascalCase único (`UserRegisteredEvent`, não `user_reg`)
- [ ] `occurredAt` está em UTC sem offset (+00 ou Z)
- [ ] Payload contém apenas dados de negócio, nunca metadados de infra
- [ ] TX outbox + mutação do agregado no mesmo TX

Antes de consumir evento:
- [ ] Validação idempotência por `eventId`
- [ ] Retry com backoff exponencial após falha persistente
- [ ] Dead-letter queue após N retries (N=5)

---

## 🔍 Debugging Events

### Visualizar Outbox no Postgres:
```sql
SELECT event_id, event_type, occurred_at, aggregate_id, payload 
FROM outbox_event 
WHERE published = false
ORDER BY occurred_at DESC 
LIMIT 20;
```

### Verificar se evento já consumido (consumer-side):
```sql
-- consumer pode manter tabela de eventos consumidos
SELECT * FROM event_log 
WHERE event_id = ? AND aggregate_type = 'user-account';
```

---

## 📚 Referências

- [Event Sourcing Patterns](https://www.oreilly.com/library/view/event-sourcing/9781492050083/)
- [Transactional Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html)
- [UUIDv7 RFC Draft](https://datatracker.ietf.org/doc/draft-peabody-dispatch-new-uuid-format/)

---

## 📝 Notas de Versão

| Versão | Mudanças | Data |
|--------|----------|------|
| 1.0 | Auth slice, outbox pattern, UUIDv7 | 2024-01 |