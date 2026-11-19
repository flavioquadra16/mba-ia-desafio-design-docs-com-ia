# Da reunião ao documento: webhooks de pedido

Este repositório documenta a feature de webhooks de notificação de status de pedidos a partir da call registrada em `TRANSCRICAO.md` e do OMS que já está em `src/`. O código da aplicação não foi alterado. O que muda para o time de engenharia são os design docs em `docs/`.

O enunciado original do desafio está no repositório base [devfullcycle/mba-ia-desafio-design-docs-com-ia](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia). O texto abaixo descreve como esta entrega foi produzida.

## Sobre o desafio

A aplicação é um Order Management System em Node.js e TypeScript, com MySQL via Prisma. Pedidos, estoque e histórico de status já existem. Notificação externa não existe: não há fila, evento nem webhook. Três clientes B2B pediram para ser avisados quando o status muda, e a decisão técnica ficou só na transcrição de uma call de cerca de 55 minutos.

A tarefa foi transformar essa call, cruzada com o código real, num pacote em que cada documento responde uma pergunta diferente. O PRD diz por que a feature existe e o que entra. O RFC propõe a arquitetura e deixa em aberto o que a reunião não fechou. Os ADRs registram cada decisão. O FDD diz como implementar. O tracker aponta a fala ou o arquivo de onde cada item saiu. Nada que não esteja na transcrição ou no código entrou como requisito.

## Ferramentas de IA utilizadas

- **Cursor, com o agente Grok 4.7.** Leu `TRANSCRICAO.md` e o código do OMS, extraiu o que foi decidido e o que foi descartado, e escreveu os markdowns. Também foi a ferramenta de revisão: cada caminho de arquivo citado foi conferido no repositório antes de permanecer no texto.
- **Nenhuma outra ferramenta de geração.** Não houve ChatGPT, Claude nem Gemini em paralelo. A checagem de contradição foi feita relendo a transcrição e os arquivos de `src/`, `prisma/` e `tests/` na mesma sessão.

## Workflow adotado

1. **Exploração.** Leitura integral da transcrição e dos pontos de encaixe: `OrderService.changeStatus`, máquina de status, erros, `requireRole`, middleware de erro, logger, schema Prisma e o prefixo `/api/v1`.
2. **Filtro.** Separação explícita entre decisão fechada, requisito, item adiado e item recusado. E-mail, dashboard, rate limit, arquivamento de 30 dias, webhook de entrada e exactly-once saíram da lista de requisitos antes de qualquer documento nascer.
3. **ADRs.** As decisões que a reunião fechou foram escritas primeiro, uma por arquivo. Elas viraram o esqueleto do resto.
4. **RFC.** A proposta ficou curta, com as alternativas que foram de fato discutidas e as três perguntas que continuaram abertas. Cada decisão aponta para o ADR, sem repetir schema nem payload.
5. **FDD.** Contratos, fluxos, erros `WEBHOOK_*` e a seção de integração com arquivos reais. O detalhe que o RFC não carrega mora aqui.
6. **PRD.** Consolidação de problema, público, métricas, escopo e riscos, com link para os ADRs em vez de reexplicar a arquitetura.
7. **Tracker.** Uma linha por item identificável, com timestamp ou caminho. Onde o FDD fixou uma constante que a call não numerou, a linha declara isso.
8. **Este README.** Por último, com o processo já acontecido.

A aplicação em `src/`, `prisma/`, `tests/` e os arquivos de configuração não foram editados. `TRANSCRICAO.md` também não.

## Prompts customizados

O primeiro prompt obrigou o filtro. Sem ele, a call vira uma lista em que tudo o que alguém falou parece requisito.

```text
Leia TRANSCRICAO.md e separe em quatro listas, só com o que a fala sustenta:
1. Decisões fechadas (quem fechou e o timestamp).
2. Requisitos funcionais que alguém pediu e ninguém retirou.
3. Itens explicitamente recusados ou adiados (e-mail, dashboard, rate limit, arquivamento, entrada, exactly-once, vários workers).
4. Frases que parecem requisito mas foram corrigidas minutos depois.
Não complete lacuna com prática de mercado. Se a reunião não escolheu, marque como em aberto.
Para cada item, devolva [hh:mm] e o nome de quem falou.
```

