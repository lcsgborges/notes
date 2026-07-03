# Arrays

**Array** é uma estrutura de dados que armazena vários valores do **mesmo tipo** em posições consecutivas, acessadas por um índice.

Exemplo:

```text
Índice:     0    1    2    3    4
         +----+----+----+----+----+
Array:   | 10 | 20 | 30 | 40 | 50 |
         +----+----+----+----+----+
```

## Declarando um array em C

```c
int nums[5];
```

Isso cria um array com espaço para 5 inteiros (cada inteiro em C equivale a 4 bytes, portanto, essa estrutura tem 20 bytes).

## Inicializando um array

```c
int nums[5] = {10, 20, 30, 40, 50};
```

Isso já cria um array com valores declarados em cada índice (0-4), ou podemos deixar que o compilador descubra o tamanho do array:

```c
int nums[] = {10, 20, 30, 40, 50};
```

Isso também funciona!

## Acessando elementos

```c
printf("%d\n", nums[0]); // 10
printf("%d\n", nums[2]); // 30
```

## Alterando um valor

```c
int nums[5] = {10, 20, 30, 40, 50};

nums[2] = 99;
```

Agora ao imprimir o elemento `nums[2]` teremos `99` e não `30`.

## Percorrendo um array

```c
#include <stdio.h>

#define TAM 5

int main() {
    int nums[TAM] = {10, 20, 30, 40, 50};
    
    for (int i = 0; i < TAM; i++) {
        printf("%d ", nums[i]);
    }
    
    return 0;
}
```

## O array na memória

Suponha que cada `int` ocupe 4 bytes e o array comece no endereço `0x1000`:

```c
int nums[5] = {10, 20, 30, 40, 50};
```

Então temos:

| Endereço | Valor |
| :------: | :---: |
| 0x1000 | 10 |
| 0x1004 | 20 |
| 0x1008 | 30 |
| 0x1012 | 40 |
| 0x1016 | 50 |

## Relação com ponteiros

Em C, o nome do array representa o endereço do primeiro elemento:

```c
numeros == &numeros[0];
```

Por isso estas duas expressões são equivalentes:

```c
numeros[2]
```

e 

```c
*(numeros + 2) // Aritmética de ponteiros
```

## Resumindo

- Um **array** é uma sequência de elementos do mesmo tipo.
- Os índices começam em 0.
- Os elementos ficam **contíguos na memória**.
- O nome do array aponta para o primeiro elemento.
- `array[i]` é equivalente a `*(array + i)` em C.
