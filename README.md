<div align="center">

# Rota Vital

### Emissão de guias de transporte com computação *serverless*

Função AWS Lambda em Python 3.12, executada localmente com AWS SAM CLI

<br>

![CESAR School](https://img.shields.io/badge/CESAR_School-000000?style=for-the-badge)
![ADS](https://img.shields.io/badge/ADS-0052CC?style=for-the-badge)
![Infraestrutura de Software](https://img.shields.io/badge/Infraestrutura_de_Software_2026.2-2E7D32?style=for-the-badge)
<br>
![Python](https://img.shields.io/badge/Python-3.12-3776AB?style=flat-square&logo=python&logoColor=white)
![AWS SAM](https://img.shields.io/badge/AWS_SAM-CLI-FF9900?style=flat-square&logo=amazonaws&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Desktop-2496ED?style=flat-square&logo=docker&logoColor=white)

**Atividade Individual 1 · Unidade II · Caminho A (SAM CLI local)**

</div>

---

## Sumário

1. [Visão geral](#1-visão-geral)
2. [Estrutura do repositório](#2-estrutura-do-repositório)
3. [Pré-requisitos](#3-pré-requisitos)
4. [Como executar](#4-como-executar)
5. [Evidências](#5-evidências)
6. [Respostas às perguntas](#6-respostas-às-perguntas)
7. [Uso de Inteligência Artificial](#7-uso-de-inteligência-artificial)
8. [Checklist dos requisitos da atividade](#8-checklist-dos-requisitos-da-atividade)

---

## 1. Visão geral

Este projeto implementa uma função *serverless* que gera a **guia de transporte** de uma requisição de hemocomponentes do sistema **Rota Vital**. Ela é acionada de duas formas: por **evento** (`sam local invoke`) e por **HTTP** (`sam local start-api`, rota `POST /guias`).

**Caminho escolhido:** Caminho A, com AWS SAM CLI rodando inteiro no computador (sem conta em provedor de nuvem).

O experimento observa quatro fundamentos do modelo *serverless*:

| Fundamento | O que foi observado |
| --- | --- |
| Ciclo de vida do ambiente | Diferença de latência entre *cold start* e *warm start* |
| Ausência de estado garantido | Comportamento de uma variável global entre chamadas |
| Gatilhos | Invocação por evento e por HTTP |
| Limite de execução | Estouro de `Timeout` e ajuste da configuração |

### Contrato da função

| Entrada (`body` JSON) | Resposta |
| --- | --- |
| Sem `requisicaoId` | `400` com `{"erro": "requisicaoId obrigatório"}` |
| Com `requisicaoId` | `201` com `guia`, `hospital`, `bolsas` e `invocacaoNesteAmbiente` |
| Com `"demorar": true` | Dorme 5 s (usado para provocar o *timeout* de 3 s) |

<details>
<summary><b>Ver o código do handler (<code>hello_world/app.py</code>)</b></summary>

```python
import json, time

contador = 0  # variável global: só sobrevive se o ambiente for reaproveitado

def lambda_handler(event, context):
    global contador
    contador += 1
    pedido = json.loads(event.get("body") or "{}")

    if "requisicaoId" not in pedido:
        return {"statusCode": 400,
                "body": json.dumps({"erro": "requisicaoId obrigatório"})}

    if pedido.get("demorar"):
        time.sleep(5)  # passa do Timeout de 3 s do template

    guia = {"guia": "G-" + pedido["requisicaoId"],
            "hospital": pedido.get("hospital"),
            "bolsas": pedido.get("bolsas", []),
            "invocacaoNesteAmbiente": contador}
    return {"statusCode": 201, "body": json.dumps(guia)}
```

</details>

---

## 2. Estrutura do repositório

```text
rota-vital-serverless/
├── events/
│   ├── event.json              # Evento válido (formato API Gateway proxy)
│   └── event-invalido.json     # Evento sem requisicaoId (testa HTTP 400)
├── evidencias/                 # Prints dos comandos e saídas (ver seção 5)
├── hello_world/
│   ├── app.py                  # lambda_handler (Python 3.12)
│   └── requirements.txt
├── tests/                      # Testes unitários e de integração
├── samconfig.toml              # Configuração do SAM CLI
└── template.yaml               # Infraestrutura como código (rota POST /guias)
```

---

## 3. Pré-requisitos

| Componente | Papel | Verificação |
| --- | --- | --- |
| Docker Desktop | Emula os contêineres do Lambda (precisa estar aberto) | `docker --version` |
| AWS SAM CLI | Build e simulação local de API Gateway e Lambda | `sam --version` |
| Python 3.12 | Runtime da função | `python --version` |
| Git Bash ou WSL | Necessário no Windows para os comandos `curl` funcionarem como escritos | `curl --version` |

---

## 4. Como executar

### Passo 1 · Clonar e fazer o build

```bash
git clone <URL_DO_REPOSITORIO>
cd rota-vital-serverless
sam build
```

> O projeto foi criado com `sam init` (Hello World Example, Python 3.12, pacote zip). No `template.yaml`, o caminho da API foi trocado de `/hello` para `/guias` e o método de `get` para `post`.

### Passo 2 · Invocar a função com um evento

O evento `events/event.json` pode ser gerado com:

```bash
sam local generate-event apigateway aws-proxy --method POST --path guias \
  --body '{"requisicaoId":"R-001","hospital":"H-Recife","bolsas":["O-NEG-0142"]}' \
  > events/event.json
```

Execute a invocação **duas vezes** e anote o tempo de cada uma:

```bash
sam local invoke -e events/event.json            # esperado: 201 Created
sam local invoke -e events/event-invalido.json   # esperado: 400 Bad Request
```

| Invocação | Tipo | Tempo registrado |
| --- | --- | --- |
| 1ª | Cold start | _preencher_ |
| 2ª | Nova invocação | _preencher_ |

### Passo 3 · Acionar por HTTP e observar o estado

Subida **sem** reaproveitamento (padrão):

```bash
sam local start-api
```

Em outro terminal, envie a requisição **três vezes**:

```bash
curl -X POST http://127.0.0.1:3000/guias \
  -H "Content-Type: application/json" \
  -d '{"requisicaoId":"R-002","hospital":"H-Caruaru","bolsas":["O-NEG-0142"]}'
```

Pare o servidor, suba novamente **com** reaproveitamento e repita as três chamadas:

```bash
sam local start-api --warm-containers LAZY
```

| Modo | `invocacaoNesteAmbiente` nas 3 chamadas |
| --- | --- |
| Padrão (sem `--warm-containers`) | `1`, `1`, `1` |
| `--warm-containers LAZY` | `1`, `2`, `3` |

### Passo 4 · Provocar o limite de tempo

Com `Timeout: 3` em `Globals.Function` do `template.yaml`:

```bash
curl -X POST http://127.0.0.1:3000/guias \
  -H "Content-Type: application/json" \
  -d '{"requisicaoId":"R-003","demorar":true}'
```

Resultado: erro `502 Bad Gateway` no cliente e `Task timed out after 3.00 seconds` no log do servidor.

Depois, ajuste o limite e repita:

```yaml
Globals:
  Function:
    Timeout: 10
```

```bash
sam build
# repita o curl acima → 201 Created
```

---

## 5. Evidências

Todos os prints estão na pasta [`evidencias/`](evidencias/).

| ID | Cenário | Resultado observado | Evidência |
| --- | --- | --- | --- |
| 01 | Invocação por evento (sucesso) | Guia gerada, retorno `201` | [01-invoke-201.png](evidencias/01-invoke-201.png) |
| 02 | Invocação por evento (erro) | Payload sem `requisicaoId`, retorno `400` | [02-invoke-400.png](evidencias/02-invoke-400.png) |
| 03a | API local | `POST /guias` na porta `3000` | [03a-start-api.png](evidencias/03a-start-api.png) |
| 03b | Warm container | Contador `1`, `2`, `3` com `--warm-containers LAZY` | [03b-estado-warm.png](evidencias/03b-estado-warm.png) |
| 03c | Cold start (servidor) | Montagem do contêiner e carga do runtime | [03c-coldstart-server.png](evidencias/03c-coldstart-server.png) |
| 03d | Cold start (cliente) | Latência inicial, contador em `1` | [03d-coldstart-client.png](evidencias/03d-coldstart-client.png) |
| 04a | Timeout (cliente) | `502 Bad Gateway` após 3 s | [04a-timeout-client.png](evidencias/04a-timeout-client.png) |
| 04b | Timeout (log) | `Task timed out after 3.00 seconds` | [04b-timeout-log.png](evidencias/04b-timeout-log.png) |
| 05 | Concorrência | Chamadas simultâneas em instâncias isoladas | [05-concorrencia.png](evidencias/05-concorrencia.png) |
| 06 | Validação de payload | Verificação de estrutura e tipos do corpo | [06-validacao-payload.png](evidencias/06-validacao-payload.png) |

---

## 6. Respostas às perguntas

### 1. Por que a primeira invocação demorou mais? O que acontece na fase de Init?

A primeira chamada sofre o **cold start**. O ciclo de vida de um ambiente de execução tem três fases: `Init`, `Invoke` e `Shutdown`. Na fase de **Init**, a plataforma:

1. Cria o ambiente isolado (microVM Firecracker na nuvem; contêiner Docker no SAM local).
2. Carrega o *runtime* da linguagem (Python 3.12).
3. Carrega o código e executa tudo que está fora do `lambda_handler` (imports de `json` e `time` e a criação de `contador = 0`).

Quando o ambiente é reaproveitado (**warm start**), o `Init` é pulado e só ocorre a fase `Invoke`, o que reduz a latência.

### 2. O que `invocacaoNesteAmbiente` mostrou com e sem `--warm-containers`?

| Modo | Comportamento | Valor observado |
| --- | --- | --- |
| Sem `--warm-containers` | Cada requisição cria e destrói um contêiner; o `contador` é recriado | sempre `1` |
| `--warm-containers LAZY` | O contêiner é mantido após a primeira requisição; o contexto global persiste | `1`, `2`, `3`… |

**Implicação:** funções *serverless* são **stateless**. O estado em variável global só sobrevive se o ambiente for reaproveitado, e o provedor **não garante** isso: o ambiente pode ser encerrado por inatividade, reciclado ou duplicado para atender picos. Dados de negócio devem ficar em serviços externos (banco de dados, cache distribuído ou filas). A variável global serve apenas para otimizações, como reaproveitar conexões.

### 3. O que aconteceu com `demorar = true`? Qual o tempo máximo no Lambda e o que fazer com trabalhos longos?

- **Observado:** a função executa `time.sleep(5)`, mas o `Timeout` do `template.yaml` era de 3 s. O ambiente interrompeu a execução, registrou `Task timed out after 3.00 seconds` e o cliente recebeu `502 Bad Gateway`. Com `Timeout: 10`, a chamada terminou com `201 Created`.
- **Limite máximo no AWS Lambda:** **15 minutos (900 s)** por execução.
- **Trabalhos mais longos:**
  1. **Dividir em partes** e enfileirar no **Amazon SQS**, com várias instâncias Lambda consumindo em paralelo.
  2. **Orquestrar com AWS Step Functions**, encadeando várias funções curtas.
  3. **Migrar para contêineres** sem limite rígido de tempo, como **AWS Fargate (ECS)** ou **AWS Batch**.

### 4. Custo de 1 milhão de execuções (`us-east-1`, 128 MB, 100 ms)

Preços usados: **US$ 0,20** por milhão de requisições e **US$ 0,0000166667** por GB-s (sem considerar o *Free Tier*).

```text
Requisições   = (1.000.000 / 1.000.000) × 0,20                 = US$ 0,20

Memória       = 128 MB / 1024                                  = 0,125 GB
Duração       = 100 ms / 1000                                  = 0,1 s
Tempo total   = 1.000.000 × 0,1 s                              = 100.000 s
GB-segundos   = 0,125 GB × 100.000 s                           = 12.500 GB-s

Computação    = 12.500 × 0,0000166667                          = US$ 0,20833

Total         = 0,20 + 0,20833                                 ≈ US$ 0,41
```

**Custo total: aproximadamente US$ 0,41** por milhão de execuções.

### 5. Onde uma função assim faz sentido no Rota Vital, e onde não faz?

| Faz sentido | Não faz sentido |
| --- | --- |
| **Emitir guias e comprovantes sob demanda.** Operação rápida, disparada por evento (aprovação de solicitação) e com picos irregulares (emergências), que se beneficia de elasticidade e cobrança por uso. | **Rastreamento GPS contínuo** com conexões persistentes (WebSockets, *streams*). O ambiente é efêmero, limitado a 15 min e sem estado de sessão; contêineres dedicados (ECS/Fargate) são mais simples e baratos. |
| **Notificações assíncronas** (SMS, e-mail, *push*) a cada mudança de status logístico. | **Otimização pesada de rotas em lote**, que exige CPU e memória acima dos limites do Lambda (10 GB) ou mais de 15 min. |
| **Validação pontual de telemetria** das caixas térmicas durante o transporte. | |

---

## 7. Uso de Inteligência Artificial

Declaração conforme a política institucional **PO-CESAR-021**.

| Item | Descrição |
| --- | --- |
| **Ferramenta** | Google Gemini |
| **Finalidade** | Apoio à estruturação e formatação deste `README.md` e verificação da memória de cálculo da Questão 4 |
| **Validação** | Todos os comandos, o `template.yaml`, os eventos em `events/`, as execuções em Docker e os prints em `evidencias/` foram executados, inspecionados e validados localmente pelo autor, que pode explicar o código em sala |

---

## 8. Checklist dos requisitos da atividade

| Requisito | Onde está |
| --- | --- |
| Ambiente e projeto (SAM CLI configurado) | Seções [3](#3-pré-requisitos) e [4](#4-como-executar), Passo 1 |
| Função invocada com evento e dois tempos registrados | Passo 2, evidências 01 e 02 |
| Gatilho HTTP (`start-api` e `curl`) | Passo 3, evidência 03a |
| Experimentos: estado com e sem reaproveitamento, timeout e ajuste | Passos 3 e 4, evidências 03b a 04b |
| Cinco perguntas respondidas, com a conta de custo | [Seção 6](#6-respostas-às-perguntas) |
| README, pasta `evidencias/` e vídeo | Seções 4, 5 e vídeo na seção 5 |
| Seção "Uso de IA" | [Seção 7](#7-uso-de-inteligência-artificial) |
