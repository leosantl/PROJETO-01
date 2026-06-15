# StandControl — Backend API

API REST para o **StandControl**, plataforma SaaS de gestão de clubes e estandes de tiro (CAC).

## Stack

| Tecnologia | Função |
|---|---|
| **Node.js + TypeScript** | Runtime e tipagem |
| **Express 4** | Framework HTTP |
| **PostgreSQL** | Banco de dados relacional |
| **node-postgres (pg)** | Driver do banco |
| **JWT (jsonwebtoken)** | Autenticação stateless |
| **bcryptjs** | Hash de senhas |
| **Zod** | Validação de schemas |
| **Helmet + CORS + rate-limit** | Segurança |

---

## Estrutura de arquivos

```
standcontrol-backend/
├── src/
│   ├── index.ts             # Entry point — Express app
│   ├── db/
│   │   └── pool.ts          # Pool de conexões + helpers query/queryOne
│   ├── middleware/
│   │   └── auth.ts          # JWT verify, sameCompany, requireRole
│   ├── routes/
│   │   ├── auth.ts          # POST /auth/login, /auth/admin/login
│   │   ├── companies.ts     # CRUD empresas + KPIs
│   │   ├── clients.ts       # CRUD atiradores/clientes
│   │   ├── weapons.ts       # CRUD acervo de armas
│   │   ├── ammo.ts          # Estoque e movimentação de munição (c/ transação)
│   │   ├── schedules.ts     # CRUD agenda + resumo semanal
│   │   ├── finance.ts       # CRUD financeiro + cashflow
│   │   ├── documents.ts     # CRUD documentos (CR, CRAF, etc.) + status dinâmico
│   │   ├── users.ts         # CRUD usuários da empresa
│   │   └── admin.ts         # KPIs globais, faturas SaaS, planos (super-admin)
│   └── types/
│       └── index.ts         # Interfaces TypeScript de todas as entidades
├── scripts/
│   ├── migrate.ts           # Cria schema completo no PostgreSQL
│   └── seed.ts              # Popula com dados iniciais (equivalente aos mocks)
├── .env.example
├── package.json
└── tsconfig.json
```

---

## Configuração

### 1. Pré-requisitos

- Node.js 20+
- PostgreSQL 14+

### 2. Instalar dependências

```bash
npm install
```

### 3. Configurar variáveis de ambiente

```bash
cp .env.example .env
# Edite .env com suas credenciais do banco
```

```env
DATABASE_URL=postgresql://user:senha@localhost:5432/standcontrol
PORT=3001
NODE_ENV=development
JWT_SECRET=chave_muito_segura_min_32_chars
JWT_EXPIRES_IN=7d
CORS_ORIGIN=http://localhost:5173
```

### 4. Criar o banco

```bash
createdb standcontrol  # ou crie pelo psql/pgAdmin
```

### 5. Rodar migrações + seed

```bash
npm run db:reset       # migrate + seed juntos
# ou separado:
npm run db:migrate
npm run db:seed
```

### 6. Iniciar servidor

```bash
npm run dev            # modo desenvolvimento (tsx watch)
npm start              # produção (após npm run build)
```

---

## Endpoints

### Autenticação

| Método | Rota | Descrição |
|---|---|---|
| POST | `/auth/login` | Login de usuário da empresa |
| POST | `/auth/admin/login` | Login de super-admin |

Todas as rotas protegidas exigem: `Authorization: Bearer <token>`

---

### Empresas `/companies`

| Método | Rota | Auth | Descrição |
|---|---|---|---|
| GET | `/companies` | super-admin | Lista todas |
| GET | `/companies/:id` | mesma empresa | Detalhe |
| POST | `/companies` | super-admin | Criar |
| PATCH | `/companies/:id` | super-admin | Atualizar |
| GET | `/companies/:id/kpis` | mesma empresa | KPIs do dashboard |

---

### Clientes `/companies/:companyId/clients`

| Método | Rota | Descrição |
|---|---|---|
| GET | `/` | Lista com filtros (status, search, page, limit) |
| GET | `/:id` | Detalhe com armas e documentos |
| POST | `/` | Cadastrar atirador |
| PATCH | `/:id` | Atualizar |
| DELETE | `/:id` | Excluir |

---

### Armas `/companies/:companyId/weapons`

| Método | Rota | Descrição |
|---|---|---|
| GET | `/` | Lista com filtros (status, caliber, search) |
| GET | `/:id` | Detalhe com nome do proprietário |
| POST | `/` | Registrar arma |
| PATCH | `/:id` | Atualizar |
| DELETE | `/:id` | Excluir |

