> Traduzido de [telvax/telvax@2253a54](https://github.com/telvax/telvax/commit/2253a54139c057b14d91eeded2259ee66e3a2148) (`README.md`).

<div align="center">

# Telvax

**Uma linguagem compilada e estaticamente tipada para dados, pipelines de ML e serviços de IA.**

[![CI](https://github.com/telvax/telvax/actions/workflows/ci.yml/badge.svg)](https://github.com/telvax/telvax/actions/workflows/ci.yml)
[![Release](https://img.shields.io/github/v/release/telvax/telvax?include_prereleases&label=release)](https://github.com/telvax/telvax/releases)
[![License](https://img.shields.io/badge/license-MIT%20%2F%20Apache--2.0-blue)](#licença)

[Documentação](https://telvax.github.io/telvax/) ·
[Tour](guide/tour.md) ·
[Biblioteca padrão](https://github.com/telvax/telvax/blob/main/docs/reference/std.md) ·
[Linha de comando](https://github.com/telvax/telvax/blob/main/docs/reference/cli.md) ·
[Discussões](https://github.com/telvax/telvax/discussions)

</div>

O Telvax é uma linguagem de programação. Você escreve programas em arquivos de texto que terminam em `.tvx` e os executa com
o comando `telvax`, que você instala uma única vez. Ele foi feito para programas que trabalham com dados (tabelas, CSV,
Parquet), aprendizado de máquina (tensores) e serviços de IA (um servidor web que responde a requisições JSON).
Se você nunca o usou, comece pelo [Início rápido](#início-rápido): ele não precisa de nenhum outro software.

O Telvax se lê como Python (blocos por indentação, sem chaves nem ponto e vírgula) e verifica como uma linguagem
compilada. Os formatos dos tensores fazem parte do sistema de tipos, um operador de pipeline (`|>`) compõe etapas, e uma
função marcada com `@endpoint` vira uma rota HTTP, com a validação e o tratamento de JSON gerados para ela.

```telvax
import std.ai.embeddings.OllamaEmbedder
import std.http.Status

let embedder = OllamaEmbedder.new("nomic-embed-text")

type QueryPayload:
    query: Text
    top_k: i32

fn embed(text: Text) -> Tensor[f32, 768]:
    text |> std.text.clean_whitespace() |> std.text.truncate(512) |> embedder.embed()

@endpoint(POST, "/v1/embed")
fn handle(payload: QueryPayload) -> Result[Tensor[f32, 768], Status]:
    if payload.query.is_empty():
        return Err(Status.BadRequest("empty query"))
    Ok(embed(payload.query))
```

Isto é um serviço: ele precisa de um [Ollama](https://ollama.com) rodando para responder. O serviço de RAG completo está em
[`examples/rag-service.tvx`](https://github.com/telvax/telvax/blob/main/examples/rag-service.tvx), e roda offline com substitutos para os modelos.
Um programa que não exige nenhuma preparação está no [Início rápido](#início-rápido).

> **Alfa inicial.** Bom para experimentar a linguagem e dar feedback. Espere bugs e mudanças incompatíveis
> na biblioteca e nas ferramentas; a sintaxe da edição 2026 está congelada ([estabilidade](https://github.com/telvax/telvax/blob/main/docs/guide/stability.md)).
> O que funciona, e o que ainda não foi executado em hardware real, está listado em [`docs/design/status.md`](https://github.com/telvax/telvax/blob/main/docs/design/status.md).

## Destaques

* **Formatos de tensores nos tipos.** `Tensor[f32, 2, 3]` carrega o seu formato, então um `matmul` com formatos incompatíveis é um erro de compilação.
* **Tabelas tipadas.** `Table[Row]` é uma tabela Arrow cujas colunas são os campos de `Row`; ela lê CSV, Parquet e Arrow, e roda SQL.
* **Serviços sem cola.** `@endpoint` transforma uma função em uma rota JSON validada; `telvax build --serve` gera um único binário e um Dockerfile.
* **Erros são valores.** `Result`, `Option`, `T?`, o operador `?` e `if x is Some(v)`.
* **Ferramentas incluídas.** Formatador, verificador de tipos com trechos exatos, REPL, executor de testes com cobertura, profiler, depurador (DAP), servidor de linguagem, extensão do VS Code, gerenciador de pacotes (`tvpm`) e um kernel do Jupyter.
* **Interpretador e backend nativo.** Rode um programa na hora, ou compile-o com MLIR e LLVM; geradores de StableHLO e PTX estão disponíveis para aceleradores.

## O que você pode construir

| Caso de uso | Exemplo |
|---|---|
| Resumir um arquivo CSV / Parquet com colunas tipadas e SQL | [`examples/dataframe/sales.tvx`](https://github.com/telvax/telvax/blob/main/examples/dataframe/sales.tvx) |
| Modelos cujos formatos de tensor são verificados antes de rodar | [`examples/ml/forward.tvx`](https://github.com/telvax/telvax/blob/main/examples/ml/forward.tvx) |
| Transformar uma função em uma rota HTTP JSON validada | [`examples/service/sentiment.tvx`](https://github.com/telvax/telvax/blob/main/examples/service/sentiment.tvx) |
| Um serviço de RAG: embeddings, busca vetorial, um LLM | [`examples/rag-service.tvx`](https://github.com/telvax/telvax/blob/main/examples/rag-service.tvx) |
| Processamento de texto com `Map`, `List` e métodos de iterador | [`examples/text/word-count.tvx`](https://github.com/telvax/telvax/blob/main/examples/text/word-count.tvx) |
| Erros como valores: `Result`, `Option`, `?`, `if x is Some(v)` | [`examples/errors/parse-config.tvx`](https://github.com/telvax/telvax/blob/main/examples/errors/parse-config.tvx) |

Cada um vem com os comandos para rodá-lo em [`guide/use-cases.md`](guide/use-cases.md).

## Instalar

```sh
curl -fsSL https://raw.githubusercontent.com/telvax/telvax/main/scripts/install.sh | sh
```

Nada se abre quando termina: o Telvax é uma ferramenta de linha de comando, sem janela e sem app. O script
imprime uma linha e coloca dois programas, `telvax` e `tvpm` (o gerenciador de pacotes), em `~/.telvax/bin`.
No macOS e no Linux ele não altera o seu `PATH`, então adicione a pasta você mesmo (ponha a linha em `~/.zshrc` ou `~/.bashrc`
para mantê-la) e confira:

```sh
export PATH="$HOME/.telvax/bin:$PATH"
telvax --version        # prints: telvax, then the version you installed
```

Se o shell responder `command not found: telvax`, falta a linha do `PATH`. Se não houver binário
para o seu sistema, o script compila a partir do código-fonte (precisa do [Rust](https://rustup.rs)) e o `cargo` coloca os
programas em `~/.cargo/bin`. Para compilar à mão: `cargo install --locked --path cli`.

Duas builds são publicadas para cada sistema: o `telvax` completo (cerca de 50 MB, com SQL e Parquet)
e o `telvax-lite` (cerca de 10 MB). A build lite tem todo o resto, tabelas em memória e CSV
inclusive; `std.table.sql` e o Parquet informam um erro claro nela. `TELVAX_LITE=1` faz o script de instalação escolher a lite.

Cada release também publica uma imagem Docker para `linux/amd64` e `linux/arm64`, com `telvax` e `tvpm`:

```sh
docker run --rm -v "$PWD:/work" -w /work ghcr.io/telvax/telvax run app.tvx main
```

As tags são a versão do release (veja os [releases](https://github.com/telvax/telvax/releases)) e `latest`.

No Windows (PowerShell):

```powershell
irm https://raw.githubusercontent.com/telvax/telvax/main/scripts/install.ps1 | iex
```

No Windows o script adiciona a pasta ao seu `PATH` (abra um novo terminal). As builds para Windows são novas e só foram compiladas e testadas pelo CI, não à mão. O backend
nativo (`--features llvm`) ainda não está disponível no Windows.

## Início rápido

Instale o Telvax (veja [Instalar](#instalar)) e salve isto como `hello.tvx`:

```telvax
fn main():
    let names = ["Ada", "Alan", "Grace"]
    for name in names:
        print(f"hello, {name}")
```

```sh
telvax run hello.tvx main
```

Ele imprime:

```text
hello, Ada
hello, Alan
hello, Grace
```

`main` é a função a executar. `telvax check hello.tvx` só lê e verifica os tipos do arquivo e imprime
`` `hello.tvx` ok ``; um erro é informado com a linha e a coluna. Mais comandos:

```sh
telvax check hello.tvx          # parse and type-check; diagnostics point at the exact span
telvax repl                     # type definitions and expressions and see the result
telvax fmt --check              # every .tvx file under this directory
telvax test tests/ --coverage   # run the `test_*` functions
telvax serve examples/rag-service.tvx   # HTTP server on :8080 (offline model stand-ins)
```

Todo comando e toda flag estão em [`docs/reference/cli.md`](https://github.com/telvax/telvax/blob/main/docs/reference/cli.md). Para notebooks,
`pip install "telvax[kernel]"` adiciona um kernel do Jupyter e a magic `%%telvax` ([`python/README.md`](https://github.com/telvax/telvax/blob/main/python/README.md)). Para compilar a partir de um clone,
é preciso o Rust nightly (o `rust-toolchain.toml` o seleciona): `cargo build --release -p telvax_cli`.

## Documentação

| Leia | Para |
|---|---|
| [Tour](guide/tour.md) | a linguagem, do primeiro programa a módulos e testes |
| [Casos de uso](guide/use-cases.md) | dataframes, modelos, serviços e RAG, com os comandos para rodar cada um |
| [Biblioteca padrão](https://github.com/telvax/telvax/blob/main/docs/reference/std.md) | cada função e método de `std` |
| [Linha de comando](https://github.com/telvax/telvax/blob/main/docs/reference/cli.md) | cada comando de `telvax` e `tvpm` |
| [Estabilidade](https://github.com/telvax/telvax/blob/main/docs/guide/stability.md) | o que está congelado e o que pode mudar |
| [Status](https://github.com/telvax/telvax/blob/main/docs/design/status.md) | o que existe, o que foi verificado e como, e o que está em aberto |
| [Sintaxe](https://github.com/telvax/telvax/blob/main/docs/design/sintaxe-pythonica.md) | o design da sintaxe |

A documentação também é um site: <https://telvax.github.io/telvax/>.

## Notas sobre a linguagem

* **Valores.** Os tipos que você define e `Text` são valores; uma `List` é uma referência (as cópias compartilham elementos).
  Tensores são valores imutáveis. Literais numéricos sem tipo recebem o tipo a partir de como são usados.
* **Tensores.** `Tensor[f32, 2, 3]` carrega o seu formato: `std.tensor.matmul` com formatos
  incompatíveis é um erro de compilação. Dimensões simbólicas (`[N]`) aceitam qualquer tamanho.
* **Tabelas.** `Table[Row]` é uma tabela Arrow cujas colunas são os campos do tipo `Row`;
  `t.score` é a coluna como um tensor sem cópia. `head`, `tail`, `sort_by`, `sort_by_desc`, `select`, `group_by`, `filter`, `concat` e `split`
  são métodos; `sql` roda uma consulta. `read_ipc` / `write_ipc` usam arquivos Arrow IPC (Feather), mapeados em memória na
  leitura; `read_flight` / `write_flight` falam com um servidor Arrow Flight (compile com `--features flight`). Uma coluna do tipo `T?` pode ter nulos, e as colunas `Date` / `Timestamp` são tipos temporais do Arrow cujos
  valores são `Text` ISO-8601.
* **Módulos.** `import util.helper`, `from crate.geometry import Point, norm`,
  `from mathx.vec import dot` (uma dependência). Um nome que começa com `_` é privado e não pode
  ser importado.
* **Subconjunto nativo.** Inteiros, floats, bools, strings, tipos que você define, listas de números, tensores
  `f32`/`f64` de formato estático, laços e f-strings (ints, bools, strings). Funções seta (`x => ...`), `Table`, SQL e a maior parte
  de `std` rodam apenas no interpretador.

## Estrutura do repositório

| Área | Onde | Notas |
|---|---|---|
| Front end do compilador | `compiler/{lexer,parser,ast,semantic}` | `logos` + `chumsky` com recuperação de erros; HIR tipada; inferência de tipo de literais; formatos de tensor verificados estaticamente |
| Interpretador e runtime | `runtime/` | Tensores e tabelas sobre Arrow, CSV / Parquet / SQL (DataFusion), clientes reais de Ollama / OpenAI / pgvector, SQL no PostgreSQL |
| Backend nativo | `compiler/{ir,codegen}` | MLIR (`melior`) e LLVM (`inkwell`) 22; JIT e executáveis; quantização int8 de `matmul`. Opcional: `--features llvm` |
| Serviço HTTP | `server/`, `cli/` | `@endpoint` vira uma rota Axum; `telvax build --serve` embute o programa em um único binário |
| Módulos e pacotes | `compiler/modules`, `tools/package-manager` | `import` entre arquivos, `tvpm` com registries, um resolvedor e um lockfile |
| Suporte a editores | `tools/lsp`, `editors/vscode` | diagnósticos, hover, autocompletar contextual, ir para a definição, dicas de tipos inferidos |
| Playground web | `playground/` | o verificador e o interpretador compilados para WebAssembly, com um console (um REPL) |
| Aceleradores | `compiler/accel/` | geradores de StableHLO e PTX (não executados em hardware TPU / NVIDIA); a execução em GPU fica em `runtime` (feature `gpu`) |
| Python | `python/` | módulo PyO3; troca sem cópia de Arrow, NumPy e arrays CUDA; executor Ray |
| Distribuição | `tools/cluster` | `telvax worker` + `--workers` para `std.dist.map` |
| Benchmarks | `benchmarks/` | metas da alfa: overhead de RAG, uso de memória, vazão de ingestão |

[`docs/design/status.md`](https://github.com/telvax/telvax/blob/main/docs/design/status.md) mapeia cada item da especificação de engenharia original
para o que existe, o que foi verificado e como, e o que ainda está em aberto.

## Testes

Os programas Telvax têm o seu próprio executor de testes: uma função chamada `test_...` é um teste, as funções `fixture_...`
dão a ele os parâmetros, `cases_...` o roda com vários valores, `std.test.mock` substitui uma
função, e `--coverage` e `--fail-under N` informam e seguram as linhas que os testes executaram. O
[tour](guide/tour.md#testes) tem os detalhes. Para testar este repositório:

```sh
cargo test --workspace                       # no LLVM needed
. scripts/llvm-env.sh && cargo test --workspace --features llvm
cd python && maturin develop && python -m pytest tests
scripts/build-playground.sh --node && node --test playground/test
cd editors/vscode && npm ci && npm test
```

## Contribuindo

Leia primeiro o [CONTRIBUTING.md](https://github.com/telvax/telvax/blob/main/CONTRIBUTING.md). Uma mudança na sintaxe, no sistema de tipos ou na biblioteca
padrão começa como um RFC em [`telvax/rfcs`](https://github.com/telvax/rfcs).

## Licença

Licenciado sob MIT (`LICENSE-MIT`) ou Apache 2.0 (`LICENSE-APACHE`): você pode usar o Telvax sob
qualquer uma das duas, à sua escolha.

## Comunidade

Perguntas e ideias vão para as [Discussões](https://github.com/telvax/telvax/discussions); bugs e
pedidos de funcionalidades para as [issues](https://github.com/telvax/telvax/issues). Leia antes o [código de conduta](https://github.com/telvax/telvax/blob/main/CODE_OF_CONDUCT.md) e informe
problemas de segurança em particular, como descrito em [SECURITY.md](https://github.com/telvax/telvax/blob/main/SECURITY.md).

## Outros repositórios

| Repositório | O que contém |
|---|---|
| [`telvax/rfcs`](https://github.com/telvax/rfcs) | propostas de mudanças na linguagem e nas ferramentas |
| [`telvax/docs-pt-br`](https://github.com/telvax/docs-pt-br), [`docs-es`](https://github.com/telvax/docs-es), [`docs-zh-cn`](https://github.com/telvax/docs-zh-cn) | traduções do tour, dos casos de uso e deste README |
| [`telvax/website`](https://github.com/telvax/website) | o site público (ainda não construído; o site da documentação é `site/` aqui) |
| [`telvax/workflows`](https://github.com/telvax/workflows) | workflows compartilhados do GitHub Actions (ainda nenhum) |
| [`telvax/telvax-grammar`](https://github.com/telvax/telvax-grammar) | uma cópia da gramática TextMate, para ferramentas que a obtêm de um repositório (GitHub Linguist) |
| [`telvax/.github`](https://github.com/telvax/.github) | perfil da organização e arquivos de comunidade padrão |
