# Go

> Documentação: [https://go.dev/doc/](https://go.dev/doc/)

---

## Instalação de ferramentas úteis

```bash
go install golang.org/x/tools/gopls@latest
go install honnef.co/go/tools/cmd/staticcheck@latest
```

---

## Estrutura lógica da linguagem Go

### Módulos

Em Go, um módulo é uma coleção de pacotes versionados em conjunto. Ele é definido por um arquivo `go.mod` na raiz de sua árvore de diretórios.

O arquivo define o caminho do módulo, registra dependências e inclui a versão da linguagem Go adotada pelo módulo.

Quando o módulo for publicado, normalmente usaremos o caminho pelo qual ele poderá ser importado. Por exemplo: `go mod init github.com/lcsgborges/project-go`.

### Pacotes

Em Go, um pacote é formado pelos arquivos `.go` de um mesmo diretório que participam da compilação e declaram o mesmo nome de pacote.

Cada pasta representa um pacote. Por isso, todos os arquivos `.go` dentro da mesma pasta normalmente devem declarar o mesmo `package`.

Um pacote chamado `main` pode ser compilado como um programa executável.

Para que um `package main` seja executável, ele precisa possuir uma função `main`:

```go
func main() {
    // Ponto de entrada da aplicação.
}
```

A função `main` é o ponto de entrada da aplicação, ou seja, o local em que a execução do programa começa.

---

## “Olá, mundo!” em Go

```go
package main

import "fmt"

func main() {
	fmt.Println("Olá, mundo!")
}
```

---

## Compilação em Go

Go é uma linguagem compilada, ou seja, precisamos compilar nosso arquivo para transformá-lo em um executável. Para isso, usamos o comando `go build`. Por exemplo:

```bash
go build -o main.exe main.go
```

Nesse caso, usamos a opção `-o` para definir o nome do arquivo de saída como `main.exe`. Para executá-lo em um ambiente compatível, usamos `./main.exe`.

Além disso, em Go, conseguimos fazer compilação cruzada (_cross-compilation_), ou seja, podemos compilar um arquivo `.go` para um sistema operacional diferente do nosso, informando o sistema em que ele será executado e a arquitetura do processador. Por exemplo:

```bash
# Em uma máquina Linux, gera um executável para Windows em AMD64.

GOOS=windows GOARCH=amd64 go build -o main.exe main.go
```

Se quisermos testar o código rapidamente, podemos usar o comando `go run`. Internamente, ele compila o código, executa o programa e remove o executável temporário. Para usá-lo, basta passar o comando e o nome do arquivo. Exemplo: `go run main.go`.

---

## Nomes exportados e não exportados

Em Go, a primeira letra de um identificador determina sua visibilidade. Um nome iniciado por uma letra maiúscula Unicode é exportado e pode ser acessado por outros pacotes. Os demais identificadores ficam restritos ao próprio pacote.

Também podemos definir um nome alternativo para o pacote durante a importação. Por exemplo:

```go
package main

import output "fmt"

func main() {
	output.Println("Olá!")
}
```

No exemplo acima, atribuímos o nome local `output` ao pacote `fmt`.

Podemos importar um pacote com `.` antes do caminho, integrando seus identificadores exportados ao escopo do arquivo atual. Por exemplo, com `import . "fmt"`, podemos usar `Println` sem escrever `fmt.Println`, mas essa prática não é recomendada.

Também podemos usar `_` para importar um pacote. Isso executa a inicialização do pacote sem disponibilizar seus identificadores no arquivo que o importou.

## Funções em Go

Uma função em Go é semelhante às funções de outras linguagens: recebe argumentos, processa dados e pode retornar um resultado.

```go
func multiply(a, b int) int {
    return a * b
}
```

Podemos retornar mais de um valor. Por exemplo: `func swap(a, b int) (int, int) { return b, a }`. Também podemos nomear os valores retornados. Ao fazer isso, é possível usar um *naked return*, escrevendo apenas `return`; esse recurso deve ser usado com cautela, pois pode prejudicar a legibilidade.

Podemos ter funções que retornam outras funções, chamadas de _higher-order functions_. Por exemplo:

```go
package main

import "fmt"

func main() {
	fmt.Printf("Resultado: %d.\n", mult2(5)())
}

func mult2(x int) func() int {
	return func() int {
		return x * 2
	}
}
```

No exemplo acima, a função `mult2` recebe um número e retorna outra função, que multiplica esse número por 2 e devolve o resultado.

Essa função consegue acessar a variável `x` declarada no escopo da função `mult2`. Esse recurso é chamado de _closure_ (fechamento).

Ainda no exemplo acima, temos uma função anônima dentro de `mult2`. Nesse caso, a função anônima forma uma _closure_ porque captura uma variável do escopo externo; nem toda função anônima captura valores dessa forma.

---

## Variáveis em Go

Em Go, as variáveis podem ser declaradas em **escopo de função** ou **escopo de pacote**. As variáveis com escopo de função são visíveis apenas dentro daquela função, enquanto as de escopo de pacote são consideradas **variáveis globais** no pacote.

Ainda podemos declarar variáveis de algumas formas:

```go
var x int = 10 // Declaração com tipo explícito.
var y = 10     // Declaração com inferência de tipo.
z := 10        // Declaração curta, válida somente dentro de funções.

var name, lastName string // Duas variáveis string com valor inicial vazio.
```

Não podemos declarar apenas como `var name`, pois nesse caso, o compilador não sabe o tipo da variável. Na mesma linha, podemos declarar variáveis de tipos diferentes, caso elas já sejam inicializadas com valor: `var name, age = "João", 10`.

---

## Tipos em Go

Em Go, temos alguns tipos bem comuns:

### Inteiros com sinal

- `int`: tipo inteiro padrão; seu tamanho depende da implementação e normalmente é de 32 ou 64 bits.
- `int8`.
- `int16`.
- `int32`; `rune` é um nome alternativo para esse tipo.
- `int64`.

### Inteiros sem sinal (_unsigned_, ou seja, não negativos)

- `uint`: possui o mesmo tamanho que `int` na implementação.
- `uint8`; `byte` é um nome alternativo para esse tipo.
- `uint16`.
- `uint32`.
- `uint64`.

### Números decimais

- `float32`.
- `float64`.

### Números complexos

- `complex64`.
- `complex128`.

### Strings

- `string`.

### Booleanos

- `bool`.

### Conversão de tipos

Também podemos converter um valor de um tipo para outro quando a conversão for permitida. Exemplo:

```go
x := 10
y := float32(x)
```

No exemplo acima, estamos convertendo um valor do tipo `int` em `float32`.

Ao converter um inteiro para `string`, o valor é interpretado como um ponto de código Unicode. Exemplo: `string(65) == "A"`.

---

## Constantes em Go

Em Go, as constantes não podem ter seus valores alterados depois de inicializadas e não podem ser declaradas com a **sintaxe curta**. Exemplo: `const x int = 10`.

Constantes podem representar valores booleanos, caracteres, inteiros, números de ponto flutuante, números complexos e *strings*.

---

## Arrays em Go

Em Go, arrays são estruturas sequenciais em memória. Eles têm tamanho fixo, que não pode ser alterado ao longo da execução. Além disso, o tamanho do array precisa ser uma constante, como o literal `10`, ou um valor declarado com `const`. Exemplo:

```go
func main() {
    const x = 10
    arr := [x]int{}
}
```

Essa declaração é válida porque `x` é uma constante. Os elementos podem ou não ser inicializados na definição do array e seus valores podem ser alterados depois. Também podemos definir um valor para uma posição específica. Por exemplo, em `arr := [10]int{5: 200}`, temos `arr[5] == 200`.

---

## Laços em Go

Em Go, temos apenas o laço `for`; não existe `while` na linguagem. A sintaxe é semelhante à de C, mas não usa parênteses. Exemplo:

```go
func main() {
    sum := 0

    for i := 0; i < 10; i++ {
        sum += i
    }

    fmt.Println("Soma:", sum)
}
```

Entretanto, todos os componentes da declaração do `for` são opcionais. Podemos ter um laço apenas com `for {}`, que equivale a um `while true` em outras linguagens. Nesse caso, podemos usar a palavra `break` para sair do laço.

Temos também a palavra reservada `range`, semelhante à usada em Python. Ao percorrer uma coleção, `range` produz o índice e o elemento correspondente. Caso queiramos apenas o valor, usamos o identificador em branco `_` para ignorar o índice:

```go
package main

import "fmt"

func main() {
	fruits := [5]string{"maçã", "banana", "laranja", "kiwi", "limão"}
	for _, fruit := range fruits {
		fmt.Println(fruit)
	}
}
```

Desde o Go 1.22, podemos usar `range` com um inteiro para repetir um bloco determinada quantidade de vezes:

```go
func main() {
    for range 10 {
        fmt.Println("Olá!") // Imprime a mensagem dez vezes.
    }
}
```

---

## Condicionais em Go

As estruturas condicionais em Go são semelhantes às de outras linguagens: `if`, `else if` e `else`. Uma diferença é a possibilidade de declarar uma variável na própria instrução `if`, o que é útil, por exemplo, no tratamento de erros:

```go
func main() {
    if num := math.Sqrt(4); num < 5 {
        fmt.Println("Menor que 5")
    }
}
```

No exemplo acima, declaramos uma variável cujo valor será `num = 2`. Como esse valor é menor que 5, o programa entra no bloco do `if`.

```go
func main() {
    var x int8 = 10
    if x < 5 {
        fmt.Println("Menor que 5")
    } else if x < 10 {
        fmt.Println("Menor que 10")
    } else {
        fmt.Println("Maior ou igual a 10")
    }
}
```

---
