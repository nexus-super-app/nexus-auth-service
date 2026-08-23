# Nexus Auth Service - Roadmap de Implementação (Slice 0+)

**Bounded Context:** Authentication & Identity  
**Language:** Java 21 / Kotlin  
**Build:** Gradle (Java 21, Kotlin DSL)

---

## 📌 Slice 0: Auth Service Foundation

### Phase 1: DB Schema + Outbox Pattern ✅ Ready to Implement

**Goal:** Criar schema PostgreSQL com todas as tabelas e implementar transactional outbox pattern.

#### Tasks:
- [ ] Criar tabela `user_account` (main aggregate)
- [ ] Criar tabela `contacts` (aggregates - list of contacts)
- [ ] Criar tabela `blocked_users` (aggregates - blocked users list)
- [ ] Criar tabela `outbox_event` (infra table for event publishing)
- [ ] Criar indexes para performance e unicidade
- [ ] Implementar repositories para todas as tabelas
- [ ] Preparar schema migration SQL para Docker Compose / migrations folder

#### Outbox Pattern Implementation:
```java
@Transactional
public void registerUser(RegisterCommand command) {
    // 1. Create UserAccount aggregate
    UserAccount account = new UserAccount(command);
    userAccountRepository.save(account);
    
    // 2. Insert event in outbox (same TX)
    OutboxEvent event = OutboxEvent.builder()
        .eventId(UUIDv7.generate())
        .eventType("UserRegisteredEvent")
        .occurredAt(LocalDateTime.now())
        .aggregateType("UserAccount")
        .aggregateId(account.getId())
        .payload(command.toPayload())
        .published(false)
        .build();
    outboxRepository.save(event);
    
    // 3. Commit TX - event will be published asynchronously later
}
```

#### Event Contract (Sacro!):
- **UUIDv7 mandatory** for all IDs (time-ordered, efficient partitioning)
- **Idempotency by `eventId`** (never by timestamp or phone number)
- **Retry logic:** Exponential backoff until published = true
- **Fixed envelope format:** `eventId`, `eventType`, `occurredAt`, `aggregateId`, `payload`

---

### Phase 2: JWT & Token Management

**Goal:** Implementar geração, validação e rotação de tokens.

#### Tasks:
- [ ] Configurar Spring Security com JWT
- [ ] Implementar access token (15 min TTL)
- [ ] Implementar refresh token (7 dias TTL)
- [ ] Criar endpoint `/auth/login`
- [ ] Criar endpoint `/auth/refresh`
- [ ] Implementar logout e revocation de tokens

#### JWT Claims:
- Access Token: `userId`, `phoneNumber`, `iat`, `exp`
- Refresh Token: `type="refresh"`, `userId`, `phoneNumber`, `iat`, `exp`

---

### Phase 3: Event Publishing Infrastructure

**Goal:** Implementar consumer do outbox para publicar eventos no event bus.

#### Tasks:
- [ ] Criar event listener para `outbox_event` (published = false)
- [ ] Implementar retry logic com exponential backoff
- [ ] Adicionar dead letter queue para falhas persistentes
- [ ] Garantir idempotência por `eventId`
- [ ] Integrar com Kafka/LocalStack quando disponível

---

### Phase 4: API Endpoints & Controllers

**Goal:** Expor endpoints REST para operações do auth.

#### Tasks:
- [ ] `/auth/register` - Cadastro de novo usuário
- [ ] `/auth/login` - Login e geração de tokens
- [ ] `/auth/refresh` - Rotação de refresh token
- [ ] `/auth/logout` - Logout e revocation
- [ ] `/users/{userId}` - Get user profile (internal)
- [ ] `/users/{userId}/contacts` - Get contacts list
- [ ] `/users/{userId}/blocked-users` - Get blocked users list

---

### Phase 5: Unit & Integration Tests

**Goal:** Garantir qualidade e comportamento esperado.

#### Tasks:
- [ ] Unit tests para UserAccount aggregate
- [ ] Unit tests para outbox logic
- [ ] Unit tests para JWT token generation/validation
- [ ] Integration tests com Testcontainers + Postgres
- [ ] Event contract tests com WireMock (fake consumers)

---

## 🧪 Testing Strategy

| Test Type | Framework | Purpose |
|-----------|-----------|---------|
| Unit Tests | JUnit + AssertJ + Mockito | Aggregates, outbox logic, JWT |
| Integration Tests | Testcontainers + Spring Boot Test | DB connectivity, full flow |
| Event Contract Tests | WireMock | Fake consumers for event validation |

---

## 📦 Dependencies (Gradle)

```kotlin
// Core
implementation("org.springframework.boot:spring-boot-starter-web")
implementation("org.springframework.boot:spring-boot-starter-data-jpa")
implementation("org.springframework.boot:spring-boot-starter-security")

// JWT
implementation("io.jsonwebtoken:jjwt-api")
runtimeOnly("io.jsonwebtoken:jjwt-impl")
runtimeOnly("io.jsonwebtoken:jjwt-jackson")

// UUIDv7
implementation("com.fasterxml.uuid:java-uuid-generator") // or dedicated UUIDv7 lib

// Testing
testImplementation("org.springframework.boot:spring-boot-starter-test")
testImplementation("org.testcontainers:junit-jupiter")
testImplementation("org.testcontainers:postgresql")
```

---

## 🐳 Docker Compose (LocalStack)

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: nexus
      POSTGRES_PASSWORD: nexus
      POSTGRES_DB: nexus_auth
    volumes:
      - ./migrations:/docker-entrypoint-initdb.d

  localstack:
    image: localstack/localstack:3.0
    environment:
      LOCALSTACK_SERVICES: "s3"
    ports:
      - "4566:4566"
```

---

## 📚 References (Auth Service)

- [Transactional Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html)
- [JWT Best Practices](https://jwt.io/introduction/)
- [UUIDv7 Specification](https://github.com/uuidbri/uuidv7)