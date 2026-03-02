# 7. Arquitetura Empresarial

[← Simulação Financeira](06_simulacao_financeira.md) | [Índice](README.md) | [Modelo de Dados →](08_modelo_dados.md)

---

## 🏗️ Pilares da Plataforma

```mermaid
graph TB
    subgraph "P1 — Identity & Tenant"
        A1[Users + 2FA + OAuth]
        A2[Tenants + Settings]
        A3[Partners + White-Label]
        A4[Memberships + RBAC]
        A5[API Keys + Credentials]
        A6[Device Tracker]
    end
    
    subgraph "P2 — Project Engine"
        B1[ProjectTypes Registry]
        B2[Project + Versions]
        B3[Custom Tools]
        B4[Workflow Builder]
        B5[Squads Multi-Agent]
        B6[KnowledgeBase + GraphRAG]
    end
    
    subgraph "P3 — Vapi Orchestration"
        C1[Provisioning]
        C2[Assistants + Tools]
        C3[Campaigns Multi-Channel]
        C4[Phone Numbers]
        C5[Voices + Clone]
    end
    
    subgraph "P4 — Event & Data Layer"
        D1[Call Ingestion + QA]
        D2[Leads + Hot Lead Auto]
        D3[Omnichannel CRM]
        D4[Cost Engine + Usage]
    end
    
    subgraph "P5 — Intelligence"
        E1[Chat LLM Multi-Provider]
        E2[A/B Testing]
        E3[Auto-Tuning IA]
        E4[Semantic Memory]
        E5[GraphRAG Search]
    end
    
    subgraph "P6 — Billing & Entitlements"
        F1[Plans + Subscriptions]
        F2[Usage Tracking + Predictive]
        F3[Budget Cap + Guard]
        F4[Entitlements 19 funções]
        F5[Ledger + Markup]
    end
    
    subgraph "P7 — Observability & Compliance"
        G1[Audit Trail 90+ ações]
        G2[Error Logger 4 camadas]
        G3[Telemetry 14 eventos]
        G4[Activity Feed 50 ações]
        G5[LGPD Data Portability]
    end

    A2 --> B1
    B1 --> C1
    C1 --> D1
    D1 --> F2
    F1 --> B2
    B5 --> C2
    D2 --> D3
    E1 --> E4
```

---

## 📐 Diagrama de Componentes

```mermaid
graph LR
    subgraph "Frontend (82 LiveViews)"
        FE1[Wizard + Editor]
        FE2[Dashboard + Analytics]
        FE3[Calls + Leads + CRM]
        FE4[Campaigns]
        FE5[Chat LLM]
        FE6[Billing + Settings]
        FE7[Partner Panel]
        FE8[Admin Panel]
        FE9[Search + Notifications]
    end
    
    subgraph "Backend (52+ Contexts)"
        BE1[Accounts + Auth]
        BE2[Projects + Types]
        BE3[Vapi + Provisioning]
        BE4[Calls + Leads]
        BE5[Billing + Entitlements]
        BE6[QA + Evals]
        BE7[Integrations + Webhooks]
        BE8[Chat LLM + GraphRAG]
        BE9[Telegram + SMS]
    end
    
    subgraph "Workers (28 Oban)"
        W1[Call Ingest + QA]
        W2[Campaign Execution]
        W3[Budget + Usage]
        W4[GraphRAG Index]
        W5[Auto-Tune + Hot Lead]
    end
    
    subgraph "External"
        EX1["API Vapi"]
        EX2["Twilio / Telnyx"]
        EX3["Stripe (futuro)"]
        EX4["CRMs (HubSpot, Pipedrive)"]
        EX5["Telegram API"]
        EX6["LLMs (Gemini, OpenAI, Anthropic)"]
    end
```

---

## 🔄 Fluxos Críticos

### Fluxo 1: Provisionamento de Projeto

