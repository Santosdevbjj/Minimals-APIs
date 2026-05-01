# Minimal APIs de Veículos — .NET 8 + JWT + MySQL + Docker

> **API RESTful com arquitetura minimalista:** gerenciamento de frota com autenticação JWT por roles (Admin/Editor), três ambientes isolados por variáveis de ambiente, testes automatizados e pipeline CI/CD no GitHub Actions.

[![.NET](https://img.shields.io/badge/.NET-8.0-512BD4?style=for-the-badge&logo=dotnet&logoColor=white)](https://dotnet.microsoft.com/)
[![MySQL](https://img.shields.io/badge/MySQL-8.0-4479A1?style=for-the-badge&logo=mysql&logoColor=white)](https://www.mysql.com/)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com/)
[![JWT](https://img.shields.io/badge/JWT-Bearer-000000?style=for-the-badge&logo=jsonwebtokens&logoColor=white)](https://jwt.io/)
[![CI](https://img.shields.io/github/actions/workflow/status/Santosdevbjj/Minimals-APIs/ci-cd.yml?style=for-the-badge&logo=githubactions&logoColor=white)](https://github.com/Santosdevbjj/Minimals-APIs/actions)
[![License](https://img.shields.io/badge/Licença-MIT-green?style=for-the-badge)](LICENSE)

---

## 1. Problema de Negócio

APIs construídas com o padrão MVC tradicional do ASP.NET Core carregam um custo estrutural que não se justifica em todos os contextos: controllers, action methods, model binding explícito, middleware de roteamento por convenção. Para APIs simples e focadas — como um endpoint de CRUD de veículos ou um serviço de autenticação — esse overhead aumenta a superfície de código sem adicionar valor ao negócio.

O problema concreto que este projeto endereça é duplo:

- **Complexidade desnecessária no bootstrap de APIs internas:** times que precisam de uma API funcional com autenticação, banco de dados e múltiplos ambientes frequentemente partem de templates MVC que trazem 5x mais código do que o necessário.
- **Gestão insegura de credenciais por ambiente:** sem separação explícita de configurações entre desenvolvimento, teste e produção, é comum que credenciais de produção vazem para repositórios ou sejam compartilhadas manualmente entre desenvolvedores.

**A questão central:** como construir uma API com autenticação JWT baseada em roles, integração com banco relacional, três ambientes isolados e pipeline de CI — com o menor boilerplate possível e sem comprometer segurança?

---

## 2. Contexto

Este projeto foi desenvolvido no Bootcamp GFT Start #7 .NET (DIO), com foco específico no padrão **Minimal APIs** introduzido no .NET 6 e consolidado no .NET 8. A aplicação gerencia um cadastro de veículos com controle de acesso por perfil: administradores podem criar, editar e deletar veículos; editores podem criar e listar; usuários não autenticados não têm acesso ao catálogo.

O contexto de frota de veículos é secundário ao que o projeto realmente demonstra: **como estruturar uma API profissional com Minimal APIs** — incluindo autenticação JWT com claims de role, separação de ambientes por arquivos `.env`, seed automático de usuários no banco, migrations com EF Core e testes de integração com `WebApplicationFactory`.

A escolha do MySQL como banco de dados reflete o ecossistema mais comum em empresas que já possuem infraestrutura de banco relacional, especialmente em ambientes de médio porte onde o SQL Server seria custo de licença desnecessário.

---

## 3. Premissas

Para estruturar este projeto, foram adotadas as seguintes premissas:

- Cada ambiente (Development, Test, Production) tem credenciais isoladas em arquivos `.env` separados. Nenhum arquivo `.env` com credenciais reais é versionado — o `.gitignore` bloqueia todos eles.
- O `Program.cs` centraliza toda a configuração da API: DI, JWT, Swagger, EF Core, endpoints e middleware. Essa é uma decisão intencional do padrão Minimal APIs — não é falta de organização, é a filosofia do framework.
- O seed de usuários (`DbSeed.cs`) roda apenas se a tabela de admins estiver vazia. Isso garante idempotência: rodar o seed múltiplas vezes não duplica dados.
- A validação do VIN (máximo 17 caracteres) é feita no endpoint, não apenas no modelo. Isso detecta entradas inválidas antes de chegar ao banco.
- Migrations automáticas (`db.Database.Migrate()`) são aceitáveis em desenvolvimento e primeiro deploy. Em produção, o padrão recomendado é executar migrations separadamente antes do deploy da aplicação.

---

## 4. Estratégia da Solução

**Minimal APIs com endpoints explícitos**

Em vez de controllers e action methods, cada endpoint é declarado diretamente no `Program.cs` com `app.MapGet`, `app.MapPost`, `app.MapPut`, `app.MapDelete`. Isso reduz a indireção: o desenvolvedor lê um arquivo e entende toda a API sem navegar por múltiplas classes. O trade-off — `Program.cs` fica longo em APIs grandes — é gerenciável neste escopo e pode ser resolvido com extensões em `IEndpointRouteBuilder`.

**JWT com políticas de autorização por role**

O `AuthService` gera tokens com claims de `ClaimTypes.Role` (Admin ou Editor). No `Program.cs`, duas políticas são configuradas: `AdminOnly` requer exatamente a role Admin, e `EditorOrAdmin` aceita qualquer uma das duas. Os endpoints de deleção e edição exigem `AdminOnly`; criação e listagem aceitam `EditorOrAdmin`. Essa granularidade foi uma decisão explícita — um editor não deveria poder deletar veículos da frota.

**Separação de ambientes com Docker Compose + arquivos .env**

O `docker-compose.yml` define a estrutura base dos serviços (mysql + api) sem credenciais. O `docker-compose.override.yml` injeta as variáveis de ambiente corretas baseado no `ASPNETCORE_ENVIRONMENT`. Cada ambiente tem seu arquivo `.env` com banco de dados, usuário, senha e JWT secret próprios. Isso elimina a necessidade de modificar o código para mudar de ambiente — apenas o arquivo `.env` muda.

**Seed automático com idempotência**

O `DbSeed.EnsureSeedDataAsync()` verifica se já existem usuários antes de inserir. Isso garante que o ambiente de desenvolvimento seja funcional imediatamente após o primeiro `docker-compose up`, sem passos manuais adicionais. Os usuários seed (admin@local.test / Admin@123 e editor@local.test / Editor@123) existem apenas em Development e Test — nunca em Production.

**Testes com WebApplicationFactory**

O projeto de testes usa `WebApplicationFactory<Program>` para subir a API completa em memória. O teste `GetHome_ReturnsOk` valida que o endpoint raiz responde com sucesso, servindo como smoke test que garante que a aplicação inicializa corretamente. O teste unitário `BCrypt_Hash_And_Verify_Works` valida o comportamento do hash de senha independente da camada HTTP.

---

## 5. Decisões Técnicas e Aprendizados

### Por que Minimal APIs e não Controllers?

Controllers fazem sentido quando a API tem muitos recursos, versioning complexo, filtros de ação ou necessidade de herança entre controllers. Para uma API focada como esta — dois recursos (vehicles + admins) com poucos endpoints cada — Minimal APIs eliminam ~40% do boilerplate sem perda de funcionalidade. O roteamento explícito (`app.MapGet("/vehicles/{id:int}")`) é mais fácil de ler e debugar do que convenção por atributos.

### Por que BCrypt e não PBKDF2 ou Argon2?

BCrypt tem adoção massiva no ecossistema .NET, biblioteca matura (`BCrypt.Net-Next`), e custo computacional configurável. Para um projeto de portfólio que demonstra hashing seguro de senhas, BCrypt é a escolha mais reconhecível por recrutadores e engenheiros sênior. Argon2 seria tecnicamente superior, mas a complexidade de configuração não se justificaria no escopo deste projeto.

### Por que MySQL e não SQL Server ou PostgreSQL?

MySQL tem custo zero, imagem Docker oficial estável e é amplamente usado em empresas de médio porte no Brasil. O provider Pomelo (`Pomelo.EntityFrameworkCore.MySql`) tem suporte ativo para .NET 8 e funciona sem configuração adicional. O trade-off é que algumas funcionalidades avançadas do EF Core podem ter comportamento ligeiramente diferente entre providers — algo que foi testado durante o desenvolvimento.

### O aprendizado sobre migrations em múltiplos ambientes

Na primeira versão, `db.Database.Migrate()` rodava em todos os ambientes incluindo produção. Aprendi que isso é um antipadrão: se a migration tem um bug, ela vai rodar diretamente em produção sem janela de rollback. A correção foi documentar que em produção as migrations devem ser executadas como step separado no pipeline: `dotnet ef database update` antes do deploy da aplicação.

### O que faria diferente

Extrairia os endpoints de vehicles e admins para métodos de extensão em arquivos separados (`VehicleEndpoints.cs`, `AdminEndpoints.cs`), mantendo o `Program.cs` focado em configuração. Também adicionaria refresh token — o token atual expira em 60 minutos e exige novo login, o que é aceitável para desenvolvimento mas inadequado para produção.

---

## 6. Resultados

O projeto entregou uma API funcional com os seguintes resultados concretos:

**9 endpoints com controle de acesso granular:** GET `/` (anônimo), POST `/login` (anônimo), POST/GET `/vehicles` (EditorOrAdmin), GET/PUT/DELETE `/vehicles/{id}` (Admin para PUT/DELETE), POST/GET `/admins` (Admin).

**Três ambientes isolados funcionais:** Development com Swagger habilitado e banco `vehiclesdb_dev`, Test com banco `vehiclesdb_test` e shutdown automático dos containers, Production em modo detach com banco `vehiclesdb` de produção.

**Autenticação JWT completa:** geração de token com claims de role, validação no middleware, autorização por política nos endpoints — tudo sem dependências externas além do `Microsoft.AspNetCore.Authentication.JwtBearer`.

**Seed idempotente:** ambiente de Development funcional imediatamente após `docker-compose up`, com dois usuários pré-criados para testes manuais e automatizados.

**Pipeline CI:** build, restore e testes executam automaticamente a cada push em `main` via GitHub Actions no Ubuntu, sem dependência de ambiente local.

---

## 7. Próximos Passos

- Extrair os endpoints para arquivos de extensão separados (`VehicleEndpoints.cs`, `AdminEndpoints.cs`) para melhor organização à medida que a API cresce.
- Implementar refresh token com rotação automática, substituindo o modelo atual de token único com expiração de 60 minutos.
- Adicionar paginação no `GET /vehicles` com parâmetros `page` e `pageSize`, evitando retorno irrestrito do catálogo.
- Configurar migrations como step separado no pipeline de CI antes do deploy em produção.
- Expandir a cobertura de testes com cenários de autenticação: token expirado, role insuficiente, endpoint protegido sem token.
- Adicionar CORS explícito com whitelist de origens para habilitar consumo seguro por frontends em domínios diferentes.

---

## Quickstart

### Pré-requisitos

- Docker Desktop
- Git

### Subir o ambiente de desenvolvimento

```bash
git clone https://github.com/Santosdevbjj/Minimals-APIs.git
cd Minimals-APIs/minimal-api-vehicles

# Development (Swagger habilitado, hot reload, banco dev)
docker-compose --env-file .env.development up --build

# Swagger disponível em: http://localhost:5000/swagger
```

### Credenciais seed (apenas Development/Test)

| Usuário | Senha | Role |
|---|---|---|
| admin@local.test | Admin@123 | Admin |
| editor@local.test | Editor@123 | Editor |

### Obter token JWT

```bash
curl -X POST http://localhost:5000/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@local.test","password":"Admin@123"}'
```

### Usar o token nos endpoints protegidos

```bash
# Criar veículo (Admin ou Editor)
curl -X POST http://localhost:5000/vehicles \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <TOKEN>" \
  -d '{"make":"Toyota","model":"Corolla","vin":"JTDBR32E720012345","year":2020}'

# Listar veículos
curl -H "Authorization: Bearer <TOKEN>" http://localhost:5000/vehicles

# Deletar veículo (apenas Admin)
curl -X DELETE -H "Authorization: Bearer <TOKEN>" http://localhost:5000/vehicles/1
```

### Outros ambientes

```bash
# Test (executa containers, roda testes, encerra)
docker-compose --env-file .env.test up --build --abort-on-container-exit

# Production (modo background)
docker-compose --env-file .env.production up -d --build
docker-compose logs -f  # acompanhar logs
```

---

## Endpoints

| Método | Rota | Autenticação | Roles |
|---|---|---|---|
| `GET` | `/` | Não | — |
| `POST` | `/login` | Não | — |
| `POST` | `/vehicles` | Sim | Admin, Editor |
| `GET` | `/vehicles` | Sim | Admin, Editor |
| `GET` | `/vehicles/{id}` | Sim | Admin, Editor |
| `PUT` | `/vehicles/{id}` | Sim | Admin |
| `DELETE` | `/vehicles/{id}` | Sim | Admin |
| `POST` | `/admins` | Sim | Admin |
| `GET` | `/admins` | Sim | Admin |

---

## Migrations

```bash
# Criar migration
dotnet ef migrations add InitialCreate \
  -p src/MinimalApi.Vehicles \
  -s src/MinimalApi.Vehicles

# Aplicar (o Program.cs já aplica automaticamente em dev)
dotnet ef database update \
  -p src/MinimalApi.Vehicles \
  -s src/MinimalApi.Vehicles
```

---

## Estrutura do Projeto

```
minimal-api-vehicles/
├── .env.development          # Credenciais dev (não versionado)
├── .env.test                 # Credenciais test (não versionado)
├── .env.production           # Credenciais prod (não versionado)
├── docker-compose.yml        # Estrutura base (sem credenciais)
├── docker-compose.override.yml  # Injeção de env por ambiente
├── .github/workflows/ci-cd.yml  # Pipeline CI
│
└── src/
    ├── MinimalApi.Vehicles/
    │   ├── Program.cs             # Toda a API: DI, JWT, endpoints, Swagger
    │   ├── Models/
    │   │   ├── Vehicle.cs         # VIN + Make + Model + Year
    │   │   └── AdminUser.cs       # Email + PasswordHash + Role
    │   ├── DTOs/
    │   │   ├── VehicleDto.cs      # Entrada para criação/edição
    │   │   ├── LoginRequest.cs    # Email + Password
    │   │   └── AdminCreateDto.cs  # Criação de administrador
    │   ├── Data/
    │   │   ├── AppDbContext.cs    # EF Core context
    │   │   └── DbSeed.cs         # Seed idempotente de usuários
    │   └── Services/
    │       ├── IAuthService.cs    # Contrato de geração de token
    │       └── AuthService.cs    # JWT com claims de role
    │
    └── MinimalApi.Vehicles.Tests/
        ├── IntegrationTests/
        │   └── VehiclesApiTests.cs  # WebApplicationFactory smoke test
        └── UnitTests/
            └── AdminUserTests.cs    # BCrypt hash/verify
```

---

## Tecnologias Utilizadas

| Camada | Tecnologia | Justificativa |
|---|---|---|
| **Runtime** | .NET 8 Minimal APIs | Baixo boilerplate, roteamento explícito |
| **ORM** | EF Core 8 + Pomelo MySQL | Code-first, migrations automáticas |
| **Banco** | MySQL 8.0 | Open source, Docker oficial, sem licença |
| **Autenticação** | JWT Bearer + BCrypt | Stateless, hash seguro de senhas |
| **Documentação** | Swagger (Swashbuckle) | UI interativa com suporte a Bearer token |
| **Containers** | Docker + Docker Compose | Paridade entre ambientes dev/test/prod |
| **Testes** | xUnit + WebApplicationFactory + FluentAssertions | Smoke test + unit test de segurança |
| **CI/CD** | GitHub Actions | Build + test automático no Ubuntu |

---

## Troubleshooting

| Sintoma | Causa provável | Solução |
|---|---|---|
| Container MySQL não sobe | Porta 3307 ocupada | Alterar porta no `.env` |
| 401 em endpoint protegido | Token expirado ou JWT_SECRET diferente | Novo login ou verificar `.env` |
| Swagger não aparece | Ambiente não é Development | Checar `ASPNETCORE_ENVIRONMENT=Development` |
| Migration falha | Banco ainda não está pronto | Aguardar healthcheck do MySQL (30s) |

---

## Autor

**Sergio Santos**  
Data Engineer & Cloud Architect — 15+ anos em sistemas críticos bancários (Bradesco)  
Campus Expert DIO · Bootcamp GFT Start #7 .NET

[![Portfólio](https://img.shields.io/badge/Portfólio-Sérgio_Santos-111827?style=for-the-badge&logo=githubpages&logoColor=00eaff)](https://portfoliosantossergio.vercel.app)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Sérgio_Santos-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/santossergioluiz)
[![GitHub](https://img.shields.io/badge/GitHub-Santosdevbjj-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/Santosdevbjj)
