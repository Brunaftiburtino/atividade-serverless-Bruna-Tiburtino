# Atividade individual 1 · Serverless (Rota Vital / HemoFlow)

**Estudante:** Bruna Ferreira Tiburtino
**Disciplina:** Infraestrutura de Software · ADS · CESAR School · 2026.2
**Caminho escolhido:** A (AWS SAM CLI local, Python 3.12, Docker Desktop, Windows com VS Code)
**Vídeo (até 3 min):** _<cole aqui o link>_

Função serverless que gera a guia de transporte de uma requisição de hemocomponentes. Recebe um
JSON no `body`, valida `requisicaoId` e devolve a guia com o número da invocação no ambiente.

## Estrutura

```
template.yaml          # API POST /guias, Python 3.12, Timeout em Globals
hello_world/app.py     # handler da função
events/pedido.json     # evento no formato do API Gateway (invocação com evento)
events/demorar.json    # corpo {"requisicaoId":"R-003","demorar":true}
evidencias/            # prints dos comandos e das saídas
```

## Como rodar

Pré-requisitos: Docker Desktop aberto, AWS SAM CLI (`sam --version`) e Python 3.12.
Foi usado o VS Code com terminais Git Bash e PowerShell.

```bash
sam build

# 1) Invocação com evento (duas vezes)
sam local invoke -e events/pedido.json

# 2) Gatilho HTTP
sam local start-api
curl -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" \
  -d '{"requisicaoId":"R-002","hospital":"H-Caruaru"}'

# 3) Reaproveitando o ambiente
sam local start-api --warm-containers LAZY

# 4) Limite de tempo: Timeout 3 no template, depois 10 (rodar sam build após mudar)
curl -X POST http://127.0.0.1:3000/guias -H "Content-Type: application/json" \
  -d '{"requisicaoId":"R-003","demorar":true}'
```

No PowerShell o `curl` é outro comando, então usei `curl.exe` com as aspas do JSON escapadas
(`'{\"requisicaoId\":\"R-002\",\"hospital\":\"H-Caruaru\"}'`). O Timeout do template ficou em **10**
no estado final do repositório; os testes com limite de 3 s foram feitos antes da mudança.

## Resultados medidos (24/11/2026)

| Experimento | Resultado | Evidência |
|---|---|---|
| `sam local invoke` #1 | Init 0,80 ms · Duration 730,75 ms · `invocacaoNesteAmbiente` = 1 | `evidencias/01-invoke-1-terminal.png`, `01b-invoke-1-vscode.png` |
| `sam local invoke` #2 | Init 0,30 ms · Duration 645,44 ms · `invocacaoNesteAmbiente` = 1 | `evidencias/02-invoke-2.png` |
| `start-api` sem warm, chamada 1 | 201 · Duration 600,45 ms · contador 1 | `evidencias/03-api-sem-warm-chamada-1.png` |
| `start-api` sem warm, chamada 3 | 201 · Duration 457,63 ms · contador 1 (a chamada 2 deu timeout de cold start, 502) | `evidencias/03b-api-sem-warm-chamada-2.png` |
| `start-api` sem warm, chamada 4 | 201 · Duration 716,00 ms · contador 1 | log do servidor |
| `start-api --warm-containers LAZY`, chamada 1 | 201 · Init 1,89 ms · Duration 1797,06 ms · contador 1 | `evidencias/04-api-warm-chamada-1.png` |
| `--warm-containers LAZY`, chamadas 2 e 3 | contador 2 e 3 · Duration 109,51 ms e 116,89 ms · sem Init · log "already running" | `evidencias/04b-api-warm-chamadas-2-e-3.png` |
| `demorar:true`, Timeout 3 | `timed out after 3 seconds` · HTTP 502 / "Internal server error" | `evidencias/05-timeout-3s-demorar-json.png`, `05b-timeout-3s-inline.png` |
| `demorar:true`, Timeout 10 | 201 · Duration 5930,93 ms · guia `G-R-003` | `evidencias/06-timeout-10s-sucesso.png` |

## Respostas

**1. Por que a primeira invocação demorou mais que a segunda? O que acontece na fase de Init?**

A primeira vez não existe ambiente de execução pronto, então a plataforma precisa criá-lo: sobe
o container, carrega o runtime Python e o código da função, e executa o que está fora do handler
(os `import` e o `contador = 0`). Isso é o cold start, e só depois vem a fase Invoke, que roda o
handler. Se o ambiente for reaproveitado, a próxima chamada pula a Init e vai direto ao handler.