O segundo prompt amarrou o desenho ao código que existe, para a integração não citar arquivo imaginário.

```text
Com as decisões já filtradas, percorra o OMS e responda só com arquivo que você abriu:
- Onde changeStatus abre a transação e o que ela escreve hoje.
- Se User tem customerId. Se não tiver, o webhook não pode ler o customer do JWT.
- Como requireRole('ADMIN') já é usado e qual código de erro o ForbiddenError emite.
- Qual envelope errorMiddleware devolve para AppError e para Zod.
- Quais status existem no enum OrderStatus.
A função publishWebhookEvent entra dentro da transação existente.
Não proponha fila, OpenTelemetry nem tabela que a reunião não pediu.
Liste também o que a call mandou criar e que ainda não existe (src/worker.ts, src/modules/webhooks).
```

## Iterações e ajustes

A primeira organização do material errou em pontos concretos. O pacote final é a versão depois dessas correções. Foram quatro ciclos: extração, ADRs, FDD/RFC, e uma passagem do tracker contra a transcrição.

1. **Contagem do retry.** A primeira redação lia “5 tentativas” como cinco chamadas HTTP e não tinha onde encaixar o intervalo de 12 horas, que Larissa listou junto. Diego tinha somado os cinco intervalos e chamado o resultado de quase 15 horas entre a primeira falha e a última tentativa. A spec ficou: um envio inicial mais cinco retentativas, com as esperas 1m, 5m, 30m, 2h e 12h, e a ida para a DLQ só na falha da quinta retentativa. A ambiguidade está escrita no ADR-002, não escondida.

2. **Webhook na criação do pedido.** Como `create` já grava `PENDING` e uma linha de histórico, a primeira versão disparava evento ali também. Bruno localizou o gancho em `changeStatus`. A criação saiu do escopo. Inscrição em `PENDING` não recebe o nascimento do pedido.

3. **Customer vindo do JWT.** Marcos sugeriu customer implícito no token. Larissa corrigiu em seguida: o id vai no body ou no path. O schema confirma a correção, porque `User` não tem `customerId`. O contrato do FDD usa o corpo, no mesmo lugar em que `createOrderSchema` já recebe `customerId`.

4. **Código `WEBHOOK_SECRET_REQUIRED` e observabilidade.** Bruno citou esse código como exemplo de prefixo. A secret é gerada pela plataforma, então o código não tem caso e saiu da matriz. A primeira versão de observabilidade pedia OpenTelemetry. O repositório só tem Pino e `X-Request-Id`. O FDD ficou em log estruturado e correlação por `eventId`, sem tracer novo.

5. **Constantes que a call não numerou.** Tamanho do lote (10), secret de 32 bytes e o header `X-Signature-Previous` apareceram primeiro como se fossem decisão. Foram rebaixados a default de implementação, com a decisão de origem (lote pequeno, secret gerada por nós, grace de 24 horas) preservada no tracker.

E-mail de falha e rate limit de saída também nasceram como requisito e voltaram para fora de escopo e questão em aberto, respectivamente. A reunião é explícita nos dois.

## Como navegar a entrega

Ordem sugerida de leitura:

1. [docs/PRD.md](docs/PRD.md) — problema, escopo, métricas e o que não entra.
2. [docs/RFC.md](docs/RFC.md) — proposta, alternativas descartadas e o que continua em aberto.
3. [docs/adrs/](docs/adrs/README.md) — uma decisão por arquivo, do ADR-001 ao ADR-007.
4. [docs/FDD.md](docs/FDD.md) — contratos, fluxos, erros e integração com o código atual.
5. [docs/TRACKER.md](docs/TRACKER.md) — de onde cada item saiu.
6. [TRANSCRICAO.md](TRANSCRICAO.md) — a call, sem edição.

| Arquivo | Papel |
| --- | --- |
| `docs/PRD.md` | Por que e o quê |
| `docs/RFC.md` | Como pretendemos resolver, e o que a call não fechou |
| `docs/adrs/ADR-001` … `ADR-007` | Por que cada decisão ficou assim |
| `docs/FDD.md` | Como construir |
| `docs/TRACKER.md` | Rastro até a fala ou até o arquivo |
