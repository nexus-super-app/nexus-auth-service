# Glossário de Domínio Nexus

---

## 🧑‍💻 UserAccount (Aggregate Root)

**Responsabilidade:** Gerenciar identidade e relacionamento básico de usuários

### Campos

| Campo | Tipo | Regra de Negócio |
|-------|------|------------------|
| `id` | UUIDv7 | Emitido pelo auth-service, global |
| `phoneNumber` | String (E.164) | **Identificador único** de login (+5511999999999) |
| `profile.name` | String? | Nome do usuário (opcional) |
| `profile.email` | String? | Email secundário |
| `profile.avatarUrl` | URL? | URL pública de avatar |

### Métodos Principais

- `addContact(phoneNumber: String)` → adiciona contato (validação DNI única)
- `blockUser(otherPhoneNumber: String)` → bloqueia usuário
- `unblockUser(otherPhoneNumber: String)` → desbloqueia usuário
- `getContacts()` → lista de contatos do usuário
- `getBlockedUsers()` → lista de usuários bloqueados

---

## 📞 Contact (Aggregate)

**Responsabilidade:** Lista de contatos em segundo plano do UserAccount

### Estrutura

```kotlin
data class Contact(
    val userId: UUID,        // Who is this contact of?
    val phoneNumber: String  // E.164 format
)
```

---

## 🚫 BlockedUser (Aggregate)

**Responsabilidade:** Usuários que um dado usuário bloqueou

### Estrutura

```kotlin
data class BlockedUser(
    val userId: UUID,        // Who blocked?
    val blockedUserId: UUID  // Who was blocked?
)
```

---

## 📜 Eventos de Domínio (Domain Events)

### Contrato de Envelope (SACRO!)

```json
{
  "eventId": "UUIDv7",
  "eventType": "UserRegisteredEvent",
  "occurredAt": "2024-01-15T10:30:00.000Z",
  "aggregateId": "user-id-here",
  "payload": { ... }
}
```

### Tipos de Eventos (Auth Slice 0)

| Evento | Quando É Emitido | Payload Típico |
|--------|------------------|-----------------|
| `UserRegisteredEvent` | UserAccount criado após registro | `{phoneNumber, profile: {...}}` |
| `UserBlockedEvent` | Usuário bloqueia outro | `{targetUserId, blockedAt}` |

---

## 🔐 JWT Claims (Auth)

| Claim | Descrição | Formato |
|-------|-----------|---------|
| `userId` | ID do usuário (UUIDv7) | String |
| `phoneNumber` | Telefone de login | String (E.164) |
| `iat` | Issued at | timestamp Unix |
| `exp` | Expiration | timestamp Unix (1h-24h) |
| `scope` | Permissões | List<String> |

### Refresh Token

- **Vida útil:** 7 dias rotacionáveis
- **Validação:** Por serviço, sem JWKS inicial
- **Claims:** `userId`, `phoneNumber`, `iat`, `exp`, `type="refresh"`

---

## 🏛️ Outbox Event (Infra)

**Responsabilidade:** Garantir entrega idempotente de eventos no mesmo TX da mutação

### Tabela PostgreSQL

```sql
CREATE TABLE outbox_event (
    event_id UUID PRIMARY KEY,  -- Global, único
    event_type VARCHAR(100),
    occurred_at TIMESTAMPTZ,
    aggregate_type VARCHAR(50),  -- 'user-account'
    aggregate_id UUID,           -- Originador do evento
    payload JSONB,               -- Dados específicos
    published BOOLEAN DEFAULT FALSE  -- Para TX outbox pattern
);

CREATE INDEX idx_outbox_published ON outbox_event(published) WHERE published = false;
```

### Pattern de Publicação (CDC via Debezium ou polling)

1. TX cria row em `outbox_event` + atualiza domínio
2. Consumer lê filas `published=false`
3. Após publish: `UPDATE outbox_event SET published=true WHERE event_id=?`
4. Idempotência por `eventId` (não por timestamp!)

---

## ⏱️ UUIDv7 (Tempo-Ordenado)

**Formato:** 8 bits timestamp + 60 bits random + 22 bits random

```kotlin
val uuidv7 = Uuid.fromTimestamp(Instant.now()) // Java/UUID v4 com seed no tempo
```

**Por que não UUIDv4?**
- Ordenado cronologicamente (particionamento mais eficiente)
- Clusterização por timestamp em vez de hash

---

## 🔍 Diferença: Event vs Domain Event

| Aspecto | Event (Infra) | Domain Event (DDD) |
|---------|---------------|--------------------|
| **Origem** | Postgres outbox | Agregado do domínio |
| **Conteúdo** | Envelope + payload rico | Apenas payload (evento já tem metadata) |
| **Consumo** | Idempotência por eventId | Processamento de negócio |

---

## 🎯 Regras de Negócio Principais

1. **DNI único:** `phoneNumber` deve ser único globalmente
2. **Bloqueio simétrico?** → Decidir se UserA bloqueia UserB implica UserB não vê UserA
3. **Perfil opcional:** `name`, `email`, `avatar` são opcionais no registro
4. **Token revogável?** → Implementar black list ou short TTL por token (Phase 2)

---

## 📝 Exemplo: UserRegisteredEvent Payload

```json
{
  "phoneNumber": "+5511999999999",
  "profile": {
    "name": "João Silva",
    "email": "joao@example.com",
    "avatarUrl": null
  },
  "createdAt": "2024-01-15T10:30:00Z"
}
```