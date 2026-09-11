# Architecture Decisions

Registro de decisões já tomadas para a plataforma de Agentes de IA
multi-tenant. Decisões em aberto (que dependem de escolha do usuário) ficam
listadas na Seção 10 de `docs/ARCHITECTURE_PLAN.md`, não aqui.

Formato: cada entrada é uma mini-ADR (contexto, decisão, consequência).

---

## D001 — Multi-tenant é requisito desde o primeiro commit de schema

**Contexto**: os workflows de referência ("Secretária IA") foram construídos
para um único cliente, com o buffer de mensagens e a memória chaveados
apenas por `telefone`.

**Decisão**: toda tabela de negócio (exceto `tenants`) carrega `tenant_id`
desde a primeira migration. Toda chave de agregação (buffer, memória,
sessão de agente) usa no mínimo `tenant_id + conversation_id`, nunca só o
telefone/contato.

**Consequência**: mais uma coluna e mais um índice em quase toda tabela
desde o dia 1, mas elimina a classe inteira de bugs de colisão entre
clientes observada no projeto de referência.

---

## D002 — Payloads nativos de canal/CRM nunca cruzam a fronteira do Adapter

**Contexto**: no workflow de referência, o node `Info` lê diretamente
`$json.body.sender.phone_number`, `$json.body.conversation.id`, etc., e essas
expressões se espalham por outros nós do fluxo (agente, memória, envio).
Trocar de CRM exigiria reescrever tudo.

**Decisão**: qualquer payload recebido é normalizado para o Universal
Message (Seção 5 do brief) dentro do Inbound Gateway / adapter de canal.
Nenhum código de Core, Agent ou Tool deve referenciar um campo específico de
Chatwoot, Kommo ou DataCry.

**Consequência**: uma camada extra de mapeamento por provider, mas o Core e
os Agents ficam 100% portáveis entre tenants com CRMs diferentes.

---

## D003 — Estado IA×Humano é modelado no Core, não no CRM

**Contexto**: hoje "humano assumiu a conversa" é só a presença da label
`agente-off` num array vindo do Chatwoot — uma regra de negócio universal
implementada como detalhe de um provider específico.

**Decisão**: existe um estado interno explícito (`AI_ACTIVE`, `AI_PAUSED`,
`HUMAN_ACTIVE`, `CLOSED`) persistido pelo Core. Cada CRM Adapter traduz esse
estado de/para sua própria representação (label, campo customizado, etc.).
O Agent Orchestrator só decide responder automaticamente quando o estado
interno é `AI_ACTIVE` — nunca inspecionando labels/campos do CRM
diretamente.

**Consequência**: uma tabela/coluna `conversation_state` a mais, mas o
handoff funciona igual independentemente do CRM do tenant.

---

## D004 — Conhecimento de negócio não vive no prompt

**Contexto**: o system prompt do agente "Secretária" no workflow de
referência tem ~11 mil caracteres e contém nome da clínica, SOP e política
de atendimento específicos daquele cliente, hardcoded.

**Decisão**: prompts de agente descrevem *comportamento e papel* (é um
artefato de configuração do tipo de agente, reutilizável entre tenants do
mesmo tipo). Dados como nome da empresa, preços, profissionais, políticas e
FAQs vivem na camada de Knowledge (`knowledge_documents`/`knowledge_chunks`,
Seção 16) e/ou no Tenant Config (Seção 6), nunca escritos diretamente no
texto do prompt de um tenant específico.

**Consequência**: o mesmo prompt-base de "Agente Secretária" serve qualquer
tenant desse tipo; personalização por cliente entra via RAG/config, não via
edição de prompt.

---

## D005 — Nenhum valor do ambiente de referência é reaproveitado

**Contexto**: os workflows legados contêm IPs (`145.223.26.136`,
`191.101.234.244`), um endpoint MCP com UUID fixo
(`rayquazads.app.n8n.cloud/mcp/db6bc79d-...`), um ID de voz ElevenLabs, e um
ID de pasta do Google Drive, todos hardcoded.

**Decisão**: nenhum desses valores é copiado para a plataforma nova, nem
como exemplo em seeds/fixtures. Toda URL, ID e credencial vem de variável de
ambiente ou de config por tenant. Um `.env.example` será criado sem valores
reais quando a Fase 1 chegar em configuração de runtime.

