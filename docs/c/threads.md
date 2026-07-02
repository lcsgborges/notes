# Threads

Uma **thread** é uma linha de execução dentro de um processo.

Até agora, quando falávamos de processos, o modelo era:

```text
Processo
 ├── código
 ├── heap
 ├── globais
 ├── stack
 ├── file descriptors
 └── uma linha de execução
```

Com threads, um processo pode ter várias linhas de execução ao mesmo tempo:

```text
Processo
 ├── código compartilhado
 ├── heap compartilhado
 ├── variáveis globais compartilhadas
 ├── file descriptors compartilhados
 ├── thread 1
 │    └── stack própria
 ├── thread 2
 │    └── stack própria
 └── thread 3
      └── stack própria
```

> **Processos têm memória separada. Threads do mesmo processo compartilham a mesma memória.**

## Processo vs Thread

### Processo

Um processo tem seu próprio espaço de memória.

```text
Processo A
 └── memória A

Processo B
 └── memória B
```

Se o processo A altera uma variável global, o processo B não vê. Por isso, para processos compartilharem dados, precisamos de IPC:

- pipe
- FIFO
- socket
- message queue
- shared memory

### Thread

Threads vivem dentro do mesmo processo.

```text
Processo
 ├── thread A
 └── thread B
```

Elas compartilham:

- heap
- variáveis globais
- arquivos abertos
- sockets
- memória do processo

Mas cada thread tem sua própria:

- stack
- registradores
- fluxo de execução

## Por quê threads existem?

Threads servem para fazer várias atividades dentro do mesmo processo. Exemplos:

- Servidor atendendo vários clientes
- Programa fazendo download enquanto atualiza interface
- Worker processo tarefa em paralelo
- Um processo lendo dados enquanto outro techo processa

Em vez de criar vários processos separados, criamos várias threads dentro do mesmo processo.

## Primeiro exemplo com `pthread_create`

Em C no Linux, usamos normalmente a biblioteca POSIX Threads, conhecida como `pthreads`.

Função principal:

```c
pthread_create()
```

Exemplo:

```c
#include <stdio.h>
#include <pthread.h>

void *minha_thread(void *arg) {
    printf("Olá, eu sou uma thread");
    return NULL;
}

int main(void) {
    pthread_t t;

    if (pthread_create(&t, NULL, minha_thread, NULL) != 0) {
        perror("pthread_create");
        return 1;
    }

    pthread_join(t, NULL);

    printf("Thread terminou\n");

    return 0;
}
```

### O que aconteceu

1. `pthread_t t;`: criamos uma variável que identifica a thread.
2. `pthread_create(&t, NULL, minha_thread, NULL);`: pedimos para o sistema criar uma nova thread que começa executando a função `minha_thread()`.
3. `pthread_join(t, NULL)`: a thread principal (processo executado) espera a outra thread terminar.

Sem `pthread_join()`, o `main()` poderia terminar antes da thread executar.

## A função da thread

A função usada por `pthread_create()` precisa ter este formato:

```c
void *funcao(void *arg)
```

Ela recebe um `void *` e retorna um `void *`, pois assim podemos passar qualquer tipo de dado para a thread.

## Passando argumentos para a thread

```c
#include <stdio.h>
#include <pthread.h>

void *imprimir_numero(void *arg) {
    int *numero = (int *) arg;

    printf("Número recebido = %d\n", *numero);

    return NULL;
}

int main(void) {
    pthread_t t;
    int valor = 42;

    if (pthread_create(&t, NULL, imprimir_numero, &valor) != 0) {
        perror("pthread_create");
        return 1;
    }

    pthread_join(t, NULL);
    return 0;
}
```

## Threads compartilham variáveis globais

Exemplo:

```c
#include <stdio.h>
#include <pthread.h>

int contador = 0;

void *incrementar(void *arg) {
    contador++;
    return NULL;
}

int main(void) {
    pthread_t t1, t2;

    pthread_create(&t1, NULL, incrementar, NULL);
    pthread_create(&t2, NULL, incrementar, NULL);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("Contador = %d\n", contador);

    return 0;
}
```

## O problema: race condition

*Race Condition* é quando duas ou mais threads fazem uma operação ao mesmo tempo. Dessa forma, o resultado final pode ser incoerente, exemplo (várias threads mexendo num contador).

