# Nexus Auth Service - Visão Geral da Arquitetura

**Bounded Context:** Authentication & Identity  
**Language:** Java 21 / Kotlin  
**Implementation Status:** Slice 0 (DB Schema + Outbox) ready for implementation

---

## 🏗️ Padrão Arquitetural

- **Domain-Driven Design (DDD)** com aggregates: UserAccount, Contact, BlockedUser
- **Event-Driven Architecture** via Transactional Outbox Pattern
- **Local-first:** 100% Docker Compose (Postgres + LocalStack/MinIO)

---

## 🧩 Agregados do Auth Service

| Aggregates | Responsabilidade |
|------------|------------------|
| `UserAccount` | Registro, login, perfil, identidade principal. Identificador único: `phoneNumber`. |
| `Contact` | Lista de contatos vinculados ao UserAccount (agregado). |
| `BlockedUser` | Usuários que este usuário bloqueou (agregado). |

---

## 🔄 Fluxo de Eventos do Auth Service

```
┌─────────────┐     ┌──────────────────────┐     ┌─────────────┐
│  Action     │ ───▶│ Outbox (Postgres)   │ ───▶│ Event Bus  │
│             │     │ + Transactional      │     │              │
└─────────────┘     └──────────────────────┘     └─────────────┘
```

### Contrato de Envelope (Sacro!)

```json
{
  "eventId": "UUIDv7",           // Global, idempotência por eventId
  "eventType": "UserRegisteredEvent",
  "occurredAt": "2024-01-15T10:30:00Z",
  "aggregateId": "user-id",      // Originador do evento (userId)
  "payload": { ... }             // Dados específicos
}
```

---

## 🎯 Eventos do Auth (Slice 0+)

| Evento | Quando É Emitido | Consumidores Típicos |
|--------|------------------|----------------------|
| `UserRegisteredEvent` | Após cadastro bem-sucedido | Messaging, LiveChat |
| `UserBlockedEvent` | Ao bloquear outro usuário | Auth, Analytics |

---

## 🔐 Tokens e Identidade (Auth Service)

### JWT Claims (Access Token)
- `userId` - UUIDv7 do usuário
- `phoneNumber` - Número de telefone (login identifier)
- `iat` - Issued at
- `exp` - Expiration time

### Refresh Token
- TTL: 7 dias
- Rotacionável por service
- JWKS opcional para validação externa (Phase 2)

### UUIDv7 Mandatório
- Todos os IDs devem ser UUIDv7 (time-ordered) para particionamento eficiente
- Nunca usar UUIDv4

---

## 📊 Esquema de Tabelas (Postgres Outbox - Slice 0)

### Domain Tables

**user_account** (Main Aggregate)
```sql
CREATE TABLE user_account (
    user_id UUID PRIMARY KEY,
    phone_number VARCHAR(20) UNIQUE NOT NULL,
    profile JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_user_phone ON user_account(phone_number);
```

**contacts** (Aggregates - List of Contacts)
```sql
CREATE TABLE contacts (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES user_account(user_id),
    phone_number VARCHAR(20) NOT NULL,
    is_verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_contacts_user ON contacts(user_id);
CREATE UNIQUE INDEX idx_contacts_phone_user ON contacts(phone_number, user_id);
```

**blocked_users** (Aggregates - Blocked Users List)
```sql
CREATE TABLE blocked_users (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES user_account(user_id),
    target_user_id UUID NOT NULL,
    reason TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_blocked_users_user ON blocked_users(user_id);
CREATE UNIQUE INDEX idx_blocked_pair ON blocked_users(user_id, target_user_id);
```

### Outbox Table (Infrastructure)

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
CREATE INDEX idx_outbox_published ON outbox_event(published) WHERE published = false;
```

---

## 🧪 Testing & Validation Strategy

- **Unit tests:** JUnit + AssertJ + Mockito (aggregates + outbox logic)
- **Integration tests:** Testcontainers + Spring Boot Test
- **Event contract tests:** WireMock for fake consumers

---

## 📚 References

- [Transactional Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html)
- [Event Sourcing Patterns](https://www.oreilly.com/library/view-event-sourcing/978149205083/)
- [UUIDv7 Specification](https://github.com/uuidbri/uuidv7)

---

## 🚀 Implementation Order (Slices)

**Slice 0:** Auth Service only (DB schema + outbox pattern)  
- Postgres tables: user_account, contacts, blocked_users, outbox_event  
- Repositories for all tables  
- Event publishing logic via outbox  

**Slice 1-3:** Depend on auth-service completion (messaging, livechat, gateway)