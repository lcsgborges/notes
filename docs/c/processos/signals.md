# Sinais

Um **sinal** (*signal*) é uma notificação assíncrona entregue a um processo para informar a ocorrência de um evento. Cada sinal possui um identificador e uma disposição padrão, como encerrar, parar ou continuar o processo. Sinais comuns não são adequados para transportar dados arbitrários.

## Principais sinais

- `SIGINT`: interrupção solicitada pelo terminal, geralmente com `Ctrl+C`.
- `SIGTERM`: solicitação de término.
- `SIGKILL`: término imediato, sem possibilidade de tratamento.
- `SIGSTOP`: suspensão imediata, sem possibilidade de tratamento.
- `SIGCONT`: continua um processo suspenso.
- `SIGCHLD`: um processo filho terminou ou mudou de estado.
- `SIGSEGV`: acesso inválido à memória.
- `SIGFPE`: erro aritmético, como uma divisão inteira por zero.
- `SIGPIPE`: escrita em um *pipe* ou *socket* sem leitores.
- `SIGHUP`: desconexão do terminal controlador; por convenção, algumas aplicações também o usam para recarregar a configuração.
- `SIGALRM`: expiração de um temporizador configurado com `alarm()`.
- `SIGUSR1` e `SIGUSR2`: sinais reservados para uso da aplicação.

Cada sinal possui um número associado. No Linux, por exemplo, normalmente temos:

- `SIGINT = 2`.
- `SIGKILL = 9`.
- `SIGTERM = 15`.

Esses números podem variar entre sistemas. Portanto, devemos usar os nomes simbólicos no código e nos comandos.

## O que os processos fazem com um sinal

Para muitos sinais, o processo pode:

1. Aceitar a ação padrão do sinal.
2. Ignorar o sinal.
3. Capturar o sinal com uma função tratadora (*handler*).
4. Bloquear temporariamente sua entrega por meio de uma máscara.

Exemplo:

```text
SIGINT chegou

Opção 1: encerrar
Opção 2: ignorar
Opção 3: executar uma função tratadora
```

`SIGKILL` e `SIGSTOP` não podem ser bloqueados, ignorados nem capturados, pois o sistema precisa conseguir terminar ou suspender um processo de forma definitiva.

Então, para encerrar um processo imediatamente:

```bash
kill -KILL PID
```

O comando `kill` não serve apenas para terminar processos; ele envia sinais. Exemplos:

```bash
kill -USR1 1234
kill -HUP 1234
kill -STOP 1234
kill -CONT 1234
```

Exemplo capturando um `Ctrl + C`:

```c
#define _POSIX_C_SOURCE 200809L

#include <stdio.h>
#include <signal.h>
#include <unistd.h>

static void sigint_handler(int signal_number) {
    static const char message[] = "\nSIGINT recebido.\n";

    (void)signal_number;
    (void)write(STDOUT_FILENO, message, sizeof(message) - 1);
}

int main(void) {
    struct sigaction action = {0};
    action.sa_handler = sigint_handler;
    sigemptyset(&action.sa_mask);

    if (sigaction(SIGINT, &action, NULL) == -1) {
        perror("sigaction");
        return 1;
    }

    printf("Em execução. Pressione Ctrl+C.\n");
    fflush(stdout);

    for (;;) {
        pause();
    }
}
```

Dentro de um tratador, somente podemos chamar funções seguras nesse contexto. `printf()`, por exemplo, não é segura para sinais assíncronos; `write()` é. Outra opção comum é alterar uma variável do tipo `volatile sig_atomic_t` e tratá-la no fluxo normal do programa.

Para ignorar um sinal, podemos configurar sua ação como `SIG_IGN`. A interface `signal()` permite, por exemplo, escrever `signal(SIGINT, SIG_IGN)`.

Embora simples, `signal()` tem problemas históricos de portabilidade e comportamento. Por isso, surgiu o `sigaction()`.

Um processo pode enviar um sinal para si mesmo usando `raise()`:

```c
#include <stdio.h>
#include <signal.h>

static volatile sig_atomic_t received_signal;

static void handler(int signal_number) {
    received_signal = signal_number;
}

int main(void) {
    if (signal(SIGUSR1, handler) == SIG_ERR) {
        perror("signal");
        return 1;
    }

    if (raise(SIGUSR1) != 0) {
        fprintf(stderr, "Não foi possível enviar SIGUSR1.\n");
        return 1;
    }

    printf("Sinal recebido: %d.\n", (int)received_signal);
    return 0;
}
```

Com `pause()`, um processo fica bloqueado até que um sinal seja entregue e seu tratador retorne:

```c
for (;;) {
    pause();
}
```

Isso evita consumo de CPU desnecessário.

Com `alarm()`, podemos pedir ao sistema que envie `SIGALRM` depois de determinada quantidade de segundos:

```c
#define _POSIX_C_SOURCE 200809L

#include <stdio.h>
#include <signal.h>
#include <unistd.h>

static void sigalrm_handler(int signal_number) {
    static const char message[] = "Tempo esgotado.\n";

    (void)signal_number;
    (void)write(STDOUT_FILENO, message, sizeof(message) - 1);
}

int main(void) {
    struct sigaction action = {0};
    action.sa_handler = sigalrm_handler;
    sigemptyset(&action.sa_mask);

    if (sigaction(SIGALRM, &action, NULL) == -1) {
        perror("sigaction");
        return 1;
    }

    alarm(5);
    pause();
    return 0;
}
```
