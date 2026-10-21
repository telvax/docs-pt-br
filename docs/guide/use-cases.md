> Traduzido de [telvax/telvax@664d389](https://github.com/telvax/telvax/commit/664d3890800279185aac3fa7624632fd396b53fd) (`docs/guide/use-cases.md`).

# Casos de uso

Para que o Telvax serve, com um programa para cada caso que você pode rodar a partir da raiz do repositório.
Todo programa abaixo está em [`examples/`](https://github.com/telvax/telvax/tree/main/examples) e é executado pelos comandos mostrados.

| Você quer | Veja | Roda com |
|---|---|---|
| Resumir um arquivo CSV ou Parquet | [`dataframe/sales.tvx`](https://github.com/telvax/telvax/blob/main/examples/dataframe/sales.tvx) | `telvax run` |
| Verificar os formatos de um modelo antes de ele rodar | [`ml/forward.tvx`](https://github.com/telvax/telvax/blob/main/examples/ml/forward.tvx) | `telvax run` |
| Servir uma função por HTTP | [`service/sentiment.tvx`](https://github.com/telvax/telvax/blob/main/examples/service/sentiment.tvx) | `telvax serve` |
| Montar um serviço de RAG | [`rag-service.tvx`](https://github.com/telvax/telvax/blob/main/examples/rag-service.tvx) | `telvax serve` |
| Processar texto | [`text/word-count.tvx`](https://github.com/telvax/telvax/blob/main/examples/text/word-count.tvx) | `telvax run` |
| Tratar erros sem exceções | [`errors/parse-config.tvx`](https://github.com/telvax/telvax/blob/main/examples/errors/parse-config.tvx) | `telvax run` |
| Escrever código numérico simples | [`basics/primes.tvx`](https://github.com/telvax/telvax/blob/main/examples/basics/primes.tvx) | `telvax run`, ou nativo com `telvax build` |

## 1. Análise de dados em tabelas

Uma `Table[Row]` é uma tabela Arrow cujas colunas são os campos do tipo `Row`. Os nomes e os tipos das
colunas são verificados antes de o programa rodar, e uma coluna é lida como um tensor sem ser copiada.

```telvax
type Sale:
    id: Int
    region: Text
    units: Int
    price: Float

fn main():
    let sales: Table[Sale] = read_csv("examples/dataframe/sales.csv")
    print(f"mean price: {sales.price.mean()}")

    let by_region: Table[{region: Text, revenue: Float}] = sales.sql("SELECT region, SUM(units * price) AS revenue FROM t GROUP BY region ORDER BY revenue DESC")
    for row in by_region:
        print(f"{row.region}: {row.revenue}")
```

```sh
telvax run examples/dataframe/sales.tvx main
```

Use para relatórios, preparação de features e verificações rápidas de um arquivo de dados. Uma coluna escrita errado
(`sales.prise`) é um erro de compilação (`no column prise on type Table[Sale]`) em vez de uma falha
no meio da execução. SQL (e Parquet) precisam da build completa; o `telvax-lite` mantém as tabelas na memória e lê CSV.

## 2. Modelos com formatos verificados

Os formatos dos tensores fazem parte do tipo, então um produto de matrizes com formatos incompatíveis não compila:

```telvax
fn layer1(x: Tensor[f32, 2, 3], w: Tensor[f32, 3, 4]) -> Tensor[f32, 2, 4]:
    x.matmul(w).relu()
```

```sh
telvax run examples/ml/forward.tvx main
telvax run examples/ml/forward.tvx main --device gpu      # needs a build with --features gpu
telvax run examples/ml/forward.tvx main --quantize int8   # int8 matrix products
```

Troque os pesos em `layer1` para `Tensor[f32, 4, 4]` e `telvax check` informa
``matmul inner dimensions differ: 3 and 4`` na chamada. Use dimensões simbólicas (`Tensor[f32, N, 3]`) quando um tamanho só é conhecido em tempo de execução.
Aqui os pesos vêm de `std.tensor.from_vec`; um modelo de verdade os carregaria de um arquivo ou de uma tabela.

## 3. Servindo uma função por HTTP

`@endpoint` transforma uma função em uma rota. O tipo do parâmetro é o corpo JSON e é validado
para você; `Err(Status.BadRequest(...))` vira um 400.

```telvax ignore
@endpoint(POST, "/v1/sentiment")
fn sentiment(input: Input) -> Result[Output, Status]:
    if input.text.is_empty():
        return Err(Status.BadRequest("empty text"))
```

```sh
telvax serve examples/service/sentiment.tvx
curl -XPOST localhost:8080/v1/sentiment -H 'content-type: application/json' -d '{"text":"great and good"}'
# {"label":"positive","score":2}
curl -XPOST localhost:8080/v1/sentiment -H 'content-type: application/json' -d '{"text":""}'
# {"error":"empty text"}   (HTTP 400)
```

Para publicá-lo, `telvax build --serve examples/service/sentiment.tvx -o service --dockerfile` produz um único
binário com o programa embutido, mais um Dockerfile.

## 4. Geração aumentada por recuperação

[`examples/rag-service.tvx`](https://github.com/telvax/telvax/blob/main/examples/rag-service.tvx) gera o embedding de uma pergunta, busca num armazém de
vetores, monta um prompt com os resultados e consulta um modelo, tudo em um único `@endpoint`. As etapas são encadeadas com
`|>`:

```telvax ignore
let answer = prompt |> llm.complete() |> std.text.trim()
```

```sh
telvax serve examples/rag-service.tvx          # offline stand-ins for the model and the database
telvax serve examples/rag-service.tvx --live   # Ollama, OpenAI and PostgreSQL with pgvector
```

O modo offline permite desenvolver e testar o serviço sem nenhum desses rodando. `--live`
fala com os de verdade.

## 5. Processamento de texto

`Text`, `List` e `Map` têm os métodos que você espera, e `List` tem métodos de iterador (`map`,
`filter`, `fold`, `sum`, `enumerate`, `zip`, ...):

```telvax
fn count_words(text: Text) -> Map[Text, Int]:
    let counts: Map[Text, Int] = std.map.new()
    for word in text.to_lowercase().split(" "):
        let seen = counts.get(word).unwrap_or(0)
        counts.insert(word, seen + 1)
    counts
```

```sh
telvax run examples/text/word-count.tvx main
```

## 6. Erros como valores

Não há exceções. Uma função que pode falhar retorna `Result[T, E]` ou `T?`, o `match`
deve cobrir todos os casos, e `?` repassa um erro para quem chamou.

```telvax ignore
fn port_of(line: Text) -> Result[Int, Text]:
    let key, value = split_pair(line)?
```

```sh
telvax run examples/errors/parse-config.tvx main
```

## 7. Código numérico simples, interpretado ou nativo

Inteiros, floats, laços e tipos próprios também funcionam como numa linguagem comum. O interpretador roda
qualquer coisa; o backend nativo (`--features llvm`) compila o subconjunto descrito no
[README](https://github.com/telvax/telvax#language-notes).

```sh
telvax run examples/basics/primes.tvx main
telvax build examples/basics/primes.tvx -o primes     # native, needs --features llvm
```

## Além de um arquivo

* **Vários arquivos e pacotes:** `from util import helper` e `tvpm`, veja o [tour](tour.md#módulos-e-pacotes).
* **Várias máquinas:** `std.dist.map` com `telvax worker`, veja o [tour](tour.md#distribuição).
* **A partir do Python:** o módulo `python/` troca tabelas Arrow e arrays NumPy sem copiar.
* **No navegador:** o playground roda o verificador e o interpretador como WebAssembly.
* **Editor:** `telvax lsp` e a extensão do VS Code dão diagnósticos, hover e autocompletar.

## Quando o Telvax ainda não é a ferramenta certa

Ele está em alfa. O backend nativo cobre um subconjunto pequeno, a saída PTX e StableHLO não foi executada
em GPUs NVIDIA nem em TPUs, e a sintaxe ainda pode mudar. [`docs/design/status.md`](https://github.com/telvax/telvax/blob/main/docs/design/status.md)
lista o que foi verificado e como.
