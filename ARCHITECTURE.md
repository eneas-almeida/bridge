# Diagramas de Arquitetura - Bridge

Este documento contém diagramas detalhados da arquitetura e fluxos de comunicação do projeto Bridge.

## Arquitetura Geral do Sistema

```mermaid
graph TB
    Client[Cliente HTTP/Browser]
    API[API Service<br/>Port 8081<br/>REST Gateway]
    People[People Service<br/>Port 9090<br/>gRPC Server]
    External[JSONPlaceholder API<br/>External REST API]

    Client -->|HTTP REST| API
    API -->|gRPC/Protobuf| People
    People -->|HTTP REST| External

    style Client fill:#e1f5ff
    style API fill:#ffe1e1
    style People fill:#e1ffe1
    style External fill:#fff4e1
```

---

## Arquitetura Interna - API Service

```mermaid
graph TB
    subgraph "API Service - Clean Architecture"
        Controller[PeopleController<br/>REST Endpoints]
        UseCase1[GetPeopleUseCase]
        UseCase2[ListPeopleUseCase]
        ClientInterface[PeopleClient<br/>Interface]
        GrpcClient[PeopleGrpcClient<br/>gRPC Implementation]

        Controller -->|delegates| UseCase1
        Controller -->|delegates| UseCase2
        UseCase1 -->|uses| ClientInterface
        UseCase2 -->|uses| ClientInterface
        ClientInterface -->|implemented by| GrpcClient
    end

    GrpcClient -->|gRPC calls| PeopleService[People Service<br/>Port 9090]

    style Controller fill:#ff9999
    style UseCase1 fill:#ffcc99
    style UseCase2 fill:#ffcc99
    style ClientInterface fill:#99ccff
    style GrpcClient fill:#99ff99
```

---

## Arquitetura Interna - People Service

```mermaid
graph TB
    subgraph "People Service - Clean Architecture"
        GrpcService[PeopleGrpcService<br/>gRPC Server]
        UseCase1[GetPeopleUseCase]
        UseCase2[ListPeopleUseCase]
        ClientInterface[PeopleClient<br/>Interface]
        HttpClient[TypiCodePeopleClientImpl<br/>HTTP Client]
        Mapper[PeopleMapper<br/>MapStruct]

        GrpcService -->|delegates| UseCase1
        GrpcService -->|delegates| UseCase2
        UseCase1 -->|uses| ClientInterface
        UseCase2 -->|uses| ClientInterface
        ClientInterface -->|implemented by| HttpClient
        HttpClient -->|uses| Mapper
    end

    HttpClient -->|HTTP REST| External[JSONPlaceholder API]

    style GrpcService fill:#99ff99
    style UseCase1 fill:#ffcc99
    style UseCase2 fill:#ffcc99
    style ClientInterface fill:#99ccff
    style HttpClient fill:#ff99ff
    style Mapper fill:#ffff99
```

---

## Fluxo de Dados - GetPeople (Buscar por ID)

```mermaid
sequenceDiagram
    participant Client as Cliente
    participant Controller as PeopleController
    participant UseCase as GetPeopleUseCase
    participant GrpcClient as PeopleGrpcClient
    participant GrpcService as PeopleGrpcService
    participant PeopleUseCase as GetPeopleUseCase (People)
    participant HttpClient as TypiCodeClientImpl
    participant External as JSONPlaceholder

    Client->>Controller: GET /api/peoples/1
    activate Controller

    Controller->>UseCase: execute(1)
    activate UseCase

    UseCase->>GrpcClient: getPeopleById(1)
    activate GrpcClient

    GrpcClient->>GrpcService: gRPC: GetPeople(id=1)
    activate GrpcService

    GrpcService->>PeopleUseCase: execute(1)
    activate PeopleUseCase

    PeopleUseCase->>HttpClient: findById(1)
    activate HttpClient

    HttpClient->>External: GET /users/1
    activate External

    External-->>HttpClient: JSON {id, name, email}
    deactivate External

    HttpClient-->>PeopleUseCase: Mono<People>
    deactivate HttpClient

    PeopleUseCase-->>GrpcService: Mono<People>
    deactivate PeopleUseCase

    GrpcService-->>GrpcClient: PeopleResponse (protobuf)
    deactivate GrpcService

    GrpcClient-->>UseCase: Mono<People>
    deactivate GrpcClient

    UseCase-->>Controller: Mono<People>
    deactivate UseCase

    Controller-->>Client: JSON {id, name, email}
    deactivate Controller
```

---

## Fluxo de Dados - ListPeople (Listar Todos)

