# Handover do Motor de Disparos, para o Luiz

> Documento de passagem de bastão. Estado atual do Motor de Disparos de Marketing da Seazone e as próximas frentes.
> Perfil do handover: híbrido (código e operação). Verificação de estado feita ao vivo em 28/07/2026.
> Como o Luiz já é dono da SAI, este documento não explica a SAI. Ele descreve o Motor e a fronteira Motor mais SAI vista do lado do Motor.
> Atualizado em 30/07/2026 com o que saiu da apresentação de funcionalidades do Motor (call das 10h, com Marketing, Expansão, Parcerias e Marketplace e SZS). A Karen confirmou na call que o projeto passa a ser tocado pelo Luiz. Os pedidos e as decisões dessa apresentação estão na seção 1.3.

Referências complementares no repositório: `RESUMO.md` (estado e próximos passos), `memory/MEMORY.md` (índice da memória durável), `BACKLOG.md` (fila única de tasks), `CONTEXTO_DISPAROS.md` (histórico), `DEFINICOES_PARAMETRIZAVEIS.md` (temperatura por vertical), `db/schema.sql` (banco).

**Acessos (EKS):** produção `https://motor-disparos.seazone.dev` · staging `https://stg-motor-disparos.seazone.dev`.

---

## 1. Próximos passos

Separei em duas categorias: **decisões de infra** (escolhas de arquitetura e plataforma, geralmente cross-team, que precisam ser decididas antes de virar código) e **tasks** (trabalho concreto e escopado). Cada item tem o porquê, o que já existe e o primeiro passo.

### 1.1 Decisões de infra

#### D1 (prioridade nº 1). Trocar a base pela Base Unificada de Clientes (BUC) da Nargylla
- **Por quê**: hoje o Motor lê da camada silver da Nekt (`motor_leads_unificados`, gerada por transformações que já carregaram um de-para pipeline para vertical errado). A BUC, projeto da Nargylla Cloviel (Time de Dados), unifica Pipedrive, Sapron e outros numa camada gold canônica, com papéis, métricas e deals por pessoa. Migrar para ela dá uma fonte única, mais rica e correta.
- **O que já existe**: a BUC em produção na Nekt, schema `nekt_operacional_gold`. `gold_clientes` (cerca de 522 mil pessoas, uma linha por pessoa, status por papel: proprietário, investidor, hóspede, franqueado, parceiro, decor), `gold_metricas` (agregado de hóspede, proprietário e comercial por pessoa) e `gold_papeis` (deal a deal). O de-para "campos do Motor para colunas da BUC" já está em andamento com a Nargylla. Material em `TABELAS_NEKT_MOTOR_DISPAROS.md`. Tarefa T051 (relatório de gap).
- **Onde mexe no Motor**: a camada de sync que popula `staging.lead_unificado` (`nekt_api_client.py`, `nekt_deal_sync.py`, `sync_full.py`), o de-para de todos os campos que o Motor usa hoje (vertical, temperatura e scoring, exclusões como `tipo_de_venda`/parceiro, `valor_capital_declarado`, `rd_source`, `base_origem`, hóspede e spot, `deal_aberto_vertical`) e o `criteria_builder` caso os nomes de coluna mudem. É a dependência mais central do Motor, tudo se apoia na base.
- **Riscos**: (1) a BUC é atualizada de forma **manual, não em tempo real**, então tem atraso de horas contra Pipedrive e Sapron, o que é ruim para disparo sensível a timing (avaliar antes de migrar a automação por marco de obra). (2) Não herdar o de-para pipeline para vertical errado, alinhar o canônico dentro do projeto da Nargylla (ver `PLANO_FIX_DEPARA_VERTICAL.md`). (3) Por ser a base de tudo, a migração precisa ser faseada e com paridade validada (contagem por vertical e por temperatura antes e depois).
- **Primeiro passo**: fechar o relatório de gap de campos com a Nargylla (T051), validar a paridade de volume por vertical, e migrar primeiro em staging lendo da gold em paralelo à silver (leitura dupla) antes de cortar a fonte antiga.
- **Contato**: Nargylla Cloviel, Time de Dados (Slack DM `D0BGL7EPKHV`).

#### D2. Login único com a SAI (SSO)
- **Por quê**: o Luiz controla as duas pontas, então dá para unificar a autenticação do Motor com a da SAI.
- **O que já existe**: o Motor usa Google OAuth mais JWT próprio, e roda embutido em iframe dentro da SAI.
- **Decisão e esboço**: SSO ou token compartilhado. A SAI passa a emitir ou validar o token que o Motor consome, ou os dois usam um provedor de identidade comum. Some o login duplicado dentro do iframe.
- **Primeiro passo**: definir o provedor de identidade comum e o contrato de token entre SAI e Motor.

