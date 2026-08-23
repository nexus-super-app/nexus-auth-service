# Nexus Auth Service - DB Schema (PostgreSQL)

**Bounded Context:** Authentication & Identity  
**Database:** PostgreSQL 16+  
**Pattern:** Transactional Outbox  

---

## 📊 Tabelas do Domain

### user_account (Main Aggregate)

```sql
CREATE TABLE user_account (
    user_id UUID PRIMARY KEY,
    phone_number VARCHAR(20) UNIQUE NOT NULL,
    profile JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes for performance
CREATE INDEX idx_user_phone ON user_account(phone_number);
CREATE INDEX idx_user_created ON user_account(created_at DESC);

-- Row-level security (optional, uncomment if needed)
-- ALTER TABLE user_account ENABLE ROW LEVEL SECURITY;
```

**Columns:**
- `user_id` - UUIDv7, identificador único do usuário
- `phone_number` - Identificador de login (único)
- `profile` - JSONB com nome, email, avatar, etc.
- `created_at` / `updated_at` - Timestamps de auditoria

---

### contacts (Aggregates - List of Contacts)

```sql
CREATE TABLE contacts (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES user_account(user_id) ON DELETE CASCADE,
    phone_number VARCHAR(20) NOT NULL,
    is_verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes for performance and uniqueness
CREATE INDEX idx_contacts_user ON contacts(user_id);
CREATE UNIQUE INDEX idx_contacts_phone_user ON contacts(phone_number, user_id);
CREATE INDEX idx_contacts_created ON contacts(created_at DESC);
```

**Columns:**
- `id` - UUIDv7, identificador único do contato
- `user_id` - Owner do contato (FK para user_account)
- `phone_number` - Número de telefone do contato
- `is_verified` - Se o contato foi verificado
- `created_at` - Timestamp de criação

---

### blocked_users (Aggregates - Blocked Users List)

```sql
CREATE TABLE blocked_users (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES user_account(user_id) ON DELETE CASCADE,
    target_user_id UUID NOT NULL,
    reason TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes for performance and uniqueness
CREATE INDEX idx_blocked_users_user ON blocked_users(user_id);
CREATE UNIQUE INDEX idx_blocked_pair ON blocked_users(user_id, target_user_id);
CREATE INDEX idx_blocked_target ON blocked_users(target_user_id);
CREATE INDEX idx_blocked_created ON blocked_users(created_at DESC);
```

**Columns:**
- `id` - UUIDv7, identificador único do bloqueio
- `user_id` - Owner do bloqueio (quem bloqueou)
- `target_user_id` - Usuário que foi bloqueado
- `reason` - Motivo opcional para o bloqueio
- `created_at` - Timestamp de criação

---

## 📦 Outbox Table (Infrastructure)

```sql
CREATE TABLE outbox_event (
    event_id UUID PRIMARY KEY,
    event_type VARCHAR(100) NOT NULL,
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    aggregate_type VARCHAR(50) NOT NULL,
    aggregate_id UUID NOT NULL,
    payload JSONB NOT NULL,
    published BOOLEAN DEFAULT FALSE
);

-- Index for efficient retrieval of unpublished events
CREATE INDEX idx_outbox_published ON outbox_event(published) WHERE published = false;

-- Additional indexes for event lookup
CREATE INDEX idx_outbox_type ON outbox_event(event_type);
CREATE INDEX idx_outbox_aggregate ON outbox_event(aggregate_type, aggregate_id);
```

**Columns:**
- `event_id` - UUIDv7, identificador global do evento (idempotência)
- `event_type` - Tipo de evento (ex: UserRegisteredEvent)
- `occurred_at` - Quando o evento ocorreu no domínio
- `aggregate_type` - Tipo de aggregate originador (ex: UserAccount)
- `aggregate_id` - ID do aggregate que originou o evento
- `payload` - Dados específicos do evento (JSONB)
- `published` - Flag para indicar se foi publicado no event bus

---

## 🔄 Transactional Outbox Pattern

### Flow Diagrama

