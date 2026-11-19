# PRD: Sistema de Webhooks de Notificação de Pedidos

## Resumo e contexto da feature

O OMS passa a notificar sistemas externos quando o status de um pedido muda. A feature é outbound: a plataforma envia, o cliente recebe. Três clientes B2B pediram isso de forma formal — Atlas Comercial, MaxDistribuição e Nova Cargo. Hoje eles descobrem a mudança consultando a listagem de pedidos. A Atlas sinalizou que, sem a entrega até o fim do trimestre, pode migrar para um concorrente.

Na reunião técnica o time fechou o comportamento de produto e a forma geral da solução: evento durável junto da mudança de status, entrega por um processo separado, assinatura por endpoint e repetição controlada quando o destino falha. O prazo estimado é de três sprints, com a Atlas mirando o fim de novembro, e com dois dias úteis de revisão de segurança antes do deploy.

“Tempo real”, para esses clientes, significa qualquer atraso abaixo de 10 segundos. Não é streaming.

## Problema e motivação

A integração atual obriga o cliente a perguntar, em ciclo, se algum pedido mudou. Isso atrasa o processo dele e custa chamada desnecessária dos dois lados. O OMS já persiste cada transição em `order_status_history` e já garante a máquina de estados e o estoque, mas não empurra o fato para fora.

Sem um canal de saída, o cliente continua cego entre uma consulta e a seguinte. Com um canal síncrono no meio da mudança de status, o problema só troca de lugar: o destino lento passa a atrasar a operação interna. A motivação da feature é informar o cliente depressa sem acoplar a disponibilidade dele ao commit do pedido.

## Público-alvo e cenários de uso

**Quem recebe o evento.** Sistemas das contas B2B, no primeiro momento Atlas Comercial, MaxDistribuição e Nova Cargo. Eles expõem uma URL https e escolhem quais status querem ouvir.

**Quem configura.** Usuário autenticado na API do OMS (roles `ADMIN` e `OPERATOR`). O JWT identifica o operador, não o customer. O customer da inscrição vai explícito no cadastro. Marcos tratou esses usuários como quem representa o cliente na nossa API.

**Quem reprocessa falha permanente.** Usuário com role `ADMIN`.

### Cenários

1. A Atlas cadastra um endpoint só para `SHIPPED` e `DELIVERED`. Quando um pedido dela entra em `SHIPPED`, o sistema dela recebe um POST. Transições para `PAID` ou `PROCESSING` não geram chamada para esse endpoint.
2. A MaxDistribuição fica cerca de duas horas em manutenção. As tentativas continuam ao longo da janela combinada e a entrega ocorre sem ninguém abrir chamado, se o endpoint voltar dentro dessa janela.
3. A Nova Cargo fica indisponível por mais tempo do que a janela de retry. O evento para na fila de falha permanente. Um `ADMIN` dispara o reprocessamento quando o endpoint volta.
4. A secret de um endpoint aparece num log do cliente. Ele pede outra secret pela API e tem 24 horas em que a anterior ainda verifica assinatura, tempo de migrar os consumidores.
5. O mesmo evento chega duas vezes. O sistema do cliente reconhece o identificador e ignora a cópia. O portal de desenvolvedores, a cargo do Marcos, explica esse comportamento.

## Objetivos e métricas de sucesso

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| O cliente deixa de precisar adivinhar a mudança por consulta | Tempo entre o commit da mudança de status e a primeira tentativa de entrega, no caminho em que o destino responde | Abaixo de 10 segundos |
| Status confirmado com interessado não fica sem evento | Transições commitadas por `changeStatus` que têm ao menos um endpoint ativo inscrito no status de destino e que possuem registro de evento na mesma transação | 100% |
| A Atlas recebe a capability no prazo em que condicionou a permanência | Conclusão da feature, incluindo a revisão de segurança | Três sprints, revisão de pelo menos dois dias úteis, alinhado ao fim de novembro |

A meta de 10 segundos é a definição de tempo real dada pelos clientes e aceita na reunião. O agendamento do envio pode esperar até 2 segundos para enxergar o evento; esse piso foi aceito e cabe dentro dos 10 segundos. Não há meta numérica de redução de QPS no `GET` de pedidos: a reunião não mediu o volume atual.

