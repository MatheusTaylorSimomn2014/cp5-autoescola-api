#  Checkpoint 5 - SOA e WebServices: 
## AutoEscola API 

API **ReST** desenvolvida com **Spring Boot 3 (Java 17)** para agendamento de instruções (aulas práticas) de uma auto-escola.

## Integrantes do Grupo

| Nome | RM |
|------|----|
| Matheus Taylor | RM556211 |
| Henrique Maldonado | RM557270 |
| Yuri Silveira | RM557270 |

---

## O que a API entrega

###  Já implementado (checkpoints anteriores)
- **CRUD completo** para: Instrutores (`/api/instrutores`), Alunos (`/api/alunos`), Usuários (`/api/usuarios`) e Instruções (`/api/instrucoes`).
- **Serviço de agendamento e cancelamento** com regras de negócio:
  - **R1** — Início anterior ao término; duração máxima de 4 horas.
  - **R2** — Somente dias úteis, dentro do horário comercial (08h–18h, mesmo dia).
  - **R3** — Proibição de sobreposição de agenda do instrutor (conflito).
  - **R4** — Aluno deve estar com processo ativo (À_BAURA ou EM_ANDAMENTO).
  - **R5** — Antecedência mínima de 24h para agendar.
  - **R6** — Cancelamento apenas de instruções `AGENDADA`, com antecedência mínima de 24h; limite configurável de cancelamentos por aluno (`app.regra.limite-cancelamentos`).
- Tratamento global de erros (`@RestControllerAdvice`) com payloads padronizados.
- Banco H2 em memória para dev/testes (driver PostgreSQL incluído para produção).

### Novas Implementações do Checkpoint 5

#### 1. Consumo de API externa (ViaCEP)
- `client/ViaCepClient.java`: usa o **`RestClient`** do Spring Framework 6 para consultar `https://viacep.com.br/ws/{cep}/json/`.
- Exposta em `GET /api/cep/{cep}` (`controller/CepController.java`).
- Usada automaticamente pelo `EnderecoService`: ao cadastrar aluno/instrutor informando apenas o CEP, logradouro/bairro/cidade/UF são preenchidos pela consulta externa (com fallback para dados manuais e tratamento de indisponibilidade → HTTP 503).
- URL configurável via `app.api.viacep.base-url`.

#### 2. Documentação automática com Swagger/OpenAPI 3
- Dependência `springdoc-openapi-starter-webmvc-ui` + customização em `config/OpenApiConfig.java`.
- Anotações `@Tag`, `@Operation` e `@Schema` nos controllers e DTOs.
- Acesse:
  - **Swagger UI:** http://localhost:8080/swagger-ui.html
  - **Spec OpenAPI:** http://localhost:8080/v3/api-docs

#### 3. Configuração de CORS
- `config/CorsConfig.java` (`WebMvcConfigurer`) libera `/api/**` para os métodos `GET, POST, PUT, PATCH, DELETE, OPTIONS`, com credenciais e `maxAge`.
- Origens permitidas configuráveis via propriedade `app.cors.allowed-origins` (padrão: `http://localhost:3000,http://localhost:5173`).

#### 4. Testes automatizados por entidade
| Classe | Cobre |
|--------|-------|
| `AlunoControllerTest` | CRUD de alunos, unicidade de CPF, validações de payload |
| `InstrutorControllerTest` | CRUD de instrutores, unicidade de CNH/CPF, validações |
| `UsuarioControllerTest` | CRUD de usuários, login duplicado, perfil inválido, senha curta |
| `InstrucaoControllerTest` | Agendamento/cancelamento/conclusão + regras R1, R3, R5, R6 |
| `ViaCepClientTest` | Cliente externo com `MockRestServiceServer` (sem rede): sucesso, CEP inexistente, erro 5xx, formato inválido |

Executar: `mvn test`

---

## Como rodar

```bash
# requisito: JDK 17+ (Maven 3.9+ ou use ./mvnw se adicionar o wrapper)
mvn spring-boot:run
```

- Porta padrão: **8080**
- Swagger UI: `http://localhost:8080/swagger-ui.html`
- Console H2 (dev): `http://localhost:8080/h2-console` (JDBC URL `jdbc:h2:mem:autoescola`, user `sa`, senha vazia)

### Exemplo rápido (cURL)

```bash
# Criar instrutor (endereço enriquecido automaticamente via ViaCEP)
curl -X POST http://localhost:8080/api/instrutores -H "Content-Type: application/json" -d '{
  "nome": "Carlos Eduardo", "cpf": "123.456.789-00", "cnh": "01234567890",
  "dataNascimento": "1985-05-10", "endereco": { "cep": "01310-100" }
}'

# Criar aluno
curl -X POST http://localhost:8080/api/alunos -H "Content-Type: application/json" -d '{
  "nome": "Maria da Silva", "cpf": "987.654.321-00",
  "dataNascimento": "2000-01-15", "endereco": { "cep": "04538-133" }
}'

# Agendar uma aula prática (dia útil, 08h-18h, com >24h de antecedência)
curl -X POST http://localhost:8080/api/instrucoes -H "Content-Type: application/json" -d '{
  "alunoId": 1, "instrutorId": 1,
  "dataHoraInicio": "2026-10-20T14:00:00", "dataHoraFim": "2026-10-20T15:00:00",
  "categoria": "B", "observacao": "Aula de baliza"
}'

# Cancelar a aula
curl -X PATCH http://localhost:8080/api/instrucoes/1/cancelamento -H "Content-Type: application/json" -d '{
  "motivo": "Imprevisto do aluno"
}'

# Consulta direta à API externa ViaCEP
curl http://localhost:8080/api/cep/01310-100
```

## Estrutura do projeto

```
src/main/java/com/fiap/autoescola
├── AutoEscolaApplication.java
├── client/      -> ViaCepClient, ViaCepResponse (API externa)
├── config/      -> OpenApiConfig (Swagger), CorsConfig (CORS), RestClientConfig
├── controller/  -> Instrutor, Aluno, Usuario, Instrucao, Cep
├── dto/         -> Request/Response + ErroResponse
├── exception/   -> GlobalExceptionHandler + exceções de domínio
├── model/       -> Entidades JPA + enums de status
├── repository/  -> Spring Data JPA
└── service/     -> Regras de negócio (agendamento/cancelamento, endereço c/ ViaCEP)
```
