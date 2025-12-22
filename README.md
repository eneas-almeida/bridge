# Bridge - Documentação da Arquitetura

## Visão Geral

O projeto **Bridge** é uma arquitetura de microserviços composta por dois serviços principais que se comunicam via **gRPC**:

- **API Service**: Gateway REST que expõe endpoints HTTP e se comunica com o serviço People via gRPC
- **People Service**: Serviço backend que fornece dados de usuários via gRPC, consumindo a API externa JSONPlaceholder

```
┌──────────────┐     HTTP/REST      ┌──────────────┐      gRPC       ┌──────────────┐     HTTP
│   Cliente    │ ─────────────────> │  API Service │ ──────────────> │People Service│ ──────────> JSONPlaceholder
│  (Browser)   │                    │  (Port 8081) │                 │ (Port 9090)  │             (External API)
└──────────────┘                    └──────────────┘                 └──────────────┘
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
│              (Use Cases)                    │
└────────────────┬────────────────────────────┘
                 │
┌────────────────▼────────────────────────────┐
│            Domain Layer                     │
│      (Entities / Client Interfaces)         │
└────────────────┬────────────────────────────┘
                 │
┌────────────────▼────────────────────────────┐
│        Infrastructure Layer                 │
│   (gRPC Clients/Servers / HTTP Clients)     │
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
Cliente                 API Service                People Service              JSONPlaceholder
  │                          │                           │                            │
  │ GET /api/peoples/1       │                           │                            │
  ├─────────────────────────>│                           │                            │
  │                          │                           │                            │
  │                          │ PeopleController          │                            │
  │                          │        ↓                  │                            │
  │                          │ GetPeopleUseCase          │                            │
  │                          │        ↓                  │                            │
  │                          │ PeopleGrpcClient          │                            │
  │                          │                           │                            │
  │                          │ gRPC: GetPeople(id=1)     │                            │
  │                          ├──────────────────────────>│                            │
  │                          │                           │ PeopleGrpcService          │
  │                          │                           │        ↓                   │
  │                          │                           │ GetPeopleUseCase           │
  │                          │                           │        ↓                   │
  │                          │                           │ TypiCodePeopleClientImpl   │
  │                          │                           │                            │
  │                          │                           │ GET /users/1               │
  │                          │                           ├───────────────────────────>│
  │                          │                           │                            │
  │                          │                           │ JSON Response              │
  │                          │                           │<───────────────────────────┤
  │                          │                           │                            │
  │                          │                           │ Map to Domain Entity       │
  │                          │                           │ (People)                   │
  │                          │                           │                            │
  │                          │ PeopleResponse (protobuf) │                            │
  │                          │<──────────────────────────┤                            │
  │                          │                           │                            │
  │                          │ Map to Domain Entity      │                            │
  │                          │ (People)                  │                            │
  │                          │        ↓                  │                            │
  │                          │ Map to DTO                │                            │
  │                          │ (PeopleResponse)          │                            │
  │                          │                           │                            │
  │ JSON Response            │                           │                            │
  │<─────────────────────────┤                           │                            │
  │                          │                           │                            │
```

### Fluxo 2: Listar Todas as Pessoas

```
Cliente                 API Service                People Service              JSONPlaceholder
  │                          │                           │                            │
  │ GET /api/peoples         │                           │                            │
  ├─────────────────────────>│                           │                            │
  │                          │                           │                            │
  │                          │ PeopleController          │                            │
  │                          │        ↓                  │                            │
  │                          │ ListPeopleUseCase         │                            │
  │                          │        ↓                  │                            │
  │                          │ PeopleGrpcClient          │                            │
  │                          │                           │                            │
  │                          │ gRPC: ListPeople()        │                            │
  │                          ├──────────────────────────>│                            │
  │                          │                           │ PeopleGrpcService          │
  │                          │                           │        ↓                   │
  │                          │                           │ ListPeopleUseCase          │
  │                          │                           │        ↓                   │
  │                          │                           │ TypiCodePeopleClientImpl   │
  │                          │                           │                            │
  │                          │                           │ GET /users                 │
  │                          │                           ├───────────────────────────>│
  │                          │                           │                            │
  │                          │                           │ JSON Array Response        │
  │                          │                           │<───────────────────────────┤
  │                          │                           │                            │
  │                          │                           │ Map to Flux<People>        │
  │                          │                           │                            │
  │                          │ ListPeopleResponse        │                            │
  │                          │ (repeated PeopleResponse) │                            │
  │                          │<──────────────────────────┤                            │
  │                          │                           │                            │
  │                          │ Map to Flux<People>       │                            │
  │                          │        ↓                  │                            │
  │                          │ Map to Flux<DTO>          │                            │
  │                          │                           │                            │
  │ JSON Array Response      │                           │                            │
  │<─────────────────────────┤                           │                            │
  │                          │                           │                            │
```

---

## API Service

### Estrutura do Projeto

