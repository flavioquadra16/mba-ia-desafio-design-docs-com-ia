# ADR-002: Retry com backoff e dead-letter em tabela separada

## Status

Aceito

- **Data:** 2026-11-18
- **Decisores:** Diego, Larissa, Bruno, Marcos
- **Fonte:** `TRANSCRICAO.md` ([09:14]–[09:19], [09:48])

## Contexto

O destinatário do webhook pode estar offline, em manutenção ou lento demais para o timeout de entrega. O evento já estará commitado na outbox (ADR-001). É preciso decidir quantas vezes tentar de novo, com que espaçamento, e onde guardar o que não foi entregue para diagnóstico e reprocessamento manual.

Diego citou cliente com cerca de duas horas de manutenção planejada. Uma política curta demais encerra o evento ainda durante essa janela. Uma política sem teto deixa evento pendurado para sempre se o endpoint deixou de existir.

## Decisão

A entrega usa backoff e, esgotado o teto, move o evento para a tabela `webhook_dead_letter`.

A progressão fechada na reunião é 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas. Diego descreveu a soma desses intervalos como quase 15 horas entre a primeira falha e a última tentativa. A pergunta do Bruno (“Quantas tentativas?”) foi respondida com 5, e o fechamento da Larissa foi: “5 tentativas, backoff 1m/5m/30m/2h/12h”.

Leitura operacional deste pacote, a única que preserva ao mesmo tempo o número 5, os cinco intervalos e a janela de cerca de 14,6 horas:

- há um envio inicial;
- há 5 retentativas;
- cada falha antes da última retentativa espera o próximo intervalo da lista;
- a falha da 5ª retentativa move o evento para a DLQ;
- não existe sexta retentativa.

A DLQ é uma tabela própria, com payload, motivo da falha e timestamp. O evento sai da outbox principal. O reprocessamento é manual, pelo endpoint `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente. Ação restrita à role `ADMIN`, com log de quem executou (ADR-006).

## Alternativas Consideradas

### Três tentativas

Descartada. Diego argumentou que três tentativas em cerca de 30 minutos matariam o evento no meio de uma manutenção de duas horas, caso que o time já viu em cliente.

### Retry indefinido com backoff

Descartada. Cobre indisponibilidade longa, e deixa a outbox com evento que nunca completa quando o cliente some.

### Marcar a linha como falha permanente na própria outbox, sem tabela nova

Descartada em favor de `webhook_dead_letter`. Diego preferiu a tabela separada para a outbox principal continuar fácil de ler e para a falha permanente ficar como evidência de debug e de replay.

## Consequências

### Positivas

- Uma manutenção de algumas horas cabe na janela de retentativa sem intervenção.
- A outbox de trabalho não acumula falha permanente.
- O suporte consegue reenviar um evento parado sem alterar o pedido.

### Negativas

- Cliente fora do ar por mais que a janela deixa de receber retentativa automática. A recuperação passa a depender de alguém com role `ADMIN`.
- A frase “5 tentativas” e a soma dos cinco intervalos não fecham se o envio inicial entrar na conta das cinco. Este ADR explicita a leitura para o time não implementar quatro esperas e abandonar o degrau de 12 horas, nem uma sexta retentativa além da progressão.
- O replay recoloca o mesmo `event_id`. O cliente que já deduplicou esse id ignora o reenvio. Isso é coerente com at-least-once (ADR-004) e exige que o operador saiba quando o replay não reprocessa do lado de lá.
