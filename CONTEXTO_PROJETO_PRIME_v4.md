# 🧠 CONTEXTO PERMANENTE — CRM PRIME
## Versão 4 — Atualizado em 20/06/2026

---

## 🤝 MODO DE TRABALHO

Claude atua como **parceiro técnico** da Prime, trabalhando junto com Thiago para construir e evoluir o sistema. Isso significa:
- Explicar o raciocínio por trás de cada decisão técnica
- Detalhar os passos para que Thiago aprenda com cada implementação
- Ser proativo em identificar riscos antes de agir
- Sinalizar quando a janela de contexto estiver longa demais e sugerir migrar para um novo chat
- Nunca executar mudanças que afetem segurança ou produção sem confirmar primeiro

---

## 🏢 A EMPRESA

**Nome:** Prime Correspondente Bancário
**Segmento:** Correspondente bancário da Caixa Econômica Federal
**Tamanho:** 5 funcionários
**Produtos:** Habitacional MCMV, Habitacional SBPE, Conta Corrente, Poupança, Cartão de Crédito, Consórcio, Consignado, Seguro de Vida, Rapidex/Amparo, Múltiplo

---

## 🌐 INFRAESTRUTURA

| Item | Valor |
|---|---|
| URL produção | https://prime-crm.sistemaintegradocrmprime.workers.dev/ |
| Hospedagem | Cloudflare Pages (plano gratuito — 500 deploys/mês) |
| Repositório | GitHub → Cloudflare Pages (CI/CD automático) |
| Arquivo principal | index.html na raiz |
| Supabase projeto | dvkniqxjudcxzpbmptbb |
| Supabase URL | https://dvkniqxjudcxzpbmptbb.supabase.co |
| Worker upload R2 | https://prime-upload-worker.sistemaintegradocrmprime.workers.dev |
| Bucket R2 | prime-documentos |
| URL pública R2 | https://pub-0b305b9d5e5d4583ae8671220e88f319.r2.dev |
| Custo mensal | R$ 0 |

**Regra de ouro:** sempre resolver no banco (SQL) antes de gerar novo deploy HTML.

---

## 🔐 AUTENTICAÇÃO — ESTADO ATUAL

**Sistema:** Supabase Auth (e-mail + senha) com JWT
**persistSession:** false (sem sessão salva no navegador)
**Token:** salvo em `window._authToken` após login — usado no Worker R2 e na Edge Function
**Tabela de perfis:** `operadores` (sem FK com auth.users)

**Usuários cadastrados:**
| E-mail | Login | Nome | Role |
|---|---|---|---|
| prime@prime.crm | prime | Prime | gestor |
| leilane@prime.crm | leilane | Leilane | gestor |
| josiel@prime.crm | josiel | Josiel | usuario |
| thiago@prime.crm | thiago | Thiago | gestor |
| aline@prime.crm | aline | Aline | usuario |

**Nota:** login aceita prefixo simples (`prime`) ou e-mail completo (`prime@prime.crm`). O código normaliza automaticamente.

**Edge Function:** `admin-usuarios` (versão 2, ACTIVE)
- Ação `criar`: cria no Supabase Auth + tabela operadores simultaneamente
- Ação `deletar`: remove do Auth + operadores
- Ação `alterarRole`: atualiza role na tabela operadores
- Requer JWT válido no header Authorization
- Verifica se quem chama é gestor antes de executar

---

## 🗄️ BANCO DE DADOS — ESTADO ATUAL

### Tabelas existentes
```
clientes              ← base principal (491 registros, todos legado=true)
historico             ← interações por cliente
historico_vistorias   ← interações por vistoria
documentos            ← tabela original (pouco uso)
documentos_cliente    ← documentos vinculados (R2, Google Drive ou link)
tarefas               ← follow-ups com vencimento
perfis                ← tabela do Supabase Auth (não usar para login)
operadores            ← usuários do CRM (login customizado)
etapas_config         ← prazos R1 e R2 por etapa
corretores            ← lista de corretores internos (13 registros)
construtores          ← lista de construtores/vendedores
vistorias             ← serviço independente de vistoria/laudos
perfis_backup_20260608 ← backup antigo (não tocar)
```