#### D3. Consolidar o data plane pós-migração
- **Por quê**: antes havia dois deploys no Coolify e o banco de produção era um Postgres interno do Coolify, separado do projeto Supabase. A migração pro EKS de 30/07 resolveu a maior parte: prod e staging num só cluster, banco no Supabase `wqvgralwoymmtkbeswqh`.
- **O que falta**: fechar a validação do cutover (ver D4), fixar a fonte de verdade agora que o banco é o Supabase, e aposentar o que sobrou do Coolify quando a Kakau validar.
- **Primeiro passo**: concluir a validação e documentar o data plane novo.

#### D4. Fechar a validação da migração pro EKS (feita em 30/07)
- **O que aconteceu**: o Motor saiu do Coolify e foi pro EKS (cluster Seazone) com banco no Supabase, migração concluída na madrugada de 30/07 pela plataforma. Prod e staging já rodam no cluster com o dado migrado. O Coolify ficou desligado como rollback até a Kakau confirmar.
- **O que falta para o cutover**: (1) reapontar os webhooks da Morada, SIA e Meta para `motor-disparos-api.seazone.dev` (era o endereço antigo do Coolify, o mais urgente, pode já estar quebrado), (2) um disparo real de validação do caminho de envio no ambiente novo, feito por alguém do time com autorização, nunca automático.
- **Deploy hoje**: por merge e a esteira de CI (ArgoCD). O MCP do EKS opera (status, logs, restart, rollback, env) mas não faz deploy.

### 1.2 Expansão e Parcerias

É a frente com mais coisa agora. Junta o teste do Motor fora do marketing interno com os pedidos de Parcerias e Marketplace (Caio, Igor, Gustavo) que saíram da apresentação de 30/07. Organizada em testes e refinamento.

**Testes**

#### Testar o Motor em expansão e parcerias
- **Por quê**: validar o Motor além do marketing interno, cobrindo expansão de portfólio e parceiros externos.
- **O que já existe**: o mundo Expansão (T158), campanha externa por upload, os gates de acesso de Parcerias e `base_externa`, e o cross-sell por `base_origem`.
- **Gap conhecido**: no deal que o Motor manda para a SAI faltam três campos (`nome_condominio`, `org_id`, `lead_reativado`). O fix é só no Motor.
- **Primeiro passo**: rodar uma campanha piloto de expansão e uma de um parceiro externo em staging, conferindo o deal criado na SAI com os três campos.

#### Suprimir o deal automático nos disparos de parceiros (cross-team Motor mais SAI)
- **Por quê**: o time de Parcerias dispara para mais de 4.500 parceiros com frequência (cota de um empreendimento, pedido de indicação) e não quer um deal criado a cada disparo. Hoje eles rodam pela SIA e o Gustavo cria o card à mão só quando a conversa evolui.
- **Opções levantadas na call**: (1) um funil "fake" no Pipedrive que absorve os cards, com o Gustavo movendo à mão os que avançam para o funil real; (2) um parâmetro que suprime a criação do card para esse tipo de disparo, que precisa de desenvolvimento nos dois lados, Motor e SAI.
- **Encaminhamento**: o Luiz propôs uma call separada e o Caio pediu um mini discovery. É a decisão de fronteira Motor mais SAI mais direta que saiu da apresentação. Owners do lado do negócio: Caio Panissi e Igor Medeiros (Parcerias e Marketplace), Gustavo Sicuti (opera a prospecção de parceiros pela SIA).
- **Primeiro passo**: a call de discovery para decidir entre funil fake e parâmetro de supressão, e escopar o que muda em cada lado.

#### Vincular Google Sheets na base externa
- **Por quê**: hoje a base externa exige subir a planilha à mão. O time de Parcerias quer vincular direto com o Google Sheets.
- **Primeiro passo**: avaliar a integração com Google Sheets como fonte da base externa.

**Refinamento (cadência, audiência, exclusões)**

#### Cadência multicanal (WhatsApp mais email na mesma régua)
- **Por quê**: pedido do Roberto, para nutrição em que um canal complementa o outro. Hoje não dá para misturar WhatsApp e email na mesma régua.
- **Workaround atual**: duas campanhas intercalando os toques, com o email caindo no meio da janela do WhatsApp.
- **Primeiro passo**: mapear a régua multicanal, um canal por passo.

