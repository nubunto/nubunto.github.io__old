---
title: GraphQL + Go + Mongo + Docker
draft: true
---

E aí galera!

Tô de olho na falta de material que existe em português tanto pra GraphQL quanto pra Go, então resolvi fazer esse blog post maroto que vai ensinar vocês um pouco dos dois.

Por isso, vou assumir que o leitor já é familiar com todas as tecnologias, e apresentarei-as brevemente, pulando direto para o código.

Bora começar!

# GraphQL

GraphQL é uma especificação de uma "query language". Basicamente, uma linguagem que permite que você possa fazer queries em forma de grafos para resgatar dados de uma API.

Pra mais detalhamento sobre o que exatamente é o GraphQL, veja [esse link](https://graphql.org). Por enquanto, você pode imaginar que GraphQL é uma alternativa ao REST, que permite maior granularidade ao buscar dados.

# Go

Uma linguagem que, na minha opinião, é uma das melhores das mais modernas. Sintaxe simples, bem expressiva, muito rápida, e vem ganhando um destaque fenomenal nos últimos anos.

Muitas empresas vêm adotando no Brasil, e algumas delas até patrocinaram a GopherCon BR esse ano! [Saca só quais são](https://2017.gopherconbr.org/#sponsors).

# MongoDB

Um banco de dados JSON. É só isso mesmo. [Site aqui](https://mongodb.org) caso você queira mais a fundo ou achar a documentação!

# Começando

Se você ainda não tem um ambiente Go (i.e. uma GOPATH) ou as ferramentas, essa é a hora de ir até o [site do Go](https://golang.org) e baixar tudo. Eu te espero.

Agora, vamos criar um diretório pra começarmos a brincadeira. Eu recomendo você a utilizar seu usuário do github pra isso, então:

    $ mkdir -p $HOME/go/src/github.com/<SEU USUARIO>/graphgo

onde, óbvio, `<SEU USUÁRIO>` é seu usuário do GitHub. Se você ainda não tem um GitHub, essa é uma boa hora de fazer um. Pode ir lá, eu te espero.

Ótimo. Agora, vamos começar com o boilerplate mínimo de qualquer programa Go:

```go
package main

import "fmt"

func main() {
    fmt.Println("hello, world!")
}
```

Eu marcho pro meu 2º ano de programação com Go e ainda começo projetos inteiros assim.

Vamos ver esse arquivinho compilar. Navegue usando um terminal até seu diretório escolhido acima, e digite:

    $ go build -o graphgo
    $ ./graphgo

Se tudo der certo, você deve ter visto o seguinte:

    hello, world!

escrito no seu terminal. Se sim, ótimo! Tudo certo!

> Nota: tenha certeza de que, para o próximo passo, você tem uma instalação Go funcional, i.e. uma $GOPATH setada ou Go 1.8+

Agora, vamos instalar a lib de servidores GraphQL para Go:

    $ go get github.com/graphql-go/graphql

Se conseguiu baixar sem erros, vamos adicionar um servidor GraphQL mínimo para testarmos!

