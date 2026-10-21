> Traduzido de [telvax/telvax@7f400a6](https://github.com/telvax/telvax/commit/7f400a605113ccec612c6418625d8607023adeef) (`docs/guide/tour.md`).

# Um tour pelo Telvax

O Telvax é fácil de ler para quem conhece Python: os blocos são feitos por indentação, um bloco abre
com `:`, e não há chaves nem ponto e vírgula. Ele tem palavras próprias (`fn`, `let` / `var`,
`type`, `x => x + 1`, `Int?`) e verifica antes de o programa rodar o que o Python só descobre
no meio da execução: todo nome tem um tipo, escrito ou inferido.

O código deste guia é testado: cada bloco que não está marcado com `ignore` é verificado, formatado e,
quando tem `fn main`, executado. Rode-os a partir da raiz do repositório.

## Funções, tipos e inferência

```telvax
type Point:
    x: Float
    y: Float

fn norm(p: Point) -> Float:
    p.x * p.x + p.y * p.y

fn main() -> i32:
    let p = Point(x: 3.0, y: 4.0)
    var total = 0  # an untyped literal: its type comes from the uses
    for i in range(10):
        total += i
    print(f"norm = {norm(p)}, total = {total}")
    total
```

* Um `let` fora de uma função pode ser escrito em maiúsculas (`MAX_TEXTS`), e seus métodos e campos funcionam como em qualquer nome.
* `let x = 1` cria um nome que não pode mudar; `var x = 1` cria um que pode (`x += 1`).
  `let x: Int = 1` escreve o tipo. A última expressão de uma função é o seu valor, então
  `return` só é necessário para sair antes.
* Os tipos começam com maiúscula, como os tipos que você mesmo escreve: `Int` (64 bits), `Float`
  (64 bits), `Bool`, `Text`, `List[T]`, `Map[K, V]`, `T?` (um valor ou `None`), `Result[T, E]`,
  `Tensor[T, dims]` e `Table[Row]` (as colunas de uma tabela também podem ser `Date` e `Timestamp`). Quando o tamanho importa: `i8`, `i16`, `i32`, `u8` a `u64`,
  `f32`, `f64`.
* Uma função sem `->` retorna o tipo da sua última expressão: `fn double(x: Int): x * 2` retorna
  um `Int`. Isso vale também para métodos (`fn area(self): self.r * self.r`), e um método que implementa um
  trait retorna o que o trait declara para ele. Escreva o `->` em `main`, nos testes, nos endpoints e nas funções
  genéricas, e quando a função tem um erro.
* Um nome que começa com `_` é privado ao seu arquivo; tudo o mais pode ser importado.
* Controle de fluxo: `if` / `elif` / `else`, `while`, `for x in items`, `for i in range(n)`,
  `for i in range(a, b)`, `break`, `continue`, `pass` e `return`. `a if cond else b` é uma
  expressão. `and`, `or`, `not`, `in` e `not in` são os operadores.
* Comentários começam com `#`. Strings usam `"` ou `'`, e `f"..."` interpola `{expr}`.

## Argumentos nomeados e valores padrão

Uma chamada pode nomear seus argumentos, e um parâmetro ou um campo pode ter um valor padrão, de modo que uma configuração
se lê como uma frase e a chamada que precisa só do caso comum continua curta:

```telvax
type Query:
    text: Text
    top_k: Int = 5

fn search(q: Query, limit: Int = 10, rerank: Bool = False) -> List[Text]:
    []

let q = Query(text: "tensor shapes")  # top_k is 5
let hits = search(q, rerank: True)  # limit is 10
```

Os nomes são verificados contra a função: um nome escrito errado ou repetido é um erro de compilação.

## Pipelines

`x |> f(a)` significa `f(x, a)`, então as etapas se leem da esquerda para a direita. Uma linha que começa com `|>` continua
a linha anterior:

```telvax ignore
let answer = prompt |> llm.complete() |> std.text.trim()

let docs = query
    |> embed()
    |> store.search(3)
```

A regra é uma só: `x |> stage` é a chamada de `stage` com `x` como primeiro argumento, e a partir daí é uma chamada
como qualquer outra.

* A etapa é uma função (`x |> f`, `x |> std.text.trim`), uma chamada (`x |> f(a)`) ou uma closure
  (`x |> (n => n + 1)`).
* Nomes e valores padrão contam o valor que o pipe fornece: com `fn scale(x: Int, k: Int = 2)`, tanto
  `1 |> scale` quanto `1 |> scale(k: 3)` funcionam.
* Um sufixo depois da etapa se aplica ao resultado da chamada: `s |> parse()?` é `parse(s)?`,
  `n |> make().field` é `make(n).field`, e `n |> items()[0]` é `items(n)[0]`.
* `|>` tem precedência menor que todos os operadores: `a + b |> f` passa a soma pelo pipe, e `x |> f == 1` passa para
  `f == 1`, o que é um erro. Escreva `(x |> f()) == 1`.

## Tensores e tabelas

```telvax
fn forward(x: Tensor[f32, 2, 3], w: Tensor[f32, 3, 2]) -> Tensor[f32, 2, 2]:
    x |> std.tensor.matmul(w) |> std.tensor.relu()  # [2,3] x [3,2] -> [2,2], checked

type Row:
    id: Int
    score: f32

fn best(t: Table[Row]) -> Table[Row]:
    t |> std.table.sql("SELECT * FROM t WHERE score > 0.5 ORDER BY score DESC")

fn mean_score(t: Table[Row]) -> f32:
    std.tensor.mean(t.score)  # a zero-copy column
```

`std.table.read_csv(path)`, `read_parquet` e `read_ipc` precisam que o tipo do resultado esteja escrito
(`let t: Table[Row] = ...`), assim como `std.tensor.zeros()`, `from_vec` e `reshape`.

Uma tabela tem métodos que mantêm o tipo de suas linhas. Eles não precisam de SQL, então também funcionam no `telvax-lite`:

```telvax
type Row:
    id: Int
    score: f32

fn top_two(t: Table[Row]) -> Table[Row]:
    t.sort_by_desc("score").head(2)

fn passed(t: Table[Row]) -> Table[Row]:
    t.filter(r => r.score > 0.5)
```

`head(n)` e `tail(n)` pegam as primeiras ou as últimas `n` linhas, `sort_by(column)` e `sort_by_desc(column)` ordenam por uma
coluna, `filter(keep)` mantém as linhas para as quais `keep` é verdadeiro, e `a.concat(b)` acrescenta as linhas de `b`
(as duas tabelas precisam ter o mesmo tipo de linha). Um nome de coluna que não existe é um erro de execução que lista as colunas.

`t.select(["name", "id"])` mantém só essas colunas, nessa ordem. Os nomes são verificados enquanto o programa é
verificado, e o tipo de linha do resultado é calculado a partir deles, então nenhuma anotação é necessária:

```telvax
type Person:
    id: Int
    name: Text
    age: Int

fn names(t: Table[Person]) -> List[Text]:
    var out: List[Text] = []
    for r in t.select(["name", "id"]):
        out.push(f"{r.id}: {r.name}")
    out
```

Os nomes devem ser escritos como texto numa lista literal (não em variáveis), e uma coluna que não existe, ou que é
nomeada duas vezes, é um erro de compilação.

`t.group_by(keys, aggregations)` cria uma linha para cada valor distinto das colunas-chave, na ordem da primeira aparição.
Cada agregação é escrita `column:function` e vira uma coluna chamada `<column>_<function>`:

```telvax
type Sale:
    region: Text
    units: Int
    price: f64

fn report(t: Table[Sale]) -> List[Text]:
    var out: List[Text] = []
    for r in t.group_by(["region"], ["units:sum", "price:mean"]).sort_by_desc("units_sum"):
        out.push(f"{r.region}: {r.units_sum} units, {r.price_mean} on average")
    out
```

As funções são `sum` (um `Int` para uma coluna de inteiros, um `Float` para uma decimal), `mean` (um `Float`), `min` e
`max` (o tipo da coluna) e `count` (um `Int`, para qualquer coluna). O tipo de linha do resultado é calculado a partir dos
nomes, como em `select`. Uma coluna ou função desconhecida, uma coluna do tipo `T?`, uma soma ou média de texto e uma coluna de
resultado com nome repetido são erros de compilação. A mesma consulta em SQL continua funcionando; `group_by` também roda no `telvax-lite`.

Uma coluna do tipo `T?` pode ter valores vazios: `temp: f64?` lê células vazias de CSV, nulos do Parquet e
`None` de `from_rows` como `None`, e `t.temp` passa a ser uma `List[f64?]` (não um tensor). Uma coluna sem `?`
que encontra um valor vazio é um erro que nomeia a coluna e diz para escrever `?`.

```telvax
type Reading:
    id: Int
    temp: f64?

fn warm(t: Table[Reading]) -> Table[Reading]:
    t.filter(r => r.temp.unwrap_or(0.0) > 20.0)
```

`Date` e `Timestamp` são tipos de coluna para datas e horas. Na tabela eles são `Date32` e
`Timestamp(µs, UTC)` do Arrow, então Parquet, SQL e PyArrow veem datas de verdade. No programa um valor é um `Text` em ISO-8601:
`"2026-10-17"` e `"2026-10-17T16:53:03Z"`. `from_rows` e o CSV aceitam qualquer deslocamento (`+02:00`) e guardam em UTC; um
texto que não é uma data é um erro que diz para escrever `YYYY-MM-DD`. Ainda não há aritmética de datas: compare
o texto, ordene a tabela ou use SQL.

```telvax
type Event:
    id: Int
    day: Date
    at: Timestamp?

fn latest(t: Table[Event]) -> Table[Event]:
    t.sort_by_desc("day").head(1)
```

`std.table.read_ipc(path)` e `write_ipc(table, path)` leem e escrevem um arquivo Arrow IPC (Feather v2), que PyArrow,
Polars e pandas também leem e escrevem. `read_ipc` mapeia o arquivo na memória e as colunas apontam para dentro dele, então um
arquivo com um único lote não é copiado. Não reescreva nem trunque um arquivo enquanto uma tabela lida dele estiver em uso. Arquivos
comprimidos com LZ4 ou Zstd são decodificados (com cópia) no `telvax` completo; o `telvax-lite` lê arquivos não comprimidos.

Uma tabela pode ser cortada em pedaços. `t.split(n)` dá uma `List[Table[Row]]` de tabelas com no máximo `n` linhas cada;
elas compartilham os buffers de `t`, então nada é copiado. `std.table.read_csv_batches(path, n)`,
`read_parquet_batches` e `read_ipc_batches` leem um arquivo direto nesses pedaços (nenhuma cópia unida do arquivo
inteiro é feita), e precisam que o tipo do resultado esteja escrito: `let parts: List[Table[Row]] = ...`. Os pedaços ficam todos na
memória ao mesmo tempo; ainda não existe um leitor que os entregue um de cada vez.

```telvax
type Row:
    id: Int
    score: f32

fn total_score(path: Text) -> f32:
    let parts: List[Table[Row]] = std.table.read_parquet_batches(path, 100000)
    var sum = 0.0
    for part in parts:
        sum += part.score.sum()
    sum
```

`std.table.read_flight(url, ticket)` busca uma tabela de um servidor Arrow Flight (`DoGet`), e
`std.table.write_flight(table, url, path)` envia uma (`DoPut`). O endereço é `grpc://host:port`, ou
`grpc+tls://host:port` para TLS. Os dois precisam da permissão `net` e de uma build com a feature `flight`
(`cargo install telvax --features flight`); os binários publicados não a incluem, e sem ela eles informam
isso. Há apenas um cliente, não um servidor Flight.

## Endpoints HTTP

```telvax ignore
@endpoint(POST, "/v1/score")
fn score(input: In) -> Result[Out, Status]:
    ...
```

O tipo do parâmetro vira o corpo JSON validado (GET/DELETE o leem da query string).
Retornar `Err(Status.BadRequest("..."))` vira um HTTP 400.

Um campo com valor padrão escrito como literal (`language: Text = "en"`) ou do tipo `T?` pode ficar de fora do
corpo; ele recebe o valor padrão, ou `None`. Qualquer outro campo é obrigatório.

Um segmento `{name}` no caminho é lido da URL para o parâmetro com esse nome:

```telvax ignore
@endpoint(GET, "/v1/results/{job_id}")
fn result(job_id: Text) -> Result[Job, Status]:
    ...

@endpoint(PUT, "/v1/jobs/{id}/steps/{step}")
fn update(id: Int, step: Text, input: Patch) -> Result[Job, Status]:
    ...
```

Um parâmetro de caminho é `Text`, um tipo numérico ou `Bool`. Um valor que não converte é um HTTP 400:
`invalid path parameter id: expected i64, found `x``. O parâmetro que não está no caminho é o
payload, e há no máximo um. `/a/{x}` e `/a/{y}` são a mesma rota.

Um parâmetro do tipo `Request` recebe a requisição. Ele não faz parte do payload, e há no máximo um:

```telvax ignore
import std.http.Request

@endpoint(DELETE, "/v1/cache")
fn clear(req: Request) -> Result[Cleared, Status]:
    if req.header("x-admin-token") != Some(token):
        return Err(Status.Unauthorized("invalid token"))
    ...
```

`req.header(name)` é o primeiro valor do cabeçalho, ou `None`; o nome não diferencia maiúsculas de minúsculas.
`req.method()` e `req.path()` são `Text`.

Valores que vivem fora de uma função são compartilhados por todas as requisições. Um `let` de nível superior que guarda uma `List` ou um
`Map` pode ser alterado a partir dos endpoints, e cada `push`, `insert` ou `remove` é atômico, então um cache ou uma tabela de
jobs não precisa de mais nada:

```telvax ignore
let cache: Map[Text, Scores] = {}
let stored_at: Map[Text, Float] = {}

fn lookup(text: Text) -> Scores?:
    match stored_at.get(text):
        case Some(t):
            if std.time.now() - t < 300.0:
                cache.get(text)
            else:
                None
        case None:
            None
```

Um `var` de nível superior não é permitido, então um contador é um `Map` com uma chave. Duas chamadas seguidas sobre o mesmo valor
não são atômicas em conjunto: outra requisição pode chegar entre o `get` e o `insert`. Não há limite de tamanho nem expiração; remova
as chaves antigas você mesmo.

Um cache com limite e expiração é `std.cache.new(max, ttl_seconds)`. Ele descarta a entrada usada há mais tempo quando está
cheio, esquece uma entrada `ttl_seconds` depois de ela ser gravada, e é compartilhado por todas as requisições como o `Map` acima, então a tabela de
`stored_at` e a remoção manual não são necessárias:

```telvax ignore
let cache = std.cache.new(10000, 300.0)

fn lookup(text: Text) -> Scores?:
    cache.get(text)

fn remember(text: Text, scores: Scores):
    cache.set(text, scores)
```

`get`, `set`, `remove(key)`, `clear()` (quantas entradas ele tinha) e `len()`. As chaves são `Text`. Os valores ainda não têm o tipo verificado:
`get` devolve uma opção do que foi guardado. O limite é mantido por shard (um cache de 10.000 entradas tem 16 shards), então a
entrada que sai é a menos usada recentemente do seu shard.

Um endpoint responde `200` com o valor em JSON. Para escolher o status ou acrescentar cabeçalhos, retorne uma `Response`:

```telvax ignore
import std.http.Response

@endpoint(POST, "/v1/analyze_batch")
fn submit(input: Batch) -> Result[Response, Status]:
    let job = start(input)
    Ok(Response.new(Accepted(job_id: job.id))
        .status(202)
        .header("location", "/v1/results/" + job.id))
```

`Response.new(body)` é um `200`; `.status(code)` recebe um número de 100 a 599 (qualquer outro é um HTTP 500) e
`.header(name, value)` define um cabeçalho, substituindo um de mesmo nome, `content-type` inclusive.
`Err(Status...)` continua dando os status de erro.

### Trabalho depois da resposta

`std.task.spawn(task)` inicia uma `async fn` e retorna na hora; a tarefa continua depois de o endpoint ter respondido, de modo que
um serviço pode responder `202` e terminar mais tarde enquanto o cliente consulta:

```telvax ignore
let jobs: Map[Text, Text] = {}

async fn work(id: Text, texts: List[Text]):
    # ... the slow part ...
    jobs.insert(id, "done")

@endpoint(POST, "/v1/jobs")
fn start(input: Batch) -> Result[Response, Status]:
    let id = new_id()
    jobs.insert(id, "running")
    std.task.spawn(work(id, input.texts))
    Ok(Response.new(Accepted(job_id: id)).status(202))
```

* O valor da tarefa é descartado. Um erro nela é escrito na saída de erro padrão como `background task failed: ...`,
  a menos que um segundo argumento diga o que fazer com ele: `std.task.spawn(work(id), failed)` chama `failed(message)`, uma
  função de um `Text`, para que o job possa ser marcado como falho em vez de ficar `running`. Uma tarefa que precisa informar um
  resultado o escreve onde o programa o lê, como `jobs` acima.
* No máximo 256 tarefas rodam ao mesmo tempo; mais uma é um erro no endpoint que a iniciou.
* A tarefa compartilha os `List` e `Map` de nível superior do programa (veja a nota sobre estado compartilhado acima). Ela termina com o
  processo: parar o servidor a para.
* Só um servidor mantém a tarefa em andamento. Em `telvax run` e `telvax test`, `spawn` espera a tarefa, como `std.task.run` faz.

### Server-sent events

`Response.events(producer)` responde com `text/event-stream`. O servidor chama `producer` com um `Sink` enquanto o
cliente lê, e cada `sink.send(...)` é um evento, enviado na hora:

```telvax ignore
import std.http.Response
import std.http.Sink

@endpoint(POST, "/v1/analyze_stream")
fn stream(input: Batch) -> Result[Response, Status]:
    fn produce(sink: Sink):
        for text in input.texts:
            sink.send(score(text))
        sink.event("done", "{}")
    Ok(Response.events(produce))
```

* `send(data)` recebe um `Text`, enviado como está, ou qualquer outro valor, enviado como JSON. `event(name, data)` acrescenta uma
  linha `event:` e `comment(text)` escreve uma linha que os clientes ignoram, para manter a conexão aberta.
* O produtor é uma função escrita dentro do endpoint (uma closure é uma linha), então pode usar os parâmetros.
* O status e os cabeçalhos são enviados primeiro, então um erro no produtor não pode mudá-los: o stream termina com
  `event: error` e a mensagem. Um cliente que se desconecta faz o próximo `send` falhar, o que encerra o produtor.
* `telvax serve` e `build --serve` fazem stream; o backend nativo não serve HTTP.

## SQL no PostgreSQL

`std.db.postgres.PostgresClient` roda SQL num servidor PostgreSQL. `execute` dá o número de linhas que um comando
alterou, e `query` dá as linhas como uma lista de um `type` que você escreve:

```telvax ignore
import std.db.postgres.PostgresClient

type Job:
    id: Text
    status: Text
    attempts: Int
    result: Text?

let db = PostgresClient.connect(std.env.get("DATABASE_URL").unwrap())

fn finish(id: Text, result: Text) -> Result[Int, Text]:
    db.execute("UPDATE jobs SET status = $1, result = $2 WHERE id = $3", ("done", result, id))

fn queued() -> Result[List[Job], Text]:
    let rows: List[Job] = db.query("SELECT id, status, attempts, result FROM jobs WHERE status = $1", ["queued"])?
    Ok(rows)
```

* Os valores de `$1`, `$2` são passados separados do SQL, como lista quando são de um mesmo tipo e como tupla quando
  não são. Eles são ligados pelo servidor, então um valor não pode alterar o comando. `None` é `NULL`.
* Um campo é lido da coluna de mesmo nome, e `Text?` permite `NULL`. Uma coluna que não é um campo é
  ignorada; um campo sem coluna é um erro. Colunas `json` e `jsonb` são `Text`. Uma coluna `timestamptz`, `numeric` ou
  `uuid` é um erro que manda fazer o cast no SQL (`created_at::text`).
* Um erro do servidor é `Err(Text)` com a sua mensagem, como o nome da constraint que falhou.
* A conexão é aberta pelo primeiro comando e mantida. Há uma por cliente, usada um comando por vez,
  e ainda não há transações.
* Precisa de `--live` e da permissão `net`: não há substituto offline para SQL.

### Métricas e logs

`telvax serve app.tvx --metrics --log-format json` serve `GET /metrics` (requisições e latência por rota) e escreve uma
linha JSON por requisição com um id da requisição. Um programa conta o que só ele sabe com `std.metrics.count`, que o
servidor lista com o próprio nome, `cache_hit`:

```telvax ignore
fn lookup(key: Text) -> Text?:
    match cache.get(key):
        case Some(v):
            std.metrics.count("cache_hit")
            Some(v)
        case None:
            std.metrics.count("cache_miss")
            None
```

`--cors "*"`, `--rate-limit 100/min` e `--security-headers` põem as proteções usuais em volta de cada resposta
sem um proxy reverso. Os detalhes estão na [referência da linha de comando](https://github.com/telvax/telvax/blob/main/docs/reference/cli.md).

Um contador pode ter rótulos, um `Map[Text, Text]`: `std.metrics.count("jobs_total", {"status": "done"})` é listado como
`jobs_total{status="done"}`, com `by` como terceiro argumento. O nome é feito de letras, dígitos, `_` e `:`, um contador tem no máximo
1.000 conjuntos de rótulos (então um rótulo não deve ser feito do que um usuário envia), e `--process-time-header` acrescenta
`x-process-time-ms` a cada resposta.
`--error-format detail` escreve os erros que o servidor produz como `{"detail": ...}`, e um corpo que não se encaixa no seu tipo como `422`,
como o FastAPI faz; o padrão é `{"error": ...}` e `400`. Uma resposta que o programa monta com `Response` nunca é alterada.
Uma função que falha em tempo de execução responde `500` com `Internal server error`; o motivo vai para o log com o id da requisição.

Um programa escreve suas próprias linhas com `std.log.debug`, `info`, `warn` e `error`, no formato de `--log-format` e no
nível de `--log-level`, com campos opcionais: `std.log.warn("slow query", {"ms": "840"})`. As linhas de requisição também ganham um `level`
(`info` abaixo de 400, `warn` para 4xx, `error` para 5xx), então `--log-level warn` mantém o que falhou nos dois.

As opções do servidor podem ser escritas no programa em vez de na linha de comando, de modo que um arquivo diz como o serviço
se comporta:

```
serve:
    metrics: True
    cors: "*"
    rate_limit: "100/min"
    security_headers: True
    error_format: "detail"
```

Os valores são fixos, ou lidos do ambiente quando o servidor inicia: `cors: env("CORS_ORIGINS", "*")` é a variável
`CORS_ORIGINS`, ou `*` quando ela não está definida ou está vazia (o padrão é opcional; sem ele, a linha é como se não estivesse lá).
O texto da variável é lido como a flag o lê, então `log_format: env("LOG_FORMAT", "json")` aceita `json` ou `text`,
e `metrics: env("METRICS")` fica ligado para `1` ou `true`. Uma variável `TELVAX_*` ou uma flag vence o bloco, e `--port`, `--host` e as outras opções
sobre onde o programa roda continuam sendo flags. A lista está na [referência da linha de comando](https://github.com/telvax/telvax/blob/main/docs/reference/cli.md).

Seu próprio middleware envolve cada requisição. Uma função marcada com `@middleware` recebe a requisição e `next`, o resto da
cadeia, e dá uma `Response`; a linha `middleware:` do bloco os lista, o mais externo primeiro:

```telvax ignore
import std.http.Next
import std.http.Request
import std.http.Response

@middleware
fn version_header(req: Request, next: Next) -> Response:
    next(req).header("x-api-version", "1")

@middleware
fn admin_only(req: Request, next: Next) -> Response:
    if req.path() == "/admin" and req.header("x-admin-token") != Some(admin_token):
        return Response.new(Denied(detail: "Invalid admin token")).status(401)
    next(req)

serve:
    middleware: [admin_only, version_header]
```

`next(req)` roda o próximo middleware e depois a rota, e dá a `Response` deles, cujo status e cabeçalhos você pode mudar.
Um middleware que não chama `next` responde sozinho, e a rota não roda. Os embutidos (`metrics`, `cors`,
`rate_limit`, ...) ficam por fora e mantêm suas opções. Um middleware roda no interpretador em cada requisição, `/health` e
`/metrics` inclusive (teste `req.path()` para deixar uma rota de fora), e não pode ler nem alterar o corpo.

A única linha do bloco que não é um valor é `on_start`, uma lista de funções que rodam uma vez, em ordem, antes de o servidor
escutar: criar uma tabela, aquecer um cache, verificar uma configuração. Uma função não recebe parâmetros e não devolve nada, ou devolve um
`Result[(), Text]`; um `Err` interrompe a inicialização, com a sua mensagem na saída de erro padrão e status de saída 1.

```telvax ignore
fn init_db():
    db.execute("CREATE TABLE IF NOT EXISTS jobs (id TEXT PRIMARY KEY)", [])

serve:
    on_start: [init_db]
```

### O documento da API

Os tipos dos endpoints já descrevem a API, então o servidor a publica. `GET /openapi.json` é um documento OpenAPI 3.0:
cada rota com seus parâmetros de caminho, os parâmetros de consulta ou o corpo JSON, o tipo da resposta e
a resposta de erro (`{"error": "..."}`). `GET /docs` é uma página que as lista. Os dois existem em todo servidor, e se o
programa tem um endpoint próprio em um desses caminhos, esse é servido no lugar. `GET /healthz` responde `ok`, ou `503` enquanto o servidor está parando.

Quando o servidor recebe `SIGTERM` ou `Ctrl-C`, ele para de aceitar conexões, termina as requisições que tem e espera as tarefas
iniciadas com `std.task.spawn`, por no máximo `--shutdown-wait N` segundos (30 por padrão, ou `TELVAX_SHUTDOWN_WAIT`). Um segundo
sinal sai na hora. Se ficou trabalho por terminar, o status de saída é 1.

Uma função que retorna uma `Response` não tem resposta tipada, então o seu `200` não tem schema. Um campo do tipo `T?` ou
`Date` é descrito como é verificado, com não mais do que o seu tipo JSON.

## Módulos e pacotes

```telvax ignore
from util import clamp  # ./util.tvx
from mathx.vec import dot  # dependency `mathx`, file vec.tvx
```

Um `let` de nível superior é importado do mesmo jeito (`from settings import LIMIT`); é um único valor para o programa inteiro, então uma
`List` ou `Map` alterada em um módulo é alterada em todos. `import util.clamp` traz o mesmo nome. Um nome pode ser renomeado com `as`:
`from util import clamp as limit`, `import util.clamp as limit`, `from std.text import trim as strip`.
Nomes de `std` não precisam de import: `std.text.trim(s)`. Um módulo não é um nome: `import util` é um erro
que manda escrever `from util import name`.

```sh
tvpm new app && cd app
tvpm add mathx --registry ~/registry     # or --path ../mathx
tvpm install --registry ~/registry       # resolves, writes telvax.lock, downloads
tvpm run                                  # runs src/main.tvx with the dependencies
tvpm update                               # newest versions that fit; `tvpm update mathx` moves only mathx
```

## Distribuição

```telvax ignore
fn work(item: Job) -> Done:
    ...

fn run_all(jobs: List[Job]) -> List[Done]:
    std.dist.map(jobs, work)
```

`std.dist.map` recebe o *nome* de uma função de nível superior. Ele roda em threads locais por padrão;
`telvax run app.tvx run_all --workers host1:7077,host2:7077` envia as chamadas para máquinas que executam
`telvax worker`, e `telvax.ray.RayExecutor` (Python) as envia para o Ray.

## Quantização

`--quantize int8` (em `run`, `serve`, `jit`, `build`) faz os produtos de matrizes `f32` usarem aritmética int8 simétrica
por tensor com acumulação em `i32`. O interpretador e o backend nativo dão
resultados idênticos.

`--quantize int8-channel` e `--quantize int4-channel` (`run` e `serve`; o backend nativo
os recusa) dão a cada linha das ativações e a cada coluna dos pesos a sua própria escala, de modo que
uma coluna muito maior ou menor que as demais não perde mais a sua precisão. O modo int4
guarda os pesos em 4 bits. As escalas são calculadas a cada chamada; não há etapa de calibração.

## Aceleradores

* `--device gpu` (em `run` e `serve`, precisa de `cargo build --features gpu`) roda operações de tensor `f32` na GPU por meio do `wgpu`: Metal no macOS, Vulkan no Linux, DX12 no Windows.
* `telvax build file.tvx --emit stablehlo` imprime StableHLO para as funções de tensor do arquivo, para XLA / PJRT.
* `telvax build file.tvx --emit ptx [--sm sm_80]` imprime kernels PTX para as operações de tensor que o arquivo usa (precisa de `--features llvm`).

PTX e StableHLO são gerados e verificados, não executados: veja `docs/design/status.md`.

## Enums, `match`, `break` e `continue`

```telvax
enum Shape:
    Circle(Float)
    Rect(Float, Float)
    Empty

fn area(s: Shape) -> Float:
    match s:
        case Shape.Circle(r):
            r * r * 3.14159
        case Shape.Rect(w, h):
            w * h
        case Shape.Empty:
            0.0

fn first_over(limit: i32) -> i32:
    var found = -1
    for i in range(1, 1000):
        if i * i > limit:
            found = i
            break
    found
```

`match` deve cobrir todos os casos (o compilador nomeia as variantes que faltam) e também funciona com
`Result` (`Ok(v)` / `Err(e)`), inteiros, strings, booleanos e `_`. Os enums viajam como JSON: uma variante
sem valores é o seu nome (`"Empty"`), uma com valores é `{"Circle": 2.0}` ou `{"Rect": [2.0, 3.0]}`.
Os enums rodam no interpretador; o backend nativo ainda não os compila.

## Limitando o que um programa pode fazer

Um programa pode ler e escrever arquivos, ler variáveis de ambiente e acessar a rede. `--allow`
em `run`, `test` e `serve` lista o que ele pode fazer; sem a flag, tudo é permitido.

```sh
telvax run job.tvx main --allow fs.read          # may read files, nothing else
telvax test app.tvx --allow none                 # may touch nothing outside itself
telvax serve app.tvx --live --allow net,env
```

| Palavra | Permite |
|---|---|
| `fs.read` | `std.fs.read`, `std.fs.exists`, `read_csv`, `read_parquet`, `read_ipc` e as suas formas `_batches` |
| `fs.write` | `std.fs.write`, `write_csv`, `write_parquet`, `write_ipc` |
| `fs` | `fs.read` e `fs.write` |
| `env` | `std.env.get` |
| `net` | os clientes Ollama, OpenAI e PostgreSQL, com `--live`, e `read_flight` / `write_flight` |
| `all`, `none` | tudo, nada |

Uma chamada não permitida interrompe o programa e diz qual palavra acrescentar:
`` `std.fs.read` needs the `fs.read` capability; run with `--allow fs.read` ``. O limite é verificado quando
o programa roda, não quando é compilado, e `std.io.print` e `std.time` são sempre permitidos.

### Arquivos dentro do binário

`std.fs.embed("path")` é um `Text` com o conteúdo de um arquivo, lido quando o programa é construído e guardado dentro dele.
O caminho é um literal de texto, relativo ao diretório do arquivo principal, então `telvax build --serve` faz um único binário
que não precisa de arquivos de dados ao lado:

```telvax
let lexicon = std.fs.embed("lexicons/vader.txt")
```

Não precisa de nenhuma permissão, porque o arquivo é lido na hora da construção, não pelo programa em execução. Um arquivo que não
existe, ou um caminho que não é literal (`"data/" + name`), é um erro quando o programa é verificado.

## Testes

Uma função chamada `test_...` é um teste. `assert(cond)` e `assert_eq(a, b)` o interrompem
com uma mensagem quando falham:

```telvax
fn double(x: Int) -> Int:
    x * 2

fn test_double():
    assert_eq(double(2), 4)
```

```sh
telvax test file.tvx           # run them all, exit 1 if one fails
telvax test file.tvx double    # only the tests whose name contains `double`
telvax test tests/             # every `test_*.tvx` and `*_test.tvx` file under a directory
telvax test                    # the same, in this directory
```

### Fixtures

Um teste que recebe parâmetros os obtém de fixtures. `fn test_total(order: Order)` chama
`fixture_order()` e passa o resultado; uma fixture pode receber outras fixtures do mesmo jeito. Cada teste
recebe valores novos. Uma função `teardown_order(order: Order)`, se existir, roda depois do teste,
tenha ele passado ou não.

```telvax
fn fixture_base() -> Int:
    10

fn fixture_total(base: Int) -> Int:
    base * 2

fn teardown_total(total: Int):
    print(f"done with {total}")

fn test_total(base: Int, total: Int):
    assert_eq(total, base * 2)
```

Um parâmetro sem função `fixture_...`, ou uma fixture que retorna outro tipo, faz o teste falhar
e diz o que escrever.

### Casos

Para rodar um teste com vários valores, escreva uma função `cases_...` com o nome do teste sem
`test_`. Ela retorna uma lista; o teste roda uma vez para cada item. Uma tupla dá os primeiros parâmetros do
teste, e os parâmetros que sobram continuam sendo fixtures.

```telvax ignore
fn cases_double() -> List[(Int, Int)]:
    [(1, 2), (2, 4), (3, 6)]

fn test_double(x: Int, want: Int):
    assert_eq(double(x), want)
```

Cada execução é listada como `test_double[(2, 4)]`. Uma função `cases_...` que está vazia, ou cujos itens não
se encaixam nos parâmetros, faz o teste falhar e diz por quê.

### Mocks

`std.test.mock(target, replacement)` faz um teste chamar outra função no lugar de uma. O alvo é
uma função sua, escrita sem aspas, ou uma função de std pelo seu caminho em texto. O alvo que é uma função
deve ter o mesmo tipo da sua substituta; um alvo de std é verificado quando o teste roda.
`std.test.calls(target)` diz quantas vezes a substituta foi chamada. Todo mock é removido quando
o teste termina.

```telvax
fn price(item: Text) -> Int:
    100

fn total(item: Text) -> Int:
    price(item) * 2

fn fake_read(path: Text) -> Result[Text, Text]:
    Ok("from the test")

fn test_total():
    std.test.mock(price, item => 7)
    assert_eq(total("a"), 14)
    assert_eq(std.test.calls(price), 1)

fn test_read():
    std.test.mock("std.fs.read", fake_read)
    assert_eq(std.fs.read("any.txt"), Ok("from the test"))
```

Uma substituta não pode chamar a função que ela substitui. Chamadas por meio de métodos (`x.len()`) não podem ser
mockadas, e o próprio `std.test.*` também não.

### Cobertura

`--coverage` lista, para cada arquivo do projeto, as linhas com código e quantas delas os testes
executaram, e os números das linhas que não rodaram. `--fail-under 90` também faz a execução falhar, com código de
saída 1, quando menos de 90% das linhas rodaram, de modo que um pipeline pode manter o patamar.

```sh
telvax test tests/ --coverage
telvax test tests/ --fail-under 90
```

```text
coverage (lines that ran / lines with code):
  src/price.tvx  18/20  90.0%   missing 14-15
  total  18/20  90.0%
```

Uma linha conta quando tem um comando, a última expressão de um bloco ou o corpo de um `case`. As funções
`test_`, `fixture_`, `teardown_` e `cases_` não são contadas, nem os arquivos fora do
projeto (o diretório com `telvax.toml`, ou aquele que você testa). A contagem é de linhas, não de
ramos: uma linha com um `if` e os seus dois resultados escritos nela conta uma vez.

## Macros

Uma macro é escrita `name!(...)`. O compilador tem cinco e você não pode definir mais. Cada uma é
reescrita como uma chamada de uma função de `std` antes de o programa ser verificado.

| Macro | O que faz |
|---|---|
| `assert!(cond)` | como `assert`, e a mensagem cita a condição: `assertion failed: x > 3` |
| `assert!(cond, "message")` | interrompe com a sua mensagem quando `cond` é falso |
| `dbg!(value)` | imprime `value_as_written = result` no stderr e devolve `value` |
| `todo!()` | interrompe com `not yet implemented`; pode estar onde qualquer tipo é esperado |
| `unreachable!()` | interrompe com `unreachable code was reached` |
| `panic!("message")` | interrompe com a mensagem |

```telvax
fn pick(n: Int) -> Int:
    if n > 0:
        n
    else:
        todo!()

fn test_pick():
    let x = dbg!(pick(4))  # stderr: pick(4) = 4
    assert!(x == 4)
```

As macros rodam no interpretador, como `assert`; o backend nativo não as compila.

## O REPL

```sh
telvax repl                    # an interactive session
telvax repl app.tvx            # start with the definitions and variables of a file
```

```text
>>> let xs = [3, 1, 2]
>>> xs.len()
3
>>> fn total(v: List[Int]) -> Int:
...     v.sum()
...
>>> total(xs)
6
>>> var n = 0
>>> for x in xs:
...     n = n + x
...
>>> n
6
>>> :type xs
List[Int]
```

Você pode digitar uma definição (`fn`, `type`, `enum`, `trait`, `extend`, `import`), uma variável (`let`, `var`), um
comando ou uma expressão; o valor de uma expressão é mostrado. Um bloco termina com uma linha vazia. Os nomes duram
até `:reset`: um nome pode ser definido de novo (`let x = x + 1`, ou uma nova `fn` com o mesmo nome), e um `let`
não pode receber uma atribuição, como em um arquivo. Cada entrada é verificada com tudo o que foi definido antes dela, então um erro de tipo
mostra a linha e deixa a sessão como estava. Um número escrito sozinho (`let n = 4`) é um `Int`, para que as
funções que recebem um `Int` o aceitem. Uma closure precisa que o tipo dos seus parâmetros seja mantido
(`let inc = (n: Int) => n + 1`).

Os comandos começam com dois-pontos: `:type <expression>`, `:vars`, `:defs`, `:load <file>`, `:reset`, `:help` e
`:quit`. Tab completa nomes, campos e métodos, e as setas percorrem as entradas anteriores (guardadas em
`~/.telvax_history`; `TELVAX_HISTORY` indica outro arquivo). `--allow` limita o que a sessão pode fazer, como em
`telvax run`. Cada entrada roda uma vez: o que ela imprime ou escreve acontece uma vez, e os valores não são calculados de novo
quando a próxima entrada é digitada.

## Depuração

```sh
telvax dap       # a debug adapter on standard input and output
```

No VS Code, F5 em um arquivo Telvax o depura (veja o README da extensão). `telvax dap` fala o [Debug Adapter Protocol](https://microsoft.github.io/debug-adapter-protocol/), então um
editor que tenha um cliente para ele pode rodar um programa no interpretador, pará-lo em breakpoints e avançar passo a passo
por ele. A sua requisição de launch recebe `program` (o arquivo `.tvx`), `function` (padrão `main`), `args` (as palavras
que você daria a `telvax run`), `stopOnEntry`, `noDebug` e `allow` (a mesma lista de `--allow`). Ele para
antes de cada comando, então um breakpoint vai em uma linha que começa um comando; em qualquer outra linha ele se move
para a próxima que começa. Enquanto parado, lista as chamadas em andamento e as suas variáveis, e um nome ou
um campo de uma delas (`p.x`, `xs.0`) pode ser avaliado; outras expressões não. Ele depura o interpretador, não
programas de `telvax build`, e não tem breakpoints condicionais.

## Profiling

```sh
telvax profile app.tvx                       # runs `main` and reports where the time and memory went
telvax profile app.tvx handler 3 "text"      # another function, with its arguments
telvax profile app.tvx --top 30 --json r.json --collapsed stacks.txt --speedscope p.json
```

O relatório tem quatro partes: o tempo total e a memória (o pico, os bytes alocados e quantas
alocações); as funções, com chamadas, o tempo na própria função (`self`) e com as chamadas que ela
faz (`total`), e os bytes que ela alocou; os comandos, com quantas vezes cada um rodou e quanto demorou
até o próximo da mesma chamada começar; e as funções de `std` (`std.tensor.matmul`,
`std.table.sql`, ...), com chamadas, tempo, as linhas ou elementos que entraram e saíram, e os bytes do
resultado. `--json` escreve tudo isso em um arquivo, `--collapsed` escreve pilhas de chamadas para `flamegraph.pl` ou
`inferno`, e `--speedscope` escreve um arquivo para <https://www.speedscope.app>.

Os tempos são medidos pelo interpretador em volta do que o programa faz, então mostram onde o *programa*
gasta o seu tempo e incluem o tempo do próprio interpretador. A memória conta o processo inteiro, então com
várias threads uma função também conta o que as outras threads alocam nesse meio tempo. Programas construídos com
`telvax build` não são perfilados.

## Documentação

`telvax schema data.csv` imprime o `type` das colunas de um arquivo CSV, Parquet ou Arrow, para que você não
o escreva à mão: uma coluna com um valor vazio sai como `T?`, e `--name Row` escolhe o nome do tipo.

`telvax doc` escreve uma referência em Markdown a partir do código: cada função com a sua assinatura, cada `type` com
uma tabela de campos (e valores padrão), cada `enum` com as suas variantes, cada `trait`, e os métodos em `extend`.
As linhas de comentário `#` escritas logo acima de um item (sem linha em branco entre elas) viram a sua descrição.

```sh
telvax doc file.tvx                # print the reference of one file
telvax doc src/ -o REFERENCE.md    # every `.tvx` file under the directory, written to a file
telvax doc --public src/           # leave out names that start with `_`
```

```tvx
# Sum of the first `n` squares.
fn squares(n: Int) -> Int:
    n * (n + 1) * (2 * n + 1) / 6
```

## Formatação e lint

```sh
telvax fmt file.tvx            # rewrite in place
telvax fmt src/ tests/         # every `.tvx` file under the directories (no path: this directory)
telvax fmt --check             # list files that would change (exit 1), for CI
telvax fmt --diff src/         # show the lines that would change, write nothing (exit 1)
telvax lint file.tvx           # unused variables, unused private functions, unreachable cases
telvax lint --list-rules       # the rules and what they find
```

Cada regra de lint tem um nome (`unused-variable`, `unused-function`, `unreachable-case`) e um nível:
`allow` (não dizer nada), `warn` (o padrão) ou `deny` (um erro, de modo que `telvax lint` falha). Defina os
níveis de um pacote em `telvax.toml`, ou para uma execução com `--allow rule`, `--warn rule` e
`--deny rule`; uma flag vence o arquivo. `--deny-warnings` faz qualquer aviso falhar a execução.

```toml
[lint]
unused-variable = "deny"
unused-function = "allow"
```

A indentação faz parte da sintaxe, então `telvax fmt` nunca move uma linha entre blocos: ele define quatro
espaços por nível, os espaços dentro de uma linha, as linhas de continuação e as linhas em branco. Os diretórios pulam as pastas
ocultas, `target`, `node_modules` e `.tvpm`.

O servidor de linguagem completa nomes enquanto você digita: tipos depois de `:` e `->`, os campos, colunas e
métodos depois de um ponto (também depois de chamadas como `s.trim().`), as variantes depois de `Shape.`, o parâmetro de
uma closure que você está escrevendo (`xs.map(p => p.`) e as colunas de uma tabela dentro de `t.sql("..`. O que se encaixa no
tipo que o código espera vem primeiro.

O servidor de linguagem também formata documentos, então o VS Code pode formatar ao salvar. Ele também encontra todas as
referências a uma variável, função, global, tipo, variante de enum, campo ou método e as renomeia (uma
renomeação é recusada quando o novo nome colidiria com outro no escopo, quando um arquivo tem um erro de
sintaxe, ou quando o nome pertence a uma dependência). Elas seguem os imports para os outros arquivos do projeto.

## Strings e listas

```telvax
fn demo(s: Text) -> u64:
    let words = s.split(" ")  # List[Text]
    words.push("extra")
    if s.contains("telvax") and not s.starts_with("#"):
        return words.len()
    0
```

`Text`: `len`, `is_empty`, `trim`, `to_lowercase`, `to_uppercase`, `contains`, `starts_with`,
`ends_with`, `replace`, `split`. `List`: `len`, `is_empty`, `push`, `contains`. `len` é um `u64`, e ao lado de um número com sinal (`xs.len() > limit`, `xs.len() - 1`) os dois são lidos como `Int`. `x in xs` é
`xs.contains(x)`. Um método que não existe é um erro de compilação com uma dica de "você quis dizer".
Esses métodos rodam no interpretador; o backend nativo ainda só tem `len` e `is_empty`.

## Valores aleatórios

`std.random.uuid()` é um novo identificador a cada vez, como `"3f2b8c1e9a4d4b7f8e6a0c5d1b2a3f4e"`, e
`std.random.int(1, 6)` um número inteiro de 1 a 6, ambos incluídos. Eles vêm do sistema operacional, então um id serve
para nomear um job cujo resultado só o seu dono deve ler. Não precisam de permissão, e não há semente.

## Valores opcionais, maps e funções pequenas

```telvax
fn demo(text: Text) -> Int:
    let counts: Map[Text, Int] = std.map.new()
    for word in text.split(" "):
        let seen = counts.get(word).unwrap_or(0)  # Int?
        counts.insert(word, seen + 1)
    match counts.get("telvax"):
        case Some(n):
            n
        case None:
            0
```

* `T?` é `Some(value)` ou `None`; `match` deve cobrir os dois. Métodos: `is_some`, `is_none`,
  `unwrap` (um erro em `None`), `unwrap_or`, `unwrap_or_else(() => v)`, `expect("message")`,
  `map`, `and_then`, `filter` e `ok_or(error)`, que o transforma em um `Result`. Vários métodos retornam um: `List.first`, `last`,
  `pop`, `get`, `Text.find`, `to_int`, `to_float`, `Map.get`, `remove`. Em JSON, `None` é
  `null` e `Some(x)` é simplesmente `x`.
* `Map[K, V]` tem chaves `Text`, inteiras ou `Bool`. Ele é criado com `std.map.new()` e uma
  anotação de tipo, e tem `insert`, `get`, `remove`, `contains_key`, `len`, `is_empty`, `keys` e
  `values`. Como `List`, as cópias compartilham as mesmas entradas. As chaves são mantidas em ordem; objetos JSON usam
  chaves de string.
* `List` também tem `sort` (números, strings, bools) e `reverse`; `Text` tem `chars` e `lines`.
* Funções são valores. O tipo de uma é escrito como um cabeçalho sem o nome:
  `(Int, Int) -> Int`, `() -> Text`, `(Int) -> ()` para uma que não retorna nada. Um nome pode guardar uma
  função (`let f = twice`), e um parâmetro, um campo, um elemento de lista ou um valor de retorno também:
  `fn apply(f: (Int) -> Int, x: Int) -> Int: f(x)`, `fn adder(n: Int) -> (Int) -> Int: x => x + n`.
  O parâmetro de uma closure pode dizer o seu tipo: `(x: Int) => x + 1`. Chamar uma função com o número ou os tipos
  errados de argumentos é um erro de compilação que mostra os dois tipos.
* `x => x + 1` e `(a, b) => a + b` são funções de uma linha. Para mais, escreva uma `fn` dentro da
  função: `fn add(a: Int): a + k`. As duas capturam variáveis por valor, então atribuir a uma variável
  de fora é um erro de compilação; valores `List` e `Map` compartilhados ainda podem ser alterados de
  dentro de uma delas.
* Isto roda no interpretador; o backend nativo ainda não o compila.

## Tabelas e tensores, do jeito curto

```telvax
type Sale:
    id: Int
    region: Text
    units: Int
    price: Float

fn main():
    let sales: Table[Sale] = read_csv("examples/dataframe/sales.csv")
    print(f"mean price: {sales.price.mean()}")

    let by_region: Table[{region: Text, revenue: Float}] = sales.sql("SELECT region, SUM(units * price) AS revenue FROM t GROUP BY region")
    for row in by_region:
        print(f"{row.region}: {row.revenue}")
```

* Cada função e método da biblioteca padrão, com o seu tipo, está na [referência](https://github.com/telvax/telvax/blob/main/docs/reference/std.md).
* `print`, `read_csv`, `read_parquet` e `read_ipc` não precisam do prefixo `std.`. `print` termina a linha;
  `std.io.print` escreve sem terminá-la.
* Os métodos de tensores são as funções de `std.tensor`: `x.matmul(w).relu().softmax()`,
  `sales.price.mean()`. Uma tabela tem `sql`. São as mesmas chamadas por baixo, então
  `std.tensor.relu(x)` continua funcionando.
* Operadores em tensores: `a + b`, `a - b`, `a * b`, `a / b` (mesmo formato) e `a * k`, `k * a` para
  escalar por um número.
* `Table[{region: Text, revenue: Float}]` escreve o tipo da linha no lugar; sem tipo descartável para
  o resultado de uma consulta. As mesmas colunas significam o mesmo tipo em todo lugar.
* `fn main():` pode não retornar nada.

Veja `examples/dataframe/`. A versão em pandas do mesmo programa tem mais ou menos o mesmo tamanho, mas a
do Telvax verifica os nomes e os tipos das colunas antes de rodar.

## Erros, tuplas e iteradores

```telvax
fn split_pair(line: Text) -> Result[(Text, Text), Text]:
    let parts = line.split("=")
    if parts.len() != 2:
        return Err(f"bad line: {line}")
    Ok((parts[0].trim(), parts[1].trim()))

fn port_of(line: Text) -> Result[Int, Text]:
    let key, value = split_pair(line)?  # `?` returns the Err early
    match value.to_int():
        case Some(n):
            Ok(n)
        case None:
            Err(f"not a number: {value}")

fn main():
    let xs = [3, 1, 4, 1, 5]
    let big = xs.filter(x => x > 2).map(x => x * 10)  # [30, 40, 50]
    for i, x in xs.enumerate():
        print(f"{i}: {x}")
    let stack = [1, 2, 3]
    while stack.pop() is Some(top):
        print(f"{top}")
    if xs.first() is Some(first):
        print(f"first = {first}")
```

* `Result[T, E]` tem `is_ok`, `is_err`, `unwrap` e `expect("message")` (os dois interrompem o programa
  com o erro), `unwrap_or(v)`, `unwrap_or_else(e => v)`, `map`, `map_err`, `and_then`, e `ok()` /
  `err()`, que dão um `Option`. `and_then` deve retornar um `Result` com o mesmo tipo de erro; use
  `map_err` para mudá-lo antes. Um método que não existe é um erro de compilação.
* `expr?` em um `Result` ou `Option` retorna o erro (ou `None`) da função atual; o
  tipo de retorno da função também deve ser um `Result` ou `T?`, e os tipos de erro devem ser iguais.
* `if x is Pattern:` e `while x is Pattern:` casam um padrão e ligam os seus nomes; funcionam
  com `break` e `continue`, e `else:` roda quando o padrão não casa.
* Tuplas: o tipo `(A, B)`, valores `(1, "a")`, campos `.0` e `.1`, e desestruturação em
  `let a, b = pair`, laços `for` e `((a, b)) => ...` (um argumento de tupla desmontado).
* Métodos de `List`: `map`, `filter`, `any`, `all`, `find`, `enumerate`, `zip`, `sum`, `min`, `max`,
  `fold`, `for_each`, `skip` e `join`, e `Map.entries`. `min`, `max` e `find` retornam um
  `Option`. Isto roda no interpretador; o backend nativo ainda não o compila.

## Tipos, métodos, traits e generics

```telvax
trait Shape:
    fn area(self) -> Float: ...

    fn describe(self) -> Text:
        f"area {self.area()}"

type Circle(Shape):
    r: Float

    fn area(self) -> Float:
        self.r * self.r * 3.14159

type Stack[T]:
    items: List[T]

    fn push(self, x: T):
        self.items.push(x)
```

Um `type` lista os seus campos e depois os seus métodos. `type Circle(Shape)` diz que o tipo implementa o
trait; um método sem corpo (`: ...`) é obrigatório, um com corpo é um padrão. `extend Circle:`
acrescenta métodos a um tipo declarado em outro lugar. Chame uma função que não tem `self` como `Circle.new(1.0)`.
`[T]` depois de um nome declara um parâmetro de tipo; `T: Shape` o limita a tipos que implementam `Shape`.
Um método pode ter parâmetros de tipo próprios, além dos do seu tipo:
`fn map[U](self, f: (T) -> U) -> Stack[U]`. Eles são inferidos pela chamada (`s.map(x => x * 2)`), e
cada escolha diferente cria a sua própria instância. Um método de um trait não pode tê-los.
Uma função genérica é chamada com os seus tipos inferidos a partir dos argumentos. Quando eles não podem ser (uma
função que não recebe nada), ou para ser explícito, escreva-os antes dos parênteses:
`empty[Text]()`, `pair[Int, Text](1, "a")`. Os tipos são nomes como `Int` ou `List[Text]`.
Três limites não precisam de um trait seu: `T: Num` (`Int`, `Float` e os números com tamanho), `T: Ord`
(números e `Text`, os valores que `<` e `>` comparam) e `T: Eq` (tudo menos funções). Mais de
um: `T: (Num, Ord)`. Um tipo que não cumpre um limite é um erro na chamada que o usa, e um
trait seu chamado `Num`, `Ord` ou `Eq` tem precedência.

Veja os [casos de uso](use-cases.md) para programas completos.
