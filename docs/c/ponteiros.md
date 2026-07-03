# Ponteiros

**Ponteiro** é uma variável que guarda um endereço de memória.

Para compreender o uso de ponteiros, precisamos entender três elementos:

1. Valor.
2. Endereço.
3. Tipo apontado.

## Variável comum e ponteiro

Quando escrevemos `int x = 10`, temos uma variável chamada `x`. Ela ocupa um espaço na memória, como representado a seguir:

```text
endereço 0x2000

+----+
| 10 | <--- x
+----+
```

Quando fazemos `int *p = &x;`, `p` é um ponteiro para `int` e guarda o **endereço** de `x`.

```text
endereço 0x2000

+----+
| 10 | <--- x
+----+


endereço 0x4000
+--------+
| 0x2000 | <--- p
+--------+
```

Então:

- `p` é o endereço guardado.
- `*p` é o valor que existe naquele endereço guardado.

## Diferença entre `&` e `*`

O `&` significa **endereço de**. Exemplo:

```c
int x = 10;

printf("%p\n", (void *)&x);
```

`&x` é o endereço de `x`.

Já o `*`, em declaração:

```c
int *p;
```

significa: **`p` é um ponteiro para int**.

Em expressão:

```c
*p
```

significa: **acesse o valor apontado por `p`**.

Exemplo:

```c
#include <stdio.h>

int main(void) {
    int x = 10;
    int *p = &x;

    printf("x = %d\n", x);
    printf("&x = %p\n", (void *) &x);
    printf("p = %p\n", (void *) &p);
    printf("*p = %d\n", *p);

    return 0;
}
```

Saída conceitual:

```text
x = 10
&x = 0x7ffd...
p = 0x7ffd...
*p = 10
```

## Alterando um valor por meio de um ponteiro

```c
#include <stdio.h>

int main(void) {
    int x = 10;
    int *p = &x;

    *p = 99;

    printf("x = %d\n", x);

    return 0;
}
```

Saída:

```text
x = 99
```

Isso ocorre porque `*p = 99;` não muda o ponteiro, mas o valor guardado no endereço apontado por ele. Visualmente:

```text
antes:

x = 10
p aponta para x

depois:

*p = 99

x = 99
```

## Ponteiro também é variável

Um ponteiro também ocupa memória.

```c
int x = 10;
int y = 20;

int *p = &x;

p = &y;
```

Aqui `p` primeiro aponta para `x`. Depois, `p` passa a apontar para `y`.

```c
#include <stdio.h>

int main(void) {
    int x = 10;
    int y = 20;

    int *p = &x;

    printf("*p = %d\n", *p);

    p = &y;

    printf("*p = %d\n", *p);

    return 0;
}
```

Saída:

```text
*p = 10
*p = 20
```

## Ponteiros como parâmetros de função

Em C, argumentos são passados por valor. Exemplo:

```c
#include <stdio.h>

void mudar(int n) {
    n = 99;
}

int main(void) {
    int x = 10;

    mudar(x);

    printf("x = %d\n", x);

    return 0;
}
```

Saída:

```text
x = 10
```

Isso ocorre porque a função `mudar()` recebeu uma cópia de `x`:

```text
main:
x = 10

mudar:
n = cópia de x
```

Alterar `n` não altera `x`.

### Usando ponteiro para alterar a variável original

```c
#include <stdio.h>

void mudar(int *p) {
    *p = 99;
}

int main(void) {
    int x = 10;

    mudar(&x);

    printf("x = %d\n", x);

    return 0;
}
```

Saída:

```text
x = 99
```

Agora, passamos o endereço de `x`. A função `mudar()` recebeu `int *p` e executou `*p = 99`, ou seja, alterou o valor no endereço recebido.

Um ponteiro como parâmetro pode servir para:

- Alterar uma variável externa.
- Evitar uma cópia grande.
- Preencher resultados.
- Passar *arrays*.
- Passar *structs* grandes.
- Permitir um retorno indireto.

## Exemplo clássico: `swap`

```c
#include <stdio.h>

void swap(int *a, int *b) {
    int temp = *a;
    *a = *b;
    *b = temp;
}

int main(void) {
    int x = 10;
    int y = 20;

    swap(&x, &y);

    printf("x = %d\n", x);
    printf("y = %d\n", y);

    return 0;
}
```

