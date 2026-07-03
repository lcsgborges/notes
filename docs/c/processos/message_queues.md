# Filas de mensagens

Uma **fila de mensagens** (*message queue*) é um mecanismo de IPC em que os processos trocam dados por meio de uma **fila mantida pelo kernel**. Diferentemente de um *pipe*, que transporta um fluxo contínuo de bytes, uma fila preserva os limites entre as mensagens: **cada envio corresponde a uma mensagem separada**.

Dessa forma, um **produtor** envia mensagens, e um **consumidor** as recebe. Esse mecanismo é útil quando precisamos de:

- Tarefas discretas.
- Comandos.
- Eventos.
- Mensagens independentes.
- Produtor e consumidor desacoplados.

## Fila de mensagens e *pipe*

### Pipe

- Fluxo de bytes.
- Não preserva os limites entre as mensagens.
- Normalmente, é lido na ordem dos bytes.
- É adequado para *streaming*.

### Message Queue

- Fila de mensagens.
- Preserva os limites entre as mensagens.
- Pode trabalhar com prioridades.
- É adequada para tarefas, eventos e comandos.

Exemplo:

```text
PIPE:
"ADD 2 3\nSUB 10 4\n"

MESSAGE QUEUE:
Mensagem 1: "ADD 2 3"
Mensagem 2: "SUB 10 4"
```

## Filas de mensagens POSIX

Essa API é mais moderna e simples:

- `mq_open()`.
- `mq_send()`.
- `mq_receive()`.
- `mq_close()`.
- `mq_unlink()`.

As filas POSIX são identificadas por nomes do tipo `/algum_nome`, e dois processos usam a mesma fila abrindo o mesmo nome com `mq_open()`.

As mensagens são enviadas e recebidas com `mq_send()` e `mq_receive()`.

O descritor da fila é fechado com `mq_close()`, enquanto seu nome é removido com `mq_unlink()`.

## Filas de mensagens System V

API mais antiga:

- `msgget()`.
- `msgsnd()`.
- `msgrcv()`.
- `msgctl()`.

A API System V usa `msgget()` para criar ou obter uma fila, `msgsnd()` para adicionar mensagens e `msgrcv()` para receber mensagens.

## Fluxo básico POSIX

Um processo **produtor** faz:

1. `mq_open()`.
2. `mq_send()`.
3. `mq_close()`.

Um processo **consumidor** faz:

1. `mq_open()`.
2. `mq_receive()`.
3. `mq_close()`.
4. `mq_unlink()`.

## Criando uma fila

Em POSIX, a fila tem um nome, como `/minha_fila`. Algumas regras importantes são:

- O nome começa com `/`.
- O nome não possui barras adicionais.
- O nome não é um caminho como `/tmp/fila`.

A fila POSIX não é um arquivo comum em um diretório. Ela é um objeto IPC do sistema.

### Estrutura de atributos

Quando criamos a fila, podemos definir atributos:

```c
struct mq_attr attr;
```

Campos importantes:

```c
attr.mq_maxmsg; // máximo de mensagens na fila

attr.mq_msgsize; // tamanho máximo de cada mensagem
```

Exemplo:

```c
struct mq_attr attr;

attr.mq_flags = 0;
attr.mq_maxmsg = 10;
attr.mq_msgsize = 256;
attr.mq_curmsgs = 0;
```

O exemplo acima diz que a fila pode ter até 10 mensagens e cada mensagem pode ter até 256 bytes.

## Produtor: envio de mensagens

Exemplo `produtor.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <mqueue.h>
#include <fcntl.h>
#include <sys/stat.h>

#define QUEUE_NAME "/fila_tarefas"
#define MAX_MSG 10
#define MSG_SIZE 256

int main(void) {
    struct mq_attr attr;

    attr.mq_flags = 0;
    attr.mq_maxmsg = MAX_MSG;
    attr.mq_msgsize = MSG_SIZE;
    attr.mq_curmsgs = 0;

    mqd_t fila = mq_open(QUEUE_NAME, O_CREAT | O_WRONLY, 0644, &attr);

    if (fila == (mqd_t) -1) {
        perror("mq_open");
        return 1;
    }

    const char *mensagens[] = {
        "tarefa 1: gerar relatório",
        "tarefa 2: enviar email",
        "tarefa 3: processar pagamento"
    };

    for (int i = 0; i < 3; i++) {
        if (mq_send(fila, mensagens[i], strlen(mensagens[i]) + 1, 0) == -1) {
            perror("mq_send");
            mq_close(fila);
            return 1;
        }
        printf("Enviado: %s\n", mensagens[i]);
    }
    mq_close(fila);
    return 0;
}
```

## Consumidor: recebendo mensagens