---

### Munição `/companies/:companyId/ammo`

| Método | Rota | Descrição |
|---|---|---|
| GET | `/stock` | Lista estoque com flag `below_minimum` |
| POST | `/stock` | Criar/atualizar item do estoque |
| GET | `/movements` | Histórico de movimentações |
| POST | `/movements` | Registrar entrada ou saída (transação atômica) |

A rota de movimentação valida estoque disponível antes de realizar saídas.

---

### Agenda `/companies/:companyId/schedules`

| Método | Rota | Descrição |
|---|---|---|
| GET | `/` | Lista com filtros (date, status, type) |
| GET | `/week-summary` | Contagem por dia dos últimos 7 dias |
| GET | `/:id` | Detalhe |
| POST | `/` | Criar agendamento |
| PATCH | `/:id` | Atualizar |
| DELETE | `/:id` | Cancelar |

---

### Financeiro `/companies/:companyId/finance`

| Método | Rota | Auth | Descrição |
|---|---|---|---|
| GET | `/` | qualquer | Lista com filtros (type, status, month, year) |
| GET | `/cashflow` | qualquer | Receitas x despesas por mês (últimos 6 meses) |
| GET | `/:id` | qualquer | Detalhe |
| POST | `/` | Admin/Gerente/Financeiro | Criar lançamento |
| PATCH | `/:id` | Admin/Gerente/Financeiro | Atualizar |
| DELETE | `/:id` | Admin/Financeiro | Excluir |

---

### Documentos `/companies/:companyId/documents`

| Método | Rota | Descrição |
|---|---|---|
| GET | `/` | Lista com filtros (status, type, client_id). Status calculado dinamicamente. |
| GET | `/:id` | Detalhe |
| POST | `/` | Cadastrar (status calculado automaticamente pela data) |
| PATCH | `/:id` | Atualizar (status recalculado se expires_at mudar) |
| DELETE | `/:id` | Excluir |

---

### Usuários `/companies/:companyId/users`

| Método | Rota | Auth | Descrição |
|---|---|---|---|
| GET | `/` | qualquer | Lista usuários da empresa |
| GET | `/me` | qualquer | Dados do usuário autenticado |
| POST | `/` | Admin/Gerente | Convidar usuário |
| PATCH | `/:id` | Admin | Atualizar / trocar senha |
| DELETE | `/:id` | Admin | Remover |

---

### Admin `/admin` (super-admin only)

| Método | Rota | Descrição |
|---|---|---|
| GET | `/kpis` | Empresas ativas, trial, MRR, ARR, novos clientes |
| GET | `/companies` | Todas as empresas com contagem de usuários |
| GET | `/invoices` | Faturas de todos os tenants |
| PATCH | `/invoices/:id` | Atualizar status de fatura |
| GET | `/mrr-series` | Evolução do MRR (últimos 6 meses) |
| GET | `/plans` | Lista de planos disponíveis |

---

## Modelo de Dados

```
plans ──────────── companies ─┬── app_users
                              ├── clients ────── weapons
                              │                └── documents
                              ├── ammo_stock ─── ammo_movements
                              ├── schedules
                              ├── finance_entries
                              ├── documents
                              └── invoices
```

**Isolamento multi-tenant:** todas as tabelas operacionais possuem `company_id`, e o middleware `sameCompany` garante que um usuário só acessa dados da própria empresa.

---

## Credenciais padrão (seed)

| Tipo | E-mail | Senha |
|---|---|---|
| Super-admin | admin@standcontrol.com.br | superadmin123 |
| Admin empresa Alpha | carlos@alphaprecision.cac | alpha123 |
| Outros usuários Alpha | (ver seed.ts) | alpha123 |

> ⚠️ Troque todas as senhas antes de usar em produção.

---

## Conectar ao frontend

No frontend (TanStack Router), substitua as importações dos mocks:

```ts
// Antes (mock)
import { clients } from "@/mocks/clients";

// Depois (API)
const res = await fetch(`/companies/${companyId}/clients`, {
  headers: { Authorization: `Bearer ${token}` }
});
const { data } = await res.json();
```

Configure um proxy no `vite.config.ts`:

```ts
server: {
  proxy: {
    "/api": {
      target: "http://localhost:3001",
      rewrite: (p) => p.replace(/^\/api/, ""),
    },
  },
},
```
