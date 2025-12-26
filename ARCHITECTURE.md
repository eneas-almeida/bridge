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
        Controller[PeopleControllerImpl<br/>REST Endpoints<br/>@RequiredArgsConstructor]
        ServiceInterface[PeopleService<br/>Interface]
        ServiceImpl[PeopleServiceImpl<br/>@Service]
        RepoInterface[PeopleRepository<br/>Interface]
        RepoImpl[PeopleRepositoryImpl]
        ClientInterface[PeopleServiceClient<br/>Interface]
        GrpcClient[PeopleServiceGrpcClientImpl<br/>gRPC + MapStruct]
        Mapper[PeopleGrpcMapper<br/>MapStruct]

        Controller -->|injects| ServiceInterface
        ServiceInterface -->|implemented by| ServiceImpl
        ServiceImpl -->|uses| RepoInterface
        RepoInterface -->|implemented by| RepoImpl
        RepoImpl -->|uses| ClientInterface
        ClientInterface -->|implemented by| GrpcClient
        GrpcClient -->|uses| Mapper
    end

    GrpcClient -->|gRPC calls| PeopleService[People Service<br/>Port 9090]

    style Controller fill:#ff9999
    style ServiceInterface fill:#ffcc99
    style ServiceImpl fill:#ffcc99
    style RepoInterface fill:#99ccff
    style RepoImpl fill:#99ccff
    style GrpcClient fill:#99ff99
    style Mapper fill:#ffff99
```

---

## Arquitetura Interna - People Service

```mermaid
graph TB
    subgraph "People Service - Clean Architecture"
        GrpcService[PeopleServiceGrpcImpl<br/>gRPC Server<br/>@RequiredArgsConstructor]
        ServiceInterface[PeopleService<br/>Interface]
        ServiceImpl[PeopleServiceImpl<br/>@Service]
        RepoInterface[PeopleRepository<br/>Interface]
        RepoImpl[PeopleRepositoryImpl<br/>Strategy Pattern]
        ClientInterface[PeopleClient<br/>Interface]
        TypiClient[TypiCodeClientImpl<br/>HTTP Client]
        DummyClient[DummyClientImpl<br/>HTTP Client]
        TypiMapper[TypiCodeMapper<br/>MapStruct]
        DummyMapper[DummyMapper<br/>MapStruct]

        GrpcService -->|injects| ServiceInterface
        ServiceInterface -->|implemented by| ServiceImpl
        ServiceImpl -->|uses| RepoInterface
        RepoInterface -->|implemented by| RepoImpl
        RepoImpl -->|Strategy| ClientInterface
        ClientInterface -->|implemented by| TypiClient
        ClientInterface -->|implemented by| DummyClient
        TypiClient -->|uses| TypiMapper
        DummyClient -->|uses| DummyMapper
    end

    TypiClient -->|HTTP REST| TypiExternal[JSONPlaceholder API]
    DummyClient -->|HTTP REST| DummyExternal[DummyJSON API]

    style GrpcService fill:#99ff99
    style ServiceInterface fill:#ffcc99
    style ServiceImpl fill:#ffcc99
    style RepoInterface fill:#99ccff
    style RepoImpl fill:#99ccff
    style TypiClient fill:#ff99ff
    style DummyClient fill:#ff99ff
    style TypiMapper fill:#ffff99
    style DummyMapper fill:#ffff99
```

---

## Fluxo de Dados - GetPeople (Buscar por ID)

```mermaid
sequenceDiagram
    participant Client as Cliente
    participant Controller as PeopleControllerImpl
    participant Service as PeopleServiceImpl (API)
    participant Repo as PeopleRepositoryImpl (API)
    participant GrpcClient as PeopleServiceGrpcClientImpl
    participant Mapper as PeopleGrpcMapper
    participant GrpcService as PeopleServiceGrpcImpl
    participant PeopleService as PeopleServiceImpl (People)
    participant PeopleRepo as PeopleRepositoryImpl (People)
    participant HttpClient as TypiCodeClientImpl
    participant HttpMapper as TypiCodeMapper
    participant External as JSONPlaceholder

    Client->>Controller: GET /api/peoples/1
    activate Controller

    Controller->>Service: getById(1)
    activate Service

    Service->>Repo: findById(1)
    activate Repo

    Repo->>GrpcClient: getPeopleById(1)
    activate GrpcClient

    GrpcClient->>GrpcService: gRPC: GetPeople(id=1)
    activate GrpcService

    GrpcService->>PeopleService: getById(1)
    activate PeopleService

    PeopleService->>PeopleRepo: findById(1)
    activate PeopleRepo

    PeopleRepo->>HttpClient: getPeopleById(1)
    activate HttpClient

    HttpClient->>External: GET /users/1
    activate External

    External-->>HttpClient: JSON {id, name, email}
    deactivate External

    HttpClient->>HttpMapper: toPeopleResponse(response)
    activate HttpMapper
    HttpMapper-->>HttpClient: PeopleResponse
    deactivate HttpMapper

    HttpClient-->>PeopleRepo: Mono<PeopleResponse>
    deactivate HttpClient

    PeopleRepo-->>PeopleService: Mono<PeopleResponse>
    deactivate PeopleRepo

    PeopleService-->>GrpcService: Mono<PeopleResponse>
    deactivate PeopleService

    GrpcService-->>GrpcClient: PeopleResponseGrpc (protobuf)
    deactivate GrpcService

    GrpcClient->>Mapper: toPeopleResponse(proto)
    activate Mapper
    Mapper-->>GrpcClient: PeopleResponse
    deactivate Mapper

    GrpcClient-->>Repo: Mono<PeopleResponse>
    deactivate GrpcClient

    Repo-->>Service: Mono<PeopleResponse>
    deactivate Repo

    Service-->>Controller: Mono<PeopleResponse>
    deactivate Service

    Controller-->>Client: JSON {id, name, email}
    deactivate Controller