## Escopo

### Incluso

- Cadastro, consulta, atualização e remoção de endpoints de webhook por customer.
- Secret gerada pela plataforma, única por endpoint, com rotação e convivência de 24 horas com a secret anterior.
- Escolha dos status que cada endpoint recebe, e flag de ativo ou inativo.
- Envio outbound quando uma mudança de status é confirmada e existe endpoint ativo interessado.
- Histórico das últimas 100 entregas de um endpoint, com sucesso ou falha, payload, resposta e tempo de resposta.
- Repetição automática com a progressão fechada na reunião e, ao fim dela, guarda do evento para replay manual por `ADMIN`.
- Assinatura do corpo para o cliente validar origem e integridade.
- Identificador estável do evento para o cliente descartar repetição.
- URL de destino exclusivamente https.

### Fora de escopo

- **E-mail quando o webhook falha.** Marcos perguntou se três falhas seguidas avisariam o cliente por e-mail. Larissa tirou isso desta fase. Pode voltar depois que o impacto real for medido.
- **Painel visual.** Marcos sugeriu um dashboard para o cliente ver os webhooks. Larissa recusou para agora. A configuração é só por API. Tela é projeto do time de frontend.
- **Rate limit de saída.** Diego perguntou o que acontece se dezenas de pedidos mudarem no mesmo minuto. Ficou para observar e decidir depois. Esta fase não limita a taxa.
- **Arquivamento das entregas com mais de 30 dias.** Mencionado como higiene da tabela, explicitamente fora desta feature.
- **Webhook de entrada.** O cliente não envia evento para o OMS por este canal. O fluxo é só de saída.
- **Mais de um worker em paralelo, garantia de ordem global e entrega exactly-once.** A ordem prometida é por pedido, com um processo só. Exactly-once foi recusado.
- **Itens do pedido dentro do aviso.** O cliente que precisa do detalhe consulta o pedido. O aviso leva o resumo da transição.
- **Disparo na criação do pedido.** O pedido nasce `PENDING` no fluxo de criação, que não é o fluxo de mudança de status coberto aqui. Inscrição em `PENDING` não recebe evento de nascimento do pedido.

## Requisitos funcionais

- **RF-01.** Um usuário autenticado cadastra um webhook com URL, customer e a lista de status que deseja receber. A secret é gerada pela plataforma e devolvida nessa resposta de criação.
- **RF-02.** O customer da inscrição vem no request. Não é lido do JWT.
- **RF-03.** O mesmo usuário lista os webhooks de um customer, atualiza URL, lista de status e flag ativo, e remove um webhook.
- **RF-04.** Cada endpoint pode ouvir um subconjunto dos status do pedido. Se nenhum endpoint ativo daquele customer quer o status de destino, nenhum evento é registrado.
- **RF-05.** O cliente consulta as últimas 100 entregas de um webhook, com resultado, payload enviado, resposta do destino e duração.
- **RF-06.** O cliente solicita uma secret nova. A secret anterior continua válida por 24 horas.
- **RF-07.** Cada request de entrega é assinado com HMAC-SHA256 da secret daquele endpoint, para o destinatário conferir origem e integridade do corpo.
- **RF-08.** URL com esquema diferente de https é recusada na validação do cadastro.
- **RF-09.** Entrega que falha é repetida na progressão de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas, em cinco retentativas após o envio inicial. A falha da última retentativa guarda o evento para tratamento manual, com payload, motivo e horário.
- **RF-10.** Um `ADMIN` reprocessa um evento dessa guarda. A chamada de quem não é `ADMIN` é recusada. A plataforma registra quem fez o replay.
- **RF-11.** Toda entrega carrega um identificador estável do evento. Repetição, retry e replay usam o mesmo identificador. A garantia oferecida ao cliente é at-least-once.
- **RF-12.** O corpo do aviso descreve a transição: identificador do evento, tipo `order.status_changed`, timestamp, pedido, número do pedido, status de origem, status de destino, customer e total. Não inclui os itens.
- **RF-13.** Endpoint inativo não recebe evento novo.

## Requisitos não funcionais