## Ponteiro para retornar múltiplos resultados

Podemos retornar vários valores indiretamente usando ponteiros:

```c
#include <stdio.h>

void dividir(int a, int b, int *quociente, int *resto) {
    *quociente = a / b;
    *resto = a % b;
}

int main(void) {
    int q, r;

    dividir(10, 3, &q, &r);

    printf("quociente = %d\n", q);
    printf("resto = %d\n", r);

    return 0;
}
```

## Ponteiro nulo: `NULL`

Um ponteiro pode não apontar para nenhum objeto.

```c
int *p = NULL;
```

Isso significa: **`p` não aponta para um objeto válido**.

Antes de usar um ponteiro recebido, é bom verificar:

```c
if (p == NULL) {
    return;
}
```

Exemplo:

```c
void mudar(int *p) {
    if (p == NULL) {
        return;
    }
    
    *p = 100;
}
```

Nunca faça:

```c
int *p = NULL;
*p = 10;
```

Isso tenta acessar o endereço zero e normalmente causa uma falha de segmentação (*segmentation fault*).

## Ponteiros e arrays

Em C, *arrays* e ponteiros têm uma relação estreita.

```c
int v[3] = {10, 20, 30};
```

`v` geralmente "decai" para um ponteiro para o primeiro elemento. Em muitos contextos, `v` equivale a `&v[0]`. Exemplo:

```c
#include <stdio.h>

int main(void) {
    int v[3] = {10, 20, 30};

    printf("%p\n", (void *) v);
    printf("%p\n", (void *) &v[0]);

    return 0;
}
```

Os endereços serão iguais.

## Aritmética de ponteiros

Se:

```c
int v[3] = {10, 20, 30};
int *p = v;
```

Então, `*p` é `v[0]`, `*(p + 1)` é `v[1]` e `*(p + 2)` é `v[2]`. Exemplo:

```c
#include <stdio.h>

int main(void) {
    int v[3] = {10, 20, 30};

    int *p = v;

    printf("%d\n", *p);
    printf("%d\n", *(p + 1));
    printf("%d\n", *(p + 2));

    return 0;
}
```

Saída:

```text
10
20
30
```

É importante saber que `(p + 1)` significa "próximo `int`", pois o *array* é de inteiros. Se um `int` ocupar 4 bytes, `p + 1` avançará 4 bytes.

## `v[i]` é açúcar sintático

Em C:

```c
v[i]
```

é equivalente a:

```c
*(v + i)
```

Por isso, `v[2]` é equivalente a `*(v + 2)`.

## Array como parâmetro de função

Quando escrevemos:

```c
void imprimir(int v[], int tam)
```

Isso é praticamente o mesmo que:

```c
void imprimir(int *v, int tam)
```

Exemplo:

```c
#include <stdio.h>

void imprimir(int *v, int tam) {
    for (int i = 0; i < tam; i++) {
        printf("%d\n", v[i]);
    }
}

int main(void) {
    int numeros[] = {10, 20, 30};

    imprimir(numeros, 3);

    return 0;
}
```

A função não recebe o array inteiro. Ela recebe um ponteiro para o primeiro elemento. Por isso, precisa receber também o tamanho.

Não precisamos passar `&numeros[0]`, pois vimos que, nesse contexto, `numeros` equivale a `&numeros[0]`.

## Strings são arrays de `char`

Em C, uma *string* é um *array* de `char` terminado com `'\0'`.

```c
char nome[] = "Lucas";
```

Na memória:

```text
L u c a s \0
```

Exemplo:

```c
#include <stdio.h>

int main(void) {
    char nome[] = "Lucas";

    char *p = nome;

    printf("%c\n", *p);
    printf("%c\n", *(p + 1));
    printf("%s\n", p);

    return 0;
}
```

Saída:

```text
L
u
Lucas
```

## `char nome[]` e `char *nome`

> Essa diferença é importante!

### Array modificável

```c
char nome[] = "Lucas";
```

Aqui, o compilador cria um *array* local com os caracteres. Podemos modificá-lo:

```c
nome[0] = 'M';
```

Resultado: `Mucas`.

### Ponteiro para literal

```c
char *nome = "Lucas";
```