#### Fatiar a base externa (upload) em lotes de tamanho fixo
- **Por quê**: pela SIA o time dispara de 1.000 em 1.000 para não queimar a qualidade do número, e quer o mesmo controle ao subir uma planilha. Hoje o upload não fatia, então a base precisa vir já segmentada.
- **Primeiro passo**: portar o limite e o fatiamento por lote para o fluxo de upload.

#### Exclusões na base externa: manter (decisão)
- Na demo a Karen ia corrigir o upload que aplicava as exclusões, mas o time (Mari, com o aval do Caio) decidiu mantê-las: faz sentido não reimpactar quem recebeu um toque recente e respeitar a regra de parceiro só receber de parceiro. Fica como comportamento desejado.

### 1.3 Outras melhorias

#### Insights: filtro de gestão e separar SZS VD de SZS B2B (pedido da Gaby, T250)
- **Por quê**: no `/insights` a Gaby precisa ver SZS quebrado entre B2B (expansão) e VD (venda direta), e um filtro de gestão.
- **Regra do split** (usar os 4 conceitos porque o pipe é bagunçado, então é OU):
  - **SZS B2B** = nome do condomínio preenchido, OU canal = Expansão, OU etiqueta = B2B, OU RD [Campanha] contém EXP / EXPANSÃO / EXPANSAO.
  - **SZS VD** = todo o resto.
- **Também**: um filtro de gestão (`is_gestao`) no painel.
- **Estado**: já é a task **T250**, com investigação concluída (`PLANO_INSIGHTS_SZS_VD_B2B.md`), falta implementar. Viável por JOIN no `funil-canal` via `deal_id` para `lead_deal` (condomínio e canal) mais `rd_campanha ILIKE '%EXPANS%'`.
- **Gap conhecido**: `etiqueta=B2B` ainda não tem fonte no banco atual (pular na v1 ou mapear a origem antes). A migração para a BUC (D1) tende a resolver a disponibilidade desses campos de uma vez.
- **Primeiro passo**: implementar os 3 conceitos que já têm fonte e decidir o que fazer com `etiqueta=B2B`.

#### Automação por marco de obra (SpotSys)
- **Por quê**: gerar campanhas automáticas quando um marco de obra avança no SpotSys. É o gatilho natural de SZI.
- **O que já existe**: `staging.marco_obra`, `ver_marcos_obra`, `marcar_marco_comunicado`, o sync de Smartsheet, o alvo de fonte `api_spotsys` e o tipo de campanha gatilho já previsto no schema.
- **Esboço técnico**: um job de cron lê marcos novos ou avançados, aplica um de-para de marco para comunicação, cria e agenda uma campanha gatilho, e marca o marco como comunicado para garantir idempotência. Reaproveita o executor e a régua que já existem.
- **Depende de**: a decisão D1 (BUC), porque a BUC não é tempo real e o gatilho por marco é sensível a timing.
- **Primeiro passo**: definir o de-para marco para comunicação e o job que detecta o delta de marco.

#### Teste A/B mais visual
- **Por quê**: facilitar a leitura do comparativo entre comunicações diferentes.
- **O que já existe**: a tabela `main.campanha_teste_ab` (hoje vazia) e os analytics de template (`ver_analytics_templates`, `ver_efetividade_templates`).
- **Esboço técnico**: tela comparativa lado a lado (variante A versus B) com métricas por variante (entregue, lido, respondido, conversão), split de audiência e destaque do vencedor.
- **Primeiro passo**: popular `campanha_teste_ab` num piloto e montar a tela side-by-side.

#### Volume e base de conhecimento do que converte
- **Por quê**: acumular volume de campanhas para aprender quais hooks, timings e comunicações convertem mais.
- **O que já existe**: `/insights` já mede origem marketing, funil, campeã, além de `ver_efetividade_templates`, `ver_analytics_whatsapp` e o carimbo de entrega por lead (T252).
- **Esboço técnico**: rodar mais campanhas com marcação consistente (hook, timing, template), acumular histórico e montar um painel de aprendizado (conversão por hook, por horário e por vertical) que sirva de recomendação.
- **Primeiro passo**: criar uma taxonomia de hook e timing nos metadados da campanha, e um painel comparativo histórico.

#### Melhorar as notificações
- **O que já existe**: avisos in-app e no Slack, configuráveis na tela, com um mapa de notificações e os gaps já levantados (T268 a T270).
- **Gap conhecido**: quando um disparo falha 100%, o aviso pode silenciar em vez de gritar, justo quando mais importa.
- **Primeiro passo**: fechar os gaps mapeados e garantir que a falha total sempre notifica.