### Tabela documentos_cliente — campos e constraints
```sql
id           uuid PK default uuid_generate_v4()
cliente_id   uuid (FK implícita para clientes)
nome_arquivo text NOT NULL
tipo         text NOT NULL default 'gdrive'
             -- CHECK: tipo IN ('gdrive', 'link', 'storage', 'r2')
url          text NOT NULL
tamanho_kb   integer (nullable)
criado_por   text (nullable)
created_at   timestamptz default now()
```

**Constraint atualizada em 20/06/2026:**
```sql
CHECK (tipo = ANY (ARRAY['gdrive'::text, 'link'::text, 'storage'::text, 'r2'::text]))
```

### Enum status_enum (valores atuais)
Lead, Andamento, Condicionado, Reprovado, Aprovado, Conformidade, Assinatura, Cartório/Prefeitura, Concluído, Pós-venda

### Tabela clientes — campos completos
```sql
id, nome, cpf (UNIQUE), telefone (UNIQUE), email, produto (produto_enum),
status (status_enum), operador, corretor, indicado_por, corretor_externo,
construtor_vendedor, observacoes, origem_arquivo, dt_entrada, dt_atualizacao,
created_at, updated_at, produtos_contratados,
-- Campos Fase 1:
legado (bool), tem_card (bool), em_alerta_r1 (bool), em_alerta_r2 (bool),
proxima_interacao (timestamptz), ultima_interacao (timestamptz),
dt_entrada_etapa (timestamptz), ultima_alteracao_por (text),
renda_principal, renda_secundaria, renda_total (calculado por trigger),
valor_imovel, valor_financiamento, valor_subsidio, valor_prestacao,
numero_contrato, comissao_valor, comissao_status, comissao_obs, email
```

### Triggers ativos
- `trg_historico_atualiza_cliente`: atualiza proxima_interacao e ultima_interacao do cliente ao inserir no histórico
- `trg_clientes_entrada_etapa`: registra dt_entrada_etapa quando status muda
- `trg_clientes_renda_total`: calcula renda_total = renda_principal + renda_secundaria
- `trg_hist_vist_atualiza`: atualiza ultima_interacao da vistoria ao inserir no histórico_vistorias
- `trg_vistorias_updated`: atualiza updated_at na vistoria

### Funções RPC
- `fn_recalcular_alertas()`: recalcula em_alerta_r1 e em_alerta_r2 de todos os clientes
- `fn_buscar_cliente_card(p_cpf, p_telefone)`: busca cliente por CPF ou telefone
- `fn_alertas_login(p_operador, p_role)`: retorna clientes em alerta para o pop-up de login

### RLS
- Todas as tabelas operacionais: políticas `authenticated` (exigem JWT do Supabase Auth)
- Tabela `operadores`: RLS desabilitado (necessário para lookup após login)
- Tabela `documentos_cliente`: políticas SELECT, INSERT, DELETE para `authenticated`

### pg_cron
- Job `recalcular-alertas-prime`: roda diariamente à meia-noite UTC

### Migrações executadas
06_estrutura_fase1.sql, 07_migracao_status_fase1_v2.sql, 08_triggers_rls_fase1.sql,
09_corrigir_operadores.sql, 10_unique_cpf_telefone_vistorias_engenheiro

---

## ☁️ CLOUDFLARE R2 — ARMAZENAMENTO DE ARQUIVOS (Fase 2 — CONCLUÍDA)

