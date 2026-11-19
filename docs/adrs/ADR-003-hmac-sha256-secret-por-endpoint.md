# ADR-003: Autenticação HMAC-SHA256 com secret por endpoint

## Status

Aceito

- **Data:** 2026-11-18
- **Decisores:** Sofia, Diego, Larissa, Bruno
- **Fonte:** `TRANSCRICAO.md` ([09:19]–[09:23], [09:44]–[09:45])

## Contexto

O webhook sai da nossa infra e chega a um endpoint controlado pelo cliente, carregando dados de pedido. O cliente precisa verificar que o corpo veio da plataforma e que ninguém o alterou no caminho. Já houve caso de secret de cliente vazada em log de aplicação.

Os endpoints são cadastrados por cliente, cada um com URL própria. Um mesmo customer pode ter mais de um cadastro. A URL em claro (`http`) foi rejeitada, mas Sofia tratou essa regra como validação de schema, não como decisão de arquitetura. O mesmo vale para o teto de 64KB, registrado como requisito não funcional no PRD e no FDD.

## Decisão

Cada endpoint de webhook tem uma secret exclusiva, gerada pela plataforma e devolvida na criação. A assinatura é HMAC-SHA256 do corpo cru do request, enviada no header `X-Signature`. Não há secret global da plataforma.

A secret é rotacionável por um endpoint da API. No momento da rotação, a secret anterior permanece válida por 24 horas, em paralelo com a nova, para o cliente migrar os sistemas dele. Depois disso, a anterior deixa de valer.

Também seguem nesta decisão os headers de correlação fechados na mesma conversa: `X-Event-Id`, `X-Timestamp` (instante do envio, para o cliente detectar replay se quiser) e `X-Webhook-Id` (qual cadastro originou a chamada, quando o cliente tem vários).

A secret precisa ser recuperável no worker para assinar. Ela não pode ser guardada só como hash irreversível. Cifra adicional em repouso não foi discutida.

## Alternativas Consideradas

### Uma secret global da plataforma

Descartada. O vazamento de uma única secret comprometeria todos os clientes. Sofia exigiu secret única por endpoint.

### Assinatura sem rotação

Descartada. Sem uma janela de sobreposição, trocar a secret quebra na hora todos os verificadores do cliente. A reunião fechou grace period de 24 horas por causa do precedente de secret vazada em log.

## Consequências

### Positivas

- O cliente consegue rejeitar payload adulterado ou forjado, com um algoritmo de biblioteca comum.
- O raio de vazamento fica limitado ao endpoint cuja secret escapou.
- A rotação não exige corte seco: as duas secrets convivem por 24 horas.

### Negativas

- A plataforma armazena material de assinatura recuperável. Log, erro e resposta de listagem não podem devolver a secret depois da criação ou da rotação.
- Durante o grace period o worker precisa produzir uma assinatura que a secret antiga ainda verifique. O formato exato dos headers nessa janela está especificado no FDD; a reunião não nomeou um segundo header.
- A geração da secret passa pela revisão de segurança da Sofia antes do deploy. O tamanho do material aleatório não foi fixado na reunião.
