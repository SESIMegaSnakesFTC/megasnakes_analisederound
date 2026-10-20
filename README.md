# 🐍 Mega Snakes Round Analyzer

Aplicativo web para registrar, analisar e acompanhar rounds de partida, com **login por usuário** e **sincronização na nuvem** via Supabase. Cada pessoa tem seus próprios dados, isolados por Row Level Security (RLS).

---

## 📑 Índice

- [🧠 Visão geral](#-visão-geral)
- [🏗️ Arquitetura](#️-arquitetura)
- [📂 Estrutura de arquivos](#-estrutura-de-arquivos)
- [🛠️ Instalação e configuração](#️-instalação-e-configuração)
- [💻 Como executar](#-como-executar)
- [🗄️ Modelo de dados](#️-modelo-de-dados)
- [🔐 Segurança](#-segurança)
- [🧪 Testes](#-testes)
- [🐛 Solução de problemas](#-solução-de-problemas)

---

## 🧠 Visão geral

O **Mega Snakes Round Analyzer** permite que cada usuário:

- 👤 Crie uma conta e faça login
- 📝 Registre rounds com nome, pontuação e observações
- 🔍 Guarde análises detalhadas de cada round
- ⏱️ Mantenha um cronômetro e um histórico de ações
- ☁️ Tenha tudo salvo automaticamente na nuvem, acessível de qualquer dispositivo

Todo o estado do aplicativo é armazenado como **um único JSON por usuário**, o que simplifica a persistência e a evolução do app sem exigir migrações a cada nova funcionalidade.

---

## 🏗️ Arquitetura

```
┌──────────────────┐        HTTPS         ┌─────────────────────────┐
│  Frontend (web)  │ ───────────────────► │        Supabase         │
│  HTML + JS       │   chave pública      │  ┌───────────────────┐  │
│  supabase-js     │ ◄─────────────────── │  │ Auth (auth.users) │  │
└──────────────────┘     dados do user    │  └─────────┬─────────┘  │
                                          │     triggers│           │
                                          │  ┌─────────▼─────────┐  │
                                          │  │ profiles          │  │
                                          │  │ user_state (JSONB)│  │
                                          │  └───────────────────┘  │
                                          │      🔒 RLS ativo       │
                                          └─────────────────────────┘
```

| Camada | Responsabilidade |
|---|---|
| 🖥️ Frontend | Interface, cronômetro, análise de rounds |
| 🔑 Supabase Auth | Cadastro, login e sessão |
| 🗄️ Postgres | Armazena perfil e estado de cada usuário |
| 🔒 RLS | Garante que cada usuário acesse apenas seus dados |
| ⚡ Triggers | Criam perfil e estado inicial automaticamente no cadastro |

---

## 📂 Estrutura de arquivos

> 💡 Ajuste os nomes conforme o seu repositório.

```
.
├── index.html              # Interface principal
├── supabase-config.js      # URL e chave pública do Supabase
├── supabase-schema.sql     # Schema, políticas RLS e triggers
└── README.md
```

---

## 🛠️ Instalação e configuração

### 1️⃣ Crie um projeto no Supabase

Acesse [supabase.com](https://supabase.com), crie um projeto e anote a **Project URL** e a **chave pública** (*publishable / anon key*) em **Project Settings → API**.

### 2️⃣ Execute o SQL do banco

1. Abra o **SQL Editor** no painel do Supabase
2. Cole o conteúdo de `supabase-schema.sql`
3. Clique em **Run**

Isso cria as tabelas `profiles` e `user_state`, as políticas RLS, as funções e os triggers.

> ⚠️ **Atenção:** o script usa `create policy` e `create trigger` sem `if not exists`. Executá-lo duas vezes gera erro de objeto já existente (veja [🐛 Solução de problemas](#-solução-de-problemas)).

### 3️⃣ Configure as credenciais

Edite o `supabase-config.js`:

```js
window.SUPABASE_CONFIG = {
  URL: 'https://SEU_PROJECT_ID.supabase.co',
  PUBLISHABLE_KEY: 'SUA_CHAVE_PUBLICA'
};
```

> 🚫 **Nunca** coloque a `service_role key` ou qualquer *secret key* neste arquivo. Ele é carregado no navegador e fica visível a qualquer visitante.

O próprio arquivo valida a configuração e registra no console:

| Mensagem | Significado |
|---|---|
| ✅ `Supabase configurado com sucesso.` | Credenciais preenchidas |
| ⚠️ `Supabase não está configurado.` | Ainda há valores de exemplo |

### 4️⃣ Configure a autenticação

Em **Authentication → Providers**, habilite o login por e-mail e senha. Em **URL Configuration**, adicione o endereço onde o app será hospedado.

---

## 💻 Como executar

Como o frontend é estático, basta servir os arquivos com qualquer servidor HTTP:

```bash
# Python
python3 -m http.server 8000

# ou Node.js
npx serve .
```

Depois acesse `http://localhost:8000` no navegador. ✅

---

## 🗄️ Modelo de dados

### 👤 `profiles`

| Coluna | Tipo | Descrição |
|---|---|---|
| `id` | `uuid` (PK) | Referencia `auth.users(id)` |
| `full_name` | `text` | Nome do metadata ou parte do e-mail |
| `email` | `text` | E-mail do usuário |
| `created_at` / `updated_at` | `timestamptz` | Controle de datas |

### 💾 `user_state`

| Coluna | Tipo | Descrição |
|---|---|---|
| `id` | `uuid` (PK) | Gerado automaticamente |
| `user_id` | `uuid` (único) | Dono dos dados |
| `data` | `jsonb` | Estado completo do app |
| `created_at` / `updated_at` | `timestamptz` | Controle de datas |

### 📦 Estrutura do JSON em `user_state.data`

```json
{
  "rounds": [
    {
      "id": "unique-id",
      "name": "Round 1",
      "score": 105,
      "notes": "Good performance",
      "timestamp": "2026-10-05T12:00:00.000Z"
    }
  ],
  "analyses": ["Análise detalhada do round 1"],
  "timer": { "elapsedSeconds": 0, "isRunning": false },
  "settings": { "theme": "light", "lastUpdatedAt": "2026-10-05T12:00:00.000Z" },
  "history": ["Ação 1: Round criado", "Ação 2: Pontuação salva"],
  "lastSyncedAt": "2026-10-05T12:00:00.000Z"
}
```

### ⚡ Automações (triggers)

Ao criar um usuário em `auth.users`, dois triggers disparam:

| Trigger | Função | Efeito |
|---|---|---|
| `on_auth_user_created` | `handle_new_user()` | Cria o registro em `profiles` |
| `on_auth_user_initialize_state` | `initialize_user_state()` | Cria um `user_state` vazio |

---

## 🔐 Segurança

- 🔒 **RLS ativado** em `profiles` e `user_state`
- 🪪 Todas as políticas comparam `auth.uid()` com o dono da linha
- 🧹 `on delete cascade`: remover um usuário apaga todos os seus dados
- 🔑 O frontend usa apenas a **chave pública**; é a RLS que protege os dados, por isso **não desative** as políticas
- 🙈 Não armazene segredos nem dados sensíveis no JSON de `user_state`

| Tabela | Ler | Criar | Atualizar | Deletar |
|---|:---:|:---:|:---:|:---:|
| `profiles` | ✅ | ✅ | ✅ | ❌ |
| `user_state` | ✅ | ✅ | ✅ | ✅ |

*Todas as permissões valem apenas para os dados do próprio usuário.*

---

## 🧪 Testes

Para validar o isolamento de dados:

1. Crie **dois usuários** diferentes
2. Faça login com cada um e salve alguns rounds
3. Execute a consulta abaixo a partir do app (com a sessão do usuário):

```js
const { data } = await supabase.from('user_state').select('*');
console.log(data); // deve conter apenas 1 registro: o do usuário logado

const { data: perfil } = await supabase.from('profiles').select('*');
console.log(perfil); // deve conter apenas o perfil do usuário logado
```

> 💡 **Dica:** rodar `select * from user_state` direto no SQL Editor do painel usa um papel administrativo e **ignora a RLS**. Para testar de verdade, consulte pelo app ou pelo cliente `supabase-js` autenticado.

✅ **Resultado esperado:** cada usuário enxerga somente o próprio perfil e o próprio estado.

---

## 🐛 Solução de problemas

| Sintoma | Causa provável | Solução |
|---|---|---|
| ⚠️ Console exibe "Supabase não está configurado" | `supabase-config.js` com valores de exemplo | Preencha URL e chave pública |
| ❌ `policy ... already exists` ao rodar o SQL | Script executado mais de uma vez | Remova a política com `drop policy` ou limpe o schema e rode de novo |
| ❌ `trigger ... already exists` | Mesmo caso acima | Use `drop trigger if exists ... on auth.users;` antes de recriar |
| 📭 Consulta retorna lista vazia | Usuário não autenticado ou registro não criado | Confirme o login e verifique se os triggers existem |
| 🚫 Erro 401/403 ao salvar | RLS bloqueando a operação | Confirme que `user_id` é igual a `auth.uid()` |
| 👤 Usuário antigo sem perfil/estado | Criado antes dos triggers | Insira manualmente os registros em `profiles` e `user_state` |

---

## 📄 Licença

Defina aqui a licença do projeto (ex.: MIT).