- **RNF-01.** No caminho feliz, a primeira tentativa de entrega ocorre em menos de 10 segundos após o commit. O mecanismo de agenda pode acrescentar até 2 segundos, e esse atraso foi aceito.
- **RNF-02.** A drenagem da outbox roda em processo separado da API. Reiniciar a API não é o que interrompe o envio.
- **RNF-03.** Corpo de evento acima de 64KB não é enviado e não é truncado. É erro.
- **RNF-04.** Chamada HTTP ao destino que não conclui em 10 segundos conta como falha e entra na política de repetição.
- **RNF-05.** A ordem de entrega é por pedido, e apenas enquanto houver um único processo de envio. Não há promessa de ordem global entre pedidos.
- **RNF-06.** A indisponibilidade do destino não desfaz a mudança de status e não segura a transação dos outros pedidos. A rede fica fora dessa transação.
- **RNF-07.** O destino só é aceito em TLS (https). Isso é validação de entrada, não um modo alternativo de transporte.

## Decisões e trade-offs principais

- **Evento na mesma transação do status, em vez de HTTP no meio dela.** Ganha consistência e tira o cliente lento do caminho crítico. Custa uma escrita a mais e adia a saída real para um processo de fundo. Detalhe em [ADR-001](adrs/ADR-001-outbox-no-mysql.md).
- **MySQL que já existe, em vez de um broker novo.** O time pequeno evita infra extra e mantém o evento atômico com o status. A contrapartida é polling e crescimento da tabela. [ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-005](adrs/ADR-005-worker-polling-em-processo-separado.md).
- **Cinco retentativas numa janela de cerca de 15 horas, depois intervenção humana.** Cobre a manutenção de duas horas já vista em cliente e não deixa evento eterno. Quem ficar fora além da janela depende de replay. [ADR-002](adrs/ADR-002-retry-backoff-e-dlq.md).
- **Secret por endpoint, com 24 horas de sobreposição na troca.** Reduz o estrago de um vazamento, que já aconteceu, e dá tempo de migrar. A plataforma precisa guardar secret recuperável. [ADR-003](adrs/ADR-003-hmac-sha256-secret-por-endpoint.md).
- **Pelo menos uma vez, com id estável, em vez de exactly-once.** O cliente deduplica. A plataforma não entra num protocolo distribuído com ele. [ADR-004](adrs/ADR-004-at-least-once-com-x-event-id.md).
- **Um worker fora do processo da API, lendo a cada 2 segundos.** Cabe nos 10 segundos e sobrevive ao restart da API. Não escala horizontalmente nesta fase. [ADR-005](adrs/ADR-005-worker-polling-em-processo-separado.md).
- **O módulo segue o OMS.** Menos superfície nova para errar. Herda também o que o código ainda não faz, como vínculo usuário–customer. [ADR-006](adrs/ADR-006-reuso-dos-padroes-do-projeto.md).
- **O JSON é fotografado na hora da transição.** O aviso continua verdadeiro depois que o pedido anda. Ele pode divergir do estado atual, de propósito. [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md).

## Dependências

- MySQL e Prisma já usados pelo OMS, inclusive a transação de `changeStatus` e a máquina de status do pedido.
- Autenticação JWT existente, com roles `ADMIN` e `OPERATOR`, e a checagem de role já usada nas rotas de usuário.
- Revisão de segurança da Sofia, pelo menos dois dias úteis, focada em HMAC e na geração da secret, antes do deploy.
- Comunicação do Marcos com Atlas, MaxDistribuição e Nova Cargo, e o texto no portal de desenvolvedores sobre deduplicação por identificador de evento. O portal em si não é entregue por este módulo.
- Nenhuma dependência nova de infraestrutura. Não há broker para provisionar.

## Riscos e mitigação

