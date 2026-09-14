# Architecture Plan — Agent Platform

> Status: DRAFT for review. No implementation has started. This document is the
> output of Phase 0 (repository inspection) only, per the project brief.

## 1. Estado atual do repositório

O repositório contém **apenas exports de workflows n8n** (a referência
"Secretária IA"), sem nenhum código de aplicação, banco de dados, testes ou
infraestrutura própria. Não há um segundo projeto ("Agente de Vendas") neste
repo — ele foi citado apenas como referência conceitual externa.

```
.
├── README.md
├── .gitignore
├── Secretaria_IA.json                              (2415 linhas, JSON válido)
├── 2__MCP_Google_Calendar.json                      (323 linhas, JSON INVÁLIDO)
├── 3__Baixar_e_enviar_arquivo_do_Google_Drive.json  (230 linhas, JSON INVÁLIDO)
├── 4__Escalar_humano.json                           (434 linhas, JSON INVÁLIDO)
├── 5__Enviar_agendamento.json                       (138 linhas, JSON INVÁLIDO)
└── 6__Assistente_interno.json                       (1056 linhas, JSON INVÁLIDO)
```

Não existem diretórios `/src`, `/database`, `/n8n`, `/docs`, `/scripts` ou
`/tests` — serão criados do zero, não reorganizados.

### 1.1 Workflow principal: `Secretaria_IA.json`

Único arquivo válido e importável hoje. 60 nós. Fluxo observado:

1. **Webhook** (Chatwoot) → **Info** (`Set`, mapeia `$json.body.*` do payload
   Chatwoot direto para variáveis de trabalho) → **Mensagem chegando?**
   (`Filter`: `tipo == incoming` AND `etiquetas` não contém `agente-off`).
2. **Enfileirar mensagem** (Postgres insert em `n8n_fila_mensagens`) → **Esperar**
   (`Wait`) → **Buscar mensagens** (select por `telefone`, ordenado por
   `timestamp`) → **Mensagem encavalada?** (`Code`: compara `id_mensagem` do
   item mais recente da fila com o da execução atual; se diferente, aborta —
   este é o mecanismo de "debounce"/buffer).
3. Suporte a áudio: **Download áudio** → **Transcrever audio** (OpenAI) antes
   de seguir.
4. **Secretária** (`@n8n/n8n-nodes-langchain.agent`) — o agente principal, com:
   - **Memory**: `memoryPostgresChat`, `sessionKey = telefone`, tabela
     `n8n_historico_mensagens`, janela de 50 mensagens.
   - Tools: `Refletir` (think), `Reagir mensagem` (HTTP), `MCP Google Calendar`
     (via `mcpClientTool` apontando para um endpoint SSE hospedado em
     `n8n.cloud`), `Listar arquivos` (Google Drive), `Baixar e enviar arquivo`
     (sub-workflow), `Enviar alerta de cancelamento` (Telegram), `Escalar
     humano` (sub-workflow).
5. Pós-processamento: decide texto vs. áudio (SSML via `chainLlm`,
   texto-para-fala via ElevenLabs), envia pela API HTTP do Chatwoot, e reseta
   o `automation_status`/status de digitação.
6. Um segundo agente, **Assistente de confirmação**, roda em um
   `scheduleTrigger` diário e usa `MCP Google Calendar.` + `Postgres Chat
   Memory` (sessão fixa `"assistente_confirmacao"`) para lembrar pacientes do
   dia seguinte.

### 1.2 Workflows secundários (JSON inválido — não importáveis)

Todos os outros 5 arquivos **contêm marcadores de conflito de merge Git não
resolvidos** (`<<<<<<< HEAD` / `=======` / `>>>>>>> 724ac98...`), tornando-os
JSON inválido hoje:

