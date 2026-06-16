# 📋 FASE 2 — BRIEFING COMPLETO
## CRM Prime — Redesign do Funil e Novas Funcionalidades

---

## 1. NOVOS STATUS DO FUNIL (substituem status_enum atual)

Baseado no Trello atual da Prime. Ordem das colunas no Kanban:

| # | Status | Descrição | Prazo alerta padrão |
|---|---|---|---|
| 1 | **Lead** | Cliente chegou via WhatsApp ou indicação. Ainda não iniciou atendimento formal. | 2 dias |
| 2 | **Andamento** | Atendimento iniciado: coleta de dados, documentos, simulações em curso. | A definir |
| 3 | **Condicionado** | Cliente com condicionantes (dívidas, rating, renda). Possibilidade real, mas precisa de ação. Revisitar. | A definir |
| 4 | **Reprovado** | Cliente reprovado por algum motivo. Não descartado — precisa ser revisitado após prazo. | A definir |
| 5 | **Aprovado** | Cliente aprovado. Precisa de acompanhamento ativo para evoluir. | A definir |
| 6 | **Conformidade** | Envio de documentos do cliente e do imóvel para aprovação pela Caixa. Acompanhamento intenso. | A definir |
| 7 | **Assinatura** | Cliente vai assinar o contrato de financiamento. Acompanhamento necessário. | A definir |
| 8 | **Cartório/Prefeitura** | Pós-assinatura: registro do contrato em cartório e/ou prefeitura. | A definir |
| 9 | **Concluído** | Contrato registrado e enviado à Caixa para liberação dos valores. Etapa final. | — |

**Coluna separada (não é status de funil):**
- **Vistoria/Laudos** — imóveis em processo de vistoria técnica ou laudo. Tratamento independente do funil principal.

---

## 2. SISTEMA DE ALERTAS POR TEMPO

### Regra geral
Cada cliente tem um campo `proxima_interacao` (data/hora) preenchido a cada atendimento.
Se não preenchido, o sistema assume `ultima_interacao + prazo_da_etapa`.
Ao atingir o prazo, o card aparece destacado no Kanban e gera alerta no Dashboard.

### Campos necessários no banco
```sql
ultima_interacao    timestamptz   -- atualizado a cada registro no histórico
proxima_interacao   timestamptz   -- preenchido pelo operador ao registrar interação
prazo_alerta_dias   integer       -- herdado da etapa, pode ser sobrescrito por cliente
em_alerta           boolean       -- calculado: proxima_interacao < now()
```

### Prazos por etapa (a serem confirmados pelo Thiago)
- Lead: **2 dias** (único já definido)
- Demais etapas: preencher antes da Fase 2 começar

### Como aparece para o usuário
- Card no Kanban com borda vermelha se em alerta
- Contador de dias em atraso visível no card
- Métrica no Dashboard: "X clientes precisam de contato hoje"
- Filtro na tela Clientes: "Somente em alerta"

---

## 3. CAMPOS NOVOS NA FICHA DO CLIENTE

### 3a. Operador (funcionário operacional da Prime)
```sql
operador   text   -- quem está operando/acompanhando o processo
```
- Aparece no formulário de cadastro e na ficha
- Visível para todos
- Filtrável na tela de Clientes

### 3b. Comissionamento (só gestores)
```sql
comissao_valor      numeric     -- valor previsto ou pago
comissao_status     text        -- 'Previsto' | 'A receber' | 'Pago'
comissao_obs        text        -- observações sobre o comissionamento
```
- **Invisível para role = 'usuario'**
- Visível e editável apenas para role = 'gestor'
- Aparece como seção separada na ficha do cliente

### 3c. Separação de origens (substituir campo "parceiro")
Hoje existe só "parceiro" misturando tudo. Separar em:

```sql
indicado_por        text    -- quem indicou o cliente (pessoa física, parceiro)
corretor_externo    text    -- corretor de imóveis (externo à Prime)
construtor_vendedor text    -- construtora ou vendedor do imóvel
```

Remover ou renomear o campo `parceiro` atual.

### 3d. Corretores internos (campo `corretor` atual)
Manter como está mas revisar a lista. Opções fixas mais recorrentes
+ opção "Outros" que abre campo de texto livre.