```mermaid
sequenceDiagram
    participant Client as Cliente
    participant Controller as PeopleController
    participant UseCase as ListPeopleUseCase
    participant GrpcClient as PeopleGrpcClient
    participant GrpcService as PeopleGrpcService
    participant PeopleUseCase as ListPeopleUseCase (People)
    participant HttpClient as TypiCodeClientImpl
    participant External as JSONPlaceholder

    Client->>Controller: GET /api/peoples
    activate Controller

    Controller->>UseCase: execute()
    activate UseCase

    UseCase->>GrpcClient: listPeople()
    activate GrpcClient

    GrpcClient->>GrpcService: gRPC: ListPeople()
    activate GrpcService

    GrpcService->>PeopleUseCase: execute()
    activate PeopleUseCase

    PeopleUseCase->>HttpClient: listAll()
    activate HttpClient

    HttpClient->>External: GET /users
    activate External

    External-->>HttpClient: JSON Array [{id, name, email}, ...]
    deactivate External

    HttpClient-->>PeopleUseCase: Flux<People>
    deactivate HttpClient

    PeopleUseCase-->>GrpcService: Flux<People>
    deactivate PeopleUseCase

    GrpcService-->>GrpcClient: ListPeopleResponse (repeated)
    deactivate GrpcService

    GrpcClient-->>UseCase: Flux<People>
    deactivate GrpcClient

    UseCase-->>Controller: Flux<People>
    deactivate UseCase

    Controller-->>Client: JSON Array [{id, name, email}, ...]
    deactivate Controller
```

---

## Camadas da Clean Architecture

```mermaid
graph TB
    subgraph "Clean Architecture Layers"
        direction TB

        subgraph Presentation["Presentation Layer (Adapters)"]
            REST[REST Controllers<br/>PeopleController]
            GRPC[gRPC Services<br/>PeopleGrpcService]
        end

        subgraph Application["Application Layer (Use Cases)"]
            UC1[GetPeopleUseCase]
            UC2[ListPeopleUseCase]
        end

        subgraph Domain["Domain Layer (Business Logic)"]
            Entity[People Entity]
            Interface[PeopleClient Interface]
        end

        subgraph Infrastructure["Infrastructure Layer (Adapters)"]
            GrpcImpl[PeopleGrpcClient]
            HttpImpl[TypiCodeClientImpl]
            Mapper[Mappers]
        end

        REST --> UC1
        REST --> UC2
        GRPC --> UC1
        GRPC --> UC2

        UC1 --> Interface
        UC2 --> Interface

        Interface --> Entity

        GrpcImpl -.implements.-> Interface
        HttpImpl -.implements.-> Interface

        GrpcImpl --> Mapper
        HttpImpl --> Mapper
    end

    style Presentation fill:#ffcccc
    style Application fill:#ffffcc
    style Domain fill:#ccffcc
    style Infrastructure fill:#ccccff
```

---

## Modelo de Dados

```mermaid
classDiagram
    class People {
        +Integer id
        +String name
        +String email
    }

    class PeopleResponse_DTO {
        +Integer id
        +String name
        +String email
        +fromDomain(People) PeopleResponse
    }

    class PeopleRequest_Proto {
        +int32 id
    }

    class PeopleResponse_Proto {
        +int32 id
        +string name
        +string email
    }

    class ListPeopleResponse_Proto {
        +repeated PeopleResponse people
    }

    People --> PeopleResponse_DTO : maps to
    People --> PeopleResponse_Proto : maps to
    PeopleResponse_Proto --> ListPeopleResponse_Proto : contains
```

---

## Configuração de Rede

```mermaid
graph LR
    subgraph "API Service Container"
        API[API Service<br/>:8081 HTTP<br/>gRPC Client]
    end

    subgraph "People Service Container"
        People[People Service<br/>:9090 gRPC Server]
    end

    subgraph "External Network"
        External[JSONPlaceholder<br/>HTTPS]
    end

    Client[HTTP Client] -->|Port 8081| API
    API -->|Port 9090<br/>PLAINTEXT| People
    People -->|HTTPS| External

    style API fill:#ffe1e1
    style People fill:#e1ffe1
    style External fill:#fff4e1
```

---

## Padrões de Comunicação

### 1. Comunicação Síncrona Reativa

```mermaid
graph LR
    A[HTTP Request] -->|Mono/Flux| B[Controller]
    B -->|Mono/Flux| C[Use Case]
    C -->|Mono/Flux| D[Client]
    D -->|Blocking Call<br/>wrapped in Mono| E[gRPC/HTTP]
    E -->|Mono/Flux| D
    D -->|Mono/Flux| C
    C -->|Mono/Flux| B
    B -->|JSON Stream| A
```

### 2. Transformação de Dados