Vi isso no experimento com `--warm-containers LAZY`: a primeira chamada levou 1797,06 ms, e as
duas seguintes levaram 109,51 ms e 116,89 ms, sem `Init Duration` no log, que passou a dizer que a
função já estava rodando. No `sam local invoke` as duas execuções ficaram parecidas (730,75 ms e
645,44 ms), porque cada invocação sobe um container novo, então a diferença da teoria não aparece
ali. Também vi o custo do cold start de outra forma: três vezes uma primeira chamada passou dos 3 s
do Timeout e foi interrompida, ou seja, o tempo de preparar o ambiente conta dentro do limite.

**2. O que `invocacaoNesteAmbiente` mostrou com e sem `--warm-containers`? O que isso diz sobre estado em variável global?**

Sem a flag, o campo foi 1 em todas as chamadas válidas, porque cada requisição usou um container
novo e a variável global voltou a zero. Com `--warm-containers LAZY`, o campo foi 1, 2 e 3: o
container ficou vivo e a variável sobreviveu entre as chamadas.

Isso mostra que variável global só é um cache oportunista. Não dá para contar com ela, porque
o ambiente pode ser destruído a qualquer momento, e, com várias instâncias em paralelo, cada
uma teria seu próprio contador. Qualquer estado que precise ser correto (como o número de uma guia
ou o estoque de bolsas) deve ficar em um serviço externo, como banco de dados, DynamoDB ou S3.

**3. O que aconteceu com `demorar = true`? Qual o tempo máximo de uma execução no Lambda e o que fazer com um trabalho mais longo?**

Com Timeout 3, a função dorme 5 s e é interrompida: o log do servidor mostrou
`Function 'GuiaFunction' timed out after 3 seconds`, e a chamada HTTP devolveu 502 com
`{"message":"Internal server error"}`. Depois de mudar o Timeout para 10 e rodar `sam build`, a
mesma chamada terminou com 201, `Duration` de 5930,93 ms (os 5 s do `sleep` mais o custo do
emulador local). Na primeira tentativa de aumentar o limite o erro continuou, porque eu não tinha
refeito o `sam build`, e o SAM usa a cópia do template em `.aws-sam/build`.

O tempo máximo de uma execução no Lambda é 15 minutos. Trabalho mais longo deve ser dividido em
etapas menores (por exemplo, com Step Functions ou filas como SQS) ou rodar em um serviço sem esse
limite, como um container no ECS/Fargate ou um job em lote.

**4. Custo de 1 milhão de execuções (128 MB, 100 ms, us-east-1)**

- Requisições: 1.000.000 × US$ 0,20 por milhão = **US$ 0,20**
- Computação: 128 MB = 0,125 GB; 0,125 GB × 0,1 s = 0,0125 GB-s por execução
  - 1.000.000 × 0,0125 = 12.500 GB-s
  - 12.500 × US$ 0,0000166667 = **US$ 0,2083**
- **Total ≈ US$ 0,41**, sem considerar a faixa gratuita do Lambda, que cobriria esse volume no mês.

**5. Em que parte do Rota Vital uma função assim faria sentido, e em qual não faria?**

Faz sentido em tarefas curtas, sem estado e disparadas por evento: gerar a guia de transporte
quando uma requisição é criada, validar a requisição, avisar o hospital de uma entrega, e tratar
leituras de temperatura da cadeia fria, emitindo um alerta quando saírem da faixa. Nesses casos o
volume varia muito e pagar só pelo uso compensa.

Não faz sentido onde há estado em memória ou processamento longo e contínuo: o estoque FEFO e o
índice por tabela hash mantidos no backend Spring Boot, o cálculo de rotas sobre o grafo e as
conexões persistentes, como um fluxo contínuo de telemetria. Isso precisa de um processo sempre
ativo, e não de funções que podem ser destruídas entre chamadas.

## Uso de IA

- **Ferramenta:** Claude (Anthropic).
- **Finalidade:** gerar o esqueleto do projeto SAM (`template.yaml`, estrutura de pastas), montar o
  rascunho do README e da conta de custo, e orientar a resolução de erros de ambiente no Windows
  (PowerShell × Git Bash, PATH do SAM, `curl.exe`, Docker).
- **Como validei:** instalei e rodei tudo na minha máquina (`sam build`, `sam local invoke`,
  `sam local start-api`, com e sem `--warm-containers`), conferi os números no terminal e guardei
  os prints em `evidencias/`. Investiguei dois erros com a ajuda da IA: timeouts de cold start e o
  `Timeout` que não mudava sem refazer o `sam build`. Refiz a conta de custo à mão.
  
