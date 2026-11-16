> Traduzido de [telvax/telvax@4c5a64d](https://github.com/telvax/telvax/commit/4c5a64ddd71f1b78a06434638f6d7ef77a6be530) (`docs/reference/cli.md`).

# Linha de comando

`telvax` é um único programa com vários comandos: `telvax <comando> [opções]`. `telvax --version` imprime a
versão e `telvax --help` imprime a lista abaixo. `telvax doctor` diz o que o build que você tem consegue fazer. O gerenciador de pacotes, `tvpm`, está [no fim](#tvpm).

## Sinopse

```text
usage:
  telvax check [--json] [file.tvx...]                       no file: the package in this directory
  telvax lint <file.tvx> [--allow|--warn|--deny rule]... [--deny-warnings] [--list-rules]
                                                            unused variables and functions, unreachable cases
  telvax fmt [--check] [--diff] [file.tvx|dir]...             format in place (`-` reads stdin; no path: this directory)
  telvax schema <file.csv|.parquet|.arrow> [--name Row]      the `type` for the columns of a data file
  telvax doc [--public] [-o out.md] [file.tvx|dir]...      Markdown reference from the types, signatures and `#` comments
  telvax test [file.tvx|dir] [name] [--allow list] [--coverage] [--fail-under N]
                                                            run the `test_*` functions (a directory: `test_*.tvx`, `*_test.tvx`)
  telvax run <file.tvx> [function] [args...] [--allow list] [--quantize int8|int8-channel|int4-channel] [--device gpu]
  telvax profile <file.tvx> [function] [args...] [--allow list] [--top N] [--json f] [--collapsed f] [--speedscope f]
                                                            time per function, line and std operation, and memory
  telvax doctor                                             what this build can do and how it is set up
  telvax repl [file.tvx...] [--allow list]                  type definitions, variables and expressions and see the result
  telvax dap                                                debug adapter over stdio (breakpoints, stepping, variables)
  telvax lsp                                                language server over stdio
  telvax worker [--listen host:port] [--token T]            run calls for a coordinator
  telvax serve <file.tvx> [--port N] [--host H] [--seed docs.jsonl] [--live] [--allow list] [--quantize int8|int8-channel|int4-channel] [--device gpu] [--metrics] [--log-format json|text] [--log-level debug|info|warn|error] [--cors origins] [--rate-limit N/min] [--security-headers] [--process-time-header] [--shutdown-wait N] [--error-format telvax|detail] [--watch]
  telvax build --serve <file.tvx> [-o out] [--dockerfile]   single-binary server (+ Dockerfile)
  telvax jit <file.tvx> <function> [args...]                 (native, needs --features llvm)
  telvax build <file.tvx> [-o out] [--emit exe|obj|llvm-ir|mlir|ptx|stablehlo] [-O 0-3] [--sm sm_80]
                                                    (all but stablehlo need --features llvm)

--allow takes a list of fs.read, fs.write, fs, env, net, all, none. Without it everything is allowed.
```

Este bloco é a saída de `telvax --help`, e um teste verifica que ele continua sendo.

## Status de saída

| Status | Significado |
|---|---|
| 0 | O comando fez o que foi pedido. |
| 1 | Falhou: um programa com erros, um teste que falha, um arquivo que precisa ser formatado, um arquivo ausente. O motivo é impresso depois de `error:`. |
| 2 | Foi chamado de forma errada: um comando desconhecido (com uma sugestão quando há um parecido) ou opções que não combinam. |

## Opções que vários comandos compartilham

`--allow lista`
: As permissões que o programa tem: `fs.read`, `fs.write`, `fs` (as duas), `env`, `net`, `all` ou `none`, separadas por vírgulas
  (`--allow fs.read,net`). Uma função que precisa de uma permissão que não foi dada é interrompida com um erro. Sem `--allow`,
  tudo é permitido. Aceita pelos comandos `run`, `test`, `profile`, `repl` e `serve`. (`lint` tem o seu próprio
  `--allow regra`, veja abaixo.)

`--quantize int8 | int8-channel | int4-channel | none`
: Executa os produtos de matrizes `f32` com aritmética de inteiros. Veja a seção Quantização do [tour](../guide/tour.md). Aceita pelos comandos `run`, `serve` e `build`.

`--device cpu | gpu`
: Onde as operações de tensores rodam; `gpu` exige um build com o recurso `gpu` e uma GPU (`telvax doctor` diz se o build o tem). Aceita pelos comandos `run` e `serve`.

`--workers host:porta,host:porta`
: Envia as chamadas de `std.dist.map` para máquinas que executam `telvax worker`. A variável de ambiente `TELVAX_WORKERS` é o
  valor padrão. Aceita pelos comandos `run` e `serve`.

## Comandos

### `telvax check`

Analisa a sintaxe e verifica os tipos dos arquivos indicados ou, sem nenhum, do pacote no diretório atual (o que tem um
`telvax.toml`). Imprime cada erro com o seu lugar e falha se houver algum.
`--json` imprime os diagnósticos como JSON, um objeto para cada um, com `file`, `line`, `column`, `end_line`, `end_column`,
`severity`, `message` e `notes`, para editores e scripts.

### `telvax lint`

Verifica um arquivo e depois relata o código que não é usado: variáveis, funções, casos que nunca podem ser alcançados.
`--list-rules` imprime cada regra com o que ela encontra. O nível de uma regra (`allow`, `warn`, `deny`) vem da
tabela `[lint]` de `telvax.toml` e depois das opções `--allow regra`, `--warn regra` e `--deny regra`.
`--deny-warnings` faz um aviso falhar o comando.

### `telvax fmt`

Reescreve arquivos no único formato que o Telvax tem e imprime `formatted <arquivo>` para cada um que mudou. Uma pasta significa
todos os arquivos `.tvx` dentro dela; nenhum caminho significa o diretório atual; `-` lê o código da entrada padrão e
escreve o resultado na saída padrão. `--check` não altera nada e falha se um arquivo mudaria, que é o que um job de
CI quer. `--diff` mostra o que mudaria.

### `telvax schema`

Lê um arquivo `.csv`, `.parquet` ou `.arrow` e imprime o `type` das suas colunas, para colar acima de
`let t: Table[Sales] = read_csv("sales.csv")`. Um CSV é lido por inteiro: o cabeçalho dá os nomes e os valores dão os
tipos (`Int`, `Float`, `Bool`, `Date`, `Timestamp`, senão `Text`), e uma coluna com um valor vazio é escrita `T?`. Para
Parquet e Arrow, usa-se o esquema do próprio arquivo, inclusive quais colunas podem ser vazias. O tipo recebe o nome do arquivo
(`sales_2024.csv` dá `Sales2024`); `--name Row` escolhe outro nome. Uma coluna cujo nome não é um nome de campo válido
(`unit price`) é relatada, não renomeada.

### `telvax doc`

Escreve uma referência em Markdown das funções, tipos, enums e traits dos arquivos indicados (uma pasta significa todos os arquivos `.tvx`
dentro dela), com as linhas de comentário `#` escritas logo acima de cada um. `--public` deixa de fora os nomes que
começam com `_`. `-o out.md` escreve um arquivo em vez de usar a saída padrão.

### `telvax test`

Executa as funções chamadas `test_...` de um arquivo; um teste passa quando retorna e falha quando para com um erro
(`assert`, `assert_eq`, `fail`, qualquer erro de execução). Uma pasta executa todos os arquivos `test_*.tvx` e `*_test.tvx` dentro dela. Uma
segunda palavra, `name`, executa apenas os testes cujo nome a contém. Imprime `N passed, M failed` e falha se algum teste
falhou. `--coverage` também relata quais linhas do programa os testes executaram; `--fail-under N` (0 a 100) transforma isso em uma
barreira: o comando falha se menos de `N` por cento das linhas foram executadas. As funções de `std.test`, como
`mock`, estão na [referência da biblioteca](https://github.com/telvax/telvax/blob/main/docs/reference/std.md).

### `telvax run`

Chama uma função de um programa: `telvax run app.tvx main` ou apenas `telvax run app.tvx` para chamar `main`. As palavras depois do
nome da função são os seus argumentos, em ordem. Um parâmetro `Text` recebe a palavra como está; qualquer outro tipo recebe JSON (`3`,
`true`, `[1, 2]`, `{"a": 1}`). O valor que a função retorna é impresso, a menos que ela não retorne nada.

### `telvax profile`

Executa uma função como `run` e relata para onde foram o tempo e a memória: por função, por linha e por operação de `std`.
`--top N` limita o relatório às `N` instruções que mais demoraram. `--json arquivo`, `--collapsed arquivo` (para ferramentas de flame graph) e
`--speedscope arquivo` também escrevem o relatório nesses formatos.

### `telvax doctor`

Imprime a versão, as edições de sintaxe que ele lê, o que este build consegue fazer (`tables`, serviços live, GPU, backend
nativo) e as variáveis de ambiente que importam (`TELVAX_LIVE`, `OLLAMA_HOST`, `OPENAI_API_KEY`,
`TELVAX_WORKERS`, `TELVAX_CLUSTER_TOKEN`). Uma chave ou um token é mostrado como `set` ou `not set`, nunca o seu valor. Coloque a
saída em um relatório de bug.

### `telvax repl`

Inicia uma sessão interativa: você digita definições de tipos, variáveis e expressões e vê o resultado. Os arquivos indicados na
linha de comando são carregados primeiro. A mesma sessão está no playground. Dentro da sessão, `:help` lista os seus comandos.

### `telvax dap`

O adaptador de depuração (pontos de parada, execução passo a passo, variáveis), que fala o Debug Adapter Protocol pela entrada e saída
padrão. Os editores o iniciam; a extensão do VS Code o faz. Você não o executa à mão.

### `telvax lsp`

O servidor de linguagem, que fala o Language Server Protocol pela entrada e saída padrão. Os editores o iniciam; a
extensão do VS Code o faz. Você não o executa à mão.

### `telvax worker`

Executa chamadas para um coordenador, para que `--workers` possa espalhar `std.dist.map` por várias máquinas. Ele escuta em
`--listen host:porta` (padrão `127.0.0.1:7077`) e, quando `--token T` ou a variável de ambiente
`TELVAX_CLUSTER_TOKEN` está definida, responde apenas a um coordenador que tenha o mesmo token.

### `telvax serve`

Serve as funções `@endpoint` de um programa por HTTP, com a requisição validada e a resposta escrita como JSON,
até ser parado (Ctrl-C ou SIGTERM). Ele escuta em `--host` (padrão `127.0.0.1`) e `--port` (padrão `8080`). Um programa
sem funções `@endpoint` é um erro. `--live` usa os serviços reais de Ollama, OpenAI e PostgreSQL em vez dos
substitutos offline (e exige `net`). `--seed docs.jsonl` preenche o armazenamento de vetores antes da primeira requisição: um objeto JSON
por linha, com os campos de texto `collection`, `id` e `text`; cada texto é transformado em embedding e armazenado.

`--watch` serve para o desenvolvimento. Ele observa os arquivos do programa (o arquivo de entrada, os módulos que ele importa, os arquivos que
ele embute com `std.fs.embed`, e `telvax.toml` e `telvax.lock` do seu projeto) e, quando um muda, verifica o
programa e inicia o servidor de novo no mesmo host e porta. Se a verificação falha, os erros são impressos com as suas
linhas e o servidor que está rodando é mantido, então um erro de digitação não derruba o serviço. O novo servidor executa `on_start`
de novo, e o que o antigo guardava na memória se perde. Se o programa tem erros quando `--watch` começa, ele espera por uma mudança em vez de sair.
Ctrl-C e SIGTERM param o servidor e o `--watch` juntos. As outras opções são passadas ao servidor como estão.

`--metrics` serve `GET /metrics` no formato Prometheus: `http_requests_total{method,path,status}`, o
histograma `http_request_duration_seconds{method,path}`, os contadores que o programa cria com
`std.metrics.count(name)` e os medidores que ele define com `std.metrics.set(name, value)` (vale o último valor), cada um com o seu próprio nome
(`name`, ou `name{label="value"}` com rótulos) e o seu próprio `# TYPE`. Um nome é um contador ou um medidor, não os dois. `path` é a rota como está escrita no programa, então o número
de séries não cresce com os ids nas URLs. `--log-format json` (ou `text`) escreve uma linha por requisição na
saída de erro padrão: hora, nível, id da requisição, método, caminho sem a query string, status e duração. O nível é
`info` para um status abaixo de 400, `warn` para 4xx e `error` para 5xx; `--log-level` (ou `TELVAX_LOG_LEVEL`) é o menor nível
que é escrito, `info` por padrão, então `--log-level warn` mantém só as requisições que falharam. As linhas de
`std.log` seguem o mesmo formato e nível. Com qualquer uma das opções, toda
resposta tem um cabeçalho `x-request-id`, o que o cliente enviou se for curto e simples, senão um novo; a
função pode lê-lo com `req.header("x-request-id")`. Um programa iniciado com `telvax build --serve` aceita as mesmas
opções, ou `TELVAX_METRICS=1` e `TELVAX_LOG_FORMAT=json`; a opção vence a variável.

Uma função que falha em tempo de execução (um índice fora dos limites, uma divisão por zero) responde `500` com `Internal server error`, nos dois
formatos de erro, porque a mensagem do interpretador nomeia o interior do programa. A mensagem vai para a saída de erro padrão
com o id da requisição, como uma linha `error GET /path 500 request_id=...: runtime error: ...` (um objeto JSON com `--log-format json`).
Uma função que retorna `Err(Status.Internal(message))` continua enviando a sua `message`, porque o programa a escolheu.

Mais três opções atuam em todas as respostas. `--cors ORIGENS` (`*`, ou origens separadas por vírgulas) permite que uma página de navegador de
uma origem permitida leia a resposta: ela acrescenta `access-control-allow-origin` e responde um `OPTIONS` de preflight com `204`;
credenciais não são suportadas. `--rate-limit N/unidade` (`second`, `minute` ou `hour`) permite `N` requisições de um mesmo cliente em
cada janela e, depois disso, responde `429` com `retry-after`; o cliente é o endereço da conexão ou o primeiro
endereço de `x-forwarded-for` com `--trust-proxy`. `/healthz` nunca é limitado. `--shutdown-wait N` (ou `TELVAX_SHUTDOWN_WAIT`) é o número de segundos que um servidor que foi parado
espera pelas suas requisições e tarefas em segundo plano (30 por padrão). `--security-headers` acrescenta
`x-content-type-options: nosniff`, `x-frame-options: DENY`, `referrer-policy: no-referrer` e `cache-control: no-store`
às respostas que não os definem. As variáveis são `TELVAX_CORS`, `TELVAX_RATE_LIMIT`, `TELVAX_SECURITY_HEADERS`
e `TELVAX_TRUST_PROXY`. As contagens do limite de requisições ficam na memória, por processo.

Essas opções também podem ser escritas no programa, em um bloco `serve:` do arquivo de entrada:

```
serve:
    metrics: True
    log_format: "json"
    log_level: "info"
    cors: "*"
    rate_limit: "100/min"
    security_headers: True
    process_time_header: True
    trust_proxy: False
    shutdown_wait: 30
    error_format: "detail"
    on_start: [init_db]
```

Cada linha é uma das opções acima (`--log-format` é `log_format`), com um valor fixo: `True` ou `False`, um número inteiro
de segundos ou um texto. Uma linha pode, em vez disso, ler uma variável quando o servidor inicia, `env("NAME")` ou
`env("NAME", "default")`, como em `cors: env("CORS_ORIGINS", "*")`: o valor é o texto da variável, ou o valor padrão quando
a variável não está definida ou está vazia (sem valor padrão, a linha é deixada de fora), lido como a opção o lê. Uma opção `True or False`
fica ligada com `1` ou `true`; o valor padrão de uma delas é `1`, `true`, `0` ou `false`. `telvax check` recusa um valor padrão que a opção
recusaria; uma variável que a opção recusa (`LOG_LEVEL=loud`) interrompe o início com
`serve: LOG_LEVEL="loud" is not a value of `log_level`: ...`. Nada é convertido: `RATE_LIMIT_PER_MINUTE=100` não é
`100/min`. `telvax check` relata um nome que não está na lista, um valor do tipo errado ou que a
opção recusaria, um nome escrito duas vezes, um segundo bloco `serve:` e um bloco em um arquivo que não é o arquivo de entrada. Uma
variável (`TELVAX_METRICS`, ...) vence o bloco, e uma opção vence a variável. `--port`, `--host`, `--live`,
`--allow`, `--seed`, `--quantize` e `--device` não estão no bloco: dependem de onde o programa roda. Um binário
de `telvax build --serve` mantém o bloco.

`middleware: [nomes]` lista funções marcadas com `@middleware`, da forma `fn name(req: Request, next: Next) -> Response`
(`Request`, `Response` e `Next` de `std.http`), da mais externa para a mais interna. Elas rodam por dentro das opções embutidas e envolvem cada
requisição, as rotas e também `/health`, `/metrics`, `/openapi.json` e `/docs`. Um middleware que não chama `next`
responde sozinho; um que o chama recebe o `Response` do resto da cadeia, cujo status e cabeçalhos ele pode mudar
(o corpo é repassado como estava), e `next` pode ser chamado uma vez. Um middleware que falha responde com `500`. `telvax check`
recusa um nome que não é uma função marcada com `@middleware`, uma assinatura errada e um nome escrito duas vezes, e avisa sobre um
`@middleware` do arquivo de entrada que a lista não menciona. Eles rodam no interpretador, o que custa mais do que uma opção
embutida.

`on_start: [nomes]` é a única linha que não é um literal: uma lista de funções de nível superior sem parâmetros, cada uma sem retorno
ou com `Result[(), Text]`, que rodam uma vez, na ordem escrita, antes de a porta ser aberta (então `/health` não
responde até que terminem). Elas rodam com as permissões do programa (`--live`, `--allow`). Uma função que retorna
um `Err`, ou que falha, interrompe o início: a mensagem vai para a saída de erro padrão como `on_start: NAME failed: MESSAGE` e o
status de saída é 1. `telvax check` relata um nome que não é uma função, uma função com parâmetros ou com outro tipo de retorno,
e um nome escrito duas vezes. Uma função pode vir de outro arquivo (`from db import init_db`). Elas não rodam em `telvax run`
nem em `telvax test`.

### `telvax build --serve`

Cria um único executável que contém o programa e o servidor: `telvax build --serve app.tvx -o app`. Ele copia o
`telvax` que executa o comando e acrescenta o programa, então tem o mesmo tamanho do `telvax`. Quando é iniciado, ele
serve o programa: escuta em `0.0.0.0`, em `PORT` se estiver definida, e `--port` e `--host` ainda vencem. `HOST` define
o endereço, e `TELVAX_LIVE=1` liga `--live`. `app --healthcheck [/caminho]` pede ao servidor desta máquina
`/caminho` (`/healthz` se omitido; a porta é `--port`, `PORT` ou 8080) e sai com 0 para uma resposta 2xx, e com 1 nos outros casos: um
contêiner não tem `curl` para isso. `--dockerfile` também escreve um `Dockerfile` para ele ao lado do programa: ele copia o
diretório inteiro (os outros módulos e os arquivos lidos por `std.fs.embed` precisam estar lá; um `.dockerignore` deixa de fora
o resto), faz o build a partir de `ghcr.io/telvax/telvax` (ou da imagem em `--build-arg TELVAX_IMAGE=...`) e tem um `HEALTHCHECK`
que chama `--healthcheck`.

### `telvax build`

Compila um programa com o backend nativo (LLVM e MLIR), então precisa de um `telvax` compilado com `--features llvm`; os
binários publicados não são, e `telvax doctor` informa isso. `--emit` escolhe a saída: `exe` (padrão, um programa com o nome do
arquivo), `obj` (um arquivo objeto), `llvm-ir` e `mlir` (impressos), `ptx` (impresso; `--sm sm_80` escolhe a geração
da GPU) e `stablehlo` (impresso). `-o` dá o nome do arquivo de saída. `-O 0` a `-O 3` é o nível de otimização
(padrão 2). `--emit ptx` e `--emit stablehlo` não precisam do backend nativo.

### `telvax jit`

Compila uma função com o backend nativo e a chama em seguida: `telvax jit app.tvx norm 3.0 4.0`. Exige `--features llvm`.

## `tvpm`

`tvpm` é o gerenciador de pacotes. Ele lê `telvax.toml`, escreve `telvax.lock` e executa o comando `telvax` correspondente com as
dependências instaladas. `tvpm --help` imprime a lista de comandos. `tvpm` encontra o projeto no diretório atual ou no
diretório pai mais próximo que tem um `telvax.toml`. Status de saída: 0 quando o comando fez o que foi pedido, 1 quando falhou (o motivo
é impresso depois de `error:`), 2 para um comando desconhecido.

### Registro

Um registro é um diretório ou uma URL `http(s)://` com o mesmo layout: `index/<nome>.json` tem uma linha JSON para cada versão
publicada (`name`, `vers`, `deps`, `cksum`, `yanked`) e `packages/<nome>-<versão>.nvpkg` é um tar compactado com gzip contendo
`telvax.toml`, `src/**/*.tvx` e os arquivos `README.md` e `LICENSE*`, se houver. Qualquer hospedagem de arquivos estáticos serve.

`--registry <onde>`, antes ou depois do comando, ou a variável de ambiente `TVPM_REGISTRY` diz qual usar. Sem
um, uma dependência do registro é um erro que diz isso. Os pacotes baixados são descompactados em `$TVPM_HOME`, ou
`~/.tvpm`, e o seu checksum é conferido com o índice.

`tvpm search`, `publish` e `yank` escrevem em um registro de **diretório** ou o leem; não funcionam com uma URL.

### Comandos

`tvpm new <nome> [--lib]`
: Cria a pasta `<nome>` com `telvax.toml`, `src/main.tvx` (`src/lib.tvx` com `--lib`) e um `.gitignore`. O nome é
  formado por letras minúsculas, dígitos e `_` isolados, começa com uma letra e não é `std`, `crate`, `self` nem `super`.

`tvpm init`
: Faz o mesmo no diretório atual, com o nome dele. Falha se `telvax.toml` já existir.

`tvpm add <nome>[@req] [--path DIR]`
: Acrescenta uma dependência a `telvax.toml`. `name@1.2` escreve esse requisito; sem um, `tvpm` procura no registro a versão mais nova que não
  foi retirada (yanked) e escreve `^<versão>`. `--path DIR` depende de um diretório (relativo ao projeto) cujo pacote
  tem esse nome. Ele não trava nem baixa: execute `tvpm install`.

`tvpm remove <nome>`
: Remove uma dependência de `telvax.toml`.

`tvpm lock`
: Resolve as versões e escreve `telvax.lock`. Imprime cada pacote travado.

`tvpm install`
: Trava se `telvax.lock` está ausente ou não corresponde a `telvax.toml` e depois baixa os pacotes do registro que não estão instalados.

`tvpm update [nome...]`
: Move as versões travadas para as mais novas que cabem. Com nomes, só esses pacotes do registro se movem, e os outros ficam onde
  o lock os tem, a menos que um requisito os force. Imprime `updated name 1.0.0 -> 1.1.0` para cada mudança.

`tvpm tree`
: Imprime a árvore de dependências.

`tvpm search <texto>`
: Lista os pacotes do registro de diretório cujo nome contém `texto`, cada um com a sua versão mais nova que não foi retirada.

`tvpm publish`
: Verifica o pacote como `telvax check`, o empacota e o acrescenta ao registro de diretório. Falha se o pacote não passa
  na verificação, se essa versão já foi publicada (aumente `version` antes) ou se ele tem uma dependência de caminho: um pacote publicado
  só pode depender de pacotes do registro.

`tvpm yank <nome> <versão> [--undo]`
: Marca uma versão publicada como retirada (yanked). Uma nova resolução a ignora, e um `telvax.lock` que já a nomeia ainda a instala.
  `--undo` a marca de novo como disponível.

`tvpm check`, `run [fn]`, `serve [opções]`, `build [opções]`
: Travam e instalam se for preciso e depois executam `telvax check`, `run`, `serve` ou `build` no arquivo de entrada, com o resto das palavras
  como argumentos. `run` sem uma função chama `main`. `tvpm` inicia o `telvax` que está ao lado dele, ou o que está em `TELVAX_BIN`, ou
  o que está em `PATH`.

### `telvax.toml`

```toml
[package]
name = "app"                  # required
version = "0.1.0"             # required, semantic version
edition = "2026"              # the syntax the package is written in; "2026" when absent
description = "..."           # optional
entry = "src/app.tvx"         # optional; default src/main.tvx, then src/lib.tvx

[dependencies]
mathx = "^1.2"                         # from the registry
shared = { path = "../shared" }        # from a directory
pinned = { version = "=1.4.2" }        # the same as a string; not together with `path`

[lint]                                 # what `telvax lint` does for each rule: allow, warn or deny
unused-variable = "deny"
unused-function = "allow"
```

Um requisito de versão é o do Cargo: `1.2` e `^1.2` significam `>=1.2.0, <2.0.0`, `~1.2` significa `>=1.2.0, <1.3.0`, `=1.2.3`
significa apenas essa versão, e `>=1, <3` combina dois. Cada nome de pacote resolve para uma só versão; quando dois pacotes precisam de
faixas diferentes de um terceiro, `tvpm` escolhe a versão mais nova que serve para os dois e falha se não houver nenhuma. Uma `edition` que este `telvax` não lê é um erro que diz quais ele lê.

### `telvax.lock`

`tvpm lock` o escreve; não o edite. Ele tem um `[[package]]` para o projeto e para cada pacote de que ele depende, com
`name`, `version`, `source` (`registry` ou `path+<dir>`; nenhum para o próprio projeto), `checksum` para um pacote do registro e
`dependencies`, uma lista de nomes. Faça commit dele: assim todas as máquinas usam as mesmas versões. Ele fica desatualizado quando `telvax.toml` tem
uma dependência ou um requisito que ele não cobre, e `install`, `check`, `run`, `serve` e `build` travam de novo nesse caso.