```c
#include <stdio.h>
#include <pthread.h>

#define N_THREADS 4
#define N_INC 1000000

int contador = 0;

void *incrementar(void *arg) {
    for (int i = 0; i < N_INC; i++) {
        contador++;
    }

    return NULL;
}

int main(void) {
    pthread_t threads[N_THREADS];

    for (int i = 0; i < N_THREADS; i++) {
        pthread_create(&threads[i], NULL, incrementar, NULL);
    }

    for (int i = 0; i < N_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }

    printf("Esperado: %d\n", N_THREADS * N_INC);
    printf("Obtido:   %d\n", contador);

    return 0;
}
```

Existem algumas formas de resolver isso, detalhado logo abaixo:

### Mutex

Mutex significa algo como **mutual exclusion**. Ou seja, "Só uma thread pode entrar nessa região crítica por vez".

**Região crítica** é o trecho que acessa dado compartilhado. Exemplo: `contador++`. Esse trecho precisa ser protegido.

Corrigindo com `pthread_mutex_t`:

```c
#include <stdio.h>
#include <pthread.h>

#define N_THREADS 4
#define N_INC 1000000

int contador = 0;

pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void *incrementar(void *arg) {
    for (int i = 0; i < N_INC; i++) {
        pthread_mutex_lock(&mutex);

        contador++;

        pthread_mutex_unlock(&mutex);
    }

    return NULL;
}

int main(void) {
    pthread_t threads[N_THREADS];

    for (int i = 0; i < N_THREADS; i++) {
        pthread_create(&threads[i], NULL, incrementar, NULL);
    }

    for (int i = 0; i < N_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }

    printf("Esperado: %d\n", NTHREADS * N_INC);
    printf("Obtido: %d\n", contador);

    pthread_mutex_destroy(&mutex);

    return 0;
}
```

#### Como o mutex funciona

Quando uma thread faz: `pthread_mutex_lock(&mutex);`:

1. Ela tenta pegar o mutex.
2. Se ninguém está usando, ela entra.
3. Se outra thread já está dentro, ela espera.
4. Depois ao terminar faz `pthread_mutex_unlock(&mutex)`, liberando o mutex para outra thread.

Visualmente:

```text
Thread A pega mutex
Thread A altera contador
Thread A solta mutex

Thread B pega mutex
Thread B altera contador
Thread B solta mutex
```

## Threads e heap

Como threads compartilham o mesmo processo, elas compartilham o heap. Exemplo:

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

void *thread_func(void *arg) {
    int *p = (int *) arg;
    *p = 99;

    return NULL;
}

int main(void) {
    pthread_t t;

    int *valor = malloc(sizeof(int));

    if (valor == NULL) {
        perror("malloc");
        return 1;
    }

    *valor = 10;

    pthread_create(&t, NULL, thread_func, valor);
    pthread_join(t, NULL);

    printf("Valor = %d\n", valor);

    free(valor);

    return 0;
}
```

A thread altera a memória alocada no heap, e o `main()` vê a mudança. Isso funciona porque ambas estão no mesmo processo.

## Threads e stack

Cada thread tem sua própria stack. Exemplo:

```c
void *funcao(void *arg) {
    int local = 10;

    printf("Endereço local: %p\n", (void *)&local);

    return NULL;
}
```

Se criarmos várias threads, cada uma terá seu próprio `local`.

Variáveis locais normais não são compartilhadas entre threads, porque ficam na stack de cada thread.

## Erro comum: retornar ponteiro para variável local

Errado:

```c
void *funcao(void *arg) {
    int x = 10;

    return &x;
}
```

`x` está na stack da thread. Quando a função termina, `x` deixa de existir.

O certo é usar heap:

```c
#include <stdio.h>
#include <pthread.h>
#include <stdlib.h>

void *funcao(void *arg) {
    int *x = malloc(sizeof(int));

    if (x == NULL) {
        return NULL;
    }

    *x = 10;

    return x;
}