Exemplo `consumidor.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <mqueue.h>
#include <fcntl.h>
#include <sys/stat.h>

#define QUEUE_NAME "/fila_tarefas"
#define MSG_SIZE 256

int main(void) {
    mqd_t fila = mq_open(QUEUE_NAME, O_RDONLY);

    if (fila == (mqd_t) -1) {
        perror("mq_open");
        return 1;
    }

    char buffer[MSG_SIZE];
    unsigned int prioridade;

    for (int i = 0; i < 3; i++) {
        ssize_t n = mq_receive(fila, buffer, sizeof(buffer), &prioridade);

        if (n == -1) {
            perror("mq_receive");
            mq_close(fila);
            return 1;
        }

        printf("Recebido: %s | prioridade %u\n", buffer, prioridade);
    }

    mq_close(fila);

    /*
     * Remove a fila do sistema.
     * Faça isso quando ela não for mais necessária.
     */
    mq_unlink(QUEUE_NAME);
    return 0;
}
```

Compilando:

```bash
gcc produtor.c -o produtor -lrt
gcc consumidor.c -o consumidor -lrt
```

## O que é `mqd_t`?

Quando executamos `mqd_t fila = mq_open(...)`, obtemos um **descritor de fila de mensagens**. Ele é semelhante a um descritor de arquivo, mas é específico para filas de mensagens POSIX.

## Prioridade das mensagens

As filas de mensagens POSIX aceitam prioridades:

```c
mq_send(fila, mensagem, tamanho, prioridade);
```

Ao receber, a fila **entrega primeiro a mensagem de maior prioridade**. Mensagens com a mesma prioridade preservam a ordem de chegada. A documentação do `mq_send()` descreve que mensagens são posicionadas em ordem decrescente de prioridade, mantendo a ordem entre as mensagens de mesma prioridade.

Exemplo `prioridade.c`:

```c
#include <stdio.h>
#include <string.h>
#include <mqueue.h>
#include <fcntl.h>
#include <sys/stat.h>

#define QUEUE_NAME "/fila_prioridade"
#define MSG_SIZE 128
#define MAX_MSG 10

int main() {
    struct mq_attr attr = {0};
    attr.mq_maxmsg = MAX_MSG;
    attr.mq_msgsize = MSG_SIZE;

    mq_unlink(QUEUE_NAME);

    mqd_t fila = mq_open(QUEUE_NAME, O_CREAT | O_RDWR, 0664, &attr);

    if (fila == (mqd_t) -1) {
        perror("mq_open");
        return 1;
    }
     
    // strlen() retorna apenas o tamanho visível da string. Somamos 1 para incluir o '\0'.
    mq_send(fila, "mensagem baixa", strlen("mensagem baixa") + 1, 1);
    mq_send(fila, "mensagem alta", strlen("mensagem alta") + 1, 10);
    mq_send(fila, "mensagem média", strlen("mensagem média") + 1, 5);

    char buffer[MSG_SIZE];
    unsigned int prio;

    for (int i = 0; i < 3; i++) {
        if (mq_receive(fila, buffer, sizeof(buffer), &prio) == -1) {
            perror("mq_receive");
            break;
        }

        printf("Recebido: %s | prioridade = %u\n", buffer, prio);
    }
    mq_close(fila);
    mq_unlink(QUEUE_NAME);
    return 0;
}
```

Saída esperada:

```text
Recebido: mensagem alta | prioridade = 10
Recebido: mensagem média | prioridade = 5
Recebido: mensagem baixa | prioridade = 1
```

## Bloqueio

Por padrão:

- `mq_receive()` bloqueia se a fila estiver vazia.
- `mq_send()` bloqueia se a fila estiver cheia.

Isso é útil porque o consumidor pode simplesmente esperar trabalho. Exemplo:

```c
mq_receive(fila, buffer, sizeof(buffer), NULL);
```

Se não houver mensagens, o processo ficará bloqueado. O kernel o acordará quando alguém enviar uma mensagem.

Podemos abrir a fila com `O_NONBLOCK` (**modo não bloqueante**). Exemplo:

```c
mqd_t fila = mq_open(QUEUE_NAME, O_RDONLY | O_NONBLOCK);
```

Se a fila estiver vazia, `mq_receive()` não ficará bloqueada: a chamada falhará imediatamente. Exemplo:

```c
#include <stdio.h>
#include <errno.h>
#include <mqueue.h>
#include <fcntl.h>

#define QUEUE_NAME "/fila_tarefas"
#define MSG_SIZE 256

int main() {
    mqd_t fila = mq_open(QUEUE_NAME, O_RDONLY | O_NONBLOCK);

    if (fila == (mqd_t) -1) {
        perror("mq_open");
        return 1;
    }

    char buffer[MSG_SIZE];

    ssize_t n = mq_receive(fila, buffer, sizeof(buffer), NULL);

    if (n == -1) {
        if (errno == EAGAIN) {
            printf("Fila vazia. Não vou bloquear.\n");
        } else {
            perror("mq_receive");
        }
    }
    mq_close(fila);
    return 0;
}
```

