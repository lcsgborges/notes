# OpenMP

## Por que utilizar programação paralela?

Os dois principais motivos são:

- **Reduzir o tempo** necessário para solucionar um problema.
- **Resolver problemas mais complexos** e de maior dimensão.

## Modelos de programação paralela

### Programação em memória compartilhada (OpenMP e Cilk)

- Programação usando processos ou threads.
- Decomposição do trabalho por domínio ou função, com granularidade fina, média ou grossa.
- Comunicação por meio de **memória compartilhada**.
- Sincronização por meio de mecanismos de exclusão mútua.

### Programação em memória distribuída (MPI)

- Programação usando processos distribuídos.
- Decomposição do domínio com granularidade grossa.
- Comunicação e sincronização por **troca de mensagens**.

## Introdução ao OpenMP

Tipos e protótipos de funções no arquivo:

```c
#include <omp.h>
```

A maioria das construções do OpenMP consiste em diretivas de compilação:

```c
#pragma omp construct [clause ...]

// Exemplo:

#pragma omp parallel private(var1, var2) shared(var3, var4)
{

}
```

> Em C, `#pragma` é uma diretiva de pré-processamento que modifica ou controla comportamentos do compilador que não fazem parte da linguagem em si. De forma resumida, ela passa comandos especiais ao compilador.

Notas de compilação com **GCC**:

```bash
gcc code.c -o code.out -fopenmp
```

### Funções OpenMP

Algumas funções comuns do OpenMP:

```c
// Arquivo de interface da biblioteca OpenMP.
#include <omp.h>

// Retorna o identificador da thread atual.
int omp_get_thread_num(void);

// Define o número de threads solicitado para regiões paralelas posteriores.
void omp_set_num_threads(int num_threads);

// Retorna o número de threads da equipe atual ou 1 fora de uma região paralela.
int omp_get_num_threads(void);
```

Também podemos solicitar um número de *threads* com a variável de ambiente `OMP_NUM_THREADS`. Exemplo: `export OMP_NUM_THREADS=4`.

### Diretivas OpenMP

Algumas diretivas comuns do OpenMP:

```c
// Cria uma região paralela e define atributos de compartilhamento.

// Cada thread possui sua própria cópia das variáveis privadas.
#pragma omp parallel private(...) shared(...)
{

    // Apenas uma thread da equipe executa o bloco seguinte.
    #pragma omp single
    {
        // Trabalho executado uma vez.
    }
}
```

#### Cláusula shared

Especifica variáveis cujo armazenamento é compartilhado entre as *threads*. As regras padrão dependem do local e da categoria de cada variável; a cláusula `default(none)` pode exigir que os atributos relevantes sejam declarados explicitamente.

#### Cláusula private

Especifica um conjunto de variáveis privadas. Cada *thread* recebe uma nova instância, cujo valor inicial é indeterminado. Ao final da região, essas instâncias são descartadas, e seus valores não são copiados para as variáveis originais.

### Exercício 1: “Olá, mundo!” em OpenMP

```c
#include <stdio.h>
#include <omp.h>

int main(void) {
    #pragma omp parallel
    {
        int thread_id = omp_get_thread_num();
        int num_threads = omp_get_num_threads();

        printf(
            "Thread %d de %d: olá, mundo!\n",
            thread_id,
            num_threads
        );
    }

    return 0;
}

// Para compilar: gcc -fopenmp code01.c -o code.out
```

No OpenMP, podemos usar `#pragma omp for`, que permite dividir as iterações de um laço entre as threads sem definir manualmente um chunk, um início e um fim para cada uma, como seria necessário em uma distribuição manual com **MPI**.

### Interações das threads

OpenMP é um modelo de _multithreading_ de memória compartilhada; portanto, as threads se comunicam por meio de variáveis compartilhadas.

