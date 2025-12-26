# Bridge - Documentação da Arquitetura

## Visão Geral

O projeto **Bridge** é uma arquitetura de microserviços composta por dois serviços principais que se comunicam via **gRPC**:

- **API Service**: Gateway REST que expõe endpoints HTTP e se comunica com o serviço People via gRPC
- **People Service**: Serviço backend que fornece dados de usuários via gRPC, com suporte a múltiplas fontes de dados externas (JSONPlaceholder ou DummyJSON)

```
┌──────────────┐     HTTP/REST      ┌──────────────┐      gRPC       ┌──────────────┐     HTTP      ┌─────────────────┐
│   Cliente    │ ─────────────────> │  API Service │ ──────────────> │People Service│ ────────────> │ External APIs   │
│  (Browser)   │                    │  (Port 8081) │                 │ (Port 9090)  │               │ - JSONPlaceholder│
└──────────────┘                    └──────────────┘                 └──────────────┘               │ - DummyJSON     │
                                                                                                      └─────────────────┘
```

---

## Índice

1. [Tecnologias Utilizadas](#tecnologias-utilizadas)
2. [Arquitetura dos Serviços](#arquitetura-dos-serviços)
3. [Contrato gRPC](#contrato-grpc)
4. [Fluxo de Comunicação Detalhado](#fluxo-de-comunicação-detalhado)
5. [API Service](#api-service)
6. [People Service](#people-service)
7. [Configuração e Execução](#configuração-e-execução)
8. [Endpoints Disponíveis](#endpoints-disponíveis)

---

## Tecnologias Utilizadas

### Comuns a Ambos os Projetos
- **Java 21**
- **Spring Boot 3.3.3**
- **Spring WebFlux** (Programação Reativa)
- **Project Reactor** (Mono/Flux)
- **Lombok**
- **Maven**

**API Service:**
- **gRPC 1.58.0**
- **Protocol Buffers 3.24.3**

**People Service:**
- **gRPC 1.59.0**
- **Protocol Buffers 3.24.4**

### Específicas do API Service
- **gRPC Client Spring Boot Starter 2.15.0**
- **MapStruct 1.5.5** (Mapeamento gRPC Proto para DTOs)

### Específicas do People Service
- **gRPC Server Spring Boot Starter 2.15.0**
- **Reactor gRPC 1.2.4** (gRPC Reativo)
- **MapStruct 1.5.5** (Mapeamento de DTOs)
- **Logstash Logback Encoder 7.4** (Logging estruturado)
- **Datadog Trace API 1.30.1** (Observabilidade)

---

## Arquitetura dos Serviços

Ambos os serviços seguem os princípios de **Clean Architecture** com separação clara de responsabilidades:

### Camadas da Arquitetura

```
┌─────────────────────────────────────────────┐
│          Presentation/Entrypoint            │
│  (REST Controllers / gRPC Services)         │
└────────────────┬────────────────────────────┘
                 │
┌────────────────▼────────────────────────────┐
│           Application Layer                 │
│         (Services + DTOs)                   │
└────────────────┬────────────────────────────┘
                 │
┌────────────────▼────────────────────────────┐
│            Domain Layer                     │
│   (Repository/Client Interfaces)            │
└────────────────┬────────────────────────────┘
                 │
┌────────────────▼────────────────────────────┐
│        Infrastructure Layer                 │
│   (Repositories / gRPC Clients/Servers)     │
│         (HTTP Clients / Mappers)            │
└─────────────────────────────────────────────┘
```

---

## Contrato gRPC

O contrato de comunicação entre os serviços é definido através de **Protocol Buffers**.

### Arquivo Proto (`person.proto`)

```protobuf
syntax = "proto3";

option java_package = "com.people.grpc";
option java_outer_classname = "ServiceProto";

package grpcservice;

service PeopleService {
  rpc GetPeople (PeopleRequestGrpc) returns (PeopleResponseGrpc);
  rpc ListPeople (ListPeopleRequestGrpc) returns (ListPeopleResponseGrpc);
}

message PeopleRequestGrpc {
  int32 id = 1;
}

message ListPeopleRequestGrpc {}

message PeopleResponseGrpc {
  int32 id = 1;
  string name = 2;
  string email = 3;
}

message ListPeopleResponseGrpc {
  repeated PeopleResponseGrpc people = 1;
}
```

### RPCs Disponíveis

| RPC | Request | Response | Descrição |
|-----|---------|----------|-----------|
| `GetPeople` | `PeopleRequestGrpc` (id) | `PeopleResponseGrpc` | Busca uma pessoa por ID |
| `ListPeople` | `ListPeopleRequestGrpc` (vazio) | `ListPeopleResponseGrpc` | Lista todas as pessoas |

---

## Fluxo de Comunicação Detalhado

### Fluxo 1: Buscar Pessoa por ID

```
Cliente              API Service             People Service          External API
  │                       │                        │                (DummyJSON ou JSONPlaceholder)
  │ GET /api/peoples/1    │                        │                        │
  ├──────────────────────>│                        │                        │
  │                       │ PeopleController       │                        │
  │                       │        ↓               │                        │
  │                       │ PeopleService          │                        │
  │                       │        ↓               │                        │
  │                       │ PeopleRepository       │                        │
  │                       │        ↓               │                        │
  │                       │ PeopleGrpcClient       │                        │
  │                       │   (MapStruct)          │                        │
  │                       │                        │                        │
  │                       │ gRPC: GetPeople(id=1)  │                        │
  │                       ├───────────────────────>│                        │
  │                       │                        │ PeopleGrpcService      │
  │                       │                        │        ↓               │
  │                       │                        │ PeopleService          │
  │                       │                        │        ↓               │
  │                       │                        │ PeopleRepository       │
  │                       │                        │   (Strategy Pattern)   │
  │                       │                        │        ↓               │
  │                       │                        │ DummyClient ou         │
  │                       │                        │ TypiCodeClient         │
  │                       │                        │   (MapStruct)          │
  │                       │                        │                        │
  │                       │                        │ GET /users/1           │
  │                       │                        ├───────────────────────>│
  │                       │                        │                        │
  │                       │                        │ JSON Response          │
  │                       │                        │<───────────────────────┤
  │                       │                        │                        │
  │                       │                        │ Map to DTO             │
  │                       │                        │ (PeopleResponse)       │
  │                       │                        │                        │
  │                       │ PeopleResponse (gRPC)  │                        │
  │                       │<───────────────────────┤                        │
  │                       │                        │                        │
  │                       │ Map Proto to DTO       │                        │
  │                       │   (MapStruct)          │                        │
  │                       │        ↓               │                        │
  │                       │ PeopleResponse (DTO)   │                        │
  │                       │                        │                        │
  │ JSON Response         │                        │                        │
  │<──────────────────────┤                        │                        │
```

### Fluxo 2: Listar Todas as Pessoas

```
Cliente              API Service             People Service          External API
  │                       │                        │                (DummyJSON ou JSONPlaceholder)
  │ GET /api/peoples      │                        │                        │
  ├──────────────────────>│                        │                        │
  │                       │ PeopleController       │                        │
  │                       │        ↓               │                        │
  │                       │ PeopleService          │                        │
  │                       │        ↓               │                        │
  │                       │ PeopleRepository       │                        │
  │                       │        ↓               │                        │
  │                       │ PeopleGrpcClient       │                        │
  │                       │   (MapStruct)          │                        │
  │                       │                        │                        │
  │                       │ gRPC: ListPeople()     │                        │
  │                       ├───────────────────────>│                        │
  │                       │                        │ PeopleGrpcService      │
  │                       │                        │        ↓               │
  │                       │                        │ PeopleService          │
  │                       │                        │        ↓               │
  │                       │                        │ PeopleRepository       │
  │                       │                        │   (Strategy Pattern)   │
  │                       │                        │        ↓               │
  │                       │                        │ DummyClient ou         │
  │                       │                        │ TypiCodeClient         │
  │                       │                        │   (MapStruct)          │
  │                       │                        │                        │
  │                       │                        │ GET /users             │
  │                       │                        ├───────────────────────>│
  │                       │                        │                        │
  │                       │                        │ JSON Array Response    │
  │                       │                        │<───────────────────────┤
  │                       │                        │                        │
  │                       │                        │ Map to Flux<DTO>       │
  │                       │                        │   (MapStruct)          │
  │                       │                        │                        │
  │                       │ ListPeopleResponse     │                        │
  │                       │ (repeated gRPC)        │                        │
  │                       │<───────────────────────┤                        │
  │                       │                        │                        │
  │                       │ Map Proto to Flux<DTO> │                        │
  │                       │   (MapStruct)          │                        │
  │                       │                        │                        │
  │ JSON Array Response   │                        │                        │
  │<──────────────────────┤                        │                        │
```

---

## API Service

### Estrutura do Projeto

```
api/
├── src/main/java/org/api/
│   ├── presentation/
│   │   └── controller/
│   │       └── PeopleControllerImpl.java
│   ├── application/
│   │   ├── dto/
│   │   │   └── PeopleResponse.java
│   │   └── service/
│   │       ├── PeopleService.java (Interface)
│   │       └── PeopleServiceImpl.java
│   ├── domain/
│   │   ├── client/
│   │   │   └── PeopleServiceClient.java (Interface)
│   │   └── repository/
│   │       └── PeopleRepository.java (Interface)
│   └── infrastructure/
│       ├── client/
│       │   ├── PeopleServiceGrpcClientImpl.java
│       │   └── PeopleGrpcMapper.java (MapStruct)
│       ├── repository/
│       │   └── PeopleRepositoryImpl.java
│       └── config/
│           └── RepositoryConfig.java
├── src/main/proto/
│   └── person.proto
└── src/main/resources/
    └── application.yml
```

### Componentes Principais

#### 1. PeopleControllerImpl (`presentation/controller/PeopleControllerImpl.java`)

```java
@RestController
@RequestMapping("/api/peoples")
@RequiredArgsConstructor  // Lombok
public class PeopleControllerImpl {

    private final PeopleService peopleService;  // Interface!

    @GetMapping("/{id}")
    public Mono<PeopleResponse> getPeopleById(@PathVariable int id)

    @GetMapping
    public Flux<PeopleResponse> listPeople()
}
```

**Responsabilidades:**
- Expor endpoints REST HTTP
- Receber requisições do cliente
- Delegar para o serviço de aplicação
- Retornar DTOs de resposta
- **Inversão de Dependência:** Injeta interface `PeopleService`, não implementação

#### 2. PeopleService (`application/service/`)

**PeopleService (Interface)**
```java
public interface PeopleService {
    Mono<PeopleResponse> getById(int id);
    Flux<PeopleResponse> listAll();
}
```

**PeopleServiceImpl**
```java
@Service
@RequiredArgsConstructor  // Lombok
public class PeopleServiceImpl implements PeopleService {

    private final PeopleRepository peopleRepository;

    @Override
    public Mono<PeopleResponse> getById(int id) {
        return peopleRepository.findById(id);
    }

    @Override
    public Flux<PeopleResponse> listAll() {
        return peopleRepository.findAll();
    }
}
```

**Responsabilidades:**
- Orquestrar a lógica de negócio
- Delegar para o repositório
- Camada de abstração entre controller e infraestrutura

#### 3. PeopleRepositoryImpl (`infrastructure/repository/PeopleRepositoryImpl.java`)

```java
@RequiredArgsConstructor  // Lombok
public class PeopleRepositoryImpl implements PeopleRepository {

    private final PeopleServiceClient peopleClient;

    @Override
    public Mono<PeopleResponse> findById(int id) {
        return peopleClient.getPeopleById(id);
    }

    @Override
    public Flux<PeopleResponse> findAll() {
        return peopleClient.listPeople();
    }
}
```

**Responsabilidades:**
- Implementar a interface `PeopleRepository`
- Delegar para o cliente gRPC
- Abstrair a comunicação gRPC

#### 4. PeopleServiceGrpcClientImpl (`infrastructure/client/PeopleServiceGrpcClientImpl.java`)

```java
@Component
@RequiredArgsConstructor  // Lombok
public class PeopleServiceGrpcClientImpl implements PeopleServiceClient {

    @GrpcClient("people-service")
    private PeopleServiceGrpc.PeopleServiceBlockingStub peopleServiceStub;

    private final PeopleGrpcMapper peopleGrpcMapper;  // MapStruct!

    @Override
    public Mono<PeopleResponse> getPeopleById(int id) {
        return Mono.fromCallable(() -> {
            ServiceProto.PeopleRequestGrpc request =
                ServiceProto.PeopleRequestGrpc.newBuilder()
                    .setId(id)
                    .build();

            ServiceProto.PeopleResponseGrpc response =
                peopleServiceStub.getPeople(request);

            return peopleGrpcMapper.toPeopleResponse(response);  // MapStruct
        })
        .subscribeOn(Schedulers.boundedElastic());
    }
}
```

**Responsabilidades:**
- Implementar a interface `PeopleServiceClient`
- Realizar chamadas gRPC para o People Service
- Converter chamadas bloqueantes em streams reativos (Mono/Flux)
- Mapear protobuf messages para DTOs usando **MapStruct**

#### 5. PeopleGrpcMapper (`infrastructure/client/PeopleGrpcMapper.java`)

```java
@Mapper(
    componentModel = "spring",
    implementationName = "PeopleGrpcMapperImpl"
)
public interface PeopleGrpcMapper {

    @Mapping(target = "id", source = "id")
    @Mapping(target = "name", source = "name")
    @Mapping(target = "email", source = "email")
    PeopleResponse toPeopleResponse(ServiceProto.PeopleResponseGrpc response);
}
```

**Responsabilidades:**
- Mapeamento **type-safe** de gRPC Proto para DTO
- Geração de código em tempo de compilação via MapStruct
- Integração com Spring via `componentModel = "spring"`

### Configuração (`application.yml`)

```yaml
server:
  port: 8081

spring:
  application:
    name: api

grpc:
  client:
    people-service:
      address: 'static://127.0.0.1:9090'
      negotiationType: PLAINTEXT
      enable-keep-alive: true
      keep-alive-without-calls: true
      max-inbound-message-size: 4194304  # 4 MB
```

---

## People Service

### Estrutura do Projeto

```
people/
├── src/main/java/org/people/
│   ├── PeopleApplication.java
│   ├── domain/
│   │   ├── client/
│   │   │   └── PeopleClient.java (Interface)
│   │   ├── entity/
│   │   │   └── People.java
│   │   ├── enums/
│   │   │   └── DataSource.java
│   │   ├── exception/
│   │   │   ├── PeopleException.java
│   │   │   ├── BusinessRuleException.java
│   │   │   ├── ValidationException.java
│   │   │   └── PeopleNotFoundException.java
│   │   └── repository/
│   │       └── PeopleRepository.java (Interface)
│   ├── application/
│   │   ├── dto/
│   │   │   └── PeopleResponse.java
│   │   └── service/
│   │       ├── PeopleService.java (Interface)
│   │       └── PeopleServiceImpl.java
│   └── infrastructure/
│       ├── entrypoint/
│       │   └── grpc/
│       │       └── PeopleServiceGrpcImpl.java
│       ├── client/
│       │   ├── typicode/
│       │   │   ├── TypiCodeClientImpl.java
│       │   │   ├── TypiCodeResponse.java
│       │   │   └── TypiCodeMapper.java (MapStruct)
│       │   └── dummy/
│       │       ├── DummyClientImpl.java
│       │       ├── DummyResponse.java
│       │       ├── DummyListResponse.java
│       │       └── DummyMapper.java (MapStruct)
│       ├── repository/
│       │   └── PeopleRepositoryImpl.java (Strategy Pattern)
│       ├── exception/
│       │   ├── GlobalGrpcExceptionHandler.java
│       │   ├── ExternalServiceException.java
│       │   └── InternalServerException.java
│       ├── logging/
│       │   ├── Logger.java
│       │   ├── LogContext.java
│       │   ├── RequestContext.java
│       │   └── GrpcLoggingInterceptor.java
│       └── config/
│           ├── RepositoryConfig.java
│           └── client/
│               ├── TypiCodeClientConfig.java
│               └── DummyClientConfig.java
├── src/main/proto/
│   └── person.proto
└── src/main/resources/
    └── application.yml
```

### Componentes Principais

#### 1. PeopleServiceGrpcImpl (`infrastructure/entrypoint/grpc/PeopleServiceGrpcImpl.java`)

```java
@GrpcService
@RequiredArgsConstructor  // Lombok
public class PeopleServiceGrpcImpl extends ReactorPeopleServiceGrpc.PeopleServiceImplBase {

    private final PeopleService peopleService;  // Interface!

    @Override
    public Mono<PeopleResponseGrpc> getPeople(Mono<PeopleRequestGrpc> request) {
        return request
            .flatMap(req -> peopleService.getById(req.getId()))
            .map(people -> PeopleResponseGrpc.newBuilder()
                .setId(people.getId())
                .setName(people.getName())
                .setEmail(people.getEmail())
                .build());
    }

    @Override
    public Mono<ListPeopleResponseGrpc> listPeople(Mono<ListPeopleRequestGrpc> request) {
        return request
            .flatMapMany(req -> peopleService.listAll())
            .map(people -> PeopleResponseGrpc.newBuilder()
                .setId(people.getId())
                .setName(people.getName())
                .setEmail(people.getEmail())
                .build())
            .collectList()
            .map(peopleList -> ListPeopleResponseGrpc.newBuilder()
                .addAllPeople(peopleList)
                .build());
    }
}
```

**Responsabilidades:**
- Expor serviços gRPC reativos
- Receber requisições gRPC do API Service
- Delegar para o serviço de aplicação
- Mapear DTOs para protobuf messages
- **Inversão de Dependência:** Injeta interface `PeopleService`, não implementação

#### 2. PeopleService (`application/service/`)

**PeopleService (Interface)**
```java
public interface PeopleService {
    Mono<PeopleResponse> getById(Integer id);
    Flux<PeopleResponse> listAll();
}
```

**PeopleServiceImpl**
```java
@Service
@RequiredArgsConstructor  // Lombok
public class PeopleServiceImpl implements PeopleService {

    private final PeopleRepository peopleRepository;

    @Override
    public Mono<PeopleResponse> getById(Integer id) {
        return peopleRepository.findById(id);
    }

    @Override
    public Flux<PeopleResponse> listAll() {
        return peopleRepository.findAll();
    }
}
```

**Responsabilidades:**
- Orquestrar a lógica de negócio
- Delegar para o repositório
- Camada de abstração entre gRPC service e infraestrutura

#### 3. PeopleRepositoryImpl (`infrastructure/repository/PeopleRepositoryImpl.java`)

```java
@RequiredArgsConstructor  // Lombok
public class PeopleRepositoryImpl implements PeopleRepository {

    private final Map<DataSource, PeopleClient> clientStrategies;
    private final DataSource activeDataSource;

    @Override
    public Mono<PeopleResponse> findById(Integer id) {
        return getActiveClient().getPeopleById(id);
    }

    @Override
    public Flux<PeopleResponse> findAll() {
        return getActiveClient().listPeople();
    }

    private PeopleClient getActiveClient() {
        return clientStrategies.get(activeDataSource);
    }
}
```

**Responsabilidades:**
- Implementar o padrão **Strategy** para seleção dinâmica de fonte de dados
- Gerenciar múltiplas implementações de PeopleClient (TypiCode, Dummy)
- Rotear requisições para a API externa configurada via `client.active-datasource`
- Abstrair a complexidade de múltiplas APIs para o serviço

**Configuração (RepositoryConfig.java):**
```java
@Configuration
public class RepositoryConfig {

    @Bean
    public PeopleRepository peopleRepository(
            TypiCodeClientImpl typiCodeClient,
            DummyClientImpl dummyClient,
            @Value("${client.active-datasource:TYPICODE}") String activeDataSourceStr) {

        Map<DataSource, PeopleClient> clientStrategies = new HashMap<>();
        clientStrategies.put(DataSource.TYPICODE, typiCodeClient);
        clientStrategies.put(DataSource.DUMMY, dummyClient);

        DataSource activeDataSource = DataSource.valueOf(activeDataSourceStr.toUpperCase());

        return new PeopleRepositoryImpl(clientStrategies, activeDataSource);
    }
}
```

A fonte de dados ativa é configurada no `application.yml`:
```yaml
client:
  active-datasource: DUMMY  # ou TYPICODE
```

#### 4. TypiCodeClientImpl (`infrastructure/client/typicode/TypiCodeClientImpl.java`)

```java
@Component
@RequiredArgsConstructor  // Lombok
public class TypiCodeClientImpl implements PeopleClient {

    @Qualifier("typiCodeWebClient")
    private final WebClient typiCodeWebClient;

    private final TypiCodeMapper mapper;  // MapStruct

    @Override
    public Mono<PeopleResponse> getPeopleById(Integer id) {
        return typiCodeWebClient
            .get()
            .uri("/users/{id}", id)
            .retrieve()
            .bodyToMono(TypiCodeResponse.class)
            .map(mapper::toPeopleResponse);  // MapStruct
    }

    @Override
    public Flux<PeopleResponse> listPeople() {
        return typiCodeWebClient
            .get()
            .uri("/users")
            .retrieve()
            .bodyToFlux(TypiCodeResponse.class)
            .map(mapper::toPeopleResponse);  // MapStruct
    }
}
```

**Responsabilidades:**
- Implementar a interface PeopleClient para JSONPlaceholder API
- Realizar chamadas HTTP reativas para `https://jsonplaceholder.typicode.com`
- Mapear respostas JSON para DTOs usando **MapStruct**
- Tratamento de erros reativo

**API Externa:** [JSONPlaceholder](https://jsonplaceholder.typicode.com/)

#### 5. TypiCodeMapper (`infrastructure/client/typicode/TypiCodeMapper.java`)

```java
@Mapper(
    componentModel = "spring",
    implementationName = "TypiCodeMapperImpl"
)
public interface TypiCodeMapper {

    @Mapping(target = "id", source = "id")
    @Mapping(target = "name", source = "name")
    @Mapping(target = "email", source = "email")
    PeopleResponse toPeopleResponse(TypiCodeResponse response);
}
```

**Responsabilidades:**
- Mapeamento **type-safe** de campos da API JSONPlaceholder
- Geração de código em tempo de compilação via MapStruct

#### 6. DummyClientImpl (`infrastructure/client/dummy/DummyClientImpl.java`)

```java
@Component
@RequiredArgsConstructor  // Lombok
public class DummyClientImpl implements PeopleClient {

    @Qualifier("dummyWebClient")
    private final WebClient dummyWebClient;

    private final DummyMapper mapper;  // MapStruct

    @Override
    public Mono<PeopleResponse> getPeopleById(Integer id) {
        return dummyWebClient
            .get()
            .uri("/users/{id}", id)
            .retrieve()
            .bodyToMono(DummyResponse.class)
            .map(mapper::toPeopleResponse);  // MapStruct
    }

    @Override
    public Flux<PeopleResponse> listPeople() {
        return dummyWebClient
            .get()
            .uri("/users")
            .retrieve()
            .bodyToMono(DummyListResponse.class)
            .flatMapMany(response -> Flux.fromIterable(response.users()))
            .map(mapper::toPeopleResponse);  // MapStruct
    }
}
```

**Responsabilidades:**
- Implementar a interface PeopleClient para DummyJSON API
- Realizar chamadas HTTP reativas para `https://dummyjson.com`
- Mapear respostas JSON para DTOs usando **MapStruct**
- Combinar firstName e lastName em um único campo name via MapStruct
- Tratamento de erros reativo

**API Externa:** [DummyJSON](https://dummyjson.com/)

#### 7. DummyMapper (`infrastructure/client/dummy/DummyMapper.java`)

```java
@Mapper(
    componentModel = "spring",
    implementationName = "DummyMapperImpl"
)
public interface DummyMapper {

    @Mapping(target = "name",
        expression = "java(response.firstName() + \" \" + response.lastName())")
    @Mapping(target = "email", source = "email")
    @Mapping(target = "id", source = "id")
    PeopleResponse toPeopleResponse(DummyResponse response);
}
```

**Responsabilidades:**
- Mapeamento **customizado** da API DummyJSON
- Combinar campos `firstName` e `lastName` em `name` via expressão Java
- Geração de código em tempo de compilação via MapStruct

### Configuração (`application.yml`)

```yaml
spring:
  application:
    name: people

grpc:
  server:
    port: 9090

client:
  active-datasource: TYPICODE  # Options: TYPICODE, DUMMY
  typicode:
    base-url: https://jsonplaceholder.typicode.com
  dummy:
    base-url: https://dummyjson.com
```

### APIs Externas Suportadas

O People Service suporta múltiplas APIs externas através do padrão **Strategy**, permitindo trocar facilmente a fonte de dados via configuração.

#### JSONPlaceholder (TYPICODE)

**URL Base:** `https://jsonplaceholder.typicode.com`

**Endpoints utilizados:**
- `GET /users/{id}` - Buscar usuário por ID
- `GET /users` - Listar todos os usuários

**Estrutura de resposta:**
```json
{
  "id": 1,
  "name": "Leanne Graham",
  "email": "Sincere@april.biz",
  "username": "Bret",
  "address": { ... },
  "phone": "1-770-736-8031 x56442",
  "website": "hildegard.org",
  "company": { ... }
}
```

**Características:**
- API gratuita e pública para testes
- 10 usuários disponíveis
- Campos diretamente mapeados (id, name, email)

#### DummyJSON (DUMMY)

**URL Base:** `https://dummyjson.com`

**Endpoints utilizados:**
- `GET /users/{id}` - Buscar usuário por ID
- `GET /users` - Listar todos os usuários

**Estrutura de resposta:**
```json
{
  "id": 1,
  "firstName": "Emily",
  "lastName": "Johnson",
  "email": "emily.johnson@x.dummyjson.com",
  "phone": "+81 965-431-3024",
  "username": "emilys",
  "birthDate": "1996-5-30",
  "image": "https://dummyjson.com/icon/emilys/128",
  ...
}
```

**Características:**
- API gratuita com dados realistas
- Maior quantidade de usuários disponíveis
- Requer mapeamento customizado (firstName + lastName → name)

#### Seleção de Fonte de Dados

A seleção é feita através da propriedade `client.active-datasource`:

```yaml
# Para usar JSONPlaceholder
client:
  active-datasource: TYPICODE

# Para usar DummyJSON
client:
  active-datasource: DUMMY
```

O padrão Strategy permite adicionar novas APIs externas facilmente, bastando:
1. Criar uma nova implementação de `PeopleClient`
2. Criar um `Mapper` MapStruct para conversão de respostas
3. Adicionar um novo valor no enum `DataSource`
4. Registrar o client no `RepositoryConfig.java`
5. Configurar a URL base no `application.yml`

---

## Estrutura Maven

O projeto utiliza uma estrutura **Maven Multi-Module** para facilitar o build e o gerenciamento de dependências.

### POM Parent/Aggregator

Na raiz do projeto, existe um `pom.xml` que funciona como **aggregator**, gerenciando os módulos:

```xml
<groupId>org</groupId>
<artifactId>bridge</artifactId>
<version>1.0-SNAPSHOT</version>
<packaging>pom</packaging>

<modules>
    <module>api</module>
    <module>people</module>
</modules>
```

**Benefícios:**
- **Build unificado**: `mvn clean install` na raiz compila ambos os serviços
- **Gerenciamento centralizado**: Versões e configurações podem ser compartilhadas
- **Ordem de build**: Maven resolve automaticamente a ordem de compilação dos módulos

---

## Configuração e Execução

### Pré-requisitos

- Java 21+
- Maven 3.6+
- Conexão com a internet (para acessar JSONPlaceholder)

### Compilação

Na raiz do projeto:

```bash
# Compilar todos os módulos
mvn clean install

# Ou compilar individualmente
cd people
mvn clean install

cd ../api
mvn clean install
```

### Executar os Serviços

**Importante:** O People Service deve ser iniciado ANTES do API Service.

#### 1. Iniciar o People Service

```bash
cd people
mvn spring-boot:run
```

O serviço estará disponível em:
- gRPC Server: `localhost:9090`

#### 2. Iniciar o API Service

```bash
cd api
mvn spring-boot:run
```

O serviço estará disponível em:
- REST API: `http://localhost:8081`

### Verificar a Saúde dos Serviços

```bash
# Verificar People Service (via API Service)
curl http://localhost:8081/api/peoples/1

# Listar todas as pessoas
curl http://localhost:8081/api/peoples
```

---

## Endpoints Disponíveis

### API Service (REST)

#### GET /api/peoples/{id}

Busca uma pessoa por ID.

**Request:**
```http
GET http://localhost:8081/api/peoples/1
```

**Response:**
```json
{
  "id": 1,
  "name": "Leanne Graham",
  "email": "Sincere@april.biz"
}
```

**Status Codes:**
- `200 OK`: Pessoa encontrada
- `500 Internal Server Error`: Erro ao buscar pessoa

---

#### GET /api/peoples

Lista todas as pessoas.

**Request:**
```http
GET http://localhost:8081/api/peoples
```

**Response:**
```json
[
  {
    "id": 1,
    "name": "Leanne Graham",
    "email": "Sincere@april.biz"
  },
  {
    "id": 2,
    "name": "Ervin Howell",
    "email": "Shanna@melissa.tv"
  }
]
```

**Status Codes:**
- `200 OK`: Lista retornada com sucesso
- `500 Internal Server Error`: Erro ao listar pessoas

---

## Tratamento de Erros

### API Service

Erros são propagados do People Service para o API Service e retornados como HTTP 500.

### People Service

O People Service possui um sistema robusto de tratamento de exceções:

**Hierarquia de Exceções:**

```
PeopleException (Base)
├── BusinessRuleException (Regras de negócio)
├── ValidationException (Validações)
└── PeopleNotFoundException (Recurso não encontrado)

Infrastructure Exceptions
├── ExternalServiceException (Erros de serviços externos)
└── InternalServerException (Erros internos do servidor)
```

**GlobalGrpcExceptionHandler:**
- Intercepta todas as exceções dos serviços gRPC
- Converte exceções de domínio em status codes gRPC apropriados
- Adiciona contexto e mensagens de erro estruturadas
- Registra erros no sistema de logging

**Status Codes gRPC Retornados:**
- `NOT_FOUND`: Quando recurso não é encontrado
- `INVALID_ARGUMENT`: Para erros de validação
- `FAILED_PRECONDITION`: Para violações de regras de negócio
- `UNAVAILABLE`: Quando serviços externos estão indisponíveis
- `INTERNAL`: Para erros internos não esperados

---

## Observabilidade

### Logs Importantes

**API Service:**
- Requisições HTTP recebidas
- Chamadas gRPC realizadas
- Erros de comunicação

**People Service:**
- Requisições gRPC recebidas
- Chamadas HTTP para JSONPlaceholder
- Erros de integração

### Logging Estruturado (People Service)

O People Service implementa logging estruturado usando **Logstash Logback Encoder**:

**Componentes:**
- **Logger**: Facade customizado para logging estruturado
- **LogContext**: Contexto de log com dados da requisição
- **RequestContext**: Armazena informações de contexto da requisição
- **GrpcLoggingInterceptor**: Interceptor gRPC para log automático de requisições

**Benefícios:**
- Logs em formato JSON para fácil parsing
- Rastreabilidade de requisições com correlation IDs
- Integração com ferramentas de agregação de logs (ELK Stack, Datadog)

### Monitoramento e Observabilidade

**People Service** possui integração com:
- **Datadog APM**: Trace distribuído e métricas de aplicação
- **Logstash**: Logs estruturados em JSON
- **gRPC Server Reflection**: Introspecção de serviços gRPC

**Ambos os serviços** suportam:
- Spring Boot Actuator (pode ser adicionado)
- Métricas Micrometer
- Health checks

---

## Considerações de Segurança

### Configuração Atual

- **Comunicação gRPC**: PLAINTEXT (sem TLS)
- **Comunicação HTTP**: Sem autenticação

### Recomendações para Produção

1. Habilitar TLS/SSL para gRPC
2. Implementar autenticação/autorização (JWT, OAuth2)
3. Configurar rate limiting
4. Adicionar circuit breakers (Resilience4j)
5. Implementar timeouts apropriados
6. Validar inputs

---

## Próximos Passos

- [ ] Adicionar testes de integração
- [ ] Implementar circuit breaker para chamadas externas
- [ ] Adicionar autenticação e autorização
- [ ] Configurar TLS para gRPC
- [ ] Adicionar métricas e tracing distribuído
- [ ] Implementar cache para reduzir chamadas à API externa
- [ ] Adicionar documentação OpenAPI/Swagger para REST API
- [ ] Configurar Docker e Docker Compose
- [ ] Implementar health checks

---

## Referências

### Frameworks e Tecnologias
- [gRPC Documentation](https://grpc.io/docs/)
- [Protocol Buffers](https://developers.google.com/protocol-buffers)
- [Spring WebFlux](https://docs.spring.io/spring-framework/reference/web/webflux.html)
- [Project Reactor](https://projectreactor.io/docs)
- [MapStruct](https://mapstruct.org/)

### APIs Externas
- [JSONPlaceholder API](https://jsonplaceholder.typicode.com/) - API de teste com dados fake
- [DummyJSON API](https://dummyjson.com/) - API de teste com dados realistas

<hr />

<div>
  <sub>Conteúdo criado por <a href="https://github.com/eneas-almeida">Enéas Almeida</a></sub>
</div>