```
api/
├── src/main/java/org/api/
│   ├── presentation/
│   │   ├── controller/
│   │   │   └── PeopleController.java
│   │   └── dto/
│   │       └── PeopleResponse.java
│   ├── application/
│   │   └── usecase/
│   │       ├── GetPeopleUseCase.java
│   │       └── ListPeopleUseCase.java
│   ├── domain/
│   │   ├── client/
│   │   │   └── PeopleClient.java (Interface)
│   │   └── entity/
│   │       └── People.java
│   └── infrastructure/
│       └── grpc/
│           └── PeopleGrpcClient.java
├── src/main/proto/
│   └── person.proto
└── src/main/resources/
    └── application.yml
```

### Componentes Principais

#### 1. PeopleController (`presentation/controller/PeopleController.java`)

```java
@RestController
@RequestMapping("/api/peoples")
public class PeopleController {

    @GetMapping("/{id}")
    public Mono<PeopleResponse> getPeopleById(@PathVariable int id)

    @GetMapping
    public Flux<PeopleResponse> listPeople()
}
```

**Responsabilidades:**
- Expor endpoints REST HTTP
- Receber requisições do cliente
- Delegar para os Use Cases
- Mapear entidades de domínio para DTOs de resposta

#### 2. Use Cases (`application/usecase/`)

**GetPeopleUseCase**
```java
public class GetPeopleUseCase {
    public Mono<People> execute(Integer peopleId)
}
```

**ListPeopleUseCase**
```java
public class ListPeopleUseCase {
    public Flux<People> execute()
}
```

**Responsabilidades:**
- Orquestrar a lógica de negócio
- Chamar os clientes de integração (gRPC)

#### 3. PeopleGrpcClient (`infrastructure/grpc/PeopleGrpcClient.java`)

```java
@Component
public class PeopleGrpcClient implements PeopleClient {

    @GrpcClient("people-service")
    private PeopleServiceBlockingStub stub;

    public Mono<People> getPeopleById(int id)
    public Flux<People> listPeople()
}
```

**Responsabilidades:**
- Implementar a interface PeopleClient
- Realizar chamadas gRPC para o People Service
- Converter chamadas bloqueantes em streams reativos (Mono/Flux)
- Mapear protobuf messages para entidades de domínio

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
│   │   └── entity/
│   │       └── People.java
│   ├── usecase/
│   │   ├── GetPeopleUseCase.java
│   │   └── ListPeopleUseCase.java
│   └── infrastructure/
│       ├── entrypoint/
│       │   └── grpc/
│       │       └── PeopleGrpcService.java
│       ├── adapter/
│       │   └── typicode/
│       │       ├── TypiCodePeopleClientImpl.java
│       │       ├── PeopleResponse.java
│       │       └── PeopleMapper.java
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
│           ├── UseCaseConfig.java
│           └── TypiCodeClientConfig.java
├── src/main/proto/
│   └── person.proto
└── src/main/resources/
    └── application.yml
```

### Componentes Principais

#### 1. PeopleGrpcService (`infrastructure/entrypoint/grpc/PeopleGrpcService.java`)

```java
@GrpcService
public class PeopleGrpcService extends PeopleServiceImplBase {

    @Override
    public void getPeople(PeopleRequest request,
                          StreamObserver<PeopleResponse> responseObserver)

    @Override
    public void listPeople(ListPeopleRequest request,
                           StreamObserver<ListPeopleResponse> responseObserver)
}
```

**Responsabilidades:**
- Expor serviços gRPC
- Receber requisições gRPC do API Service
- Delegar para os Use Cases
- Mapear entidades de domínio para protobuf messages

#### 2. Use Cases (`usecase/`)

**GetPeopleUseCase**
```java
public class GetPeopleUseCase {
    public Mono<People> execute(Integer peopleId)
}
```

**ListPeopleUseCase**
```java
public class ListPeopleUseCase {
    public Flux<People> execute()
}
```

**Responsabilidades:**
- Orquestrar a lógica de negócio
- Chamar o cliente HTTP para JSONPlaceholder

#### 3. TypiCodePeopleClientImpl (`infrastructure/adapter/typicode/TypiCodePeopleClientImpl.java`)

```java
@Component
public class TypiCodePeopleClientImpl implements PeopleClient {

    @Autowired
    private WebClient typiCodeWebClient;

    public Mono<People> findById(Integer id)
    public Flux<People> listAll()
}
```

**Responsabilidades:**
- Implementar a interface PeopleClient
- Realizar chamadas HTTP para a API externa JSONPlaceholder
- Mapear respostas JSON para entidades de domínio usando MapStruct

#### 4. PeopleMapper (`infrastructure/adapter/typicode/PeopleMapper.java`)

```java
@Mapper(componentModel = "spring")
public interface PeopleMapper {
    People toPeople(PeopleResponse response);
}
```

**Responsabilidades:**
- Mapeamento type-safe de DTOs para entidades de domínio
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
  typicode:
    base-url: https://jsonplaceholder.typicode.com
```

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

- [gRPC Documentation](https://grpc.io/docs/)
- [Protocol Buffers](https://developers.google.com/protocol-buffers)
- [Spring WebFlux](https://docs.spring.io/spring-framework/reference/web/webflux.html)
- [Project Reactor](https://projectreactor.io/docs)
- [JSONPlaceholder API](https://jsonplaceholder.typicode.com/)

<hr />

<div>
  <sub>Conteúdo criado por <a href="https://github.com/eneas-almeida">Enéas Almeida</a></sub>
</div>