| Arquivo | Conflitos | Import no n8n |
|---|---|---|
| `2__MCP_Google_Calendar.json` | 8 blocos | ❌ falha |
| `3__Baixar_e_enviar_arquivo_do_Google_Drive.json` | 6 blocos | ❌ falha |
| `4__Escalar_humano.json` | 15 blocos | ❌ falha |
| `5__Enviar_agendamento.json` | 4 blocos | ❌ falha |
| `6__Assistente_interno.json` | 33 blocos | ❌ falha |
| `Secretaria_IA.json` | 0 | ✅ ok |

O commit `85ecfdb` ("Versão inicial com arquivos atualizados da Marlene") é a
fonte provável desses conflitos, seguido por um merge (`84ea01b`) que não
resolveu os marcadores nos arquivos secundários. Conforme instrução do
projeto, **não vou tentar reconciliar automaticamente essas duas versões** —
elas serão tratadas apenas como referência conceitual (ex.: para entender o
formato esperado de `handoff`, `calendar`, etc.), não como fonte de verdade.

## 2. Problemas encontrados

### 2.1 Segredos e valores sensíveis hardcoded

Nenhuma chave de API literal foi encontrada (credenciais n8n ficam fora do
export), mas há infraestrutura e identificadores fixos no JSON:

- IPs de servidor hardcoded, usados como URL base da API Chatwoot:
  `145.223.26.136:3000` e `191.101.234.244:3000` (em `Secretaria_IA.json`,
  `4__Escalar_humano.json`, `6__Assistente_interno.json`).
- Endpoint MCP hospedado fixo, com UUID de instância embutido:
  `https://rayquazads.app.n8n.cloud/mcp/db6bc79d-ba32-.../sse` (e uma
  variação com typo `rayquazadz`, sugerindo dado copiado/colado
  inconsistentemente entre arquivos).
- ID de voz da ElevenLabs hardcoded no path da URL:
  `.../text-to-speech/33B4UnXyTNbgLmdEDh5P`.
- ID de pasta do Google Drive hardcoded:
  `drive.google.com/drive/folders/1Pu2rkCPFLDCauWoYZVVezBeswFmOtdHi`.
- Webhook IDs fixos por workflow (esperado em exports n8n, mas precisam ser
  regenerados/tratados como config por ambiente, nunca reaproveitados).

**Nenhum desses valores será reutilizado na nova plataforma.** Servem apenas
como exemplo do tipo de acoplamento a evitar (ver Seção 24 do brief original).

### 2.2 Acoplamento excessivo (a razão de existir deste projeto)

- **CRM único e hardcoded**: todo o pipeline assume payload do Chatwoot
  (`$json.body.sender.phone_number`, `$json.body.conversation.id`,
  `$json.body.account.id`) diretamente nos nós de negócio. Não há camada de
  normalização — trocar de CRM exigiria reescrever o workflow inteiro.
- **Provider de LLM único**: apenas `lmChatOpenAi` é usado; não há roteamento,
  fallback ou abstração de modelo.
- **Handoff acoplado ao Chatwoot**: humano-vs-IA é resolvido checando se a
  label `agente-off` está presente no array `etiquetas` vindo do payload —
  regra de negócio universal implementada como detalhe de um CRM específico.
- **Buffer de mensagens chaveado só por telefone**: `Enfileirar mensagem`,
  `Buscar mensagens` e `Limpar fila de mensagens` usam exclusivamente a coluna
  `telefone` como chave. Sem `tenant_id`, dois clientes diferentes com o
  mesmo número de telefone (ou o mesmo número reaproveitado entre tenants)
  colidiriam. Não há chave composta `tenant_id + conversation_id` como o
  projeto exige.
- **Memória sem tenant**: `n8n_historico_mensagens` é uma tabela única,
  `sessionKey = telefone`. Mesmo risco de colisão entre tenants, e nenhuma
  separação entre histórico bruto / resumo / contexto operacional.