Aqui, `nome` aponta para uma *string* literal, geralmente armazenada em uma região de somente leitura. Tentar modificá-la causa um comportamento indefinido.

Forma correta para literal:

```c
const char *nome = "Lucas";
```

Assim deixamos claro que não vamos modificar essa string.

## `const` com ponteiros

### Ponteiro para valor constante

```c
const int *p;
```

ou:

```c
int const *p;
```

Significa:

- Não posso alterar o valor apontado por `p`.
- Mas posso mudar `p` para apontar para outro lugar.

Exemplo:

```c
int x = 10;
int y = 20;

const int *p = &x;

p = &y; // Permitido

*p = 50; // Erro
```

### Ponteiro constante

```c
int *const p = &x;
```

Significa:

- Não posso mudar `p` para apontar para outro lugar.
- Posso alterar o valor apontado.

```c
int x = 10;
int y = 20;

int *const p = &x;

*p = 30; // Permitido

p = &y; // Erro
```

### Ponteiro constante para valor constante

```c
const int *const p = &x;
```

Significa:

- Não posso mudar `p`.
- Não posso mudar `*p`.

### Regras para ler declarações com `const`

Leia perto do `*`:

```c
const int *p
```

`p` aponta para `const int`. **O valor é constante**.

---

```c
int *const p;
```

`p` é constante. **O ponteiro não muda**.

## Ponteiros para structs

Suponha:

```c
typedef struct {
    int id;
    char nome[50];
} Pessoa;
```

Podemos criar:

```c
Pessoa p = {1, "Lucas"};

Pessoa *ptr = &p;
```

Para acessar campos:

```c
(*ptr).id
```

Essa forma é pouco usada. C oferece o operador `->`:

```c
ptr->id
```

Exemplo:

```c
#include <stdio.h>

typedef struct {
    int id;
    char nome[50];
} Pessoa;

int main(void) {
    Pessoa p = {1, "Lucas"};
    Pessoa *ptr = &p;

    printf("ID: %d\n", ptr->id);
    printf("Nome: %s\n", ptr->nome);

    ptr->id = 99;

    printf("Novo ID: %d\n", ptr->id);
    
    return 0;
}
```

## Passando struct para função

Se passarmos uma *struct* por valor, copiaremos toda a estrutura.

```c
void imprimir(Pessoa p) {
    printf("%s\n", p.nome);
}
```

Para *structs* grandes, é melhor passar um ponteiro:

```c
void imprimir(const Pessoa *p) {
    printf("%s\n", p->nome);
}
```

Usamos `const` porque a função só lê.

Exemplo:

```c
#include <stdio.h>

typedef struct {
    int id;
    char nome[50];
} Pessoa;

void imprimir_pessoa(const Pessoa *p) {
    if (p == NULL) 
        return;

    printf("ID: %d\n", p->id);
    printf("Nome: %s\n", p->nome);
}

void mudar_id(Pessoa *p, int novo_id) {
    if (p == NULL)
        return;

    p->id = novo_id;
}

int main(void) {
    Pessoa pessoa = {1, "Lucas"};

    imprimir_pessoa(&pessoa);

    mudar_id(&pessoa, 10);

    imprimir_pessoa(&pessoa);

    return 0;
}
```

## Retornando ponteiros de função

Uma função pode retornar um ponteiro, mas precisamos saber **para onde esse ponteiro aponta**.

### Errado: retornar endereço de variável local

```c
int *criar_numero(void) {
    int x = 10;
    return &x; // Errado
}
```

`x` está na pilha da função. Quando a função termina, `x` deixa de existir. O ponteiro fica pendente; isso é chamado de *dangling pointer*.

### Correto: retornar ponteiro para heap

```c
#include <stdio.h>
#include <stdlib.h>

int *criar_numero(void) {
    int *p = malloc(sizeof(int));

    if (p == NULL)
        return NULL;

    *p = 10;

    return p;
}

int main(void) {
    int *n = criar_numero();

    if (n == NULL)
        return 1;

    printf("%d\n", *n);

    free(n);

    return 0;
}
```

Aqui, a memória continua existindo depois que a função termina porque foi alocada no *heap*. Entretanto, quem chama a função precisa liberá-la com `free(n)`.
