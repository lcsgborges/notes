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

## Processo e *thread*

### Processo

Um processo tem seu próprio espaço de memória.

```text
Processo A
 └── memória A

Processo B
 └── memória B
```

Se o processo A alterar uma variável global, o processo B não verá a mudança. Por isso, para que os processos compartilhem dados, precisamos de IPC:

- *Pipe*.
- FIFO.
- *Socket*.
- Fila de mensagens (*message queue*).
- Memória compartilhada (*shared memory*).

### Thread

Threads vivem dentro do mesmo processo.

```text
Processo
 ├── thread A
 └── thread B
```

Elas compartilham:

- *Heap*.
- Variáveis globais.
- Arquivos abertos.
- *Sockets*.
- Memória do processo.

Entretanto, cada *thread* tem seus próprios recursos:

- Pilha.
- Registradores.
- Fluxo de execução.

## Por que as *threads* existem?

As *threads* permitem realizar várias atividades dentro do mesmo processo. Exemplos:

- Um servidor que atende vários clientes.
- Um programa que baixa um arquivo enquanto atualiza a interface.
- Um *worker* que processa uma tarefa em paralelo.
- Uma *thread* que lê dados enquanto outra os processa.

Em vez de criar vários processos separados, criamos várias threads dentro do mesmo processo.

## Primeiro exemplo com `pthread_create`

Em C, no Linux, normalmente usamos a biblioteca POSIX Threads, conhecida como Pthreads.

Função principal:

```c
pthread_create()
```

Exemplo:

```c
#include <stdio.h>
#include <pthread.h>

void *minha_thread(void *arg) {
    printf("Olá, eu sou uma thread.\n");
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
3. `pthread_join(t, NULL)`: a *thread* principal do processo espera que a outra *thread* termine.

Sem `pthread_join()`, a função `main()` poderia terminar antes que a *thread* fosse executada.

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

Uma **condição de corrida** (*race condition*) ocorre quando duas ou mais *threads* acessam dados compartilhados concorrentemente, pelo menos uma delas realiza uma escrita e não há a sincronização necessária. Dessa forma, o resultado final pode ser incoerente, como no exemplo de várias *threads* alterando um contador.

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

Existem algumas formas de resolver esse problema, detalhadas a seguir.

### Mutex

*Mutex* significa **exclusão mútua** (*mutual exclusion*), ou seja, somente uma *thread* pode entrar na região crítica por vez.

Uma **região crítica** é um trecho que acessa dados compartilhados. Por exemplo, `contador++` precisa ser protegido.

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

    printf("Esperado: %d\n", N_THREADS * N_INC);
    printf("Obtido: %d\n", contador);

    pthread_mutex_destroy(&mutex);

    return 0;
}
```

#### Como o mutex funciona

Quando uma thread faz: `pthread_mutex_lock(&mutex);`:

1. Ela tenta pegar o mutex.
2. Se ninguém estiver usando o *mutex*, ela entrará.
3. Se outra *thread* já estiver na região crítica, ela esperará.
4. Ao terminar, executará `pthread_mutex_unlock(&mutex)`, liberando o *mutex* para outra *thread*.

Visualmente:

```text
Thread A pega mutex
Thread A altera contador
Thread A solta mutex

Thread B pega mutex
Thread B altera contador
Thread B solta mutex
```

## *Threads* e *heap*

Como as *threads* pertencem ao mesmo processo, elas compartilham o *heap*. Exemplo:

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

    printf("Valor = %d\n", *valor);

    free(valor);

    return 0;
}
```

A *thread* altera a memória alocada no *heap*, e a função `main()` vê a mudança. Isso funciona porque ambas fazem parte do mesmo processo.

## *Threads* e pilha

Cada *thread* tem sua própria pilha. Exemplo:

```c
void *funcao(void *arg) {
    int local = 10;

    printf("Endereço local: %p\n", (void *)&local);

    return NULL;
}
```

Se criarmos várias *threads*, cada uma terá sua própria variável `local`.

Variáveis locais comuns não são compartilhadas entre *threads*, porque ficam na pilha de cada uma.

## Erro comum: retornar ponteiro para variável local

Errado:

```c
void *funcao(void *arg) {
    int x = 10;

    return &x;
}
```

`x` está na pilha da *thread*. Quando a função termina, `x` deixa de existir.

O correto é usar o *heap*:

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
        printf("Valor retornado = %d\n", *valor);
        free(valor);
    }
    return 0;
}
```