int main(void) {
    pthread_t t;

    pthread_create(&t, NULL, funcao, NULL);

    void *retorno;

    pthread_join(t, &retorno);

    int *valor = (int *)retorno;

    if (valor != NULL) {
        printf("Valor retorna = %d\n", *valor);
        free(valor);
    }
    return 0;
}
```

## `pthread_join()`

`pthread_join()` espera a thread terminar. É parecido conceitualmente com `wait()` para processos. Diferença:

- `wait()`: espera processo filho
- `pthread_join()`: espera thread

Se quisermos pegar o retorno da thread:

```c
void *retorno;
pthread_join(thread, &retorno);
```

## `pthread_detach()`

Às vezes não queremos esperar uma thread com `pthread_join()`, nesse caso, podemos destacá-la:

```c
pthread_detach(t);
```

Uma *thread detached* libera seus recursos automaticamente quando termina. Não conseguimos dar `join()` nela depois.

Regra prática:

- Quer esperar o resultado? **use `pthread_join()`**
- Não quer esperar e não preciso do retorno? **use `pthread_detach()`**

## Threads e file descriptors

Threads compartilham file descriptors. Se uma thread abre um arquivo:

```c
int fd = open("arquivo.txt", O_RDONLY);
```

outra thread do mesmo processo pode usar esse mesmo `fd`, desde que tenha acesso à variável ou receba o valor. Isso vale para:

- arquivos
- pipes
- sockets
- FIFOs

Esse compartilhamento é útil, mas também pode gerar problemas se não protegermos o acesso com mutex.

## Threads são paralelismo ou concorrência?

Uma thread permite concorrência.

- **Concorrência**: A concorrência é a capacidade do sistema de **gerenciar múltiplas tarefas** alternando entre elas, dando a sensação de que ocorrem ao mesmo tempo, embora geralmente sejam processadas de forma sequencial por um único núcleo.
- **Paralelismo**: O paralelismo é a **execução literal e simultânea** de duas ou mais tarefas ao mesmo tempo, exigindo múltiplos núcleos de processador.

Se sua máquina tem vários núcleos de CPU, threads podem rodar em paralelo. Mas mesmo em um único núcleo, o sistema operacional pode alternar entre threads rapidamente.

## O scheduler controla threads

O kernel agenda threads para executar. Com processos, o shceduler escolhe processos/threads executáveis. 

Com threads POSIX no Linux, cada thread é tratada pelo kernel como uma **unidade agendável**. Então a ordem de execução **não** é garantida.

Isso significa que não podemos confiar que uma Thread A vai executar antes de uma Thread B. Para que isso ocorra, precisamos usar **sincronização**.

## Exemplo útil: servidor com thread por cliente

Em TCP Sockets, uma arquitetura comum é:

1. Servidor aceita cliente
2. Cria thread para atender aquele cliente
3. Volta para `accept()`

Modelo:

```text
main thread
 ├── accept()
 ├── accept()
 └── accept()

worker threads
 ├── cliente 1
 ├── cliente 2
 └── cliente 3
```

Exemplo sem socket real:

```c
#include <stdio.h>
#include <pthread.h>
#include <stdlib.h>
#include <unistd.h>

void *atender_cliente(void *arg) {
    int cliente_fd = *(int *)arg;
    free(arg);

    printf("Atendendo cliente %d\n", cliente_fd);
    sleep(2);
    printf("Cliente %d finalizado\n", cliente_fd);

    return NULL;
}

int main(void) {
    for (int i = 0; i < 5; i++) {
        pthread_t t;

        int *id = malloc(sizeof(int));

        if (id == NULL) {
            perror("malloc");
            return 1;
        }

        *id = i;

        if (pthread_create(&t, NULL, atender_cliente, id) != 0) {
            perror("pthread_create");
            free(id);
            return 1;
        }

        pthread_detach(t);
    }
    sleep(5);
    return 0;
}
```

## Quando usar threads

Use threads quando:

- várias tarefas compartilham muito estado
- quer evitar custo de criar processos
- quer atender várias conexões dentro do mesmo processo
- quer paralelizar trabalho em múltiplos núcleos
- quer manter uma tarefa bloqueante sem travar o programa inteiro

## Quando evitar thrads

Evite ou tenha cuidado quando:

- o programa é simples e sequencial
- o compartilhamento de estado vai ficar confuso
- você não quer lidar com race condition
- precisa isolamento forte entre tarefas
- um erro em uma tarefa não pode derrubar tudo

Com threads, todas vivem no mesmo processo. Portanto, se uma thread causar `segmentation fault`, o processo inteiro cai.

## Resumo

> **Threads compartilham a memória do processo, mas cada thread tem sua própria stack e seu próprio fluxo de execução.**

> **Sempre que duas threads acessam o mesmo dado compartilhado e pelo menos uma escreve, você precisa de sincronização.**