```
┌─────────────┐     ┌──────────────────────┐     ┌─────────────┐
│  Action     │ ───▶│ Outbox (Postgres)   │ ───▶│ Event Bus  │
│             │     │ + Transactional      │     │              │
└─────────────┘     └──────────────────────┘     └─────────────┘
```

### Implementation Example (Java/Spring)

```java
@Service
public class UserAccountService {
    
    @Autowired private UserAccountRepository userAccountRepository;
    @Autowired private OutboxEventRepository outboxRepository;
    
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
}
```

### Event Contract (Sacro!)

```json
{
  "eventId": "UUIDv7",           // Global, idempotência por eventId
  "eventType": "UserRegisteredEvent",
  "occurredAt": "2024-01-15T10:30:00Z",
  "aggregateId": "user-id",      // Originador do evento (userId)
  "payload": { ... }             // Dados específicos
}
```

**Rules:**
- **UUIDv7 mandatory** for all IDs (time-ordered, efficient partitioning)
- **Idempotency by `eventId`** (never by timestamp or phone number)
- **Retry logic:** Exponential backoff until published = true
- **Dead letter queue:** Para eventos que falham persistentemente

---

## 🧪 Migration SQL para Docker Compose

```sql
-- Run this in a fresh Postgres container for initial setup
-- Or use Flyway/Liquibase for versioned migrations

-- Domain tables
CREATE TABLE user_account (
    user_id UUID PRIMARY KEY,
    phone_number VARCHAR(20) UNIQUE NOT NULL,
    profile JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_user_phone ON user_account(phone_number);
CREATE INDEX idx_user_created ON user_account(created_at DESC);

CREATE TABLE contacts (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES user_account(user_id) ON DELETE CASCADE,
    phone_number VARCHAR(20) NOT NULL,
    is_verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_contacts_user ON contacts(user_id);
CREATE UNIQUE INDEX idx_contacts_phone_user ON contacts(phone_number, user_id);
CREATE INDEX idx_contacts_created ON contacts(created_at DESC);

CREATE TABLE blocked_users (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES user_account(user_id) ON DELETE CASCADE,
    target_user_id UUID NOT NULL,
    reason TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_blocked_users_user ON blocked_users(user_id);
CREATE UNIQUE INDEX idx_blocked_pair ON blocked_users(user_id, target_user_id);
CREATE INDEX idx_blocked_target ON blocked_users(target_user_id);
CREATE INDEX idx_blocked_created ON blocked_users(created_at DESC);

-- Outbox table (infrastructure)
CREATE TABLE outbox_event (
    event_id UUID PRIMARY KEY,
    event_type VARCHAR(100) NOT NULL,
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    aggregate_type VARCHAR(50) NOT NULL,
    aggregate_id UUID NOT NULL,
    payload JSONB NOT NULL,
    published BOOLEAN DEFAULT FALSE
);

CREATE INDEX idx_outbox_published ON outbox_event(published) WHERE published = false;
CREATE INDEX idx_outbox_type ON outbox_event(event_type);
CREATE INDEX idx_outbox_aggregate ON outbox_event(aggregate_type, aggregate_id);
```

---

## 📦 Docker Compose (Postgres + LocalStack)

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: nexus
      POSTGRES_PASSWORD: nexus
      POSTGRES_DB: nexus_auth
    volumes:
      - ./migrations:/docker-entrypoint-initdb.d
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U nexus -d nexus_auth"]
      interval: 10s
      timeout: 5s
      retries: 5

  localstack:
    image: localstack/localstack:3.0-alpine
    environment:
      LOCALSTACK_SERVICES: "s3"
      DEFAULT_REGION: us-east-1
    ports:
      - "4566:4566"
    volumes:
      - ./localstack-data:/var/lib/localstack

volumes:
  postgres_data:
```

---

## 📚 References

- [Transactional Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html)
- [PostgreSQL UUID](https://www.postgresql.org/docs/current/datatype-uuid.html)
- [UUIDv7 Specification](https://github.com/uuidbri/uuidv7)