# Shared Memory

**Shared Memory** é um mecanismo de IPC onde dois ou mais processos acessam a **mesma região de memória**.

A ideia é:

```mermaid
    flowchart TD
    A[Processo A]
    B[Processo B]
    C[Mesma região na memória]
    A --> C
    B --> C
```

Aqui os processos **não enviam dados através de `read()`/`write()`. Eles acessam diretamente uma área de memória compartilhada.

## O problema que Shared Memory resolve

Normalmente, processos são isolados. Se temos `int x = 10` no **Processo A**, o **Processo B** não consegue acessar esse `x`.

Isso é segurança e isolamento, mas às vezes queremos que dois processos compartilhem dados rapidamente. Exemplo:

- Um processo produz dados, outro processo consome dados
- Um processo escreve estado atual, outro processo lê esse estado
- Vários processos acessam uma tabela comum

Com `pipe`/`socket`/`message queue`, os dados passam pelo kernel em forma de bytes ou mensagens. Com `shared memory`, os processos acessam a mesma região diretamente.

### Por quê Shared Memory é rápida?

Porque evita ficar copiando dados entre processos. Isso é muito eficiente para grandes volumes de dados.

## O grande problema: sincronização

Shared memory é rápida, mas perigosa. Imagine:

```text
Processo A está escrevendo uma struct
Processo B lê no meio da escrita
```

O processo B pode ver dados incompletos. Então a regra é: **shared memory quase sempre precisa de sincronização**. Normalmente usamos:

- semáforos
- mutexes compartilhados
- futexes
- cond vars compartilhadas
- spinlocks

## Formas comuns de Shared Memory em C/Linux

As principais formas são:

1. POSIX Shared Memory:
    - `shm_open()`
    - `ftruncate()`
    - `mmap()`
2. `mmap()` anônimo com MAP_SHARED
    - útil entre pai e filho após `fork()`
3. System V Shared Memory
    - `shmget()`
    - `shmat()`
    - `shmdt()`
    - `shmctl()`

A mais limpa é a **POSIX Shared Memory**.

## Fluxo POSIX Shared Memory

O processo que cria a memória faz:

```c
shm_open()
ftruncate()
mmap()
// Usar a memória
munmap()
close()
shm_unlink()
```

Outro processo faz:

```c
shm_open()
mmap()
// Usar a memória
munmap()
close()
```

A ideia:

- `shm_open()`: cria/abre um objeto de memória compartilhada
- `ftruncate()`: define o tamanho
- `mmap()`: mapeia esse objeto no espaço de memória do processo.

O ponto mais importante é `mmap()`. Ele faz uma região aparecer dentro da memória virtual do processo.

## Primeiro exemplo: escritor e leitor

Vamos criar dois programas:

1. `writer.c`: cria a memória compartilhada e escreve uma mensagem.
2. `reader.c`: abre a memória compartilhada e lê a mensagem.

Nome da memória compartilhada: `/minha_memoria`.

Em POSIX, o nome geralmente começa com `/`.

### `writer.c`

```c
#include <stdio.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/mman.h>
#include <sys/stat.h>

#define SHM_NAME "/minha_memoria"
#define SHM_SIZE 4096

int main(void) {
    int fd = shm_open(SHM_NAME, O_CREAT | O_RDWR, 0666);

    if (fd == -1) {
        perror("shm_open");
        return 1;
    }

    if (ftruncate(fd, SHM_NAME) == -1) {
        perror("ftruncate");
        close(fd);
        return 1;
    }

    void *ptr = mmap(NULL, SHM_SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
   
    if (ptr == MAP_FAILED) {
        perror("mmap");
        close(fd);
        return 1;
    }  

    const char *msg = "Olá pela memória compartilhada!";

    strcpy((char *) ptr, msg);

    printf("Writer escreveu: %s\n", msg);

    munmap(str, SHM_SIZE);
    
    close(fd);
    return 0;
}
```

### `reader.c`

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/mman.h>
#include <sys/stat.h>

#define SHM_NAME "/minha_memoria"
#define SHM_SIZE 4096

