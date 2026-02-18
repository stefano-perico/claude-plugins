# Patterns Mermaid pour Symfony

Exemples de diagrammes Mermaid par type de processus Symfony.

## Table des matières

1. [Saga Pattern](#saga-pattern)
2. [CQRS / Command Handler](#cqrs--command-handler)
3. [Event-Driven / Messenger](#event-driven--messenger)
4. [Architecture Hexagonale](#architecture-hexagonale)
5. [Workflow Symfony](#workflow-symfony)
6. [API / Controller](#api--controller)

---

## Saga Pattern

### Flowchart - Flux complet

```mermaid
flowchart TD
    subgraph "Déclenchement"
        E1[Event: start_saga] --> W[SagaWorkflow]
    end

    subgraph "Étape 1: Initialisation"
        W --> START[StartSagaCommand]
        START --> H1[StartSagaHandler]
        H1 --> |Crée saga| STEP1_CMD[Step1Command]
    end

    subgraph "Étape 2: Traitement"
        STEP1_CMD --> H2[Step1Handler]
        H2 --> |Traitement| STEP2_CMD[Step2Command]
    end

    subgraph "Étape 3: Finalisation"
        STEP2_CMD --> H3[Step2Handler]
        H3 --> |Succès| COMPLETE[✓ SAGA COMPLETED]
    end

    subgraph "Compensation"
        H1 --> |Erreur| FAIL1[FAILED step 1]
        H2 --> |Erreur| FAIL2[FAILED step 2]
        H3 --> |Erreur| FAIL3[FAILED step 3]
        FAIL2 --> COMPENSATE[CompensateStep1]
        FAIL3 --> COMPENSATE2[CompensateStep2]
    end

    style COMPLETE fill:#90EE90
    style FAIL1 fill:#FFB6C1
    style FAIL2 fill:#FFB6C1
    style FAIL3 fill:#FFB6C1
```

### State - Machine à états

```mermaid
stateDiagram-v2
    [*] --> STARTED: StartCommand
    STARTED --> STEP_1: Step1Command
    STEP_1 --> STEP_2: Step2Command
    STEP_2 --> COMPLETED: Succès

    STEP_1 --> FAILED: Erreur
    STEP_2 --> COMPENSATING: Erreur
    COMPENSATING --> FAILED: Compensation terminée

    COMPLETED --> [*]
    FAILED --> [*]
```

### Sequence - Orchestration

```mermaid
sequenceDiagram
    participant E as Event
    participant W as Workflow
    participant H1 as StartHandler
    participant H2 as Step1Handler
    participant R as SagaRepository

    E->>W: start_saga
    W->>H1: StartCommand
    H1->>R: save(saga)
    H1->>H2: Step1Command
    H2->>R: update(saga)
    H2-->>E: saga.completed
```

---

## CQRS / Command Handler

### Flowchart - Command/Query séparation

```mermaid
flowchart LR
    subgraph "Write Side"
        C[Controller] --> CMD[Command]
        CMD --> H[CommandHandler]
        H --> R[Repository]
        R --> DB[(Database)]
    end

    subgraph "Read Side"
        C2[Controller] --> Q[Query]
        Q --> QH[QueryHandler]
        QH --> RR[ReadRepository]
        RR --> DB
    end
```

### Sequence - Command flow

```mermaid
sequenceDiagram
    participant C as Controller
    participant B as CommandBus
    participant H as Handler
    participant R as Repository
    participant E as EventDispatcher

    C->>B: dispatch(CreateUserCommand)
    B->>H: handle(command)
    H->>R: save(user)
    H->>E: dispatch(UserCreatedEvent)
    E-->>C: Response
```

### Class - Structure CQRS

```mermaid
classDiagram
    class CommandInterface {
        <<interface>>
    }
    class CreateUserCommand {
        +string email
        +string name
    }
    class CommandHandlerInterface {
        <<interface>>
        +__invoke(CommandInterface)
    }
    class CreateUserHandler {
        -UserRepository repository
        +__invoke(CreateUserCommand)
    }

    CommandInterface <|.. CreateUserCommand
    CommandHandlerInterface <|.. CreateUserHandler
    CreateUserHandler ..> CreateUserCommand
```

---

## Event-Driven / Messenger

### Flowchart - Event propagation

```mermaid
flowchart TD
    subgraph "Producer"
        S[Service] --> E[Event]
        E --> D[EventDispatcher]
    end

    subgraph "Messenger"
        D --> T[Transport/RabbitMQ]
        T --> W[Worker]
    end

    subgraph "Consumers"
        W --> H1[Handler1]
        W --> H2[Handler2]
        W --> H3[Handler3]
    end

    H1 --> A1[Action 1]
    H2 --> A2[Action 2]
    H3 --> A3[Action 3]
```

### Sequence - Async processing

```mermaid
sequenceDiagram
    participant S as Service
    participant M as MessageBus
    participant T as Transport
    participant W as Worker
    participant H as Handler

    S->>M: dispatch(Message)
    M->>T: send to queue
    Note over T: Async
    T->>W: consume
    W->>H: handle(Message)
    H-->>W: ack/nack
```

---

## Architecture Hexagonale

### Flowchart - Ports & Adapters

```mermaid
flowchart TB
    subgraph "Infrastructure"
        API[REST API]
        CLI[Console]
        MQ[Message Queue]
    end

    subgraph "Application"
        UC1[UseCase1]
        UC2[UseCase2]
    end

    subgraph "Domain"
        E[Entity]
        VS[ValueObject]
        P[Port Interface]
    end

    subgraph "Infrastructure Out"
        DB[(PostgreSQL)]
        EXT[External API]
        FS[FileSystem]
    end

    API --> UC1
    CLI --> UC1
    MQ --> UC2

    UC1 --> E
    UC2 --> E
    E --> P

    P --> DB
    P --> EXT
    P --> FS
```

### Class - Port/Adapter

```mermaid
classDiagram
    class RepositoryInterface {
        <<interface>>
        +save(Entity) void
        +findById(string) Entity
    }

    class DoctrineRepository {
        -EntityManager em
        +save(Entity) void
        +findById(string) Entity
    }

    class InMemoryRepository {
        -array storage
        +save(Entity) void
        +findById(string) Entity
    }

    RepositoryInterface <|.. DoctrineRepository
    RepositoryInterface <|.. InMemoryRepository
```

### Flowchart - ServiceLocator pattern

```mermaid
flowchart TD
    H[Handler] --> SL[ServiceLocator]
    SL --> |key: 'provider_a'| A1[ProviderAAdapter]
    SL --> |key: 'provider_b'| A2[ProviderBAdapter]

    A1 --> I[ProviderInterface]
    A2 --> I

    subgraph "Résolution dynamique"
        H --> |getDepositaire| KEY[key]
        KEY --> SL
    end
```

---

## Workflow Symfony

### State - Workflow places

```mermaid
stateDiagram-v2
    [*] --> draft
    draft --> pending_review: submit
    pending_review --> approved: approve
    pending_review --> rejected: reject
    rejected --> draft: revise
    approved --> published: publish
    published --> archived: archive
    published --> [*]
    archived --> [*]
```

### Flowchart - Workflow avec guards

```mermaid
flowchart TD
    A[Draft] --> |submit| B{Guard: isComplete?}
    B --> |yes| C[Pending Review]
    B --> |no| A

    C --> |approve| D{Guard: hasPermission?}
    D --> |yes| E[Approved]
    D --> |no| C

    C --> |reject| F[Rejected]
    F --> |revise| A

    E --> |publish| G[Published]
```

---

## API / Controller

### Sequence - REST endpoint

```mermaid
sequenceDiagram
    participant C as Client
    participant R as Router
    participant CT as Controller
    participant V as Validator
    participant S as Service
    participant DB as Database

    C->>R: POST /api/users
    R->>CT: createAction(Request)
    CT->>V: validate(DTO)
    V-->>CT: ValidationResult
    alt Valid
        CT->>S: createUser(DTO)
        S->>DB: persist
        S-->>CT: User
        CT-->>C: 201 Created
    else Invalid
        CT-->>C: 400 Bad Request
    end
```

### Flowchart - Error handling

```mermaid
flowchart TD
    REQ[Request] --> CT[Controller]
    CT --> TRY{Try}

    TRY --> |success| RES[Response 200]
    TRY --> |ValidationException| E400[Response 400]
    TRY --> |NotFoundException| E404[Response 404]
    TRY --> |AccessDeniedException| E403[Response 403]
    TRY --> |Exception| E500[Response 500]

    E400 --> LOG[Logger]
    E404 --> LOG
    E403 --> LOG
    E500 --> LOG
```

---

## Styles et couleurs

### Palette recommandée

```mermaid
flowchart LR
    A[Success] --> B[Warning] --> C[Error] --> D[Info] --> E[Neutral]

    style A fill:#90EE90
    style B fill:#FFD700
    style C fill:#FFB6C1
    style D fill:#87CEEB
    style E fill:#D3D3D3
```

### Subgraphs pour phases

```mermaid
flowchart TD
    subgraph "Phase 1: Input"
        A[Step A]
    end

    subgraph "Phase 2: Process"
        B[Step B]
        C[Step C]
    end

    subgraph "Phase 3: Output"
        D[Step D]
    end

    A --> B
    B --> C
    C --> D
```

## Conseils

1. **Nommer les nœuds** : Utiliser des noms courts mais explicites
2. **Grouper logiquement** : Utiliser `subgraph` pour les phases/couches
3. **Couleurs cohérentes** : Vert=succès, Rouge=erreur, Jaune=warning
4. **Limiter la complexité** : Max 15-20 nœuds par diagramme
5. **Direction** : `TD` pour flux vertical, `LR` pour horizontal