#### Melhorar a tela de detalhes da campanha
- **O que já existe**: o detalhe mostra status, audiência, disparos, passos da cadência, ocorrências e stats, além do deal e da conversa por disparo.
- **Gap**: a tela cresceu em pedaços e pede uma passada de organização e clareza para quem abre uma campanha.
- **Primeiro passo**: levantar com a Gaby o que ela mais procura ali e reorganizar em cima disso.

#### Playbook por funcionalidade (expandir o "Sobre")
- **Por quê**: pedido do Roberto, um lugar de consulta das funcionalidades e do como fazer. A seção "Sobre" existe mas não cobre funcionalidade por funcionalidade.
- **Primeiro passo**: expandir o "Sobre" com um passo a passo por funcionalidade. A Karen avaliou como esforço baixo.

---

## 2. Pendências e riscos abertos

- **Performance**: preview de audiência estoura `statement_timeout` no caso SZS amplo. Falta materializar uma vez.
- **Cutover EKS (30/07)**: migração pro EKS em validação. Reapontar os webhooks Morada/SIA/Meta pro endereço novo `motor-disparos-api.seazone.dev` (urgente) e fazer o disparo real de validação. O Coolify segue parado como rollback até a Kakau confirmar.
- **Data plane local**: depois da migração, o banco de produção é o Supabase. Confirmar a que ambiente a leitura local (REST e MCP) aponta antes de concluir volumetria.
- **RBAC (trava de disparo)**: na apresentação de 30/07 o Roberto viu campos de coordenador abertos ao acessar. O fluxo esperado é entrar como visualizador e um admin aprovar o acesso solicitado, e só a Gaby (e a Karen) disparam no marketing de base interna. Vale um double-check dessa trava para garantir que ninguém dispara sem querer.

---

## 3. Arquivos-chave do backend

Os arquivos para ter na cabeça ao pegar o código:
- `backend/app/main.py`: bootstrap, schedulers (cron, executor, sync).
- `backend/app/services/executor_service.py`: lógica de disparo (SAI e Morada), preparação, retries.
- `backend/app/services/regua_engine.py`: avanço de cadência.
- `backend/app/services/_sai_client.py`: integração com a SAI.
- `backend/app/services/reconcile_sai_service.py`: paridade de status contra a SAI.
- `backend/app/services/erro_wa_map.py`: mapa de erros da Meta.
- `backend/app/api/v1/campanhas.py`: endpoints de campanha.
- `db/schema.sql` e `db/seed.sql`: estrutura de dados.

---

## 4. O que é o Motor, em uma frase

O Motor segmenta leads, orquestra cadências de comunicação (WhatsApp e email) e mede resultado por vertical. Ele **não executa envios**. Gera ordens de disparo e delega a execução para a SAI, para a Morada (MIA) e para o email. A regra de ouro do projeto é ZERO disparos sem autorização explícita.

---

## 5. O que o Motor FAZ e o que NÃO faz

**Faz**
- Segmenta a base por vertical, subtipo, temperatura, origem e filtros flexíveis.
- Monta a audiência de uma campanha, com preview e checagem de conflito e cooldown.
- Cria campanhas de disparo único (blast) e cadências multi passo (régua).
- Gera e gerencia templates, incluindo submissão para aprovação na Meta.
- Aplica exclusões, opt-outs e cooldown antes de liberar qualquer lead.
- Emite a ordem de disparo para a SAI ou para a Morada, com os parâmetros de roteamento.
- Reconcilia o status real dos disparos contra a SAI (não confia só no otimismo do envio).
- Mede o funil por frente em `/insights` (entregue, lido, respondido, conversão, campeã).
- Notifica no Slack, de forma configurável pela própria tela.

**Não faz**
- Não envia mensagem. Quem envia é a SAI, a Morada ou o provedor de email.
- Não decide sozinho disparar em produção. Todo envio real depende de autorização.
- Não guarda variável por lead no envio via SAI. A SAI recebe só nome, telefone e email.
- Não é a fonte da verdade dos dados de lead. As bases vêm da Nekt, da Sapron e de upload externo.
- Não resolve as barreiras de canal da SIA (rate limit, WAF). Isso é do lado da SIA, hoje território do Luiz.

---

## 6. Tipos de disparo

