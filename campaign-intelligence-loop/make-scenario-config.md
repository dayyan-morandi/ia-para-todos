# Make Scenario — Campaign Intelligence Loop
## Configuração completa: Meta API → Claude → Meta API

**Duração estimada para montar:** 45–60 minutos  
**Custo operacional:** ~150 operações/execução · execução diária = ~4.500 ops/mês (dentro do plano gratuito se < 1.000 leads/mês em outros cenários)

---

## PRÉ-REQUISITOS

Antes de criar o cenário, tenha em mãos:

| Variável | Como obter |
|---|---|
| `META_ACCESS_TOKEN` | Meta Business → Settings → System Users → Generate Token |
| `META_AD_ACCOUNT_ID` | Meta Ads Manager → URL da conta (número após `act_`) |
| `CLAUDE_API_KEY` | console.anthropic.com → API Keys |
| `NOTIFICATION_EMAIL` | Seu email para alertas |

---

## ESTRUTURA DO CENÁRIO (7 módulos)

```
[1. Schedule]
    ↓
[2. GET Meta API — métricas dos ad sets]
    ↓
[3. Formatar dados para Claude]
    ↓
[4. POST Claude API — decisão]
    ↓
[5. Parse JSON da resposta]
    ↓
[6. Iterator — percorre cada decisão]
    ↓
[7. Router]
    ├─ scale_up  → [8A. PATCH Meta API — aumentar budget]
    ├─ pause     → [8B. POST Meta API — pausar ad set]
    ├─ alert     → [8C. Email — notificação]
    └─ hold      → [8D. (nada — apenas log)]
         ↓ (todos os branches)
[9. Google Sheets — log auditável]
```

---

## MÓDULO 1 — Schedule (Trigger)

No Make: **Create a new scenario → Add a trigger → Built-in → Scheduling**

Configurações:
```
Run scenario: Every day
Time: 07:00
Timezone: America/Sao_Paulo
```

> Não existe um módulo visual de agendamento — o scheduling é configurado no painel do cenário (ícone de relógio no topo da tela).

---

## MÓDULO 2 — GET Meta Ads API (métricas)

**Módulo:** `HTTP → Make a request`

```
URL:
https://graph.facebook.com/v19.0/act_{{META_AD_ACCOUNT_ID}}/adsets

Method: GET

Query String (adicione cada um separado):
  fields = id,name,status,daily_budget,insights.date_preset(last_7d){spend,impressions,clicks,actions,ctr,frequency,cost_per_action_type}
  access_token = {{META_ACCESS_TOKEN}}
  limit = 50

Parse response: YES
```

**Nota:** `daily_budget` vem em centavos. Ex: R$50/dia = `5000`.  
**Nota:** `cost_per_action_type` retorna um array — o CPA de compra está no item com `action_type: "offsite_conversion.fb_pixel_purchase"`.

---

## MÓDULO 3 — Formatar dados para Claude

**Módulo:** `Tools → Set multiple variables` ou use um **Text aggregator** para montar o JSON.

O objetivo é transformar a resposta crua da Meta API em um array limpo para o Claude. Monte este JSON manualmente combinando os campos do Módulo 2:

```
Variável: ad_sets_payload

Valor (template Make):
[
  {{#each data}}
  {
    "ad_set_id": "{{id}}",
    "ad_set_name": "{{name}}",
    "daily_budget_brl": {{divide daily_budget 100}},
    "spend_7d_brl": {{insights.data.0.spend}},
    "impressions_7d": {{insights.data.0.impressions}},
    "clicks_7d": {{insights.data.0.clicks}},
    "ctr_pct": {{insights.data.0.ctr}},
    "frequency": {{insights.data.0.frequency}},
    "conversions_7d": "extrair de actions onde action_type = offsite_conversion.fb_pixel_purchase",
    "cpa_brl": "extrair de cost_per_action_type onde action_type = offsite_conversion.fb_pixel_purchase"
  }
  {{/each}}
]
```

> Para extrair conversões e CPA do array de actions, use o módulo **Array aggregator** ou o **Iterator** antes deste passo para processar cada ad set.

---

## MÓDULO 4 — POST Claude API (decisão)

**Módulo:** `HTTP → Make a request`

```
URL: https://api.anthropic.com/v1/messages

Method: POST

Headers:
  x-api-key: {{CLAUDE_API_KEY}}
  anthropic-version: 2023-06-01
  content-type: application/json

Body type: Raw (JSON)

Body:
{
  "model": "claude-haiku-4-5-20251001",
  "max_tokens": 1024,
  "system": "[cole aqui o system prompt do arquivo claude-decision-agent-prompt.md]",
  "messages": [
    {
      "role": "user",
      "content": "Analise os seguintes ad sets e retorne suas decisões:\n\n{{ad_sets_payload}}"
    }
  ]
}

Parse response: YES
```

**Custo estimado por execução:** ~R$0,05–0,10 usando claude-haiku (modelo mais barato, rápido e suficiente para esse uso).

---

