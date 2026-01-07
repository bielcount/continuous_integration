## O que é uma Pipeline
é a sequência automática de etapas que o código percorre após uma mudança. <br>


```txt
Pipeline
 ├── Job A (VM 1)
 │     ├── step 1
 │     ├── step 2
 │     └── step 3
 ├── Job B (VM 2)
 │     ├── step 1
 │     └── step 2
 ```
 
Push / PR <br>
   ↓
Checkout do código <br>
   ↓
Setup ambiente <br>
   ↓
Build <br>
   ↓
Testes <br>
   ↓
Análise <br>
   ↓
aprovação ou reprovação

## Pra que serve uma pipeline?
serve para detectar erros antes que cheguem em produção, impedindo merge na branch caso os testes falhem


## O QUE COMPÕE UM PIPELINE?

| Elemento  | O que é              |
| --------- | -------------------- |
| Trigger   | O evento (push, PR)  |
| Jobs      | Blocos de execução   |
| Steps     | Passos dentro do job |
| Runner    | Máquina que executa (ambiente isolado e descartavel) |
| Artefatos | Arquivos gerados     |
| Status    | Sucesso ou falha     |

## exemplo de pipeline
```yml
on: pull_request

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
      - run: mvn test
```

## Trigger (gatilho)
O trigger define quando a pipeline será executada, ouve eventos no seu repositório git.

push <br>
pull_request <br>
agendado (cron) <br>
manual

```yml
on: pull_request 
#sempre que um pull request é executado no seu repositório, a pipeline é executada.
```

## Jobs
é uma macro etapa (bloco de execução) da pipeline, possui virtual machine (VM) própria e podem ser executadas em paralelo.
sua falha resulta em falha na pipeline.

cada job:
- não vê arquivos de outro job
- não vê banco de outro job
- não vê variáveis de outro job (sem artifacts)

```yml
jobs:
  build:
    runs-on: ubuntu-latest
```
aqui estamos dizendo que existe um job chamado build, que roda em uma vm ubuntu.

## Steps
são micro etapas de um job que são executadas em ordem sequencial e a sua falha resulta na falha do job. <br>
compartilham o mesmo ambiente (job)
podem ser:
- run (comandos)
- uses (ações reutilizaveis)

```yml
steps:
  - uses: actions/checkout@v4 # baixa o código do repositório, similar ao git clone
  - uses: actions/setup-java@v4 # instala o Java (@v4 é a versão da action, é um boilerplate obrigatório)
  - run: mvn test # usa o mvn para rodar o script que está nomeado como test, poderia ser qualquer comando shell
```

## Runner
O runner é a máquina onde tudo acontece, é criada uma virtual machine e vem com algumas coisas instaladas por padrão.
Pode ser:

hospedado pelo GitHub (ubuntu-latest)

self-hosted (servidor da empresa)

```yml
runs-on: ubuntu-latest
```
O GitHub cria uma VM do zero, executa o job e destrói a máquina no final.

## Artefatos
são arquivos gerados pela pipeline, como:

- builds (.jar, .zip)
- relatórios de teste
- logs

```yml
- uses: actions/upload-artifact
```

podem ser baixados, usados em outros jobs e armazenados temporariamente.

## Exemplo de uma pipeline Java + Maven

```yml
on: pull_request

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Baixar código # name é usado apenas para nomear a step, não altera comportamento e deixa os logs mais organizados e legiveis
        uses: actions/checkout@v4

      - name: Configurar Java
        uses: actions/setup-java@v4 # baixa o java
        with:
          java-version: '17' # define a versão do java
          distribution: 'temurin' # define a distribuição (quem empacota e mantem a o java, Oracle? Java? temurin? ...)

      - name: Rodar testes
        run: mvn test
```

## Status
No final, a pipeline retorna um status:

✔ Success → código pode ser mergeado

❌ Failure → algo deu errado

# Sintáxe

## Uses
uses é uma ação (Action) pronta e reutilizavel, o GitHub baixa um repositório de ação e executa o script configurado nela

```yml
- uses: actions/checkout@v4 # baixa o seu repositório
```

## Run
é usado para executar comandos shell diretamente na virtual machine.

```yml
- run: mvn test # roda testes unitarios
```

## with
é usado para passar parametros para uma action

```yml
- uses: actions/setup-java@v4
  with:
    java-version: '17'
    distribution: 'temurin'
```
sem with a action usa valores default (se existirem)

## env
define variaveis e pode exitir em:

- step
- job
- workflow inteiro

```yml
- name: Rodar aplicação
  run: java App
  env:
    PORT: 8080

```

## Secrets
são variaveis que não devem ser expostas publicamente 

uso incorreto:
```yml
env:
  DB_PASSWORD: 123456
```

uso correto:
```yml
env:
  DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
```

devem ser configuradas em:
GitHub → Settings → Secrets and variables → Actions

## Services
Service é um processo auxiliar, geralmente um container, que roda em paralelo ao job para fornecer dependências externas.
usados para conexões com bancos, Api's, cache, etc.

- Um service não executa código do seu projeto
- Ele fica “ligado” enquanto o job roda
- Seu job se conecta a ele

Service = infraestrutura temporária

runner cria a VM -> GitHub sobe containers Docker definidos em services

Esses containers:

- iniciam antes dos steps
- permanecem rodando durante o job
- Seus steps se conectam a eles via rede

Ao fim do job, tudo é destruído

```yml
jobs:
  test:
    runs-on: ubuntu-latest

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: testdb
          MYSQL_USER: test
          MYSQL_PASSWORD: test
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping -h localhost -ptest"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
```