| Tipo | O que é | Como funciona no Motor |
|------|---------|------------------------|
| **Blast** | Uma mensagem pontual para uma audiência | `tipo=blast`. Executor lança tudo de uma vez, a SAI fatia em lotes de 40. |
| **Sequência (régua/cadência)** | N passos ao longo do tempo | `tipo=sequencia` com `regua_id`. Timing por data de referência (D-60, D-30, D-0) ou por delay (1x por semana). O `regua_engine` avança passo a passo. |
| **Gatilho** _(em breve)_ | Disparo disparado por um evento | **Ainda não no ar.** Previsto no schema, é a base da automação por marco de obra (ver os próximos passos). |
| **Recorrente** _(em breve)_ | Disparo que se repete | **Ainda não no ar.** Previsto no schema. |
| **Modo teste** | Qualquer campanha marcada como teste | `is_teste=true`. Não conta em `disparos_count`. Em staging, só o subtipo de teste pode disparar. |

Detalhe operacional da cadência: o botão "Disparar" solta o passo 1 na hora (idempotente). Os passos seguintes avançam pelo cron das 08h, ou manualmente via `POST /reguas/{id}/avancar`.

---

## 7. Perfis de lead

**Verticais**: SZI (investimento), SZS (serviços), MKP (marketplace). Decor aparece como pipeline no dado.

**Subtipos**: 30 subtipos cadastrados (13 SZI, 9 SZS, 8 MKP). Verificado ao vivo em 28/07. Configuráveis na tela de subtipos.

**Temperatura**: quente, morno, frio. As regras por vertical estão em `DEFINICOES_PARAMETRIZAVEIS.md`. O scoring tem 21 regras cadastradas (`lead_score_config`), com ranking em `v_lead_score_ranking`.

**Origem (cross-sell)**: o filtro `base_origem` separa a base de ORIGEM da vertical de DESTINO. Permite disparar para uma base de uma vertical vendendo outra. A base de hóspedes entra pelo sentinela `base_origem="hospede"`.

Observação: os perfis de USUÁRIO do sistema (papel por área, RBAC) são outra coisa. Ficam na tela de usuários e controlam o acesso às seções (Marketing, Expansão, Parcerias).

---

## 8. Bases usadas

| Base | O que traz | Onde entra |
|------|-----------|------------|
| **Base de leads atual** | De onde a audiência sai hoje: `staging.lead_unificado`, populada da camada **silver** da Nekt (`motor_leads_unificados`). Em produção são **279.239 leads** no total, **124.324 elegíveis** (ao vivo, 28/07). O store Supabase sincronizado tem 171.504. Não confundir com a BUC, que ainda **não está em uso** (é a Frente 1). | Origem principal da audiência. Vem da Nekt (camada silver). |
| **Nekt (Pipedrive)** | Leads e deals, cerca de 221 mil deals. | Fonte de dados via Nekt Data API. Sync diário às 21h. |
| **Sapron** | Relação imóvel para proprietário e base de hóspedes. | Sync diário às 21h. |
| **Base externa (upload)** | Bases de parceiros e listas externas. | Mundo Expansão e Parcerias, por upload. Alimenta a frente de parcerias. |
| **SpotSys / Spotômetro** | Empreendimentos e marcos de obra. | `staging.marco_obra`. Alimenta a frente de automação por marco de obra. |
| **Smartsheet** | Marcos e cobertura. | `sync_smartsheet`. |

---

## 9. Infra usada

**Repositório**: monorepo `sz-disparos-mkt`, dentro deste projeto.
- `frontend/`: Next.js 16, React 19, Tailwind 4, shadcn/ui.
- `backend/`: FastAPI, Python 3.12, SQLAlchemy async, asyncpg, Pydantic v2.
- `db/`: `schema.sql` (2 schemas), `seed.sql`, migrations.
- `mcp/`: servidor MCP com as ferramentas operacionais (17 tools) que os operadores usam pelo Claude.

**Banco (Supabase Postgres, projeto `wqvgralwoymmtkbeswqh`)**, dois schemas por camada:
- `staging`: camada de dados de lead. Tabelas `lead_unificado`, `lead_deal`, `marco_obra` e a view `v_leads_elegiveis`.
- `main`: camada operacional do Motor. `campanha`, `campanha_passo`, `disparo`, `template`, `exclusao`, `subtipo_lead`, `lead_score_config`, `comunicacao`, `variavel_template`, `cron_log`, `audit_log`, `metrica_disparo`, entre outras. Views `v_dashboard_campanhas`, `v_leads_em_cooldown`, `v_lead_score_ranking`, `v_leads_por_subtipo`.
- Acesso via REST API (`/rest/v1/`) com header `Accept-Profile`, porque o host direto só resolve IPv6 e a WSL não alcança.