**Consequência**: nenhuma, além da disciplina de sempre revisar trechos
copiados dos workflows legados antes de reaproveitá-los conceitualmente.

---

## D006 — Arquivos legados com conflito de merge não são "corrigidos" às cegas

**Contexto**: `2__MCP_Google_Calendar.json`, `3__Baixar_e_enviar_arquivo_do_Google_Drive.json`,
`4__Escalar_humano.json`, `5__Enviar_agendamento.json` e
`6__Assistente_interno.json` têm marcadores de conflito Git não resolvidos
(`<<<<<<<`/`=======`/`>>>>>>>`) e não são JSON válido hoje.

**Decisão**: esses arquivos não serão editados para "resolver" o conflito
escolhendo um lado arbitrariamente. Eles são preservados como estão (serão
movidos para `/n8n/legacy` na Fase 1, sem alteração de conteúdo) e usados
apenas como referência conceitual manual, nunca importados ou executados.
`Secretaria_IA.json` (o único arquivo válido) é a única referência
executável do comportamento legado.

**Consequência**: nenhuma extração automática de lógica desses 5 arquivos;
qualquer conceito útil neles precisa ser lido e reimplementado manualmente
com revisão humana.

---

## D007 — Postgres + pgvector é o backend de conhecimento padrão; Supabase não é obrigatório

**Contexto**: o projeto de referência "Agente de Vendas" usa
Supabase/vector store; o brief pede para não tornar isso obrigatório.

**Decisão**: a camada de Knowledge assume PostgreSQL com a extensão
`pgvector` como padrão, já que o Core inteiro já depende de PostgreSQL para
tenants/mensagens/memória. Supabase pode ser usado por um tenant específico
apenas se necessário, através do mesmo `StorageAdapter`/`KnowledgeAdapter`
que qualquer outro backend de vetor implementaria.

**Consequência**: um único banco relacional para operar em produção
(menos peças móveis), condicionado a validar que o Postgres do ambiente-alvo
suporta `pgvector` (risco listado em `ARCHITECTURE_PLAN.md`).

---

## D008 — Model Registry separa LLM de Embeddings

**Contexto**: o brief exige suportar, por exemplo, LLM = Anthropic e
Embeddings = OpenAI simultaneamente.

**Decisão**: o Model Registry (Seção 12) trata "provider de chat/completions"
e "provider de embeddings" como dimensões independentes — um tenant escolhe
um profile de LLM (`FAST`/`STANDARD`/`ADVANCED`/...) e, separadamente, um
provider de embeddings para a camada de Knowledge.

**Consequência**: o LLM Router e o Embeddings Provider são componentes
distintos desde o desenho inicial, mesmo que inicialmente só um provider de
embeddings esteja implementado.

---

## D009 — Fases seguem exatamente a ordem proposta no brief, sem pular etapas

**Contexto**: o brief lista Fases 1–7 (Fundação → primeiro fluxo funcional →
Handoff → Calendar → Knowledge/RAG → Kommo → DataCry).

**Decisão**: essa ordem é adotada como está. Calendar, RAG e integrações de
CRM adicionais só entram depois de existir um fluxo ponta-a-ponta simples
(Chatwoot → Core → LLM → Chatwoot) funcionando e validado.

**Consequência**: nenhuma feature "adiantada" (ex.: não implementar Calendar
antes de o buffer/handoff básico estarem sólidos), mesmo que pareça
conveniente implementar tudo junto.

---

## D010 — Este documento e o plano são o único entregável desta primeira tarefa

**Contexto**: a tarefa 25 do brief pede explicitamente para não avançar para
implementação grande antes de aprovação.

**Decisão**: nenhum código, migration, ou estrutura de diretório nova
(`/src`, `/database`, etc.) foi criado nesta etapa — apenas
`docs/ARCHITECTURE_PLAN.md` e este arquivo. A criação da estrutura real do
projeto aguarda aprovação explícita do usuário sobre os pontos em aberto
(Seção 10 do plano).

**Consequência**: nenhuma mudança de código para revisar ainda; o próximo
passo é uma decisão do usuário, não mais análise.
