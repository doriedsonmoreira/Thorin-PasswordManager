# Thorin Password Manager

> Gerenciador de senhas projetado com foco em segurança, isolamento de responsabilidades e suporte à arquitetura **Zero-Knowledge**.

![Node.js](https://img.shields.io/badge/Node.js-Backend-339933?logo=node.js&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-Strict-3178C6?logo=typescript&logoColor=white)
![Express](https://img.shields.io/badge/Express-5.x-000000?logo=express&logoColor=white)
![Prisma](https://img.shields.io/badge/Prisma-7.x-2D3748?logo=prisma&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-4169E1?logo=postgresql&logoColor=white)
![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)

---

## Visão Geral

O **Thorin Password Manager** é um gerenciador de credenciais desenvolvido para armazenar e organizar logins de aplicações de forma segura. O projeto está sendo construído de forma incremental, começando por uma API REST robusta em **Node.js + TypeScript**, com persistência em **PostgreSQL** via **Prisma ORM v7** e fundação criptográfica preparada para operação *zero-knowledge*.

---

## Funcionalidades

### Implementadas
- **Cadastro de usuários (`POST /register`)**:
  - Validação de campos obrigatórios (`name`, `email`, `password`) e prevenção de e-mails duplicados.
  - Hash seguro da senha de autenticação com `bcrypt` (10 *salt rounds*).
  - Geração automática de um `salt` criptográfico único por usuário (`16 bytes` aleatórios em hexadecimal) para derivação de chave *zero-knowledge*.
- **Autenticação JWT (`POST /login`)**:
  - Validação segura das credenciais via `bcrypt.compare`.
  - Emissão de token JWT (`jsonwebtoken`) com expiração de `30m`.
- **Perfil do usuário autenticado (`GET /profile`)**:
  - Protegido por middleware de autenticação Bearer Token (`authMiddleware`).
  - Retorno dos dados do usuário sem exposição do campo `password`.
- **Tratamento centralizado de erros**:
  - Classes de erro HTTP semânticas (`ApiError`, `BadRequestError`, `UnauthorizedError`, `NotFoundError`) interceptadas por middleware dedicado.
- **Proteção HTTP básica**:
  - Aplicação de cabeçalhos de segurança padrão com `helmet`.
- **Modelagem relacional e migrations versionadas**:
  - Tabelas `users`, `credentials` e `apps` configuradas com relacionamentos, UUIDs e regras de integridade referencial.

### Roadmap (Em Desenvolvimento)
- [ ] CRUD completo de credenciais (`Credentials`) com criptografia ponta a ponta (*zero-knowledge*).
- [ ] Gestão de aplicações/serviços (`App`).
- [ ] Refresh tokens e estratégia de revogação de sessões.
- [ ] Validação de schemas de entrada (ex.: Zod) e rate limiting em rotas de autenticação.
- [ ] Suíte de testes automatizados (unitários e de integração).
- [ ] Aplicação Frontend para gerenciamento visual do cofre.

---

## Decisões Arquiteturais

### 1. Arquitetura em Camadas
O backend segue uma separação clara de responsabilidades para facilitar manutenção, testabilidade e evolução do domínio:

```mermaid
flowchart LR
    Client["Cliente / HTTP"] --> Routes["Routes"]
    Routes --> Middlewares["Middlewares (Helmet, Auth, Error)"]
    Middlewares --> Controllers["Controllers"]
    Controllers --> Repositories["Repositories"]
    Repositories --> Prisma["Prisma Client (@prisma/adapter-pg)"]
    Prisma --> DB[("PostgreSQL 15")]
```

- **Routes (`src/routes`)**: registra endpoints públicos e aplica middlewares de proteção às rotas privadas.
- **Middlewares (`src/middlewares`)**: concentra preocupações transversais (autenticação JWT e tratamento global de erros).
- **Controllers (`src/controllers`)**: orquestra validação, regras de aplicação (hash, geração de salt, emissão de token) e respostas HTTP.
- **Repositories (`src/repositories`)**: encapsula o acesso ao banco via Prisma, isolando a camada de persistência da regra de negócio.

### 2. Preparação para Criptografia Zero-Knowledge
Cada usuário recebe um `salt` aleatório único gerado via `crypto.randomBytes(16).toString("hex")` no momento do cadastro e armazenado na tabela `users`.
- **Por quê?** Em uma arquitetura *zero-knowledge*, o `salt` permite que o cliente derive localmente a chave de criptografia do cofre a partir da senha mestra do usuário, sem que o servidor precise conhecer a chave usada para cifrar ou decifrar as credenciais armazenadas.
- O campo `password` usado para autenticação é tratado separadamente com `bcrypt` e sempre omitido das respostas da API e do objeto `req.user` (`Omit<User, "password">`).

### 3. Sessões Stateless com JWT de Curta Duração
- A autenticação usa tokens **JWT** enviados via cabeçalho `Authorization: Bearer <token>`.
- A expiração foi definida em **30 minutos** (`expiresIn: "30m"`) para reduzir a janela de risco caso um token seja interceptado.
- O `authMiddleware` valida a assinatura do token, confirma a existência do usuário no banco e injeta o usuário tipado em `req.user` (com extensão global de tipos em `src/@types/express.d.ts`).

### 4. Modelagem Relacional e Regras de Exclusão
As entidades utilizam **UUIDs** (`@db.Uuid`) como chaves primárias para evitar enumeração sequencial de registros na API.

```mermaid
erDiagram
    USERS ||--o{ CREDENTIALS : "possui (Cascade)"
    APPS ||--o{ CREDENTIALS : "classifica (Restrict)"

    USERS {
        uuid id PK
        varchar name
        varchar email UK
        varchar password
        varchar salt
        timestamp created_at
        timestamp updated_at
    }

    CREDENTIALS {
        uuid id PK
        varchar title
        varchar usernameOrEmail
        varchar password
        uuid userId FK
        uuid appId FK
        timestamp created_at
        timestamp updated_at
    }

    APPS {
        uuid id PK
        varchar name
        text image_url
        timestamp created_at
        timestamp updated_at
    }
```

- **`User` → `Credentials` (`onDelete: Cascade`)**: ao excluir uma conta de usuário, todas as suas credenciais são automaticamente removidas, garantindo privacidade e evitando dados órfãos.
- **`App` → `Credentials` (`onDelete: Restrict`)**: bloqueia a exclusão de um aplicativo enquanto existirem credenciais associadas a ele, prevenindo perda acidental de referências.

### 5. Prisma ORM v7 com Driver Adapter (`@prisma/adapter-pg`)
- Adotou-se a configuração moderna do Prisma v7 (`prisma.config.ts`) integrada ao adaptador nativo PostgreSQL (`@prisma/adapter-pg`).
- O Prisma Client é gerado em `Backend/src/generated/prisma` (ignorado pelo Git) em formato CommonJS (`moduleFormat = "cjs"`), mantendo o repositório limpo e desacoplado de artefatos gerados.

### 6. TypeScript Estrito e Tratamento Centralizado de Erros
- O `tsconfig.json` habilita checagens rigorosas (`strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `isolatedModules`) para capturar falhas em tempo de compilação.
- Com o suporte do **Express 5** a rejeições em handlers assíncronos, erros lançados nos controllers (como `BadRequestError` e `UnauthorizedError`) fluem automaticamente para o `errorMiddleware`, padronizando respostas de erro JSON.

### 7. Ambiente Reprodutível com Docker
- O banco **PostgreSQL 15 Alpine** é provisionado via `Backend/docker-compose.yml` com volume persistente (`postgres_data`) e credenciais parametrizadas por variáveis de ambiente.

---

## Stack Tecnológica

| Categoria | Tecnologia |
| --- | --- |
| Runtime | Node.js |
| Linguagem | TypeScript (`^7.0.2`) |
| Framework HTTP | Express (`^5.2.1`) |
| Segurança HTTP | Helmet (`^8.3.0`) |
| Autenticação & Hash | `jsonwebtoken` (`^9.0.3`), `bcrypt` (`^6.0.0`), `crypto` (nativo) |
| ORM & Banco de Dados | Prisma (`^7.9.1`), `@prisma/adapter-pg`, PostgreSQL 15 |
| Infraestrutura Local | Docker & Docker Compose |
| Desenvolvimento | `concurrently`, `nodemon`, `cross-env`, `rimraf`, `dotenv` |

---

## Estrutura do Projeto

```text
Thorin-PasswordManager/
├── .gitignore
├── LICENSE
├── README.md
└── Backend/
    ├── .gitignore
    ├── docker-compose.yml
    ├── package.json
    ├── prisma.config.ts
    ├── tsconfig.json
    ├── prisma/
    │   ├── schema.prisma
    │   └── migrations/
    │       ├── 20260811011852_start_define_scope/
    │       ├── 20260812021050_add_delete_rules/
    │       └── 20260821232349_add_salt_to_user/
    └── src/
        ├── @types/
        │   └── express.d.ts
        ├── controllers/
        │   └── AuthController.ts
        ├── helpers/
        │   └── api-errors.ts
        ├── lib/
        │   └── prisma.ts
        ├── middlewares/
        │   ├── auth.Middlewares.ts
        │   └── error.ts
        ├── repositories/
        │   └── UserRepository.ts
        ├── routes/
        │   └── routes.ts
        └── index.ts
```

---

## Endpoints da API

| Método | Rota | Autenticação | Descrição |
| --- | --- | --- | --- |
| `GET` | `/` | Pública | Healthcheck do servidor |
| `POST` | `/register` | Pública | Registra um novo usuário e gera seu `salt` |
| `POST` | `/login` | Pública | Autentica o usuário e retorna dados + JWT |
| `GET` | `/profile` | Bearer JWT | Retorna os dados do usuário autenticado |

### Exemplos de Requisição

#### Registrar usuário (`POST /register`)
```json
{
  "name": "Doriedson Moreira",
  "email": "usuario@exemplo.com",
  "password": "sua-senha-forte"
}
```

#### Login (`POST /login`)
```json
{
  "email": "usuario@exemplo.com",
  "password": "sua-senha-forte"
}
```

**Resposta (`200 OK`):**
```json
{
  "user": {
    "id": "uuid-do-usuario",
    "name": "Doriedson Moreira",
    "email": "usuario@exemplo.com",
    "salt": "salt-hexadecimal-gerado",
    "createdAt": "2026-08-21T21:00:00.000Z",
    "updatedAt": "2026-08-21T21:00:00.000Z"
  },
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

#### Consultar perfil (`GET /profile`)
Enviar cabeçalho:
```http
Authorization: Bearer <seu-token-jwt>
```

---

## Como Executar Localmente

### Pré-requisitos
- **Node.js** (versão LTS recomendada) e **npm**
- **Docker** e **Docker Compose**

### Passo a passo

1. **Clone o repositório e acesse a pasta do backend:**
   ```bash
   git clone https://github.com/doriedsonmoreira/Thorin-PasswordManager.git
   cd Thorin-PasswordManager/Backend
   ```

2. **Configure as variáveis de ambiente:**
   Crie um arquivo `.env` dentro de `Backend/` com base nas variáveis abaixo:
   ```env
   PORT=3000

   POSTGRES_USER=postgres
   POSTGRES_PASSWORD=postgres
   POSTGRES_DB=password_manager

   DATABASE_URL="postgresql://postgres:postgres@localhost:5432/password_manager?schema=public"
   JWT_PASS="substitua-por-uma-chave-secreta-forte"
   ```

3. **Suba o banco de dados PostgreSQL com Docker:**
   ```bash
   docker compose up -d
   ```

4. **Instale as dependências:**
   ```bash
   npm install
   ```

5. **Gere o Prisma Client e execute as migrations:**
   ```bash
   npx prisma generate
   npx prisma migrate dev
   ```

6. **Inicie o servidor:**
   - Em modo de desenvolvimento (watch + reload automático):
     ```bash
     npm run dev
     ```
   - Em modo de produção:
     ```bash
     npm run build
     npm start
     ```

A API estará disponível em `http://localhost:3000`.

---

## Scripts Disponíveis (`Backend/package.json`)

| Comando | Descrição |
| --- | --- |
| `npm run dev` | Compila TypeScript em modo watch e reinicia o servidor com `nodemon` |
| `npm run build` | Limpa a pasta `dist/` e compila o projeto TypeScript |
| `npm start` | Executa o build compilado (`dist/index.js`) em modo produção |

---

## Licença

Este projeto está licenciado sob a **MIT License** — consulte o arquivo [LICENSE](./LICENSE) para mais detalhes.
# Thorin-PasswordManager
Security for your credentials
