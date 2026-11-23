Markdown
# Rota Vital — Arquitetura Serverless (AWS SAM + Lambda)

> **Instituição:** CESAR School  
> **Curso:** Análise e Desenvolvimento de Sistemas (ADS) — 3º Período (Turma Embarque)  
> **Período Letivo:** 2026.2  
> **Disciplina:** Infraestrutura de Software  
> **Discente:** Letícia Gomes da Silva  
> **Docente:** Raoni Monteiro de Oliveira  

---

## Visão Geral do Projeto

Protótipo de microsserviço serverless desenvolvido com **AWS SAM (Serverless Application Model)** e **Python 3.12** para emissão e validação de guias de requisição hospitalar (`POST /guias`). O experimento demonstra na prática os conceitos fundamentais de computação sem servidor (FaaS): **execução isolada**, **retenção de estado em memória (*warm start*)**, **inicialização a frio (*cold start*)**, **limites de tempo de execução (*timeout*)**, **concorrência** e **validação de contratos HTTP**.

---

## Estrutura do Repositório

```text
rota-vital-serverless/
├── events/
│   ├── event.json                  # Payload válido de requisição hospitalar
│   └── event-invalido.json         # Payload vazio para teste de validação (HTTP 400)
├── evidencias/                     # Capturas de tela dos 6 experimentos realizados
│   ├── 01-invoke-201.png
│   ├── 02-invoke-400.png
│   ├── 03a-start-api.png
│   ├── 03b-estado-warm.png
│   ├── 03c-coldstart-server.png
│   ├── 03d-coldstart-client.png
│   ├── 04a-timeout-client.png
│   ├── 04b-timeout-log.png
│   ├── 05-concorrencia.png
│   └── 06-validacao-payload.png
├── hello_world/
│   ├── app.py                      # Handler da função AWS Lambda (Python 3.12)
│   └── requirements.txt            # Dependências da função
├── template.yaml                   # Infraestrutura como Código (IaC) AWS SAM
└── README.md                       # Documentação acadêmica e análise das evidências
Tecnologias e Execução Local
Linguagem / Runtime: Python 3.12

Infraestrutura como Código (IaC): AWS SAM (template.yaml)

Emulação Local: AWS SAM CLI + Docker Desktop

Comandos Utilizados
Build do pacote da aplicação:

Bash
sam build
Invocação direta da função Lambda (sem API Gateway):

Bash
sam local invoke HelloWorldFunction -e events/event.json
Inicialização da API HTTP local:

Bash
sam local start-api
Evidências e Análise dos Experimentos
1. Invocação Direta com Payload Válido (HTTP 201)
Arquivo: evidencias/01-invoke-201.png

Procedimento: Execução direta da função via CLI com sam local invoke HelloWorldFunction -e events/event.json.

Comportamento observado: A função Lambda interpretou o JSON de entrada, processou a requisição hospitalar, gerou o identificador da guia (G-REQ-...) e retornou status HTTP 201 Created.

2. Invocação Direta com Payload Inválido (HTTP 400)
Arquivo: evidencias/02-invoke-400.png

Procedimento: Execução de sam local invoke HelloWorldFunction -e events/event-invalido.json contendo "body": "{}".

Comportamento observado: A camada de validação rejeitou a requisição por ausência do campo obrigatório requisicaoId, retornando status HTTP 400 Bad Request e impedindo o processamento indevido.

3. Estado em Memória (Warm Start) vs. Inicialização a Frio (Cold Start)
Arquivos: 03a-start-api.png, 03b-estado-warm.png, 03c-coldstart-server.png e 03d-coldstart-client.png

Análise Técnica:

Variáveis declaradas fora do escopo do lambda_handler permanecem alocadas na memória do contêiner enquanto o ambiente de execução estiver ativo (warm). Na primeira chamada HTTP, invocacaoNesteAmbiente retornou 1; na segunda chamada consecutiva, retornou 2.

Ao encerrar o servidor (Ctrl + C) e reiniciar o sam local start-api, um novo contêiner Docker (SAM_CONTAINER_ID) foi provisionado do zero (cold start), reiniciando o contador em memória para 1. Isso evidencia que funções serverless são stateless por arquitetura e não garantem persistência em memória local entre ciclos de vida distintos.

4. Limite de Tempo de Execução (Timeout de 3 Segundos)
Arquivos: evidencias/04a-timeout-client.png e evidencias/04b-timeout-log.png

Procedimento: Envio de requisição HTTP POST /guias com "demorar": true para forçar um tempo de processamento superior ao limite Timeout: 3 definido no template.yaml.

Análise Técnica: Exatamente após 3 segundos, o runtime interrompeu forçadamente a execução (Function 'HelloWorldFunction' timed out after 3 seconds), e o API Gateway local retornou {"message":"Internal server error"} ao cliente.

5. Comportamento de Concorrência
Arquivo: evidencias/05-concorrencia.png

Procedimento: Disparo simultâneo de duas requisições via PowerShell utilizando Start-Job (REQ-1 lenta com demorar: true em segundo plano e REQ-2 imediata em primeiro plano).

Análise Técnica: Enquanto na nuvem AWS o serviço Lambda escala horizontalmente provisionando instâncias isoladas em paralelo para atender picos simultâneos, a emulação local com sam local start-api processa as requisições em fila no contêiner local: a REQ-2 foi atendida normalmente (G-REQ-2, invocacaoNesteAmbiente: 1) enquanto o Job1 (REQ-1) estourou o timeout de 3 segundos (Internal server error), comprovado pelo comando Receive-Job -Id 1.

6. Validação de Contrato HTTP via API Gateway Local
Arquivo: evidencias/06-validacao-payload.png

Procedimento: Envio de POST http://127.0.0.1:3000/guias com corpo JSON vazio ({}).

Análise Técnica: A integração HTTP repassou o evento ao handler Lambda, que validou o contrato de entrada e devolveu o erro semântico {"erro": "requisicaoId obrigatório"} com status HTTP 400 Bad Request.
