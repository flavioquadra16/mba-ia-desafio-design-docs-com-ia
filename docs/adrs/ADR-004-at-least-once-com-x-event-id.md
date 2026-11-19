# ADR-004: Garantia at-least-once com X-Event-Id

## Status

Aceito

- **Data:** 2026-11-18
- **Decisores:** Diego, Larissa, Sofia, Marcos
- **Fonte:** `TRANSCRICAO.md` ([09:24]–[09:26])

## Contexto

O worker pode enviar o HTTP, cair antes de marcar a linha como entregue, e enviar de novo no próximo ciclo. Retry (ADR-002) também reenvia o mesmo fato. O cliente precisa distinguir uma mudança nova de uma repetição.

Diego colocou exactly-once como a alternativa mais forte, e a afastou: exigiria coordenação dos dois lados e complexidade que o caso não paga. O padrão citado na reunião é o de Stripe e GitHub: entrega pelo menos uma vez, com identificador estável para o recebedor deduplicar.

## Decisão

A garantia da plataforma é at-least-once. Não há garantia exactly-once.

Cada linha da outbox nasce com um UUID. Esse valor segue no header `X-Event-Id` e no campo `event_id` do payload. Ele é criado na inserção e não muda no retry nem no replay administrativo: replay recoloca o mesmo evento (ADR-002).

O cliente deduplica do lado dele. Marcos assume documentar esse contrato no portal de desenvolvedores. A plataforma não mantém confirmação de “processado pelo cliente” além do HTTP de sucesso da entrega.

## Alternativas Consideradas

### Exactly-once

Descartada. Diego argumentou que a coordenação entre os dois lados fica complexa demais, e que at-least-once com `event_id` cobre a necessidade. Sofia registrou que a deduplicação fica com o cliente; a reunião aceitou essa divisão.

## Consequências

### Positivas

- O desenho sobrevive a crash entre o POST e a marcação de entregue sem perder o evento e sem inventar um protocolo de duas fases com o cliente.
- O identificador é estável e barato: é a chave da própria outbox.
- O contrato é reconhecível para quem já integra webhooks de mercado.

### Negativas

- O cliente que não persistir `X-Event-Id` aplica a mesma mudança duas vezes. A plataforma não consegue impedir isso.
- Um replay administrativo do mesmo id é invisível para quem já deduplicou a primeira entrega bem-sucedida. O operador precisa de outro caminho se o objetivo for forçar reprocessamento lógico do lado do cliente. A reunião não pediu um id novo no replay.
- “Pelo menos uma vez” também significa que sucesso HTTP e marcação local não são um único commit distribuído. Duplicata nessa janela é comportamento esperado, não incidente.
