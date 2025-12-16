# People Service

Serviço de gerenciamento de usuários baseado em gRPC que consome dados da API pública JSONPlaceholder. O projeto foi desenvolvido utilizando Spring Boot e segue princípios de Clean Architecture.

## 📋 Índice

- [Visão Geral](#-visão-geral)
- [Arquitetura](#-arquitetura)
- [Tecnologias](#-tecnologias)
- [Estrutura do Projeto](#-estrutura-do-projeto)
- [Pré-requisitos](#-pré-requisitos)
- [Configuração](#-configuração)
- [Como Executar](#-como-executar)
- [API gRPC](#-api-grpc)
- [Testes](#-testes)
- [Detalhes Técnicos](#-detalhes-técnicos)

## 🎯 Visão Geral

O **People Service** é um microserviço que expõe uma API gRPC para consulta de informações de usuários. O serviço atua como um intermediário entre clientes gRPC e a API REST pública do JSONPlaceholder, aplicando os conceitos de Clean Architecture para garantir separação de responsabilidades e facilitar a manutenção.

### Funcionalidades Principais

- **Buscar usuário por ID**: Retorna informações detalhadas de um usuário específico
- **Listar todos os usuários**: Retorna uma lista com todos os usuários disponíveis
- **Comunicação reativa**: Utiliza WebFlux para chamadas HTTP não-bloqueantes
- **Interface gRPC**: API de alto desempenho para comunicação entre serviços

## 🏗 Arquitetura

O projeto segue os princípios da **Clean Architecture**, organizando o código em camadas bem definidas:

```
┌─────────────────────────────────────────┐
│      Entrypoint (gRPC Service)          │
│   - UserGrpcService                     │
└─────────────┬───────────────────────────┘
              │
┌─────────────▼───────────────────────────┐
│          Use Cases                      │
│   - GetUserUseCase                      │
│   - ListUsersUseCase                    │
└─────────────┬───────────────────────────┘
              │
┌─────────────▼───────────────────────────┐
│       Domain Layer                      │
│   - User (Entity)                       │
│   - UserClient (Interface)              │
└─────────────┬───────────────────────────┘
              │
┌─────────────▼───────────────────────────┐
│      Infrastructure Adapter             │
│   - TypicodeUserClientAdapter           │
│   - UserMapper                          │
└─────────────────────────────────────────┘
```

### Camadas

1. **Domain** (`org.people.domain`)
   - Contém as entidades de negócio e interfaces de cliente
   - Livre de dependências externas
   - Define os contratos que devem ser implementados

2. **Use Cases** (`org.people.usecase`)
   - Implementa a lógica de negócio da aplicação
   - Orquestra as interações entre domain e infrastructure
   - Mantém independência de frameworks

3. **Infrastructure** (`org.people.infrastructure`)
   - **Adapters**: Implementações concretas das interfaces de domínio
   - **Entrypoints**: Pontos de entrada da aplicação (gRPC)
   - **Config**: Configurações e beans do Spring

## 🚀 Tecnologias

### Core
- **Java 21**: Versão LTS mais recente com recursos modernos
- **Spring Boot 3.3.3**: Framework principal para desenvolvimento
- **Maven**: Gerenciamento de dependências e build

### Comunicação
- **gRPC 1.58.0**: Framework RPC de alto desempenho
- **Protocol Buffers 3.24.3**: Serialização de dados
- **Spring WebFlux**: Cliente HTTP reativo e não-bloqueante

### Utilitários
- **Lombok**: Redução de boilerplate code
- **MapStruct 1.5.5**: Mapeamento automático entre objetos
- **net.devh:grpc-server-spring-boot-starter**: Integração gRPC com Spring Boot

### Programação Reativa
- **Project Reactor**: Implementação do Reactive Streams
  - `Mono<T>`: Para operações que retornam 0 ou 1 elemento
  - `Flux<T>`: Para operações que retornam 0 a N elementos

## 📁 Estrutura do Projeto

```
people/
├── src/
│   ├── main/
│   │   ├── java/org/people/
│   │   │   ├── domain/
│   │   │   │   ├── client/
│   │   │   │   │   └── UserClient.java              # Interface do cliente de usuários
│   │   │   │   └── entity/
│   │   │   │       └── User.java                    # Entidade de domínio
│   │   │   │
│   │   │   ├── usecase/
│   │   │   │   ├── GetUserUseCase.java              # Caso de uso: buscar usuário
│   │   │   │   └── ListUsersUseCase.java            # Caso de uso: listar usuários
│   │   │   │
│   │   │   ├── infrastructure/
│   │   │   │   ├── adapter/typicode/
│   │   │   │   │   ├── TypicodeUserClientAdapter.java  # Implementação do cliente
│   │   │   │   │   ├── UserMapper.java                  # Mapper entre DTOs e entidades
│   │   │   │   │   └── UserResponse.java                # DTO de resposta da API
│   │   │   │   │
│   │   │   │   ├── config/
│   │   │   │   │   └── UseCaseConfig.java            # Configuração dos use cases
│   │   │   │   │
│   │   │   │   └── entrypoint/grpc/
│   │   │   │       └── UserGrpcService.java          # Serviço gRPC
│   │   │   │
│   │   │   └── PeopleApplication.java                # Classe principal
│   │   │
│   │   ├── proto/
│   │   │   └── service.proto                         # Definição do serviço gRPC
│   │   │
│   │   └── resources/
│   │       └── application.yml                       # Configurações da aplicação
│   │
│   └── test/
│       └── java/org/people/
│           └── PeopleApplicationTests.java
│
├── target/                                          # Arquivos compilados
├── pom.xml                                          # Configuração Maven
├── mvnw                                             # Maven Wrapper (Unix)
└── mvnw.cmd                                         # Maven Wrapper (Windows)
```

## 📋 Pré-requisitos

- **Java Development Kit (JDK) 21** ou superior
- **Maven 3.6+** (ou utilize o Maven Wrapper incluído no projeto)
- **Git** (para clonar o repositório)
- Conexão com a internet (para acessar a API JSONPlaceholder)

### Verificar Instalações

```bash
# Verificar versão do Java
java -version

# Verificar versão do Maven
mvn -version
```

## ⚙ Configuração

### Variáveis de Ambiente

O projeto utiliza as seguintes variáveis de ambiente:

| Variável | Descrição | Valor Padrão | Obrigatório |
|----------|-----------|--------------|-------------|
| `TYPICODE_BASE_URL` | URL base da API JSONPlaceholder | `https://jsonplaceholder.typicode.com` | Não |

### Arquivo application.yml

```yaml
spring:
  application:
    name: people

grpc:
  server:
    port: 9090

client:
  typicode:
    base-url: ${TYPICODE_BASE_URL:https://jsonplaceholder.typicode.com}
```

### Customização

Para alterar a URL base da API externa, você pode:

1. **Definir variável de ambiente**:
```bash
export TYPICODE_BASE_URL=https://sua-api.exemplo.com
```

2. **Ou modificar diretamente no application.yml**:
```yaml
client:
  typicode:
    base-url: https://sua-api.exemplo.com
```

## 🏃 Como Executar

### Usando Maven Wrapper (Recomendado)

#### Windows
```cmd
# Limpar e compilar o projeto
.\mvnw.cmd clean install

# Executar a aplicação
.\mvnw.cmd spring-boot:run
```

#### Unix/Linux/MacOS
```bash
# Limpar e compilar o projeto
./mvnw clean install

# Executar a aplicação
./mvnw spring-boot:run
```

### Usando Maven Local

```bash
# Limpar e compilar
mvn clean install

# Executar
mvn spring-boot:run
```

### Executando o JAR Gerado

```bash
# Gerar o JAR
mvn clean package

# Executar o JAR
java -jar target/people-0.0.1-SNAPSHOT.jar
```

### Verificar Execução

Após iniciar a aplicação, você verá logs indicando que o servidor gRPC está rodando:

```
gRPC Server started, listening on address: *, port: 9090
```

## 🔌 API gRPC

### Definição do Serviço (Protocol Buffers)

O serviço gRPC está definido em `src/main/proto/service.proto`:

```protobuf
syntax = "proto3";

option java_package = "com.people.grpc";
option java_outer_classname = "ServiceProto";

package grpcservice;

service UserService {
  rpc GetUser (UserRequest) returns (UserResponse);
  rpc ListUsers (ListUsersRequest) returns (ListUsersResponse);
}

message UserRequest {
  int32 id = 1;
}

message ListUsersRequest {
}

message UserResponse {
  int32 id = 1;
  string name = 2;
  string email = 3;
}

message ListUsersResponse {
  repeated UserResponse users = 1;
}
```

### Endpoints Disponíveis

#### 1. GetUser
Busca um usuário específico por ID.

**Request:**
```protobuf
message UserRequest {
  int32 id = 1;
}
```

**Response:**
```protobuf
message UserResponse {
  int32 id = 1;
  string name = 2;
  string email = 3;
}
```

**Exemplo de Uso com grpcurl:**
```bash
grpcurl -plaintext -d '{"id": 1}' localhost:9090 grpcservice.UserService/GetUser
```

**Resposta Esperada:**
```json
{
  "id": 1,
  "name": "Leanne Graham",
  "email": "Sincere@april.biz"
}
```

#### 2. ListUsers
Lista todos os usuários disponíveis.

**Request:**
```protobuf
message ListUsersRequest {
}
```

**Response:**
```protobuf
message ListUsersResponse {
  repeated UserResponse users = 1;
}
```

**Exemplo de Uso com grpcurl:**
```bash
grpcurl -plaintext localhost:9090 grpcservice.UserService/ListUsers
```

**Resposta Esperada:**
```json
{
  "users": [
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
    // ... mais usuários
  ]
}
```

### Testando com grpcurl

Para testar os endpoints, você pode usar a ferramenta [grpcurl](https://github.com/fullstorydev/grpcurl):

```bash
# Instalar grpcurl (exemplo para Linux)
go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest

# Listar serviços disponíveis
grpcurl -plaintext localhost:9090 list

# Descrever um serviço
grpcurl -plaintext localhost:9090 describe grpcservice.UserService
```

### Tratamento de Erros

O serviço retorna erros gRPC padrão:

- **OK (0)**: Operação bem-sucedida
- **INTERNAL (13)**: Erro ao buscar dados da API externa
  - Exemplo: `Error fetching user: 404 Not Found`

## 🧪 Testes

### Executar Todos os Testes

```bash
# Com Maven Wrapper
./mvnw test

# Com Maven local
mvn test
```

### Executar Testes com Cobertura

```bash
mvn clean test jacoco:report
```

O relatório de cobertura estará disponível em: `target/site/jacoco/index.html`

### Testes Disponíveis

- **PeopleApplicationTests**: Testes de contexto da aplicação Spring Boot

## 🔧 Detalhes Técnicos

### Programação Reativa

O projeto utiliza **Project Reactor** para operações assíncronas e não-bloqueantes:

```java
// Exemplo de uso de Mono
public Mono<User> execute(Integer userId) {
    return userClient.findById(userId);
}

// Exemplo de uso de Flux
public Flux<User> execute() {
    return userClient.listAll();
}
```

**Vantagens:**
- Maior eficiência no uso de recursos
- Melhor escalabilidade
- Não bloqueia threads durante operações I/O

### MapStruct

O MapStruct é utilizado para conversão automática entre DTOs e entidades:

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
    User toUser(UserResponse userResponse);
}
```

**Benefícios:**
- Geração de código em tempo de compilação
- Type-safe
- Alto desempenho

### gRPC vs REST

**Por que gRPC?**

| Característica | gRPC | REST |
|----------------|------|------|
| **Formato** | Protocol Buffers (binário) | JSON (texto) |
| **Performance** | Mais rápido | Mais lento |
| **Tamanho payload** | Menor | Maior |
| **Tipo de contrato** | Fortemente tipado | Baseado em convenções |
| **Streaming** | Bidirecional | Limitado |
| **Geração de código** | Automática | Manual ou com ferramentas |

### Compilação do Protocol Buffers

O plugin `protobuf-maven-plugin` compila automaticamente os arquivos `.proto`:

```xml
<plugin>
    <groupId>org.xolstice.maven.plugins</groupId>
    <artifactId>protobuf-maven-plugin</artifactId>
    <version>0.6.1</version>
</plugin>
```

**Classes Geradas:**
- `ServiceProto.java`: Mensagens do Protocol Buffers
- `UserServiceGrpc.java`: Stub e interface do serviço gRPC

Localizadas em: `target/generated-sources/protobuf/`

### Configuração de Use Cases

Os use cases são configurados como beans Spring em `UseCaseConfig`:

```java
@Configuration
public class UseCaseConfig {

    @Bean
    public GetUserUseCase getUserUseCase(UserClient userClient) {
        return new GetUserUseCase(userClient);
    }

    @Bean
    public ListUsersUseCase listUsersUseCase(UserClient userClient) {
        return new ListUsersUseCase(userClient);
    }
}
```

Esta abordagem mantém os use cases independentes do Spring, facilitando testes.

### WebClient Configuração

O `WebClient` do Spring WebFlux é configurado para fazer chamadas HTTP reativas:

```java
@Component
public class TypicodeUserClientAdapter implements UserClient {
    private final WebClient webClient;

    public TypicodeUserClientAdapter(WebClient.Builder webClientBuilder) {
        this.webClient = webClientBuilder
            .baseUrl(this.baseUrl)
            .build();
    }
}
```

**Características:**
- Non-blocking I/O
- Suporte a backpressure
- Composição funcional de requisições

## 📦 Build e Deploy

### Gerar Artefato de Produção

```bash
# Compilar sem executar testes
mvn clean package -DskipTests

# Compilar com testes
mvn clean package
```

O JAR executável será gerado em: `target/people-0.0.1-SNAPSHOT.jar`

### Executar em Produção

```bash
java -jar \
  -Dspring.profiles.active=prod \
  -DTYPICODE_BASE_URL=https://api.exemplo.com \
  target/people-0.0.1-SNAPSHOT.jar
```

### Docker (Exemplo de Dockerfile)

```dockerfile
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

COPY target/people-0.0.1-SNAPSHOT.jar app.jar

EXPOSE 9090

ENV TYPICODE_BASE_URL=https://jsonplaceholder.typicode.com

ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Build e Run:**
```bash
# Build da imagem
docker build -t people-service:latest .

# Executar container
docker run -p 9090:9090 people-service:latest
```

## 🤝 Contribuindo

1. Fork o projeto
2. Crie uma branch para sua feature (`git checkout -b feature/MinhaFeature`)
3. Commit suas mudanças (`git commit -m 'Adiciona MinhaFeature'`)
4. Push para a branch (`git push origin feature/MinhaFeature`)
5. Abra um Pull Request

### Padrões de Código

- Seguir convenções Java padrão
- Manter Clean Architecture
- Adicionar testes para novas funcionalidades
- Documentar código complexo

## 📝 Licença

Este projeto é um exemplo educacional e está disponível para uso livre.

## 📧 Contato

Para dúvidas ou sugestões, abra uma issue no repositório.

---

**Desenvolvido com Java + Spring Boot + gRPC**