## MÓDULO 5 — Parse resposta do Claude

**Módulo:** `JSON → Parse JSON`

```
JSON string: {{4.body.content.0.text}}
```

Isso extrai o campo `text` da resposta da API do Claude, que contém o JSON de decisões.

---

## MÓDULO 6 — Iterator (percorre decisões)

**Módulo:** `Flow control → Iterator`

```
Array: {{5.decisions}}
```

O Iterator vai criar um bundle separado para cada ad set — o Router no módulo seguinte processa um por vez.

---

## MÓDULO 7 — Router

**Módulo:** `Flow control → Router`

Configure 4 branches com filtros:

```
Branch 1 — scale_up
  Condição: {{action}} = scale_up

Branch 2 — pause
  Condição: {{action}} = pause

Branch 3 — alert ou alert_creative
  Condição: {{action}} contains "alert"

Branch 4 — hold / insufficient_data
  Condição: (else — sem filtro)
```

---

## MÓDULO 8A — Escalar budget (branch scale_up)

**Módulo:** `HTTP → Make a request`

```
URL: https://graph.facebook.com/v19.0/{{ad_set_id}}

Method: POST

Body type: application/x-www-form-urlencoded

Body:
  daily_budget = {{round(multiply(current_daily_budget_brl, budget_multiplier, 100))}}
  access_token = {{META_ACCESS_TOKEN}}
```

> `daily_budget` precisa ser enviado em centavos. Multiplique o valor em BRL por 100.

---

## MÓDULO 8B — Pausar ad set (branch pause)

**Módulo:** `HTTP → Make a request`

```
URL: https://graph.facebook.com/v19.0/{{ad_set_id}}

Method: POST

Body type: application/x-www-form-urlencoded

Body:
  status = PAUSED
  access_token = {{META_ACCESS_TOKEN}}
```

---

## MÓDULO 8C — Email de alerta (branch alert)

**Módulo:** `Email → Send an email` (ou Gmail)

```
To: {{NOTIFICATION_EMAIL}}
Subject: ⚠️ Alerta de campanha — {{ad_set_name}}
Body:
Ad Set: {{ad_set_name}}
Ação recomendada: {{action}}
Motivo: {{reason}}

Métricas (últimos 7 dias):
• CPA: R${{metrics.cpa_brl}}
• CTR: {{metrics.ctr_pct}}%
• Frequência: {{metrics.frequency}}
• Gasto: R${{metrics.spend_7d_brl}}
• Conversões: {{metrics.conversions_7d}}

---
Campaign Intelligence Loop · IA para Todos
```

---

## MÓDULO 9 — Log no Google Sheets (todos os branches)

**Módulo:** `Google Sheets → Add a row`  

Crie uma planilha chamada `Campaign Intelligence Log` com as colunas:

```
A: Data
B: Ad Set ID
C: Ad Set Name
D: Action
E: Reason
F: CPA (R$)
G: CTR (%)
H: Frequência
I: Gasto 7d (R$)
J: Conversões 7d
K: Budget Anterior (R$)
L: Budget Novo (R$)
M: Campaign Health
```

Configuração do módulo:
```
Spreadsheet: [selecione sua planilha]
Sheet: Log
Row values:
  A: {{now}}
  B: {{ad_set_id}}
  C: {{ad_set_name}}
  D: {{action}}
  E: {{reason}}
  F: {{metrics.cpa_brl}}
  G: {{metrics.ctr_pct}}
  H: {{metrics.frequency}}
  I: {{metrics.spend_7d_brl}}
  J: {{metrics.conversions_7d}}
  K: {{current_daily_budget_brl}}
  L: {{new_daily_budget_brl}}
  M: {{5.campaign_health}}
```

---

## CHECKLIST DE TESTE

Antes de ativar o cenário em produção:

- [ ] Rode manualmente com `Run once` e verifique se o Módulo 2 retorna dados dos ad sets
- [ ] Confirme que o JSON enviado ao Claude no Módulo 4 está bem formatado (veja na aba de execução)
- [ ] Verifique que o Claude retorna JSON válido (não texto corrido)
- [ ] Teste o branch `scale_up` com um ad set de teste — confirme que o budget mudou no Ads Manager
- [ ] Teste o branch `pause` — confirme que o status ficou PAUSED
- [ ] Confirme que o email de alerta chega corretamente
- [ ] Confirme que a planilha está sendo preenchida

---

## ATIVAR EM PRODUÇÃO

1. No cenário, clique no toggle **ON/OFF** → ligue
2. Configure o scheduling (07h diário)
3. Defina `Max errors`: 3
4. Defina `Data loss`: OFF (nunca descarte execuções com erro)

---

## ESTIMATIVA DE CUSTO MENSAL

| Item | Custo |
|---|---|
| Make (plano Core) | $9/mês — 10.000 ops |
| Claude Haiku (30 execuções/mês) | ~$0.50/mês |
| Meta API | Grátis |
| Google Sheets | Grátis |
| **Total** | **~$9,50/mês** |