## `pthread_join()`

`pthread_join()` espera que a *thread* termine. Esse comportamento é conceitualmente parecido com o de `wait()` para processos. As diferenças são:

- `wait()`: espera um processo filho.
- `pthread_join()`: espera uma *thread*.

Se quisermos pegar o retorno da thread:

```c
void *retorno;
pthread_join(thread, &retorno);
```

## `pthread_detach()`

Às vezes, não queremos esperar uma *thread* com `pthread_join()`; nesse caso, podemos destacá-la:

```c
pthread_detach(t);
```

Uma *thread detached* libera seus recursos automaticamente quando termina. Depois disso, não podemos chamar `pthread_join()` para ela.

Regra prática:

- Quer esperar o resultado? **Use `pthread_join()`.**
- Não quer esperar nem precisa do retorno? **Use `pthread_detach()`.**

## *Threads* e descritores de arquivo

As *threads* compartilham descritores de arquivo. Se uma *thread* abrir um arquivo:

```c
int fd = open("arquivo.txt", O_RDONLY);
```

outra *thread* do mesmo processo poderá usar o mesmo `fd`, desde que tenha acesso à variável ou receba o valor. Isso vale para:

- Arquivos.
- *Pipes*.
- *Sockets*.
- FIFOs.

Esse compartilhamento é útil, mas também pode gerar problemas se não protegermos o acesso com um *mutex*.

## Threads são paralelismo ou concorrência?

Uma thread permite concorrência.

- **Concorrência**: capacidade do sistema de **gerenciar múltiplas tarefas**, alternando entre elas. Em um único núcleo, elas progridem de forma intercalada.
- **Paralelismo**: **execução simultânea** de duas ou mais tarefas, o que exige múltiplos núcleos de processamento.

Se uma máquina tiver vários núcleos de CPU, as *threads* poderão ser executadas em paralelo. Mesmo em um único núcleo, o sistema operacional poderá alternar rapidamente entre elas.

## O escalonador controla as *threads*

O kernel agenda as *threads* para execução. O escalonador (*scheduler*) escolhe os processos e as *threads* aptos a executar.

Com threads POSIX no Linux, cada thread é tratada pelo kernel como uma **unidade agendável**. Então a ordem de execução **não** é garantida.

Isso significa que não podemos pressupor que uma *thread* A será executada antes de uma *thread* B. Para garantir essa ordem, precisamos usar **sincronização**.

## Exemplo útil: servidor com uma *thread* por cliente

Com *sockets* TCP, uma arquitetura comum é:

1. O servidor aceita um cliente.
2. Cria uma *thread* para atender esse cliente.
3. Volta para `accept()`.

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

- Várias tarefas compartilham muito estado.
- Você quer evitar o custo de criar processos.
- Você quer atender várias conexões dentro do mesmo processo.
- Você quer paralelizar o trabalho em múltiplos núcleos.
- Você quer manter uma tarefa bloqueante sem travar o programa inteiro.

## Quando evitar *threads*

Evite ou tenha cuidado quando:

- O programa é simples e sequencial.
- O compartilhamento de estado ficará confuso.
- Você não quer lidar com condições de corrida.
- Você precisa de isolamento forte entre as tarefas.
- Um erro em uma tarefa não pode derrubar todo o programa.

Todas as *threads* pertencem ao mesmo processo. Portanto, se uma delas causar uma falha de segmentação (*segmentation fault*), todo o processo será encerrado.

## Resumo

> **As threads compartilham a memória do processo, mas cada uma tem sua própria pilha e seu próprio fluxo de execução.**

> **Sempre que duas threads acessarem o mesmo dado compartilhado e pelo menos uma delas realizar uma escrita, será necessário usar sincronização.**
