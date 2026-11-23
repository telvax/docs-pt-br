> Traduzido de [telvax/telvax@2134c83](https://github.com/telvax/telvax/commit/2134c838344b99b31e44f7ba0bb2bde2ea0017e5) (`SECURITY.md`).

# Política de segurança

## Versões suportadas

O Telvax é um alfa inicial. Só a última versão recebe correções de segurança; as versões mais antigas não são
corrigidas, então atualize antes de relatar.

## Relatando uma vulnerabilidade

Relate as vulnerabilidades **em particular** pelo GitHub: abra a aba Security do repositório e escolha
"Report a vulnerability". Não abra uma issue, um pull request nem uma discussão públicos para um
problema de segurança.

Inclua o que puder entre: a versão afetada (`telvax --version`), o seu sistema operacional, um programa
`.tvx` mínimo ou uma requisição que reproduza o problema, e o que você esperava que acontecesse.

## O que esperar

* Nosso objetivo é confirmar o recebimento de um relato em até **7 dias** e dizer se o consideramos uma
  vulnerabilidade em até **14 dias**. Este é um projeto pequeno, então esses prazos são metas, não garantias.
* Uma vulnerabilidade confirmada é corrigida em um novo lançamento, e o aviso é publicado depois que o
  lançamento sai. Damos crédito a quem relata e quer ser mencionado.
* Pedimos que nos dê um tempo razoável para corrigir o problema antes de divulgá-lo publicamente.

## Escopo

Dentro do escopo: os binários `telvax` e `tvpm`, o servidor de linguagem, o servidor HTTP que `telvax serve`
e `telvax build --serve` produzem (tratamento de requisições, validação), o gerenciador de pacotes e o seu
cliente de registro, os scripts de instalação e a extensão do VS Code.

Fora do escopo: o conteúdo dos programas que você escolhe executar (o interpretador executa o código que recebe),
a negação de serviço por entradas deliberadamente enormes para um servidor de desenvolvimento, e os problemas em serviços
de terceiros que o Telvax pode chamar (Ollama, OpenAI, PostgreSQL).