Probabilidade e impacto abaixo são a leitura dos riscos que a reunião descreveu. A reunião não atribuiu percentual.

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| O endpoint do cliente fica fora do ar por horas. Já existe precedente de manutenção planejada de cerca de duas horas. | Média | Médio. O status interno permanece correto; o aviso atrasa ou, se passar da janela, para de tentar sozinho. | Cinco retentativas ao longo de cerca de 14,6 horas e, depois, guarda com replay manual por `ADMIN`. |
| A secret vaza, como já vazou em log de cliente. | Média | Alto se uma secret abrisse todos os clientes. Com secret por endpoint, o estrago fica nesse cadastro, e ainda assim permite forjar aviso daquele endpoint. | Secret individual, rotação com 24 horas de grace, TLS obrigatório, secret omitida nas consultas posteriores à criação. |
| A Atlas cumpre a ameaça de migrar se a feature passar do fim do trimestre. | Média | Alto. Perda do cliente que trouxe o prazo. | Escopo desta fase fechado em três sprints, com a revisão da Sofia já reservada no fim. E-mail, painel e rate limit não competem com esse prazo. |

Um quarto efeito, tratado como comportamento combinado e não como falha surpresa: a entrega pode repetir. O impacto fica baixo para quem deduplica pelo identificador e alto para quem não deduplica. A mitigação é o id estável mais a documentação no portal.

## Critérios de aceitação

1. Dado um usuário autenticado e um customer existente, quando ele cadastra uma URL https e uma lista de status, então a resposta de criação contém a secret e as consultas seguintes desse webhook não a repetem.
2. Dado um cadastro com URL http, quando a API valida o corpo, então o cadastro é recusado.
3. Dado um endpoint ativo inscrito em `SHIPPED` para aquele customer, quando `changeStatus` confirma a ida para `SHIPPED`, então o destino recebe um POST com o resumo da transição, sem os itens, e com identificador estável e assinatura.
4. Dado que nenhum endpoint ativo daquele customer está inscrito no status de destino, quando o status muda, então não existe evento registrado para essa transição.
5. Dado um destino que não responde a tempo ou responde fora de sucesso, quando as retentativas se esgotam, então o evento fica disponível para replay e um usuário sem role `ADMIN` não consegue reprocessá-lo.
6. Dado um replay feito por `ADMIN`, quando a operação conclui, então o mesmo identificador de evento volta a ser entregável e existe registro de quem executou a ação.
7. Dada uma rotação de secret, quando ainda não se passaram 24 horas, então a secret anterior ainda verifica assinatura; depois desse prazo, deixa de verificar.
8. Dado o mesmo evento entregue duas vezes, quando o cliente compara o identificador, então as duas chamadas carregam o mesmo valor.
9. Dado um webhook com entregas, quando o cliente consulta o histórico, então recebe no máximo as 100 mais recentes, com resultado, payload, resposta e duração.
10. Dado um corpo de evento acima de 64KB, quando a plataforma for registrá-lo, então ela falha sem truncar e sem enviar o corpo cortado.

## Estratégia de testes e validação

A suíte atual do OMS é Vitest com Supertest (`tests/orders.test.ts`, script `test` do `package.json`). A feature entra nessa suíte. Larissa reservou testes de ponta a ponta na mesma estimativa da integração com o service de pedidos. A revisão da Sofia é validação humana obrigatória, não substituída por teste automático.

**Contrato da API de gestão.** Cadastro https devolve secret uma vez. URL http falha. Lista, alteração, remoção e histórico de 100 entregas respeitam autenticação. Replay sem `ADMIN` falha; com `ADMIN`, recoloca o evento e deixa rastro do usuário.

**Transação de status.** Com inscrição, a mesma transação que muda o status grava o evento. Sem inscrição, não grava. Falha ao gravar o evento desfaz status, histórico e estoque. `create` de pedido, que já grava `PENDING` e histórico, não grava evento de webhook. Transição inválida continua sem evento, como já é hoje quando a transação aborta.

**Entrega.** Vetor conhecido de HMAC-SHA256 sobre o corpo. Timeout de 10 segundos conta como falha. A progressão de espera é a da reunião, incluindo o degrau de 12 horas, com id igual em todas as tentativas. Corpo acima de 64KB não é truncado. Dois eventos do mesmo pedido não saem fora da ordem enquanto o mais antigo não tiver sido entregue ou movido para a guarda de falha.

**O que a suíte não prova.** Que a Atlas vai abandonar o polling, e que o destino do cliente deduplica. O primeiro é acompanhamento de produto depois do lançamento. O segundo é responsabilidade do cliente, conferida na documentação do portal.
