> Traduzido de [telvax/telvax@1e3cb31](https://github.com/telvax/telvax/commit/1e3cb318a8ca48be7bb78dea5ac1e8a1db835cc0) (`CONTRIBUTING.md`).

# Como contribuir com o Telvax

Obrigado por experimentar o Telvax. Na versão 0.0.x, a contribuição mais útil é um relato de bug ou um programa que
não se comportou como você esperava.

## Relatando problemas

Abra uma issue e inclua:

* a saída de `telvax --version` e o seu sistema operacional;
* o menor programa `.tvx` que mostra o problema;
* o que você executou (`telvax run ...`, `telvax serve ...`) e a saída completa.

Ideias para a linguagem também são bem-vindas, de preferência com um pequeno exemplo do código que você gostaria de poder escrever.

Uma mudança na sintaxe, no sistema de tipos ou na biblioteca padrão começa como um RFC em
[telvax/rfcs](https://github.com/telvax/rfcs): veja o README dele para conhecer o processo. O pull request aqui
faz referência ao RFC aceito. As traduções da documentação estão em `telvax/docs-pt-br`, `docs-es` e
`docs-zh-cn`; os arquivos em inglês em `docs/` continuam sendo a fonte.

## Compilando

O Rust nightly é obrigatório; `rust-toolchain.toml` o seleciona.

```sh
cargo build --release -p telvax_cli      # target/release/telvax
cargo test --workspace                  # default build, no LLVM needed
```

Peças opcionais:

| Recurso | O que acrescenta | Exige |
|---|---|---|
| `--features telvax_cli/llvm` | código nativo, `--emit ptx` | LLVM 22 com MLIR (`. scripts/llvm-env.sh`) |
| `--features telvax_cli/gpu` | `--device gpu` | uma GPU com Metal, Vulkan ou DX12 |

Os bindings de Python (`python/`), a extensão do VS Code (`editors/vscode`), o playground em Wasm
(`playground/`) e o site de documentação (`site/`, publicado por versão, veja o README dele) têm o seu próprio README e os seus testes.

## Como as mudanças chegam à `main`

A `main` é protegida: nada é enviado diretamente a ela, nem mesmo pelo mantenedor. O trabalho acontece em um
branch e chega à `main` por um pull request, e o pull request precisa passar em todas as verificações
(formatação, clippy, testes em Linux e Windows, os builds de LLVM e GPU, o build lite, os
bindings de Python, o playground em um navegador real, a extensão em um VS Code real, a imagem
Docker, `cargo deny` e a varredura de segredos) antes de poder ser mesclado. Os merges são feitos com squash, então o
título do pull request vira a mensagem do commit: mantenha-o curto e no imperativo (`feat: add ...`).

```sh
git switch -c fix/short-name
# ...edit, commit...
git push -u origin fix/short-name
gh pr create --fill
gh pr merge --auto --squash      # merges by itself once every check is green
```

Você ainda pode executar as verificações localmente enquanto trabalha (veja abaixo), mas o que vale é um pull request
verde.

## Antes de enviar um pull request

```sh
cargo fmt --all
cargo clippy --workspace --all-targets
cargo test --workspace
cargo deny check
```

* O código nos documentos é testado (`cli/tests/docs.rs`): um bloco ```` ```telvax ```` precisa passar em `telvax check` e
  `telvax fmt --check`, e ser executado quando tem `fn main`. Marque um trecho de programa com ```` ```telvax ignore ````.
  Os arquivos em `examples/` passam pelas mesmas verificações.
* Acrescente um teste junto com a mudança. Os testes do compilador ficam ao lado do crate (`compiler/*/tests`), os testes do
  runtime em `runtime/tests`.
* Mantenha os commits curtos e no imperativo, com os prefixos que já existem: `feat:`, `fix:`, `docs:`,
  `test:`, `refactor:`, `style:`, `ci:`, `build:`, `chore:`.
* Se uma mudança altera a linguagem, atualize `docs/guide/tour.md` e `CHANGELOG.md`.
* Os programas em `tests/editions/` são congelados (veja `docs/guide/stability.md`). Uma mudança não pode fazer
  um deles falhar, e não pode editar nenhum. Se não consegue evitar isso, é uma mudança incompatível: uma nova edição e
  um título `feat!:`. Uma sintaxe nova ganha um novo programa congelado e uma linha no `CHECKSUMS` daquele diretório.
* `docs/design/status.md` diz o que foi e o que não foi executado em hardware real. Mantenha-o honesto: não
  marque algo como funcionando só porque compila.

## Onde as coisas estão

Veja a tabela no README. Um mapa rápido da linguagem em si: `compiler/lexer` (tokens),
`compiler/parser` (sintaxe), `compiler/semantic` (tipos e verificações), `runtime` (interpretador),
`compiler/ir` e `compiler/codegen` (nativo), `compiler/accel` (PTX e StableHLO).

Uma nova função da biblioteca padrão vai em `compiler/semantic/src/stdlib.rs` (o seu tipo) e
`runtime/src/std_impl.rs` (o que ela faz). Descreva-a também em `compiler/semantic/src/stdlib_docs.rs`: um teste
falha sem a descrição, e `docs/reference/std.md` é escrito a partir dela com
`UPDATE_REFERENCE=1 cargo test -p telvax_hir --test reference`.

## Lançamentos

A `main` é lançada automaticamente **uma vez por semana** (segunda-feira, 12:00 UTC), e o **título** de cada pull request mesclado desde o último lançamento (um
[Conventional Commit](https://www.conventionalcommits.org)) escolhe a versão, então nenhum arquivo precisa ser
editado. Um lançamento leva uma hora, e um lançamento leva todos os merges da semana: o título de **cada**
commit desde a última tag conta, e vence o maior incremento (um `feat:` e depois um `fix:` é um lançamento
minor). Para lançar mais cedo, execute o workflow Release à mão:

| Título | Versão | Exemplo |
|---|---|---|
| `feat!: ...` ou uma linha `BREAKING CHANGE:` no corpo | major (`1.2.3` para `2.0.0`; enquanto o major é 0, minor) | `feat!: rename the pipe operator` |
| `feat: ...` | minor (`0.1.2` para `0.2.0`) | `feat: add traits` |
| `fix:`, `perf:`, `refactor:`, `build:`, `revert:` ou sem prefixo | patch (`0.1.2` para `0.1.3`) | `fix: empty table in sql` |
| `docs:`, `ci:`, `test:`, `style:`, `chore:` | nenhum lançamento | `docs: add a use case` |

Acrescente `[skip release]` a um título e esse merge não conta para nada; isso não cancela os outros. Para escolher você mesmo uma versão, aumente
`version` no `Cargo.toml` da raiz acima da última tag. As notas do lançamento são a seção do `CHANGELOG.md`
dessa versão, se você escreveu uma, seguida da lista de pull requests mesclados.

## Segurança

Veja [SECURITY.md](security.md).