```mermaid
sequenceDiagram
    participant U as Usuário
    participant W as WizardLive (3 passos)
    participant PE as Project Engine
    participant PR as Provisioning (Oban)
    participant V as Vapi API

    U->>W: Preenche tipo → config → deploy
    W->>PE: Cria projeto draft
    PE->>PE: Gera config + cria versão SHA256
    U->>W: Clica "Publicar"
    W->>PR: Enfileira job Oban
    PR->>V: POST /tool (para cada tool)
    V-->>PR: tool_ids
    PR->>V: POST /assistant (com tool_ids)
    V-->>PR: assistant_id → VapiResource
    PR->>V: POST /phone-number
    V-->>PR: phone_number_id
    PR->>PE: Salva IDs + ativa projeto
    PE-->>U: Projeto publicado ✅
```

### Fluxo 2: Ingestão de Chamada

```mermaid
sequenceDiagram
    participant V as Vapi
    participant WH as Webhook Controller
    participant I as CallIngestWorker (Oban)
    participant DB as Database
    participant PS as PubSub
    participant QA as CallQAWorker
    participant HL as HotLeadWorker

    V->>WH: POST /webhooks/vapi
    WH->>I: Enfileira com idempotência (external_id)
    I->>DB: Salva call + transcript
    I->>DB: Extrai + salva lead
    I->>DB: Calcula custo → usage_daily
    I->>PS: Broadcast call_ended
    I->>QA: Enfileira eval automático
    QA->>DB: success_score ≥/< threshold?
    QA->>HL: Se score alto → HotLeadWorker
    HL->>DB: Auto-liga closer em 30s
    PS-->>Dashboard: UI atualiza em tempo real
```

### Fluxo 3: Campanha Multi-Channel

```mermaid
graph LR
    A[Criar Campanha] --> B{Canal?}
    B -->|Voice| C[CampaignDialerWorker]
    B -->|SMS| D[CampaignSmsWorker]
    B -->|Telegram| E[CampaignTelegramWorker]
    C --> F[DispatchCallWorker]
    D --> G[SmsChannel.send_sms]
    E --> H[Telegram.Notifier]
    F --> I[CallIngestWorker]
    G --> J[TwilioWebhookController]
    I --> K[Lead + CRM Omnichannel]
    J --> K
```

### Fluxo 4: Budget Cap + Billing Preditivo

```mermaid
sequenceDiagram
    participant I as Ingestion
    participant U as UsageAggregatorWorker
    participant B as BudgetGuard GenServer
    participant P as Predictive
    participant T as Tenant

    I->>U: Atualiza minutos
    U->>B: Verifica budget_cap
    B->>B: total_cost >= budget_cap?
    alt Excedeu
        B->>T: Bloqueia tenant
        B->>PubSub: budget_blocked event
        B->>Telegram: Alerta budget via Notifier
        B->>Email: UserNotifier.deliver_budget_alert
    else Dentro
        U->>P: burn_rate_forecast (média 7 dias)
        P->>P: days_remaining <= threshold?
        alt Vai acabar
            P->>Telegram: Alerta preditivo
            P->>Email: Alerta preditivo
        end
    end
```

---

## 🔐 Hierarquia Multi-Tenant

```mermaid
graph TB
    A["Você (Platform Owner / Admin)"]
    A --> B["Partner A (Agência)"]
    A --> C["Partner B (Franquia)"]
    A --> D["Direct Tenant X"]
    B --> E[Tenant 1]
    B --> F[Tenant 2]
    B --> G[Tenant 3]
    C --> H[Unidade 1]
    C --> I[Unidade 2]
    C --> J[Unidade 3]
    
    style A fill:#e74c3c,color:white
    style B fill:#8e44ad,color:white
    style C fill:#8e44ad,color:white
    style D fill:#3498db,color:white
```

**RBAC por Membership:**
- `admin` — Tudo (edit, publish, billing, settings, invite, feature flags)
- `operator` — Edit + publish + view dashboard
- `viewer` — Somente visualização
- `custom` — Definido por TenantRole.permissions (JSONB)

---

## 🧱 Módulos Elixir (52+ Contexts)