```

---

## Fluxo de Dados - ListPeople (Listar Todos)

```mermaid
sequenceDiagram
    participant Client as Cliente
    participant Controller as PeopleControllerImpl
    participant Service as PeopleServiceImpl (API)
    participant Repo as PeopleRepositoryImpl (API)
    participant GrpcClient as PeopleServiceGrpcClientImpl
    participant Mapper as PeopleGrpcMapper
    participant GrpcService as PeopleServiceGrpcImpl
    participant PeopleService as PeopleServiceImpl (People)
    participant PeopleRepo as PeopleRepositoryImpl (People)
    participant HttpClient as TypiCodeClientImpl
    participant HttpMapper as TypiCodeMapper
    participant External as JSONPlaceholder

    Client->>Controller: GET /api/peoples
    activate Controller

    Controller->>Service: listAll()
    activate Service

    Service->>Repo: findAll()
    activate Repo

    Repo->>GrpcClient: listPeople()
    activate GrpcClient

    GrpcClient->>GrpcService: gRPC: ListPeople()
    activate GrpcService

    GrpcService->>PeopleService: listAll()
    activate PeopleService

    PeopleService->>PeopleRepo: findAll()
    activate PeopleRepo

    PeopleRepo->>HttpClient: listPeople()
    activate HttpClient

    HttpClient->>External: GET /users
    activate External

    External-->>HttpClient: JSON Array [{id, name, email}, ...]
    deactivate External

    loop For each user
        HttpClient->>HttpMapper: toPeopleResponse(response)
        activate HttpMapper
        HttpMapper-->>HttpClient: PeopleResponse
        deactivate HttpMapper
    end

    HttpClient-->>PeopleRepo: Flux<PeopleResponse>
    deactivate HttpClient

    PeopleRepo-->>PeopleService: Flux<PeopleResponse>
    deactivate PeopleRepo

    PeopleService-->>GrpcService: Flux<PeopleResponse>
    deactivate PeopleService

    GrpcService-->>GrpcClient: ListPeopleResponseGrpc (repeated)
    deactivate GrpcService

    loop For each proto
        GrpcClient->>Mapper: toPeopleResponse(proto)
        activate Mapper
        Mapper-->>GrpcClient: PeopleResponse
        deactivate Mapper
    end

    GrpcClient-->>Repo: Flux<PeopleResponse>
    deactivate GrpcClient

    Repo-->>Service: Flux<PeopleResponse>
    deactivate Repo

    Service-->>Controller: Flux<PeopleResponse>
    deactivate Service

    Controller-->>Client: JSON Array [{id, name, email}, ...]
    deactivate Controller