int main(void) {
    int fd = shm_open(SHM_NAME, O_RDONLY, 0666);

    if (fd == -1) {
        perror("shm_open");
        return 1;
    }

    void *ptr = mmap(NULL, SHM_SIZE, PROT_READ, MAP_SHARED, fd, 0);

    if (ptr == MAP_FAILED) {
        perror("mmap");
        close(fd);
        return 1;
    }

    printf("Reader leu: %s\n", (char *)ptr);

    munmap(ptr, SHM_SIZE); // Desfaz o mapeamento
    close(fd);

    // Remove o objeto de shared memory do sistema.
    // Faça isso quando não precisar mais dele
    shm_unlink(SHM_NAME);
    return 0;
}
```

## `shm_open()` retorna file descriptor

```c
int fd = shm_open(...)
```

Ele retorna um file descriptor, parecido com `open()`. Mas esse `fd` representa um **objeto de memória compartilhada**, não um arquivo comum. Depois usamos esse `fd` no `mmap()`.

```mermaid
    flowchart LR
    A[shm_open()]
    B[fd do objeto de shared memory]
    C[mmap()]
    D[ponteiro para a memória compartilhada]
    A -> B
    B -> C
    C -> D
```

## `ftruncate()` define o tamanho

Quando criamos um objeto com `shm_open()`, ele pode ter tamanho 0. Então precisamos definir o tamanho:

```c
ftruncate(fd, SHM_SIZE);
```

Sem isso, o `mmap()` pode falhar ou podemos acessar memória inválida.

- `shm_open()`: cria o objeto
- `ftruncate()`: define o tamanho
- `mmap()`: coloca esse objeto na memória do processo

## `mmap()` é o centro da ideia

A chamada:

```c
void *ptr = mmap(NULL, SHM_SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
```

Significa:

- `NULL`: kernel escolhe o endereço virtual
- `SHM_SIZE`: tamanho da região
- `PROT_READ | PROT_WRITE`: posso ler e escrever
- `MAP_SHARED`: alterações são compartilhadas com outros processos
- `fd`: objeto que será mapeado
- `0`: offset inicial

O retorno é um ponteiro `void *ptr`. Depois disso, usamos como memória comum:

```c
char *mem = ptr;
mem[0] = 'A';
strcpy(mem, "texto");
```

## `MAP_SHARED` vs `MAP_PRIVATE`

- `MAP_SHARED`: alterações são visíveis para outros processos.
- `MAP_PRIVATE`: alterações são privadas do processo (copy-on-write)

Se usar `MAP_PRIVATE`, cada processo pode ver sua própria cópia modificada.

## Shared Memory com structs

Vamos compartilhar uma struct:

```c
typedef struct {
    int id;
    double valor;
    char status[64];
} Dados;
```

Arquivo `writer_struct.c`:

```c
#include <stdio.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/mman.h>
#include <sys/stat.h>

#define SHM_NAME "/dados_compartilhados"

typedef struct {
    int id;
    double valor;
    char status[64];
} Dados;

int main(void) {
    int fd = shm_open(SHM_NAME, O_CREAT | O_RDWR, 0666);

    if (fd == -1) {
        perror("shm_open");
        return 1;
    }

    if (ftruncate(fd, sizeof(Dados)) == -1) {
        perror("ftruncate");
        close(fd);
        return 1;
    }

    Dados *dados = mmap(NULL, sizeof(Dados), PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);

    if (dados == MAP_FAILED) {
        perror("mmap");
        close(fd);
        return 1;
    }

    dados->id = 101;
    dados->valor = 250.75;
    strcpy(dados->status, "processado");

    printf("Writer escreveu struct.\n");

    munmap(dados, sizeof(Dados));
    close(fd);
    
    return 0;
}
```

Arquivo `reader_struct.c`:

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/mman.h>
#include <sys/stat.h>

#define SHM_NAME "/dados_compartilhados"

typedef struct {
    int id;
    double valor;
    char status[64];
} Dados;

int main(void) {
    int fd = shm_open(SHM_NAME, O_RDONLY, 0666);

    if (fd == -1) {
        perror("shm_open");
        return 1;
    }

    Dados *dados = mmap(NULL, sizeof(Dados), PROT_READ, MAP_SHARED, fd, 0);

    if (dados == MAP_FAILED) {
        perror("mmap");
        close(fd);
        return 1;
    }

    printf("ID: %d\n", dados->id);
    printf("Valor: %.2f\n", dados->valor);
    printf("Status: %s\n", dados->status);

    munmap(dados, sizeof(Dados)); // Desfaz o mapeamento
    close(fd);
    shm_unlink(SHM_NAME);

    return 0;
}
```



Em Shared Memory, prefira:

- Arrays Fixos
- Inteiros
- Floats/Doubles
- Structs Simples
- Offsets em vez de ponteiros

Uma vez que ponteiro é endereço virtual do processo, no Processo A não aponta necessariamente para o mesmo lugar no processo B.

Regra: **Não compartilhe ponteiros crus entre processos. Compartilhe dados**.