- Uma **condição de corrida de dados** ocorre quando duas ou mais *threads* acessam a mesma posição de memória sem a sincronização necessária, pelo menos um dos acessos é uma escrita e os acessos podem ocorrer simultaneamente.
- O resultado de uma corrida pode variar conforme o escalonamento das *threads*.
- Devemos planejar o acesso aos dados para garantir a correção e evitar sincronização desnecessária, que possui custo de desempenho.

### Sincronização

Assegura que uma ou mais threads estejam em um estado bem definido em um ponto conhecido da execução. As duas formas mais comuns são:

- **Barreira**: cada *thread* espera até que todas as demais alcancem o mesmo ponto.
- **Exclusão mútua**: permite que somente uma *thread* execute determinado bloco por vez.

#### Barreira

Exemplo esquemático:

```c
#pragma omp parallel
{
    int id = omp_get_thread_num();
    A[id] = big_calc(id);

    #pragma omp barrier
    B[id] = big_calc2(id, A);
}
```

#### Exclusão mútua (`critical`)

Uma região `critical` pode ser executada por apenas uma *thread* por vez:

```c
#include <stdio.h>

int main(void) {
    double result = 0.0;

    #pragma omp parallel for
    for (int i = 0; i < 1000; i++) {
        double value = (double)i * i;

        #pragma omp critical
        result += value;
    }

    printf("Resultado: %.0f.\n", result);
    return 0;
}
```

A diretiva `atomic` protege determinados acessos simples à memória sem criar uma região crítica arbitrária:

```c
#include <stdio.h>

int main(void) {
    long total = 0;

    #pragma omp parallel for
    for (long i = 0; i < 1000; i++) {
        #pragma omp atomic update
        total += i;
    }

    printf("Total: %ld.\n", total);
    return 0;
}
```

### Exercício 2: soma de um vetor

```c
#include <stdio.h>
#include <omp.h>

#define TAM 1000

int main(void) {
    int v[TAM];
    int sum = 0;

    for (int i = 0; i < TAM; i++) {
        v[i] = 1;
    }

    #pragma omp parallel shared(v, sum)
    {
        int sum_local = 0;

        #pragma omp for
        for (int i = 0; i < TAM; i++) {
            sum_local += v[i];
        }

        #pragma omp atomic update
        sum += sum_local;
    }

    printf("Soma: %d.\n", sum);
    return 0;
}
```

### Redução

A combinação de variáveis locais das threads em uma única variável é chamada de **redução**.

Sintaxe da cláusula: `reduction(operador:lista_de_variaveis)`.

Dentro de uma região paralela ou divisão de trabalho:

- É criada uma cópia privada de cada variável da lista.
- Cada cópia é inicializada com a identidade definida para o operador.
- As atualizações acontecem nas cópias privadas.
- Ao final, as cópias são combinadas com a variável original por meio do operador.

```c
// Exemplo:
#pragma omp for reduction(+:sum)
```

Solução do exercício 2 com redução:

```c
#include <stdio.h>
#include <omp.h>

#define TAM 100000

int main(void) {
    int sum = 0, v[TAM];

    for (int i = 0; i < TAM; i++) {
        v[i] = 1;
    }

    #pragma omp parallel for reduction(+:sum)
    for (int i = 0; i < TAM; i++) {
        sum += v[i];
    }

    printf("Soma: %d.\n", sum);
    return 0;
}
```

### Seções

A diretiva `#pragma omp section` define uma tarefa dentro de uma construção `#pragma omp sections`. As seções são distribuídas entre as *threads* da equipe, e cada uma é executada uma única vez:

```c
#include <stdio.h>
#include <omp.h>

int main(void) {
    #pragma omp parallel
    {
        #pragma omp sections
        {
            #pragma omp section
            {
                // Apenas uma thread executa esta seção.
                printf("Função 1.\n");
            }
            #pragma omp section
            {
                // Apenas uma thread executa esta seção.
                printf("Função 2.\n");
            }
        }
        // Todas as threads executam esta instrução.
        printf("Função executada por todas as threads!\n");
    }
    return 0;
}
```
