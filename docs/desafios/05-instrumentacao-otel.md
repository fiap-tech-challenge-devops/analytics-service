# Fase 4 — Instrumentação com OpenTelemetry

**Projeto:** ToggleMaster (FIAP — Fase 4)
**Escopo:** `requirements.txt`, `Dockerfile` e a propagação de contexto no `app.py`

---

## Visão geral

Terceiro serviço em Python, mas o primeiro com **forma diferente**. Os outros dois são APIs HTTP: recebem requisição, consultam banco, respondem. Este é um **consumidor de fila** — o Flask existe só para o `/health`, e o trabalho real acontece numa thread que faz long polling no SQS.

Isso muda o que a instrumentação precisa fazer. Não há requisição HTTP de entrada para iniciar o trace; há uma mensagem que chegou de outro serviço, e o trace precisa continuar de onde ele parou.

Para o padrão geral — por que `opentelemetry-bootstrap` não serve no Dockerfile multi-stage, por que os pacotes são `0.xxbY` — ver o documento equivalente do `flag-service`.

---

## O que veio do padrão

```diff
# requirements.txt
+opentelemetry-distro[otlp]==0.66b1
+opentelemetry-instrumentation-flask==0.66b1
+opentelemetry-instrumentation-botocore==0.66b1
+opentelemetry-instrumentation-logging==0.66b1
```

```diff
# Dockerfile
-CMD ["gunicorn", "--bind", "0.0.0.0:8005", "app:app"]
+CMD ["opentelemetry-instrument", "gunicorn", "--bind", "0.0.0.0:8005", "app:app"]
```

Sem `psycopg2` nem `requests` aqui: este serviço não tem banco relacional nem chama outros serviços por HTTP. Em compensação entra o **`botocore`**, que instrumenta as chamadas ao SQS e ao DynamoDB.

---

## Desafio 1 — o `traceparent` que nunca chegaria

A instrumentação do `botocore` gera spans para `receive_message` e `put_item`. Mas isso só mostra *que* houve chamada à AWS, não **de onde veio a mensagem**.

Para ligar ao produtor, o `traceparent` precisa viajar nos atributos da mensagem. E aí está a armadilha:

```diff
 response = sqs_client.receive_message(
     QueueUrl=SQS_QUEUE_URL,
     MaxNumberOfMessages=10,
     WaitTimeSeconds=20,
+    MessageAttributeNames=['All']
 )
```

**O SQS não devolve atributos de mensagem a menos que você peça.** Sem essa linha, o produtor pode injetar o `traceparent` corretamente e o consumidor nunca o recebe — a propagação parece quebrada do lado errado.

É o tipo de detalhe que não produz erro: o código roda, os spans aparecem, e só a correlação não existe.

---

## Desafio 2 — lote exige _link_, não parent-child

Cada poll traz até **10 mensagens**, vindas de execuções diferentes do `evaluation-service` e, portanto, de traces diferentes.

A convenção de mensageria do OpenTelemetry é explícita sobre isso: o mecanismo padrão é o **span link**, não a relação pai-filho, porque **um span tem um pai só** e não haveria como representar dez origens.

```python
def _producer_link(message):
    attributes = message.get("MessageAttributes") or {}
    carrier = {
        key.lower(): value["StringValue"]
        for key, value in attributes.items()
        if value.get("StringValue")
    }
    span_context = trace.get_current_span(extract(carrier)).get_span_context()
    return [Link(span_context)] if span_context.is_valid else []
```

E o span de processamento por mensagem:

```python
with tracer.start_as_current_span(
    f"process {QUEUE_NAME}",
    kind=SpanKind.CONSUMER,
    links=_producer_link(message),
    attributes={
        "messaging.system": "aws_sqs",
        "messaging.operation.name": "process",
        "messaging.destination.name": QUEUE_NAME,
        "messaging.message.id": message.get("MessageId", ""),
    },
):
    _handle_message(message)
```

Os atributos seguem a convenção: `messaging.system` e `messaging.operation.name` são obrigatórios. Atenção a uma mudança que pega quem segue tutorial antigo: **não existe mais um atributo chamado apenas `messaging.operation`**.

O corpo original do processamento foi extraído para `_handle_message`, sem alteração de lógica — o span apenas o envolve.

---

## O que isso custou em zero-code

Diferente dos outros dois serviços Python, aqui **o `app.py` mudou**. A auto-instrumentação cobre Flask e boto3, mas não sabe que aquela mensagem carrega um contexto de trace — isso é semântica da aplicação, não da biblioteca.

São 45 linhas, sendo a maior parte o span de processamento e a extração do link.

---

## Validação

A cadeia completa não é testável no ambiente local: o `.env` não tem credenciais da AWS nem URL de fila, e o serviço nem sobe sem elas — ele faz `sys.exit(1)` no import se faltar `AWS_REGION`, `AWS_SQS_URL` ou `AWS_DYNAMODB_TABLE`.

A validação acontece no cluster, onde as credenciais vêm do IRSA e a fila existe. O que se espera ver, numa requisição a `/evaluate`:

1. No `evaluation-service`: span `send <fila>` com kind `Producer`
2. No `analytics-service`: span `process <fila>` com kind `Consumer`, **ligado por link** ao span do produtor
3. Dentro dele: spans de `put_item` no DynamoDB e `delete_message` no SQS, gerados pelo `botocore`

Verificações locais feitas:

| Verificação | Resultado |
|---|---|
| `ruff check` com a 0.17.0, a mesma da esteira | `All checks passed!` |
| Ordem dos imports (`I001`) | correta — `opentelemetry` entra depois de `flask` |

---

## O que não foi feito

**Health check real.** O `/health` devolve `ok` fixo e não reflete o estado da thread consumidora. Se ela morrer, o liveness probe continua passando e o pod parece saudável — um ponto cego que a instrumentação torna mais visível, porque os spans de `process` simplesmente parariam de aparecer.

**Métrica de profundidade de fila.** Hoje quem lê a fila é o KEDA, direto no SQS. Uma métrica própria daria o mesmo dado no mesmo lugar que o resto da telemetria.

**Formatter JSON no stdout**, que entra com o `filelog`, na rodada do Loki.