## Descobrindo atributos das filas

Podemos usar `mq_getattr()`:

```c
#include <stdio.h>
#include <mqueue.h>
#include <fcntl.h>

#define QUEUE_NAME "/fila_tarefas"

int main() {
    mqd_t fila = mq_open(QUEUE_NAME, O_RDONLY);

    if (fila == (mqd_t) -1) {
        perror("mq_open");
        return 1;
    }

    struct mq_attr attr;

    if (mq_getattr(fila, &attr) == -1) {
        perror("mq_getattr");
        mq_close(fila);
        return 1;
    }

    printf("Max mensagens = %ld\n", attr.mq_maxmsg);
    printf("Tam mensagens = %ld\n", attr.mq_msgsize);
    printf("Msgs atuais = %ld\n", attr.mq_curmsgs);

    mq_close(fila);
    return 0;
}
```

## Uma fila de tarefas

Uma fila na qual os clientes enviam tarefas para um *worker* processar.

Mensagem:

```c
typedef struct {
    int id;
    char acao[64];
} Job;
```

### Produtor

```c
#include <stdio.h>
#include <string.h>
#include <mqueue.h>
#include <fcntl.h>
#include <sys/stat.h>

#define QUEUE_NAME "/fila_jobs"

typedef struct {
    int id;
    char acao[64];
} Job;

int main(void) {
    struct mq_attr attr = {0};

    attr.mq_maxmsg = 10;
    attr.mq_msgsize = sizeof(Job);

    mqd_t fila = mq_open(QUEUE_NAME, O_CREAT | O_WRONLY, 0644, &attr);

    if (fila == (mqd_t) -1) {
        perror("mq_open");
        return 1;
    }

    Job jobs[] = {
        {1, "gerar_pdf"},
        {2, "enviar_email"},
        {3, "processar_pagamento"}
    };

    for (int i = 0; i < 3; i++) {
        if (mq_send(fila, (const char *) &jobs[i], sizeof(Job), 0) == -1) {
            perror("mq_send");
            mq_close(fila);
            return 1;
        }
        printf("Job enviado: id=%d acao=%s\n", jobs[i].id, jobs[i].acao);
    }

    mq_close(fila);
    return 0;
}
```

### Worker

```c
#include <stdio.h>
#include <mqueue.h>
#include <fcntl.h>

#define QUEUE_NAME "/fila_jobs"

typedef struct {
    int id;
    char acao[64];
} Job;

int main() {
    mqd_t fila = mq_open(QUEUE_NAME, O_RDONLY);

    if (fila == (mqd_t) -1) {
        perror("mq_open");
        return 1;
    }

    while (1) {
        Job job;

        ssize_t n = mq_receive(fila, (char *) &job, sizeof(Job), NULL);

        if (n == -1) {
            perror("mq_receive");
            mq_close(fila);
            return 1;
        }

        printf("Processando job: id=%d acao=%s\n", job.id, job.acao);

        if (job.id == -1) {
            printf("Worker encerrando\n");
            break;
        }
    }
    mq_close(fila);
    mq_unlink(QUEUE_NAME);
    return 0;
}
```

### Mensagem de parada

```c
Job stop = {-1, "stop"};
mq_send(fila, (const char*) &stop, sizeof(Job), 10);
```

## `mq_close()` e `mq_unlink()`

`mq_close()` fecha o descritor da fila no processo: `mq_close(fila)`.

Já `mq_unlink()` remove o nome da fila do sistema: `mq_unlink("/fila")`. Se não chamarmos essa função, a fila poderá permanecer no sistema sem necessidade.

## Filas POSIX no Linux

No Linux, as filas POSIX geralmente aparecem por meio de um sistema de arquivos especial chamado `mqueue`, frequentemente montado em `/dev/mqueue`. Podemos consultá-lo com:

```bash
ls /dev/mqueue
```

## Quando usar uma fila de mensagens

Use quando:

- Temos mensagens discretas.
- Queremos uma arquitetura de produtor e consumidor.
- Queremos uma fila de tarefas local.
- Precisamos definir prioridades.
- Queremos desacoplar o envio do processamento.
- Queremos evitar a criação de um protocolo para separar mensagens em um fluxo de bytes.

Bons exemplos:

- Um processo envia tarefas para um *worker*.
- Um *daemon* recebe comandos internos.
- Múltiplos produtores geram eventos.
- Um consumidor processa as mensagens em determinada ordem ou prioridade.

Evite quando:

- Precisamos de *streaming* contínuo de dados.
- Precisamos de comunicação pela rede.
- Precisamos atender muitos clientes complexos com respostas individuais.
- Precisamos de alta vazão para dados grandes.