**Deploy (EKS, migrado do Coolify em 30/07/2026)**: prod e staging rodam no cluster EKS da Seazone. A migração foi feita pela plataforma (And, a pedido do Bill) na madrugada de 30/07, com todo o dado migrado. O Coolify ficou desligado, mantido de pé como rollback até a Kakau confirmar o cutover.
- Endereços: prod `motor-disparos.seazone.dev` (front) e `motor-disparos-api.seazone.dev` (API), staging `stg-motor-disparos.seazone.dev`. Atenção: prod **envia real** (`DISPATCH_ENABLED=true`), então teste só no subtipo de teste.
- Banco: o mesmo projeto Supabase `wqvgralwoymmtkbeswqh` (us-east-1) que a app já usava. O Postgres interno do Coolify saiu de cena. Trade-off consciente: latência cross-region de cerca de 120ms por query (o EKS está em sa-east-1), a monitorar no preview de audiência pesado.
- Deploy: por merge e a esteira de CI da Seazone (ArgoCD). O MCP do EKS opera (status, logs, restart, rollback, env por SSM), mas **não faz deploy**. Promoção para produção segue por trem (lote staging para main), com cherry-pick só dos nossos commits, nunca mergeando o trem inteiro do staging.
- Containers: backend (python 3.12-slim, uvicorn na 8000), frontend na 3000, healthcheck em `/health`.

**Autenticação**: Google OAuth restrito a `@seazone.com.br`, com JWT próprio.

**Schedulers (cron)** no `backend/app/main.py`:
- Sync diário às 21h BRT: leads da Nekt e hóspedes da Sapron.
- Executor a cada 5 min: lança as campanhas agendadas.
- Status sync a cada 30s: reconcilia status real dos disparos contra a SAI.
- Avanço de régua às 08h BRT: move os leads pelos passos da cadência.

**Ambiente de desenvolvimento**: WSL com cerca de 4GB de RAM. Há proteções instaladas (earlyoom, swap, hook de mem-guard, statusline com RAM livre) por causa de um travamento por falta de memória no passado. Sessão curta por task.

**Onde o Motor aparece pro usuário**: ele roda dentro da SAI, embutido em iframe (moldura fina).

---

## 10. Fronteira Motor mais SAI (a parte que interessa pro Luiz)

O Motor conversa com a SAI pelo cliente `backend/app/services/_sai_client.py`.

**Chamada principal**: `criar_campanha_sai()` faz `POST /api/campaigns` na SAI (`SAI_API_URL = https://sai.seazone.dev`). A SAI responde com um `task_id`. O Motor faz polling em `_poll_task()` até `status=done` (limite de 150s).

**Contatos que o Motor manda**: apenas `{name, phone_number, email}`. Sem variável por lead.

**Parâmetros da ordem de disparo**:

| Parâmetro | Para que serve |
|-----------|----------------|
| `agente` | Instance ID da MIA. Obrigatório. |
| `empreendimento` | Resolve o `product_id` na MIA. Obrigatório. |
| `message_template` | Nome do HSM aprovado na Meta. |
| `dispatch_channel` | Vazio significa Morada default. `'sia'` significa rota pela SIA (cadence-dispatch). |
| `rd_source` | Campo `[RD] Source` do deal. Vai como `mia_source` no dispatch da MIA. |
| `util_template` | Fallback para os templates de teto da Meta (131047 e 131049). |
| `theme_slug` | Tema SZS, repassado no dispatch SIA para rotear o fluxo. |

**Fatiamento**: a SAI fatia sozinha em lotes de 40 contatos e devolve todos os `campaign_ids`. O Motor precisa lançar TODOS os batches (não só o primeiro).

**Roteamento SIA versus Morada**: definido pela config `sai_dispatch_channel_{vertical}` (valor `'sia'` ou vazio). Editável na tela de configurações (T237).

**`rd_source` derivado**: override por campanha vence. Senão usa a config por vertical. A regra deriva da rota (rota sia gera source SIA, rota Morada gera source MIA), então um flip futuro de canal já nasce com a atribuição certa.

**Reconciliação**: além do status sync a cada 30s, há `POST /campanhas/{id}/reconciliar-sai-live`. Ele confere o status real na SAI e rebaixa disparo fantasma (marcado como enviado sem confirmação) de volta para agendado. É read-only na SAI, casa por telefone ou email, e só mexe em quem está como FALHA ou fantasma.

**Endpoints do Motor relevantes para a integração**: `POST /campanhas/{id}/executar-disparo`, `POST /campanhas/{id}/reconciliar-sai-live`, `POST /campanhas/{id}/reconciliar-sia`, `POST /reguas/{id}/avancar`.