| # | Context | Responsabilidade |
|---|---------|-----------------|
| 1 | `Accounts` | Users, Tenants, Partners, Memberships, Auth, 2FA, Devices, OAuth, API Keys, Credentials, Policy |
| 2 | `Projects` | Projects, Types, Versions, Deployments, Custom Tools, Workflows |
| 3 | `Vapi` | Client HTTP (retry + circuit breaker), Provisioning, VapiResource |
| 4 | `Calls` | Ingestion, Transcripts, CallMonitor GenServer |
| 5 | `Leads` | Leads, qualification, promote to contact |
| 6 | `Campaigns` | Campaign lifecycle, CampaignLeads, multi-channel dispatch |
| 7 | `Billing` | Plans, Subscriptions, Usage, Entitlements, BudgetGuard, Ledger, Markup, Predictive |
| 8 | `QA / Evals` | StaticValidator, EvalRunner, EvalSuites, CI.ValidationPipeline |
| 9 | `Chat` | ChatSession, LLM multi-provider (Gemini/OpenAI/Anthropic), 80+ tools, SystemKnowledge, Tracer |
| 10 | `KnowledgeBase` | File upload, indexação, search |
| 11 | `KnowledgeGraph` | GraphRAG (entities, relationships, search semântico) |
| 12 | `Omnichannel` | Contacts, Threads, Messages, cross-channel handoff |
| 13 | `Telegram` | Bot GenServer, 7 comandos, Notifier, RateLimiter, Session, LinkToken |
| 14 | `SMS` | SmsSession, SmsChannel (Twilio), DNC |
| 15 | `Voices` | CustomVoice, VoiceClone (ElevenLabs) |
| 16 | `Analytics` | Boards, Widgets configuráveis |
| 17 | `Forms` | FormTemplate, FormSubmission |
| 18 | `Integrations` | IntegrationDefinition (8 tipos), TenantIntegration |
| 19 | `Webhooks` | Outbound webhooks (23 eventos), DeliveryLog |
| 20 | `Squads` | Squad, SquadMember, multi-agent handoff |
| 21 | `Audit` | AuditLog (90+ ações), AuditConfig, AuditHook |
| 22 | `Activities` | ActivityLog (50 ações humanizadas), feed com PubSub |
| 23 | `Notifications` | In-app notifications |
| 24 | `Alerts` | Email + Webhook alertas, LLMAlertHandler |
| 25 | `FeatureFlags` | Flags com rollout %, overrides, QA circuit breaker |
| 26 | `Media` | MediaFile upload, storage |
| 27 | `Search` | Busca paralela global (9 tipos), cache ETS |
| 28 | `Config` | 664 funções tipadas + Settings DB-backed |
| 29 | `ErrorLogger` | GenServer 4 camadas (backend + frontend) |
| 30 | `DataExchange` | LGPD import/export |
| 31 | `Templates` | Biblioteca de templates de agente |
| 32 | `Mcp` | Model Context Protocol (34 tools) |

---

## ✅ Validação de Compatibilidade

| Modelo de Negócio | Suportado? | Pilares responsáveis |
|-------------------|------------|----------------------|
| Agência DFY | ✅ | P1 + P2 + P3 |
| Self-serve | ✅ | P2 + P5 + P6 + P7 |
| White-label | ✅ | P1 (Partners + WhiteLabel plug) |
| Enterprise | ✅ | P7 (Audit + LGPD) + P6 |
| Performance | ✅ | P4 (Leads + Hot Lead) + P6 |
| Outbound | ✅ | P3 (Campaigns multi-channel) |
| Franquias | ✅ | P1 (Hierarquia) |
| BYOK/BYOC | ✅ | P1 (Credentials criptografadas) |
| Marketplace | ✅ | P2 (Templates) + P6 |

> **Conclusão**: A arquitetura é **agnóstica ao modelo de negócio**. O que muda é configuração, não código.

---

[← Simulação Financeira](06_simulacao_financeira.md) | [Índice](README.md) | [Modelo de Dados →](08_modelo_dados.md)
