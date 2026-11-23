> Traduzido de [telvax/telvax@0524f5d](https://github.com/telvax/telvax/commit/0524f5db55c7dfdef801ffd31e3feb8eb2f30719) (`docs/guide/stability.md`).

# Estabilidade da sintaxe

Um programa que funciona hoje continua funcionando. A sintaxe é agrupada em **edições**. A única edição até agora
é a `2026`, e a sua sintaxe está congelada: a partir da versão que contém esta página, um programa escrito na
edição 2026 é aceito por todo Telvax posterior que leia a edição 2026.

## O que é prometido

* **Programas não quebram.** Todo programa em `tests/editions/2026/` deve continuar passando em
  `telvax check`, em `telvax fmt --check` e, quando tem funções `test_...`, em `telvax test`. O teste
  que os executa é `cli/tests/editions.rs`.
* **Os arquivos são congelados.** `tests/editions/2026/CHECKSUMS` guarda um hash de cada programa. Alterar um
  programa, ou adicionar um sem a sua linha, faz o teste falhar. Um pull request que precisa editar um programa
  congelado é uma mudança incompatível e exige uma nova edição (abaixo).
* **Sintaxe nova pode ser adicionada.** Uma palavra, uma forma ou uma função da biblioteca padrão que não faz
  nenhum programa antigo deixar de funcionar é uma mudança `feat:` comum. Os programas novos também entram no
  diretório congelado, então a promessa cresce.
* **As mensagens de erro não fazem parte dela.** O texto de um diagnóstico, os avisos do lint e a saída de
  `telvax fmt` para código que o formatador rejeitava antes podem mudar. O formatador não muda o
  layout de um programa que já está formatado.

## O que não é prometido

* A biblioteca `std` pode ganhar funções, e uma função pode ganhar uma nova sobrecarga. Remover ou mudar uma
  segue a mesma regra da sintaxe: uma nova edição.
* O backend nativo, a execução em GPU e as saídas para aceleradores estão listados em
  `docs/design/status.md`; o que roda ali pode crescer, e um programa que não compilava nativamente pode passar a compilar.
* A correção de um bug pode rejeitar um programa que só funcionava por causa do bug (um erro de tipo que o verificador deixou passar).
  A mudança diz isso em `CHANGELOG.md`.

## Escolhendo uma edição

`telvax.toml` dá o nome da edição de um pacote:

```toml
[package]
name = "app"
version = "0.1.0"
edition = "2026"
```

* Sem a linha, o pacote é lido como a edição mais recente que o compilador conhece.
  `tvpm new` a escreve, então um pacote novo já nasce fixado.
* Uma edição que o compilador não conhece é um erro que lista as que ele conhece e manda atualizar o
  Telvax se o pacote for mais novo.
* Um arquivo solto, sem `telvax.toml`, é lido como a edição mais recente.

## Uma nova edição

Uma mudança que faz um programa antigo deixar de compilar, ou muda o seu significado, entra em uma nova edição
e não na existente:

1. A edição antiga continua compilando como antes. O compilador lê as duas, escolhidas por `edition`.
2. O novo diretório `tests/editions/<ano>/` recebe os seus próprios programas congelados e o seu `CHECKSUMS`, e o
   ano é acrescentado a `EDITIONS` em `tools/package-manager/src/manifest.rs`.
3. `CHANGELOG.md` lista o que mudou, e o `telvax-migrate` (`tools/migrate`) ganha uma regra que reescreve o
   código antigo para a nova edição onde isso puder ser feito de forma mecânica.
4. Por quanto tempo uma edição antiga continua suportada é decidido quando a segunda edição é proposta; remover
   uma é uma mudança `feat!:` à parte.

## O que existe hoje

Pronto: a linha `edition` e a sua verificação, os programas congelados e o teste que os protege, e esta
página. Ainda não pronto: uma segunda edição (então a leitura de duas edições não foi testada), avisos para formas
obsoletas, e um comando `telvax migrate` dentro do binário `telvax`. A ferramenta `telvax-migrate`
converte a antiga sintaxe no estilo Rust e ainda é um programa separado, que será removido.