**Pra alinhar com a SIA (status de template)**: a SIA guarda um cache próprio de templates que não acompanha a Meta sozinho. Como o Motor cria e aprova pela Meta, ele precisa ver o status lá e então chamar o `/whatsapp-templates/sync` da SIA antes de disparar. Enquanto isso não for automático, é a origem de disparo com template ainda PENDING (relacionado ao guard T291, que hoje checa a Meta e não o cache da SIA).

---

## 11. Tratamento de erro

**Mapa de erros da Meta** (`backend/app/services/erro_wa_map.py`): mais de 15 códigos mapeados, cada um com explicação, recomendação e responsável (sai, meta, contato, motor, desconhecido). Os principais:
- `131047`: falha de reengajamento (janela de 24h sem resposta). Usar template aprovado.
- `131049`: frequency capping (engajamento baixo). Espaçar disparos, não reenviar em massa.
- `131048`: rate limit por spam. Reduzir volume.

**Enviado fantasma**: quando a SAI retorna `sent=0` ou o template está PENDING, o Motor chegava a marcar como enviado sem confirmação. A reconciliação rebaixa fantasma para agendado. Existe um guard de template PENDING, com um buraco conhecido (T291): ele checa a Meta, não a SIA.

**Anti-duplicação e idempotência**: índice de dedup por `(lead_id, campanha_id, regua_id, canal)` e trava de concorrência (`FOR UPDATE` em `preparar_disparos`), já em produção. A duplicação de deals que apareceu em julho (colisão de lançamento mais reprocesso de timeout) foi corrigida em produção. A blindagem de raiz, um `create_deal` idempotente, fica do lado da SAI (hoje território do Luiz).

**Tentar de novo (cadência, T300)**: retenta o último passo que falhou e reprograma o `proximo_tp_data`.

**Locks**: `lead_update_lock` evita deadlock entre startup/sync e reclassificação de temperatura.

**Retries manuais**: `reprocessar_falhas()` reprocessa lotes falhados por endpoint (não automático).

