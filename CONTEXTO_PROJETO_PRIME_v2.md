# 🧠 CONTEXTO PERMANENTE — CRM PRIME
## Versão 2 — Atualizado em 15/06/2026

---

## 🏢 A EMPRESA

**Nome:** Prime Correspondente Bancário
**Segmento:** Correspondente bancário da Caixa Econômica Federal
**Tamanho:** 5 funcionários
**Perfil técnico da equipe:** Básico
**Uso do sistema:** Celular + computador no escritório

**Produtos trabalhados:**
Habitacional MCMV, Habitacional SBPE, Conta Corrente, Poupança,
Cartão de Crédito, Consórcio, Consignado, Seguro de Vida,
Rapidex/Amparo, Múltiplo

---

## 🌐 INFRAESTRUTURA

| Item | Valor |
|---|---|
| URL produção | https://snazzy-kitsune-fb7ad0.netlify.app/ |
| Netlify | Plano Free — créditos atuais: ~59 (renova 03/julho) |
| Custo por deploy | 15 créditos |
| Deploys restantes até julho | ~3 (usar com extrema parcimônia) |
| Supabase projeto | dvkniqxjudcxzpbmptbb |
| Supabase URL | https://dvkniqxjudcxzpbmptbb.supabase.co |
| Repositório | GitHub → Netlify (CI/CD automático) |
| Arquivo principal | index.html na raiz do repositório |

**Regra de ouro para Netlify:** sempre tentar resolver no banco (SQL)
antes de gerar novo index.html. Cada deploy = 15 créditos consumidos.

---

## 🔐 AUTENTICAÇÃO — ESTADO ATUAL (pós-migração Fase 1)

**Sistema:** Supabase Auth (e-mail + senha) com JWT
**Tabela de perfis:** `perfis` (foi `perfis_v2` durante migração)
**Campos:** id (uuid, FK auth.users), nome, role, ativo, ultimo_acesso
**Roles:** `gestor` | `usuario`
**Login:** e-mail + senha via `sb.auth.signInWithPassword`
**persistSession:** false (usuário sempre precisa logar — sem sessão salva)
**Edge Function:** `admin-usuarios` (criar/deletar usuários com segurança)

**Usuários cadastrados:**
| E-mail | Nome | Role |
|---|---|---|
| thiagologistica047@gmail.com | Thiago | gestor |
| prime@prime.crm | Prime | gestor |
| leilane@prime.crm | Leilane | gestor |
| josiel@prime.crm | Josiel | usuario |

**Políticas RLS ativas na tabela perfis:**
- `leitura_perfis_autenticados` → SELECT para authenticated
- `update_perfis_autenticados` → UPDATE para authenticated
- `insert_perfis_service_e_auth` → INSERT para authenticated + service_role
- `service_role_perfis_v2` → ALL para service_role

---

## 🗄️ BANCO DE DADOS — ESTADO ATUAL

### Tabelas existentes
```
clientes              ← base principal (~293 registros)
historico             ← log de interações por cliente
documentos            ← documentos vinculados (estrutura pronta, pouco uso)
tarefas               ← follow-ups com vencimento
perfis                ← usuários do CRM (antes era perfis_v2)
perfis_backup_20260608 ← backup do sistema de login antigo (login+senha_hash)
```

### Tabela clientes — campos atuais
```sql
id, nome, cpf, telefone, email, produto (produto_enum), status (status_enum),
corretor, parceiro, observacoes, arquivo_vinculado, origem_arquivo,
dt_entrada, dt_atualizacao, created_at, updated_at,
produtos_contratados (text[] ou text separado por vírgula)
```

### ENUMs atuais (a serem SUBSTITUÍDOS na Fase 2)
```sql
produto_enum: Habitacional MCMV, Habitacional SBPE, Conta Corrente,
              Poupança, Cartão de Crédito, Consórcio, Consignado,
              Seguro de Vida, Rapidex/Amparo, Múltiplo

status_enum: Em andamento, Simulação, SICAQ, SIOPI, Aguardando,
             Concluído, Reprovado, Restrição, Perdeu
```
⚠️ Estes enums serão completamente redesenhados na Fase 2.
Ver arquivo FASE2_BRIEFING.md para os novos status.

### RLS
- Todas as tabelas com RLS ativo
- Tabelas operacionais (clientes, historico, tarefas, documentos):
  políticas `anon_*` substituídas por `auth_*` (exigem authenticated)
- Ver 10_migracao_auth_etapa4.sql para detalhes

---

## 💻 ESTRUTURA DO index.html ATUAL

**Bibliotecas CDN:**
- @supabase/supabase-js@2
- Chart.js@4.4.1
- @tabler/icons-webfont@3.19.0

**Páginas:**
| ID | Nome |
|---|---|
| page-login | Tela de login (e-mail + senha) |
| page-dashboard | Dashboard com 5 métricas e 4 gráficos |
| page-clientes | Tabela com filtros e exportação CSV |
| page-kanban | Funil kanban por status |
| page-tarefas | Lista de tarefas/follow-ups |
| page-detail | Ficha completa do cliente |
| page-equipe | Gestão de usuários (só gestor) |
| page-senha | Alterar senha própria |

**Variáveis globais JS:**
`sb, all, filtered, pg, PER, detailId, editId, charts, usuarioAtual, editUsuarioId`

**Funções principais:**
`doLogin, doLogout, carregarPerfil, entrarNoApp, mostrarLogin,
loadAll, go, renderMetrics, renderCharts, filterTable, renderTable,
renderKanban, openDetail, loadHist, loadTasks, saveCliente,
carregarEquipe, salvarUsuario, chamarEdgeFunction, salvarSenha,
exportarCSV, mascaraCPF, mascaraTel`

---

## 🔑 PRINCÍPIOS DE DESENVOLVIMENTO

1. **Netlify créditos:** sempre resolver no banco (SQL) antes de gerar deploy
2. **Edições cirúrgicas:** nunca reescrever o arquivo inteiro sem necessidade
3. **Supabase Auth:** nunca reverter para login por senha_hash na tabela
4. **RLS sempre ativo:** nunca desabilitar RLS nas tabelas
5. **CPF como chave única:** deduplicação sempre por CPF normalizado
6. **Dual format:** `produtos_contratados` pode ser array ou string — tratar ambos

---

## 📋 HISTÓRICO DE PROBLEMAS RESOLVIDOS

| Problema | Causa | Solução |
|---|---|---|
| Alerta segurança Supabase | RLS desativado | 06_rls_opcao1.sql aplicado |
| Login não funcionava | Tabela perfis_v2 renomeada para perfis | View perfis_v2 criada + código corrigido |
| Login travava em "Verificando" | onAuthStateChange conflitando com doLogin | Removido onAuthStateChange, fluxo direto |
| Login automático indesejado | persistSession: true com sessão salva | persistSession: false |
| Equipe vazia | Faltava política INSERT em perfis | insert_perfis_service_e_auth criada |

---

## 🚀 FASE 2 — PLANEJADA (NÃO INICIADA)

Ver arquivo `FASE2_BRIEFING.md` para detalhamento completo.

Resumo:
- Redesign completo dos status do funil (baseado no Trello atual)
- Novo sistema de alertas por tempo por etapa
- Campo de comissionamento (visível só para gestores)
- Campo de operador (funcionário operacional da Prime)
- Separação clara: corretor / parceiro/indicação / construtor/vendedor
- Kanban redesenhado com as novas colunas
- Migração da base de clientes existente para a nova estrutura
- Regras de tempo mínimo por etapa com alertas automáticos