```

---

## Camadas da Clean Architecture

```mermaid
graph TB
    subgraph "Clean Architecture Layers"
        direction TB

        subgraph Presentation["Presentation Layer (Controllers/Entrypoints)"]
            REST[REST Controllers<br/>PeopleControllerImpl<br/>@RequiredArgsConstructor]
            GRPC[gRPC Services<br/>PeopleServiceGrpcImpl<br/>@RequiredArgsConstructor]
        end

        subgraph Application["Application Layer (Services + DTOs)"]
            ServiceInterface[PeopleService<br/>Interface]
            ServiceImpl[PeopleServiceImpl<br/>@Service]
            DTO[PeopleResponse<br/>DTO]
        end

        subgraph Domain["Domain Layer (Interfaces)"]
            RepoInterface[PeopleRepository<br/>Interface]
            ClientInterface[PeopleClient/PeopleServiceClient<br/>Interface]
        end

        subgraph Infrastructure["Infrastructure Layer (Implementations)"]
            RepoImpl[PeopleRepositoryImpl]
            GrpcImpl[PeopleServiceGrpcClientImpl]
            HttpImpl[TypiCodeClientImpl<br/>DummyClientImpl]
            Mapper[MapStruct Mappers<br/>PeopleGrpcMapper<br/>TypiCodeMapper<br/>DummyMapper]
        end

        REST -->|injects| ServiceInterface
        GRPC -->|injects| ServiceInterface
        ServiceInterface -->|implemented by| ServiceImpl

        ServiceImpl -->|uses| RepoInterface
        RepoInterface -->|implemented by| RepoImpl

        RepoImpl -->|uses| ClientInterface

        GrpcImpl -.implements.-> ClientInterface
        HttpImpl -.implements.-> ClientInterface

        GrpcImpl -->|uses| Mapper
        HttpImpl -->|uses| Mapper

        ServiceImpl -->|returns| DTO
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
    B -->|Mono/Flux| C[Service]
    C -->|Mono/Flux| D[Repository]
    D -->|Mono/Flux| E[Client]
    E -->|Blocking Call<br/>wrapped in Mono| F[gRPC/HTTP]
    F -->|Mono/Flux| E
    E -->|MapStruct| G[Mapper]
    G -->|DTO| E
    E -->|Mono/Flux| D
    D -->|Mono/Flux| C
    C -->|Mono/Flux| B
    B -->|JSON Stream| A
```

### 2. Transformação de Dados

```mermaid
graph TB
    A[External JSON] -->|Jackson| B[External Response]
    B -->|MapStruct<br/>TypiCodeMapper<br/>DummyMapper| C[PeopleResponse DTO]
    C -->|Manual Mapping| D[Protobuf Message]
    D -->|gRPC Wire| E[Network]
    E -->|gRPC Wire| F[Protobuf Message]
    F -->|MapStruct<br/>PeopleGrpcMapper| G[PeopleResponse DTO]
    G -->|Jackson| H[JSON Response]

    style B fill:#ffe1e1
    style C fill:#e1ffe1
    style D fill:#e1e1ff
    style F fill:#e1e1ff
    style G fill:#e1ffe1
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
        A5[MapStruct 1.5.5]
        A6[Lombok]
    end

    subgraph "People Service Stack"
        P1[Spring Boot 3.3.3]
        P2[Spring WebFlux]
        P3[gRPC Server Starter]
        P4[Project Reactor]
        P5[Reactor gRPC 1.2.4]
        P6[MapStruct 1.5.5]
        P7[Lombok]
        P8[Logstash Encoder]
    end

    subgraph "Common Stack"
        C1[Java 21]
        C2[Maven]
        C3[gRPC 1.58.0/1.59.0]
        C4[Protobuf 3.24.3/3.24.4]
    end

    A1 -.-> C1
    P1 -.-> C1
    A2 -.-> A4
    P2 -.-> P4
    A5 -.shares config.-> P6
    A6 -.shares config.-> P7
```

---

## Ciclo de Vida da Requisição

```mermaid
stateDiagram-v2
    [*] --> ClientRequest: HTTP GET

    ClientRequest --> APIController: Route to endpoint

    APIController --> APIService: Delegate to service

    APIService --> APIRepository: Delegate to repository

    APIRepository --> GrpcClient: Call gRPC client

    GrpcClient --> GrpcSerialization: Serialize to Protobuf

    GrpcSerialization --> Network: Send via gRPC

    Network --> GrpcDeserialization: Receive Protobuf

    GrpcDeserialization --> PeopleGrpcService: Route to RPC handler

    PeopleGrpcService --> PeopleService: Delegate to service

    PeopleService --> PeopleRepository: Delegate to repository

    PeopleRepository --> HttpClient: Call HTTP client

    HttpClient --> ExternalAPI: HTTP GET JSONPlaceholder

    ExternalAPI --> HttpClient: JSON Response

    HttpClient --> MapStructMapping: MapStruct Mapper

    MapStructMapping --> PeopleRepository: Return Mono/Flux<DTO>

    PeopleRepository --> PeopleService: Return Mono/Flux<DTO>

    PeopleService --> PeopleGrpcService: Return Mono/Flux<DTO>

    PeopleGrpcService --> ProtoMapping: Map DTO to Protobuf

    ProtoMapping --> Network: Send gRPC response

    Network --> GrpcClient: Receive Protobuf response

    GrpcClient --> GrpcMapStructMapping: MapStruct Mapper

    GrpcMapStructMapping --> APIRepository: Return Mono/Flux<DTO>

    APIRepository --> APIService: Return Mono/Flux<DTO>

    APIService --> APIController: Return Mono/Flux<DTO>

    APIController --> JSONSerialization: Serialize DTO to JSON

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