Corretores confirmados até agora:
Josiel, Leilane, Manoel, Romário, Thiago, Simone, Edivan,
Ketlen, Bruno, Amanda, Ygor, Uembley

---

## 4. KANBAN REDESENHADO

### Colunas na ordem
1. Lead
2. Andamento
3. Condicionado
4. Reprovado
5. Aprovado
6. Conformidade
7. Assinatura
8. Cartório/Prefeitura
9. Concluído
10. Vistoria/Laudos *(coluna separada, cor diferente)*

### Informações visíveis no card
- Nome do cliente
- Produto
- Operador responsável
- Dias desde última interação (com cor: verde/amarelo/vermelho)
- Ícone de alerta se `em_alerta = true`
- Data da próxima interação prevista

### Registro de interação direto do card
Ao clicar no card, além de ver a ficha, deve ser fácil registrar
uma nova interação (modal rápido com: tipo, descrição, próxima interação).

---

## 5. HISTÓRICO DE INTERAÇÕES (melhorias)

Cada interação registrada deve ter:
```sql
tipo              text        -- WhatsApp, Ligação, Email, Visita, Nota, Status alterado
descricao         text        -- o que foi feito/combinado
proxima_interacao timestamptz -- quando será o próximo contato
registrado_por    text        -- operador que registrou
created_at        timestamptz -- automático
```

O campo `proxima_interacao` da interação atualiza automaticamente
o campo `proxima_interacao` da tabela clientes.

---

## 6. MIGRAÇÃO DA BASE EXISTENTE

### Regras para migração
- Só iniciar após o novo schema estar 100% definido e aprovado
- Mapeamento dos status antigos → novos status (a definir)
- Campo `ultima_interacao` será preenchido com `dt_atualizacao` atual
- Campo `proxima_interacao` será preenchido com `ultima_interacao + 2 dias` (padrão Lead)
  ajustado conforme o status do cliente
- Alertas já ficam ativos desde a migração

### Mapeamento provisório de status (a confirmar)
| Status atual | Status novo |
|---|---|
| Em andamento | Andamento |
| Simulação | Andamento |
| SICAQ | Aprovado |
| SIOPI | Conformidade |
| Aguardando | Andamento |
| Concluído | Concluído |
| Reprovado | Reprovado |
| Restrição | Condicionado |
| Perdeu | (manter como Perdeu ou arquivar) |

---

## 7. ORDEM DE EXECUÇÃO DA FASE 2

**Antes de começar qualquer código:**
1. Thiago confirma os prazos de alerta por etapa
2. Thiago confirma o mapeamento de status antigos → novos
3. Thiago confirma a lista de corretores internos final
4. Thiago confirma quais campos de comissão são necessários

**Sequência de desenvolvimento:**
1. SQL: novo schema (novos status, campos novos, tabela de prazos por etapa)
2. SQL: migração dos dados existentes
3. HTML: novo Kanban com colunas redesenhadas
4. HTML: nova ficha de cliente com campos novos
5. HTML: sistema de alertas no Dashboard
6. HTML: filtro "em alerta" na tela Clientes
7. Teste completo antes de deploy

**Regra de créditos Netlify:**
- Sempre resolver no banco antes de gerar deploy
- Agrupar todas as mudanças de HTML em no máximo 2 deploys
- Renovação de créditos: 03/julho/2026

---

## 8. PERGUNTAS ABERTAS PARA THIAGO RESPONDER

Antes de iniciar a Fase 2, preciso das respostas abaixo:

- [ ] Qual o prazo de alerta para cada etapa (dias sem interação)?
- [ ] O status "Perdeu" some do funil ou vai para uma aba de arquivo?
- [ ] "Vistoria/Laudos" — quais clientes entram ali? Só Habitacional?
- [ ] O campo comissão mostra valor bruto, líquido ou ambos?
- [ ] Há necessidade de registrar o imóvel vinculado ao cliente (endereço, valor, tipo)?
- [ ] A lista de corretores internos está completa ou falta alguém?
- [ ] Quais são os parceiros/indicadores mais frequentes para criar lista fixa?

