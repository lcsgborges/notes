# Descritores de arquivo

A frase **“tudo no Unix é um arquivo”** resume a ideia de oferecer uma interface uniforme para diferentes recursos. Um processo não acessa diretamente objetos do kernel, como arquivos, terminais, *pipes* e *sockets*. Em vez disso, usa um **descritor de arquivo** (*file descriptor*, ou `fd`), representado por um número inteiro.

Um descritor de arquivo é um número inteiro usado por um processo para se referir a um recurso aberto no kernel.

Logo, temos:

```c
int fd = open("arquivo.txt", O_RDONLY);
```

Suponha que o sistema retorne `fd = 10`. Isso significa que a entrada 10 da tabela de descritores desse processo referencia um objeto aberto no kernel. O número `10` não é o arquivo; ele é apenas uma **referência** válida naquele processo.

## A ideia histórica: "Everything is a file"

No Unix, uma das grandes ideias foi criar uma interface uniforme para muitos recursos. Em vez de cada coisa ter uma API completamente diferente, muitas operações são feitas com `read()`, `write()` e `close()`. Por exemplo, podemos usar `read()` para:

- Ler de um arquivo.
- Ler de um terminal.
- Ler de um *pipe*.
- Ler de um *socket*.

E podemos usar o `write()` para:

- Escrever em um arquivo.
- Escrever em um terminal.
- Escrever em um *pipe*.
- Escrever em um *socket*.

Por convenção, um processo normalmente inicia com três descritores:

- `0` (`stdin`): entrada padrão.
- `1` (`stdout`): saída padrão.
- `2` (`stderr`): saída padrão de erros.

Esses descritores podem apontar para um terminal, um arquivo ou outro recurso, dependendo de redirecionamentos. Quando executamos `printf("Ola\n")`, por exemplo, a biblioteca normalmente envia o conteúdo de seu *buffer* ao descritor associado a `stdout`, em uma operação conceitualmente semelhante a `write(STDOUT_FILENO, "Ola\n", 4)`.

`printf()` é uma função formatada da biblioteca padrão de C e normalmente usa um *buffer*. `write()` é a interface POSIX de mais baixo nível para escrever bytes em um descritor de arquivo.

## POSIX

**POSIX** (Portable Operating System Interface) é um padrão que define uma interface de programação para sistemas Unix-like. Ele especifica funções como:

- `open()`.
- `read()`.
- `write()`.
- `close()`.
- `pipe()`.
- `fork()`.
- Funções da família `exec`.

### `stdio.h` e POSIX

| Operações sobre arquivos | stdio.h | POSIX |
| :----------------------: | :-----: | :---: |
| Abrir | `fopen()` | `open()` |
| Fechar | `fclose()` | `close()` |
| Ler | `fread()`, `fscanf()`, `getc()` | `read()` |
| Escrever | `fwrite()`, `fprintf()`, `putc()` | `write()` |
| Outros | `fseek()`, `rewind()` | `mmap()`, `lseek()`, `poll()` |
| Vantagens | Portável, formatado e com *buffer* | Controle direto sobre descritores e opções POSIX |
| Desvantagens | O *buffering* exige atenção | Não faz parte do C padrão |

### Modos POSIX

| Modo | Significado |
| :--: | :---------: |
| `O_RDONLY` | Abre o arquivo somente para leitura |
| `O_WRONLY` | Abre o arquivo somente para escrita |
| `O_RDWR` | Abre o arquivo para leitura e escrita |
| `O_APPEND` | Faz cada escrita ocorrer no final do arquivo |
| `O_CREAT` | Cria o arquivo se ele não existir |
| Outros | [https://man7.org/linux/man-pages/man2/open.2.html](https://man7.org/linux/man-pages/man2/open.2.html) |

## Arquivos

Abrindo um arquivo de verdade:

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>

int main(void) {
    // O terceiro argumento define as permissões solicitadas.
    int fd = open("teste.txt", O_CREAT | O_WRONLY | O_TRUNC, 0644);

    if (fd == -1) {
        perror("open");
        return 1;
    }
    printf("Descritor de arquivo: %d.\n", fd);
    close(fd);
    return 0;
}
```

### O offset do arquivo

Quando lemos ou escrevemos em um arquivo regular, existe uma posição atual. Exemplo:

```text
arquivo: ABCDEFGHIJ
offset: 0
```

Se fizermos `read(fd, buffer, 3)`, lê `ABC` e o *offset* avança para 3:

```text
arquivo: ABCDEFGHIJ
offset: 3
```

Se lermos mais dois bytes, obteremos `DE`, e o *offset* passará a ser 5. Esse *offset* pertence à **descrição de arquivo aberto** (*open file description*), mantida pelo kernel e possivelmente compartilhada por mais de um descritor.

Podemos mudar o offset manualmente com `lseek()`:

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>

int main(void) {
    int fd = open("dados.txt", O_RDONLY);

    if (fd == -1) {
        perror("open");
        return 1;
    }

    char c;

    ssize_t n = read(fd, &c, 1);

    if (n != 1) {
        if (n == -1) {
            perror("read");
        } else {
            fprintf(stderr, "O arquivo está vazio.\n");
        }
        close(fd);
        return 1;
    }

    printf("Primeiro caractere: %c.\n", c);

    if (lseek(fd, 0, SEEK_SET) == -1) {
        perror("lseek");
        close(fd);
        return 1;
    }

    n = read(fd, &c, 1);

    if (n != 1) {
        if (n == -1) {
            perror("read");
        } else {
            fprintf(stderr, "O arquivo está vazio.\n");
        }
        close(fd);
        return 1;
    }

    printf("Depois de lseek(): %c.\n", c);

    close(fd);
    return 0;
}
```

Arquivos regulares aceitam `lseek()`, sejam eles de texto ou binários. *Pipes* e *sockets* não possuem uma posição desse tipo e normalmente fazem `lseek()` falhar com `ESPIPE`.

> Quando chamamos `close(fd)`, a entrada é removida da tabela de descritores do processo. A variável inteira ainda contém o mesmo número, mas ele não deve mais ser usado como descritor. A descrição de arquivo aberto é liberada quando deixa de possuir referências.

Se não fecharmos descritores que deixaram de ser necessários, o processo poderá atingir seu limite e receber o erro `Too many open files`. O limite varia conforme o sistema e a configuração; no *shell*, podemos consultar o limite flexível com `ulimit -n`.

### EOF

A chamada `read()` retorna a quantidade de bytes lidos:

- `> 0`: quantidade de bytes lidos.
- `= 0`: fim do arquivo (EOF).
- `-1`: erro, com detalhes em `errno`.

Exemplo de código que lê até o fim do arquivo:

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>

int main(void) {
    int fd = open("dados.txt", O_RDONLY);

    if (fd == -1) {
        perror("open");
        return 1;
    }

    char buffer[1024];
    ssize_t n;

    while ((n = read(fd, buffer, sizeof(buffer))) > 0) {
        size_t total = 0;

        while (total < (size_t)n) {
            ssize_t written = write(
                STDOUT_FILENO,
                buffer + total,
                (size_t)n - total
            );

            if (written == -1) {
                perror("write");
                close(fd);
                return 1;
            }

            total += (size_t)written;
        }
    }

    // Verifica se a leitura terminou por causa de um erro.
    if (n == -1) {
        perror("read");
        close(fd);
        return 1;
    }

    close(fd);
    return 0;
}
```

Esse programa é uma versão mínima do comportamento do `cat`.