### Como funciona
```
Operador arrasta arquivo no card
         ↓
  Cloudflare Worker (prime-upload-worker)
  · Valida JWT do usuário com Supabase Auth
  · Limite: 10 MB por arquivo
  · Aceita: PDF, JPG, PNG, WEBP, DOC, DOCX, XLS, XLSX
         ↓
  Cloudflare R2 (bucket: prime-documentos)
  · Caminho: clientes/{uuid_cliente}/{timestamp}_{nome_arquivo}
         ↓
  URL pública salva em documentos_cliente (tipo='r2')
         ↓
  Link clicável aparece no card do cliente
```

### Arquivos do Worker (local no computador de Thiago)
```
C:\Users\User\prime-upload-worker\
├── wrangler.toml
└── src\
    └── index.js
```

### Segredos registrados no Worker
- `SUPABASE_ANON_KEY`: chave anon do Supabase
- `R2_PUBLIC_URL`: https://pub-0b305b9d5e5d4583ae8671220e88f319.r2.dev

### Comando para atualizar o Worker (quando necessário)
```cmd
cd C:\Users\User\prime-upload-worker
wrangler deploy
```

### Compatibilidade retroativa
- Links do Google Drive existentes continuam funcionando (tipo='gdrive')
- O campo de link manual no modal de documentos continua disponível
- Novos uploads vão automaticamente para o R2

---

## 🔒 SEGURANÇA E LGPD — ANÁLISE E DECISÕES

**Princípios ativos:**
1. **RLS sempre ativo** em tabelas operacionais — exige JWT válido
2. **Supabase Auth** para autenticação — nunca reverter para senha_hash customizada
3. **Service role key** nunca no frontend — só na Edge Function no servidor
4. **CPF e telefone únicos** — constraints no banco + validação no frontend
5. **Comissão protegida** — campo invisível para operadores no frontend
6. **Token em memória** (`window._authToken`) — não persiste no browser (LGPD)
7. **Logs de alteração** — campo `ultima_alteracao_por` em toda edição de cliente

**Decisão documentada — Arquivos R2 (20/06/2026):**
O R2 usa URL pública com acesso por link direto. Isso foi analisado e é adequado para o contexto da Prime pelos seguintes motivos:
- O link contém UUID do cliente + timestamp (imprevisível — impossível de adivinhar)
- O link só aparece no CRM para usuários autenticados (RLS ativo)
- O upload exige JWT válido (Worker rejeita requisições sem login)
- Risco residual aceito: se um operador copiar e compartilhar o link fora do CRM, o arquivo fica acessível. Mitigação: orientar a equipe a não compartilhar links fora do sistema.
- Proteção total (Cloudflare Access) foi avaliada mas foge do modelo gratuito.

**Quando surgir nova funcionalidade com risco de segurança/LGPD:**
Claude deve sempre: (1) identificar o risco, (2) explicar em linguagem clara, (3) propor a mitigação, (4) registrar a decisão aqui.

---

## 💻 ESTRUTURA DO index.html ATUAL

**Versão:** v12 (20/06/2026) — upload R2 implementado
**Tamanho:** ~1972 linhas

**Bibliotecas CDN:**
- @supabase/supabase-js@2
- Chart.js@4.4.1
- @tabler/icons-webfont@3.19.0

**Páginas/abas:**
| ID | Nome | Acesso |
|---|---|---|
| page-login | Login com e-mail+senha | Público |
| page-kanban | Quadro operacional (kanban) | Todos |
| page-posvenda | Clientes em pós-venda | Todos |
| page-vistorias | Vistorias e laudos | Todos |
| page-dashboard | Dashboard com métricas | Todos |
| page-clientes | Tabela de clientes | Todos |
| page-tarefas | Tarefas e follow-ups | Todos |
| page-detail | Ficha do cliente | Todos |
| page-equipe | Gestão de usuários | Gestor only |
| page-senha | Alterar senha | Todos |