- **Conhecimento de negócio dentro do prompt**: o system prompt do agente
  "Secretária" (11k caracteres) contém o nome da clínica ("Clínica João
  Paulo"), SOP detalhado, regras de formatação de telefone, e presumivelmente
  mais adiante preços/políticas — tudo hardcoded no prompt, não em uma camada
  de Knowledge/RAG separada.
- **Sem idempotência explícita fora do buffer**: o mecanismo "mensagem
  encavalada" evita reprocessar mensagens antigas comparando o último
  `id_mensagem` da fila, mas não há proteção contra webhooks duplicados do
  Chatwoot (mesmo `id_mensagem` reentregue), nem chave de idempotência em
  nível de tabela.
- **Sem observabilidade estruturada**: não há logging de eventos
  (`llm_requested`, `tool_failed`, etc.), correlação por `trace_id`, nem
  registro de custo/latência de chamadas de LLM.
- **Estados comerciais implícitos**: não há máquina de estados explícita
  (`NEW`, `QUALIFIED`, `SCHEDULED`, ...); o único estado observável é a label
  do Chatwoot.

### 2.3 Banco de dados existente

Apenas 2 tabelas inferidas pelos workflows (não há migration/DDL no repo):

- `n8n_fila_mensagens` (`id`, `id_mensagem`, `telefone`, `mensagem`,
  `timestamp`) — fila de buffer de mensagens.
- `n8n_historico_mensagens` (`session_id`, `message`, `created_at`, ...) —
  memória de chat no formato esperado pelo node `memoryPostgresChat` do
  n8n (schema fixo da lib, não customizável livremente).

Nenhuma delas tem `tenant_id`, índice documentado, ou separação de
conhecimento/config.

### 2.4 Documentação existente

Só o `README.md` (descreve o produto para um único cliente fictício —
"Marlene" / clínica). Não há ADRs, diagramas ou docs de arquitetura prévios.
Este documento e `docs/DECISIONS.md` são os primeiros.

## 3. Arquitetura proposta (visão macro)

Mantém a separação de responsabilidades pedida no brief, sem nomes de
arquivo obrigatórios:

```
CORE     (independente de canal/CRM/LLM/agenda)
├── Inbound Gateway     — normaliza qualquer payload de canal/CRM em
│                          Universal Message
├── Tenant Resolver     — resolve tenant_id a partir da origem do evento
├── Message Buffer      — agrega mensagens picadas, chave (tenant_id, conversation_id)
├── Context Builder      — monta memória + config + knowledge para o agente
├── Agent Orchestrator   — invoca o LLM Router e as Tools, aplica AI x Humano
├── Output Gateway       — envia resposta de volta pelo canal de origem
└── Logging              — eventos estruturados com trace_id

CRM      (Router + Adapters: Kommo, Chatwoot, DataCry[stub])
LLM      (Router + Adapters: OpenAI, Anthropic, Google + Fallback)
TOOLS    (Calendar, Knowledge, Files, Human Handoff, Tasks — contratos
          validados por schema antes de qualquer execução)
AGENTS   (definições de prompt + tools habilitadas por tenant, não código)
```

Cada camada troca dados apenas através dos contratos definidos nas Seções 5
(Universal Message), 6 (Tenant Config), 9 (CRM contract) e 13 (Tool contract)
do brief original — nunca com o payload nativo de um provider vazando para
fora do respectivo Adapter.

## 4. Estrutura de diretórios proposta

```
/docs                     — ADRs, este plano, decisões
/database
  /migrations             — SQL versionado (tenants, buffer, memória, knowledge...)
  /seeds
/src
  /core                   — Inbound Gateway, Tenant Resolver, Buffer, Context
                             Builder, Agent Orchestrator, Output Gateway, Logging
  /adapters
    /crm                  — KommoAdapter, ChatwootAdapter, DataCryAdapter (stub)
    /llm                  — OpenAI, Anthropic, Google + Router/Fallback
    /channels              — normalização de payloads por canal (se necessário
                             além do que o CRM já resolve)
    /calendar              — GoogleCalendarAdapter
    /storage                — GoogleDriveAdapter
  /services                — regras de aplicação que orquestram adapters
  /schemas                 — JSON Schemas / Zod / Pydantic para validação de
                             tool calls e mensagens universais
  /types
  /utils
/n8n
  /core                    — CORE-00..90 (fluxos finos, delegam lógica ao /src
                             quando possível)
  /crm
  /llm
  /tools
  /agents
  /legacy                  — os 6 arquivos atuais, preservados como referência
                             histórica (não editados in-place)
/scripts
/tests
```

Justificativa: como o repositório hoje é 100% n8n, dois caminhos são
possíveis — (a) manter tudo em n8n com melhor modularização, ou (b) migrar
lógica crítica (buffer, roteamento, validação) para código versionado
(`/src`) e deixar o n8n como camada fina de I/O. Ver `docs/DECISIONS.md` para
a decisão e trade-offs.

## 5. Entidades de banco (rascunho inicial — Fase 1 detalha em migrations)

```
tenants
tenant_channels
tenant_crm_config
tenant_llm_config
tenant_features

contacts
conversations
messages
conversation_state        -- AI_ACTIVE / AI_PAUSED / HUMAN_ACTIVE / CLOSED

message_buffer            -- substitui n8n_fila_mensagens; chave (tenant_id, conversation_id)

agent_sessions
agent_events

llm_models                -- model registry (Seção 12 do brief)
llm_calls

tool_calls

knowledge_documents
knowledge_chunks
```

Todas com `tenant_id` (exceto `tenants` em si) e índices em `tenant_id`,
`conversation_id`, `contact_id`, `created_at`, `status`, conforme exigido.
Detalhamento de colunas fica para a migration real (Fase 1), não para este
documento.

## 6. Contratos principais (referência)

- **Universal Message** — Seção 5 do brief (já reproduzida ali; será
  implementada como schema validado).
- **Tenant Config** — Seção 6 do brief; knowledge (FAQs, preços, etc.)
  explicitamente **fora** deste config.
- **CRM Action contract** (request/response) — Seção 9 do brief.
- **Tool call contract** — `{ tool, action, arguments }` validado por schema
  antes de execução (Seção 13).
- **Handoff tool universal** — `handoff.request { reason, priority, summary }`
  (Seção 8), traduzido por cada CRM Adapter (ex.: Chatwoot → label
  `agente-off`, Kommo → campo customizado, DataCry → TODO).

## 7. Módulos e ordem de dependência

1. `schemas/types` (Universal Message, Tenant Config, Tool contracts) — sem
   dependências, todo o resto depende disso.
2. `database/migrations` — schema Postgres inicial.
3. Adapters base como **interfaces** (CRM, LLM, Calendar, Storage, Channel) —
   sem implementação completa ainda.
4. Core: Tenant Resolver → Message Buffer → Context Builder → Agent
   Orchestrator → Output Gateway.
5. Primeiro adapter concreto de cada tipo (Chatwoot, OpenAI) para viabilizar
   o fluxo ponta-a-ponta da Fase 2.

## 8. Fases de migração

Reaproveitando exatamente a sequência do brief (Seção 23):

- **Fase 1 — Fundação**: schema Postgres, migrations, Tenant Config, tipos/
  contratos, interfaces base de adapters. (Este documento + `DECISIONS.md`
  são o passo 1-4 desta fase.)
- **Fase 2 — Primeiro fluxo funcional**: Chatwoot → Inbound Gateway → Tenant
  Resolver → Message Buffer → Context Builder → LLM Router → Agent Core →
  Chatwoot. Sem Calendar, sem RAG.
- **Fase 3**: Human Handoff universal.
- **Fase 4**: Calendar (multi-provider, IDs de agenda vindos de config, não
  do prompt).
- **Fase 5**: Knowledge/RAG (Postgres + pgvector como padrão, sem exigir
  Supabase).
- **Fase 6**: Kommo Adapter.
- **Fase 7**: DataCry Adapter (stub até haver documentação da API).

## 9. Riscos

- **Ambiguidade entre "core em código" vs. "core em n8n"**: decide o quanto
  do buffer/roteamento pode viver em nós n8n padrão vs. precisa de
  código customizado (Function/Code nodes ou um serviço HTTP externo
  chamado pelo n8n). Impacta diretamente testabilidade e portabilidade.
  Ver `DECISIONS.md` (pendente de confirmação do usuário).
- **DataCry**: nenhuma documentação de API disponível ainda. Qualquer
  adapter será stub/interface com TODOs — risco de retrabalho quando a API
  real for conhecida.
- **Reaproveitamento indevido de infraestrutura antiga**: os IPs, endpoint
  MCP e IDs listados na Seção 2.1 pertencem ao ambiente do cliente de
  referência ("Marlene"/clínica) e não devem vazar para a plataforma nova
  nem para exemplos/seeds — risco de contaminação se alguém copiar trechos
  dos workflows legados sem revisar.
- **Arquivos legados quebrados**: 5 dos 6 workflows não importam no n8n hoje
  (conflitos de merge). Se alguém tentar "consertá-los" automaticamente
  escolhendo um lado do conflito às cegas, corre o risco de reintroduzir
  lógica incompatível com a versão nova da arquitetura. Tratamento: mover
  para `/n8n/legacy` sem alterar conteúdo, e extrair conceitos manualmente
  quando necessário.
- **pgvector vs. outra solução de vetor**: brief pede preferência por
  pgvector mas não obrigatoriedade — precisa validação de que o Postgres do
  ambiente-alvo suporta a extensão antes de comprometer a Fase 5.
- **Model Registry vs. simplicidade inicial**: uma tabela `llm_models` completa
  (Seção 12) é mais trabalho de fundação do que o necessário só para a Fase
  2 rodar. Risco de over-engineering se implementada 100% antes do fluxo
  básico funcionar.

## 10. Decisões ainda pendentes

Ver `docs/DECISIONS.md` para as decisões já tomadas. Pendentes de decisão do
usuário antes de avançar:

1. **Core em n8n vs. core em serviço externo (código) orquestrado pelo n8n.**
   Trade-off principal: n8n puro é mais aderente ao ambiente atual do
   usuário e não exige deploy de novo serviço, mas dificulta testes
   automatizados, versionamento de lógica complexa (buffer com concorrência,
   LLM Router com fallback) e reuso de código entre workflows. Um serviço
   HTTP em `/src` chamado por nós `HTTP Request` do n8n resolve isso mas
   introduz um novo componente de infraestrutura para hospedar/operar.
2. **Linguagem/runtime do `/src`** (caso a decisão acima aponte para um
   serviço externo) — ex. TypeScript/Node (mais próximo do ecossistema n8n)
   vs. Python (mais comum para RAG/embeddings). Sem preferência declarada
   pelo usuário ainda.
3. **Qual CRM vira o "primeiro adapter" de fato na Fase 2** — o brief usa
   Chatwoot no exemplo da Fase 2, mas o tenant real de referência também
   usa Kommo; confirmar que Chatwoot é o ponto de partida correto.
4. **Onde hospedar o Model Registry inicialmente** — tabela Postgres completa
   (Seção 12) já na Fase 1, ou uma versão mínima (provider+model+enabled) até
   a Fase 2 estar validada, expandindo depois?
5. **Escopo exato do DataCry stub** — apenas a interface `CrmAdapter` sem
   nenhum método implementado, ou stubs que retornam `NotImplementedError`
   explícito por ação, com TODOs individuais? (Recomendação será feita
   quando a interface `CrmAdapter` for desenhada na Fase 1.)
