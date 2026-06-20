# 📋 BRIEFING DO PROJETO — STATUS ATUAL
## CRM Prime — Atualizado em 20/06/2026

---

## ✅ O QUE JÁ ESTÁ PRONTO

### Fase 1 — CRM Operacional
- ✅ Banco migrado (scripts 06-10 executados)
- ✅ Kanban operacional com drag and drop (9 colunas)
- ✅ Sistema de alertas R1 e R2
- ✅ Pop-up de alertas no login
- ✅ Campos de renda e financiamento habitacional
- ✅ Comissão protegida para gestores
- ✅ Vistorias com histórico e alertas
- ✅ CPF e telefone únicos
- ✅ Autenticação segura via Supabase Auth + JWT
- ✅ RLS ativo em todas as tabelas
- ✅ Cloudflare Pages (substituiu Netlify)
- ✅ Edge Function admin-usuarios v2
- ✅ Pós-venda separado do kanban

### Fase 2 — Armazenamento R2
- ✅ Bucket R2 criado: `prime-documentos`
- ✅ Worker publicado: `prime-upload-worker`
- ✅ Upload direto no card (arrastar arquivo → R2)
- ✅ Validação JWT no Worker (segurança LGPD)
- ✅ Compatibilidade retroativa com links Google Drive
- ✅ Constraint banco atualizada (tipo aceita 'r2')
- ✅ Testado com operadora Aline — PDF e documento funcionando

---

## ⏳ PRÓXIMA AÇÃO — ETAPA 5: MIGRAÇÃO DO TRELLO

**Quando fazer:** equipe já pode usar o CRM. Migrar o Trello quando a equipe confirmar que está confortável com o kanban interno.

**Como fazer:**
1. Exportar JSON do board do Trello (Menu do board → Exportar como JSON)
2. Subir o JSON no chat para Claude processar
3. Claude gera script SQL mapeando cards → clientes por CPF/telefone
4. Executar script no Supabase (seta `tem_card = true` para os ativos)
5. Equipe valida no kanban
6. Desativar o Trello

**Regra importante:** clientes legados só recebem card via importação do Trello ou no momento da primeira interação no CRM.

**Perguntas que precisarão ser respondidas na hora:**
- Quantos cards ativos existem no Trello hoje?
- Os cards têm CPF registrado no título ou descrição?
- Quais são as listas (colunas) do board? → precisamos mapear para o status_enum do CRM

---

## 🔮 FASE 3 — WHATSAPP (planejada, sem data)

- Z-API + n8n no Railway
- Receber mensagens → criar lead automaticamente no CRM
- Enviar alertas de vencimento via WhatsApp
- Custo estimado: ~R$ 50-100/mês dependendo do volume
- **Pré-requisito:** definir budget para esta fase

---

## 💡 MELHORIAS IDENTIFICADAS (backlog)

Itens que surgiram durante o desenvolvimento e podem ser implementados futuramente:

- [ ] Operadores auto-populando do banco ao criar novos usuários (hoje é lista estática no HTML)
- [ ] Lista de construtores/vendedores para popular a tabela `construtores` (vazia atualmente)
- [ ] Exclusão de arquivos do R2 quando documento é removido do card (hoje só remove o registro do banco)
- [ ] Visualização de imagens direto no card (hoje baixa o arquivo)
- [ ] Relatório mensal PDF por corretor
- [ ] Dashboard com comparativo mês a mês

---

## 🏗️ ARQUITETURA ATUAL COMPLETA

```
┌─────────────────────────────────────────┐
│         USUÁRIO (navegador)             │
│   prime-crm.sistemaintegrado...dev      │
└──────────────┬──────────────────────────┘
               │
       ┌───────▼────────┐
       │ Cloudflare     │
       │ Pages          │  ← index.html (v12, ~1972 linhas)
       │ (hospedagem)   │    GitHub → auto-deploy
       └───────┬────────┘
               │
    ┌──────────▼──────────────────────┐
    │         SUPABASE                │
    │  · PostgreSQL (banco de dados)  │
    │  · Auth (login JWT)             │
    │  · Edge Function admin-usuarios │
    │  · RLS em todas as tabelas      │
    └──────────┬──────────────────────┘
               │
    ┌──────────▼──────────────────────┐
    │      CLOUDFLARE                 │
    │  · R2: prime-documentos         │  ← arquivos físicos
    │  · Worker: prime-upload-worker  │  ← valida JWT + salva R2
    └─────────────────────────────────┘
```

---

## 📌 DECISÕES TÉCNICAS REGISTRADAS

| Data | Decisão | Motivo |
|---|---|---|
| Jun/2026 | Token JWT em memória (window._authToken) | LGPD — não persiste no navegador |
| Jun/2026 | Service role key só na Edge Function | Segurança — nunca no frontend |
| Jun/2026 | R2 com URL pública por link | Custo zero vs Cloudflare Access pago. Risco aceito e documentado. |
| Jun/2026 | Secrets do Worker via wrangler secret put | Nunca em arquivo de código (segurança) |
| Jun/2026 | Constraint tipo_check atualizada | Adicionado 'r2' e 'storage' além de 'gdrive' e 'link' |
| Jun/2026 | Manter link Google Drive no modal | Compatibilidade retroativa com documentos antigos |