**Funcionalidades implementadas:**
- Login via Supabase Auth com normalização de e-mail
- Kanban com 9 colunas + drag and drop entre colunas
- Pop-up de alertas ao logar (R1 para operador, R1+R2 para gestor)
- Busca automática por CPF ou telefone ao criar card
- Validação de CPF e telefone únicos antes de salvar
- Campos habitacional condicionais (só para MCMV e SBPE)
- Campo comissão visível apenas para gestor
- **Upload de documentos direto no card via Cloudflare R2** ← NOVO Fase 2
- Campo de link manual (Google Drive/externo) — mantido por compatibilidade
- Botão "Criar card" para clientes legados
- Pós-venda separado do kanban principal
- Vistorias com histórico, alertas, editar, engenheiro
- Dashboard com filtro legados/ativos e alertas R1/R2
- Export CSV
- Gestão de equipe via Edge Function (criar/deletar no Auth)
- Tag "L" visual para clientes legados

---

## 🔑 PERFIS E PERMISSÕES

**Gestor:**
- Cria e deleta usuários (via Edge Function)
- Visualiza e edita campo comissão
- Altera operador responsável de qualquer card
- Recebe alertas R1 e R2 no pop-up
- Dashboard completo com tabelas R1 e R2

**Operador (usuario):**
- Acesso ao kanban, clientes, tarefas, vistorias
- Não vê comissão em nenhuma tela
- Recebe apenas alertas R1 no pop-up
- Não pode criar/deletar usuários

---

## 📋 REGRAS DE TEMPO

| Etapa | R1 (dias sem interação) | R2 (dias na etapa) |
|---|---|---|
| Lead | 1 | 5 |
| Andamento | 2 | 10 |
| Condicionado | 7 | 30 |
| Reprovado | 30 | 60 |
| Aprovado | 3 | 7 |
| Conformidade | 2 | 5 |
| Assinatura | 2 | 5 |
| Cartório/Prefeitura | 7 | 15 |
| Concluído | 7 | 30 |
| Pós-venda | 30 | 60 |

---

## 🚀 PRÓXIMOS PASSOS

**Etapa 5 — Migração do Trello (próxima ação):**
- Exportar JSON do Trello no momento da migração
- Mapear cards → clientes pelo CPF/telefone
- Script SQL: setar `tem_card = true` para os ativos
- Validação pela equipe antes de desativar o Trello
- **Pré-requisito:** equipe confortável com o kanban interno ✅

**Fase 3 (planejada):**
- WhatsApp Business API via Z-API + n8n no Railway
- Receber mensagens → criar lead automaticamente
- Enviar alertas de vencimento via WhatsApp
- Custo estimado: ~R$ 50-100/mês

---

## ⚙️ PADRÕES TÉCNICOS

- **DROP POLICY IF EXISTS** + **CREATE POLICY** (sem IF NOT EXISTS em políticas)
- **Substituições no HTML:** sempre via str_replace cirúrgico, nunca reescrever tudo
- **Migrações SQL:** numeradas sequencialmente (06_, 07_, 08_...)
- **CPF deduplicação:** normalizar removendo não-dígitos antes de comparar
- **produto enum:** string vazia vira NULL antes de salvar (evita erro de enum)
- **construtor_vendedor:** remover sufixo ` (tipo)` antes de salvar
- **Verbo de erro:** sempre toast() + console.error() no catch do saveCliente
- **Secrets do Worker:** nunca colar direto no terminal — sempre via Bloco de Notas primeiro para evitar caracteres invisíveis
- **Constraint documentos_cliente_tipo_check:** valores aceitos: gdrive, link, storage, r2

---

## 🧠 GESTÃO DO CONTEXTO

Esta conversa deve ser encerrada quando:
- A janela de contexto estiver longa (Claude vai sinalizar proativamente)
- Antes de iniciar uma nova fase ou funcionalidade grande
- Sempre atualizar CONTEXTO_PROJETO_PRIME e FASE_BRIEFING antes de encerrar

**Para iniciar novo chat:** subir os arquivos .md atualizados como contexto do projeto.