```mermaid
graph TB
    A[External JSON] -->|Jackson| B[DTO/Record]
    B -->|MapStruct| C[Domain Entity]
    C -->|Manual Mapping| D[Protobuf Message]
    D -->|gRPC Wire| E[Network]
    E -->|gRPC Wire| F[Protobuf Message]
    F -->|Manual Mapping| G[Domain Entity]
    G -->|DTO Mapper| H[Response DTO]
    H -->|Jackson| I[JSON Response]
```

---

## Tratamento de Erros

```mermaid
graph TB
    A[Client Request] --> B{API Service}

    B -->|Success| C[gRPC Call]
    B -->|Error| D[HTTP 500]

    C --> E{People Service}

    E -->|Success| F[External API Call]
    E -->|Error| G[gRPC INTERNAL Status]

    F -->|Success| H[Process Response]
    F -->|Error| I[gRPC INTERNAL Status]

    G --> J[Propagate to API]
    I --> J

    J --> K[HTTP 500]

    H --> L[Success Response]

    style D fill:#ff9999
    style G fill:#ff9999
    style I fill:#ff9999
    style K fill:#ff9999
    style L fill:#99ff99
```

---

## Stack Tecnológico por Camada

```mermaid
graph TB
    subgraph "API Service Stack"
        A1[Spring Boot 3.3.3]
        A2[Spring WebFlux]
        A3[gRPC Client Starter]
        A4[Project Reactor]
        A5[Lombok]
    end

    subgraph "People Service Stack"
        P1[Spring Boot 3.3.3]
        P2[Spring WebFlux]
        P3[gRPC Server Starter]
        P4[Project Reactor]
        P5[MapStruct]
        P6[Lombok]
    end

    subgraph "Common Stack"
        C1[Java 21]
        C2[Maven]
        C3[gRPC 1.58.0]
        C4[Protobuf 3.24.3]
    end

    A1 -.-> C1
    P1 -.-> C1
    A2 -.-> A4
    P2 -.-> P4
```

---

## Ciclo de Vida da Requisição

```mermaid
stateDiagram-v2
    [*] --> ClientRequest: HTTP GET

    ClientRequest --> APIController: Route to endpoint

    APIController --> APIUseCase: Delegate business logic

    APIUseCase --> GrpcClient: Call external service

    GrpcClient --> GrpcSerialization: Serialize to Protobuf

    GrpcSerialization --> Network: Send via gRPC

    Network --> GrpcDeserialization: Receive Protobuf

    GrpcDeserialization --> PeopleGrpcService: Route to RPC handler

    PeopleGrpcService --> PeopleUseCase: Delegate business logic

    PeopleUseCase --> HttpClient: Call external API

    HttpClient --> ExternalAPI: HTTP GET JSONPlaceholder

    ExternalAPI --> HttpClient: JSON Response

    HttpClient --> Mapping: Map to Domain Entity

    Mapping --> PeopleUseCase: Return Mono/Flux

    PeopleUseCase --> PeopleGrpcService: Return result

    PeopleGrpcService --> ProtoMapping: Map to Protobuf

    ProtoMapping --> Network: Send gRPC response

    Network --> GrpcClient: Receive response

    GrpcClient --> DomainMapping: Map to Domain Entity

    DomainMapping --> APIUseCase: Return Mono/Flux

    APIUseCase --> APIController: Return result

    APIController --> DTOMapping: Map to DTO

    DTOMapping --> JSONSerialization: Serialize to JSON

    JSONSerialization --> [*]: HTTP Response
```

---

## Deployment Diagram

```mermaid
graph TB
    subgraph "Development Environment"
        subgraph "Process 1"
            API[API Service<br/>mvn spring-boot:run<br/>Port 8081]
        end

        subgraph "Process 2"
            People[People Service<br/>mvn spring-boot:run<br/>Port 9090]
        end

        LH[localhost]
    end

    subgraph "External Services"
        JP[JSONPlaceholder<br/>jsonplaceholder.typicode.com]
    end

    Browser[Web Browser] -->|HTTP :8081| API
    API -->|gRPC :9090<br/>127.0.0.1| People
    People -->|HTTPS| JP

    API -.->|runs on| LH
    People -.->|runs on| LH
```

---

## Dependências entre Projetos

```mermaid
graph LR
    subgraph "API Module"
        API[api]
        APIProto[service.proto]
    end

    subgraph "People Module"
        People[people]
        PeopleProto[service.proto]
    end

    subgraph "Shared"
        Proto[Generated gRPC Code<br/>com.people.grpc.*]
    end

    API -->|depends on| Proto
    People -->|depends on| Proto

    APIProto -->|generates| Proto
    PeopleProto -->|generates| Proto

    APIProto -.identical to.- PeopleProto

    style Proto fill:#ffff99
```

Este documento fornece uma visão visual completa da arquitetura do sistema Bridge, facilitando o entendimento dos fluxos de comunicação e das responsabilidades de cada componente.