**Parada de envio**: arquivar, pausar e cancelar já param o disparo de fato (fix #630).

**Ponto de atenção em notificação**: quando 100% do lote falha, o aviso pode silenciar. Os avisos de Slack são configuráveis pela UI.

---

## 12. Configurações

**Variáveis de ambiente (por nome, sem valores)**: `DATABASE_URL`, `SUPABASE_URL`, `SUPABASE_SERVICE_KEY`, `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `ALLOWED_DOMAIN`, `JWT_SECRET`, `CORS_ORIGINS`, `NEKT_API_KEY`, `SYNC_SECRET`, `SAI_API_URL`, `SAI_API_KEY`, `SIA_API_URL`, `SIA_API_KEY`, `SIA_BEARER_TOKEN`, `MORADA_MCP_KEY`, `MIA_GATEWAY_URL`, `MIA_GATEWAY_API_KEY`, `META_ACCESS_TOKEN`, `META_WABA_ID`, `META_WABA_MORADA_ID`, `META_PHONE_NUMBER_ID`, `SMARTSHEET_API_KEY`, `SPOTOMETRO_API_KEY`, `DEBUG`, `DEV_MODE`, `AMBIENTE_TESTE`.

**Flags importantes**:
- `DISPATCH_ENABLED`: em staging está `true`, ou seja, staging envia real. Testar só no subtipo de teste.
- `AMBIENTE_TESTE` e `is_teste`: restringem o disparo à audiência de teste.
- `DEV_MODE`: pula auth e usa mock. Nunca em produção.
- `SPOTOMETRO_FONTE`: hoje `units_cache` (legado), alvo é `api_spotsys`.

**Config por vertical (na tabela de config)**: `sai_dispatch_channel_{vertical}`, `sai_rd_source_{vertical}`, `sai_empreendimento_{vertical}`.

**Telas de configuração**: usuários (RBAC por papel e área), exclusões (liga e desliga, default desligado com núcleo protegido), subtipos, templates e variáveis, whitelist, scoring, provider de WhatsApp (números), monitoramento (saúde de cron e sync), Slack, roteamento SIA e MIA por vertical.

**Segredos**: as chaves ficam em `sz-disparos-mkt/backend/.env` (gitignored). As chaves antigas foram expostas no git e **precisam de rotação** no console do Supabase. Isso é uma pendência de segurança.

---

## 13. Sistemas conectados, onde e quando

| Sistema | Para que | Onde e quando | Referência no código ou env |
|---------|----------|---------------|------------------------------|
| **SAI** | Executa disparos de WhatsApp, fatia em lotes de 40, cadence-dispatch | `POST sai.seazone.dev/api/campaigns` com polling. Status sync a cada 30s. | `_sai_client.py`, `SAI_API_URL` |
| **SIA** | Canal e roteamento SIA (dono: Luiz) | Quando `dispatch_channel='sia'` | `SIA_API_URL`, `SIA_API_KEY`, `SIA_BEARER_TOKEN` |
| **Morada / MIA** | Rota legada de disparo, agentes dinâmicos | `_morada_client.py`, `/morada/agents` | `MORADA_MCP_KEY`, `MIA_GATEWAY_URL` |
| **Meta / WhatsApp (2 WABAs)** | Aprovação de template HSM e envio WA | Submissão via `submeter-meta`, status Meta | `META_ACCESS_TOKEN`, `META_WABA_ID`, `META_WABA_MORADA_ID` |
| **Email (SIA para Resend/SES)** | Canal de email | Comunicações com canal email ou wa mais email | SIA Email API |
| **Nekt (Data API)** | Fonte de leads e deals (camada silver hoje; a gold BUC é o alvo da Frente 1) | `POST sql-query`, endpoint `/sync/nekt`, sync diário 21h | `NEKT_API_KEY`, `SYNC_SECRET` |
| **Sapron** | Imóvel para proprietário, base de hóspedes | Sync diário 21h | `consultar_banco` (MCP) |
| **SpotSys / Spotômetro** | Empreendimentos e marcos de obra | `ver_marcos_obra`, `staging.marco_obra` | `SPOTOMETRO_API_KEY`, alvo `api_spotsys` |
| **Smartsheet** | Marcos e cobertura | `sync_smartsheet` | `SMARTSHEET_API_KEY` |
| **Slack** | Avisos de disparo, configurável pela UI | `notificar_disparo_campanha` (T157) | webhooks |
| **Google OAuth** | Login restrito a @seazone | `auth/callback` | `GOOGLE_CLIENT_ID` |
| **Supabase** | Postgres (2 schemas) e REST | `DATABASE_URL`, REST com `Accept-Profile` | `SUPABASE_URL`, `SUPABASE_SERVICE_KEY` |

---

## 14. Estado atual, produção ao vivo (28/07/2026)

Leitura read-only na API de produção (`motor-disparos.seazone.dev`), sem PII. Números ao vivo de 28/07.

**Base e funil**:
- Leads no total: **279.239**.
- Elegíveis agora: **124.324** (quentes **6.309**, mornos **19.524**, frios **98.491**).
- Excluídos: **154.915**. Principais motivos: fora do polígono 7.081, parceiros 6.395, contato inválido 3.920, duplicados 2.912, sem interesse 2.814, expansão 626, aluguel anual 455.
- Score médio **38** (mínimo 10, máximo 205).

**Campanhas**:
- No Motor: **176** (126 correntes mais 50 arquivadas). Das correntes, **111 reais** e **15 de teste**.
- Por tipo: **122 blast**, **4 cadência**.
- Por vertical: **SZI 61**, **MKP 46**, **SZS 19**.
- Canal e área: **100% WhatsApp** e **100% marketing**. Expansão, parcerias e email ainda não rodaram pelo Motor, o que casa com as frentes futuras.
- Mensagens enviadas em campanhas reais: cerca de **9.333**.
- Categorias mais usadas: lost 57, repescagem 36, oportunidade da semana 6, proprietário 5.

**Catálogo de configuração** (do store Supabase, config estável): comunicações **50** (matriz SZI 22, SZS 13, MKP 15), exclusões **11**, subtipos **30**, variáveis **15**, regras de scoring **21**.

**Duas notas de leitura**:
- O widget de dashboard reporta **557 finalizadas** mais 20 ativas numa contagem de base mais ampla (provável contagem de sub-lotes da SAI). Não misturo esse número com o de campanhas da lista.
- Depois da migração pro EKS (30/07), o banco de produção passou a ser o próprio projeto Supabase `wqvgralwoymmtkbeswqh` (us-east-1). Antes a produção rodava num Postgres interno do Coolify e o Supabase era um **store à parte**, isso deixou de valer. Como o cutover está em validação, confirme a que ambiente a leitura local (REST e MCP) aponta antes de tirar conclusão de volumetria.

---

*Gerado em 28/07/2026, volumetria de produção verificada ao vivo na API (`motor-disparos.seazone.dev`), read-only e sem PII. Atualizado em 30/07/2026 com os pedidos e as decisões da apresentação de funcionalidades (seção 1.3).*
