# Fase 3 — Achados do `ruff` na esteira de CI

**Projeto:** ToggleMaster (FIAP — Fase 3)
**Escopo:** Correções em `app.py` e criação do `ruff.toml` para o job `Linter`

---

## Visão geral

O job `Linter` da esteira de CI reprovava com cinco achados do `ruff`. Dois eram seguros de corrigir; três são o mesmo padrão de tratamento de exceção, que **não pode ser corrigido no código sem alterar comportamento** — e, neste serviço, é o que mantém o worker vivo.

---

## O que o linter acusou

Execução de referência: [run 33431093420](https://github.com/fiap-tech-challenge-devops/analytics-service/actions/runs/33431093420), `ruff 0.16.5`.

```
app.py:1:1     I001    Import block is un-sorted or un-formatted
app.py:39:8    BLE001  Do not catch blind exception: `Exception`
app.py:84:12   BLE001  Do not catch blind exception: `Exception`
app.py:113:16  BLE001  Do not catch blind exception: `Exception`
app.py:138:34  PLW1508 Invalid type for environment variable default
Found 5 errors.
```

O job tem `continue-on-error: true`: aparece vermelho na interface, mas não reprova a esteira.

### Uma descoberta sobre a configuração

**Não existia arquivo de configuração do `ruff` neste repositório**, e o CI instala a ferramenta com `pip install ruff`, sem versão fixa.

O conjunto de regras padrão do `ruff` **cresceu** entre versões — `BLE001` e `PLW1508` não faziam parte do padrão quando o projeto começou. Sem configuração e sem versão fixa, o lint pode ficar vermelho sozinho, sem nenhuma mudança no código, só porque uma release nova ativou uma regra nova.

É o mesmo problema que levou o [`reusable-workflows`](https://github.com/fiap-tech-challenge-devops/reusable-workflows) a fixar a versão exata de toda action: uma esteira que existe para dar sinal não deveria executar código que muda sozinho.

---

## Desafio 1 — `I001`, bloco de imports desordenado

### O que estava errado

```python
import os
import sys
import threading
import json
import uuid
import time
import logging
import boto3
from botocore.exceptions import NoCredentialsError, ClientError
from flask import Flask, jsonify
from dotenv import load_dotenv
```

Biblioteca padrão e dependências de terceiros misturadas, sem ordem interna.

### Correção aplicada

Automática, via `ruff check --select I001 --fix`. O resultado separa biblioteca padrão de terceiros e ordena alfabeticamente dentro de cada grupo.

**Zero mudança de comportamento** — em Python, a ordem dos imports de módulos independentes não altera o que é carregado.

---

## Desafio 2 — `PLW1508`, tipo do valor padrão

### O que estava errado

```python
port = int(os.getenv("PORT", 8005))
```

A assinatura de `os.getenv` é `getenv(key, default)`, e a função retorna `str | None`. Passar um `int` como `default` é inconsistente com o tipo de retorno: quando a variável existe, vem `str`; quando não existe, vem `int`.

### Por que a correção é segura

O resultado é envolvido por `int()`. Com `8005` o `int()` recebe um inteiro e devolve `8005`; com `"8005"` recebe uma string e devolve `8005`. **Mesmo valor, mesmo tipo na saída.**

### Correção aplicada

```diff
-    port = int(os.getenv("PORT", 8005))
+    port = int(os.getenv("PORT", "8005"))
```

---

## Desafio 3 — `BLE001`, e por que não foi corrigido no código

### O que o linter aponta

Três ocorrências, e neste serviço elas fazem coisas diferentes entre si:

| linha | onde | papel |
|---|---|---|
| 40 | inicialização do Boto3 | último recurso antes de `sys.exit(1)` |
| 85 | `process_message` | não apaga a mensagem da fila; ela será reprocessada |
| 114 | `sqs_worker_loop` | mantém o loop vivo após uma falha |

As três seguem o mesmo formato — o `except Exception` é sempre o **último** de uma cadeia:

```python
except ClientError as e:
    log.error(f"Erro do Boto3 no loop principal do SQS: {e}")
    time.sleep(10) # Pausa antes de tentar novamente
except Exception as e:
    log.error(f"Erro inesperado no loop principal do SQS: {e}")
    time.sleep(10)
```

Os casos previstos (`NoCredentialsError`, `ClientError`, `json.JSONDecodeError`) já têm tratamento próprio. O catch-all é a rede de segurança.

### Por que a recomendação não se aplica aqui

A regra `BLE001` desaconselha capturar `Exception` porque isso engole erros imprevistos, inclusive bugs, que ficariam mais visíveis se estourassem.

O argumento vale para código de biblioteca. Aqui o programa é um **worker de fila em laço infinito**, e o objetivo é o oposto: uma exceção não prevista não pode derrubar o processo, senão o container morre e para de consumir o SQS.

A linha 114 é literalmente o que separa "uma mensagem falhou" de "o serviço caiu". Estreitar o `except` para tipos específicos faria qualquer erro fora da lista encerrar o worker.

Na linha 85 a consequência é mais sutil e igualmente importante: o bloco **não apaga a mensagem do SQS**, então ela volta para a fila e é reprocessada. Se a exceção subisse, o `sqs_worker_loop` a capturaria uma camada acima — mas com um `sleep(10)` de penalidade e sem o registro de qual mensagem falhou.

### Evidência do teste local

Durante a validação em container, sem credenciais AWS configuradas, o log registrou:

```
2026-08-31 22:04:07,824 - INFO - Iniciando o worker SQS...
2026-08-31 22:04:07,826 - ERROR - Erro inesperado no loop principal do SQS: Unable to locate credentials
2026-08-31 22:04:17,836 - ERROR - Erro inesperado no loop principal do SQS: Unable to locate credentials
```

O `except Exception` da linha 114 capturou o erro, registrou, esperou dez segundos e tentou de novo. **O container continuou de pé e o endpoint `/health` seguiu respondendo `200`** — que é exatamente o comportamento desejado para um worker cujo destino remoto está temporariamente inacessível.

Sem esse bloco, o processo teria morrido no primeiro ciclo.

### Decisão: configuração, não reescrita

Criado o `ruff.toml` na raiz:

```toml
required-version = ">=0.16,<0.17"

[lint]
ignore = ["BLE001"]
```

Duas coisas de uma vez:

| linha | efeito |
|---|---|
| `required-version` | o `ruff` recusa rodar numa versão fora da faixa, em vez de silenciosamente aplicar um conjunto de regras diferente |
| `ignore = ["BLE001"]` | desliga a regra que conflita com o desenho do worker |

O `required-version` não substitui fixar a versão no `pip install` — que seria o lugar certo, e fica registrado abaixo como pendência —, mas transforma uma mudança silenciosa de comportamento em erro explícito.

---

## Resumo das mudanças

| arquivo | mudança | motivo |
|---|---|---|
| `app.py` | bloco de imports reordenado | `I001` |
| `app.py` | `os.getenv("PORT", 8005)` → `"8005"` | `PLW1508`; `int()` produz o mesmo valor |
| `ruff.toml` *(novo)* | `ignore = ["BLE001"]` + `required-version` | catch-all é o que mantém o worker vivo |

**Nenhuma alteração de lógica.** Nenhuma rota, tratamento de exceção, consumo de fila ou escrita no DynamoDB mudou.

---

## Validação

### Lint

```
ruff 0.16.5 → All checks passed!
```

### Build

```
docker build → ok, imagem de 210 MB
```

### Execução

O serviço foi subido em container com as variáveis `AWS_REGION`, `AWS_SQS_URL` e `AWS_DYNAMODB_TABLE` definidas, mas **sem credenciais AWS** — cenário escolhido de propósito para exercitar o caminho de erro do worker.

| passo | resultado |
|---|---|
| Boot | `Clientes Boto3 inicializados na região us-east-1` |
| Worker | `Iniciando o worker SQS...` |
| Falha esperada de credencial | capturada, registrada, loop continua |
| `GET /health` | `200` `{"status":"ok"}` |

O `/health` respondendo enquanto o worker falha em laço é a confirmação de que a thread do Flask e a do SQS seguem independentes depois da reordenação de imports.

---

## Pendência registrada

**Fixar a versão do `ruff` no CI.** O `python-ci.yml` do [`reusable-workflows`](https://github.com/fiap-tech-challenge-devops/reusable-workflows) executa `pip install ruff` sem versão. O `required-version` deste repositório detecta a divergência, mas a correção de fato é `pip install ruff==0.16.5` no workflow compartilhado — mudança que afeta os três serviços em Python e exige uma tag nova.

## O que não foi testado

O caminho feliz do worker — consumir mensagem do SQS e gravar no DynamoDB — **não foi exercitado**, porque exigiria credenciais AWS reais ou um emulador de fila. As alterações deste documento não tocam nesse fluxo: são reordenação de imports, um literal de porta e um arquivo de configuração do linter.
