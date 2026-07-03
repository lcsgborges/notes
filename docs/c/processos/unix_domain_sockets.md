# Soquetes de domínio Unix

Um **soquete de domínio Unix** (*Unix domain socket*) é uma forma de comunicação entre processos que estão na **mesma máquina**. Em resumo, é um mecanismo de **IPC**.

Ele funciona como um canal de comunicação **bidirecional**, controlado pelo kernel e representado por descritores de arquivo.

## O que é um socket

Um *socket* é uma extremidade de comunicação. Cada processo utiliza uma extremidade, criada e controlada pelo kernel. Em C, um *socket* é representado por um descritor de arquivo: `int fd = socket(...)`.

## Diferença entre *pipe* e *socket*

Um ***pipe*** é um canal de comunicação unidirecional. Já um **soquete de domínio Unix** permite a comunicação nos dois sentidos pelo mesmo canal, sendo adequado para interações como:

- O cliente pergunta, e o servidor responde.
- O cliente envia um comando, e o servidor confirma o recebimento.
- O processo A envia dados, e o processo B devolve o resultado.

## Endereços de socket

As estruturas `sockaddr`, `sockaddr_un`, `sockaddr_in`, `sockaddr_in6` e `sockaddr_storage` **não são o *socket* em si; elas representam endereços de *sockets***.

A estrutura `sockaddr_*` é o **endereço usado para localizar ou identificar uma ponta desse canal**.

- ***Socket***: o canal de comunicação.
- **Endereço de *socket***: onde esse canal pode ser encontrado.

Exemplo com um soquete de domínio Unix: endereço local `/tmp/app.sock`.

Exemplo com IPv4: endereço de rede `192.168.0.10:8000`.

Exemplo com IPv6: endereço de rede `[2804:abcd::1]:8080`.

### Por que utilizar várias estruturas

Uma conexão feita com um soquete de domínio Unix é diferente de uma conexão feita com IPv4 ou IPv6. Dessa forma, as estruturas são especializadas:

- `sockaddr_un`: endereço local Unix.
- `sockaddr_in`: endereço IPv4.
- `sockaddr_in6`: endereço IPv6.

Entretanto, as funções do sistema **precisam aceitar qualquer uma delas**. A estrutura genérica usada para isso é `sockaddr`.

> Como analogia, podemos pensar em `sockaddr` como uma interface genérica e nas demais estruturas como representações especializadas.

A API de sockets usa funções genéricas como:

- `bind()`.
- `connect()`.
- `accept()`.
- `sendto()`.
- `recvfrom()`.

Essas funções não precisam distinguir diretamente se estamos usando soquetes de domínio Unix, IPv4 ou IPv6. Portanto, elas recebem um **ponteiro genérico**:

```c
struct sockaddr *
```

Exemplo:

```c
bind(fd, (struct sockaddr *) &addr, sizeof(addr));
```

Nesse exemplo, `addr` pode ser do tipo `sockaddr_un`, `sockaddr_in` ou outro tipo compatível.

O importante aqui é saber que todas elas começam com um campo que informa a **família do endereço**.

### Família de endereço

Cada estrutura tem um campo de família. Na estrutura genérica, ele é `sa_family`; nas demais:

- Soquete de domínio Unix: `sun_family`.
- IPv4: `sin_family`.
- IPv6: `sin6_family`.

Elas indicam coisas como:

- `AF_UNIX`: endereço local Unix.
- `AF_INET`: endereço IPv4.
- `AF_INET6`: endereço IPv6.

Quando o kernel recebe um `struct sockaddr *`, ele verifica primeiro a família. Exemplo conceitual:

1. "Recebi um endereço."
2. "O primeiro campo contém `AF_UNIX`."
3. "Então, interpretarei o restante como `sockaddr_un`."

### A estrutura genérica `sockaddr`

A estrutura genérica é mais ou menos assim:

```c
#include <sys/socket.h>

struct sockaddr {
    sa_family_t sa_family; // Família de endereços (por exemplo, AF_INET).
    char        sa_data[14]; // 14 bytes para o endereço.
};
```

- `sa_family_t`: família do endereço (`AF_UNIX`, `AF_INET` etc.).
- `sa_data`: espaço genérico para os dados do endereço.

Na prática, quase nunca preenchemos `sa_data` diretamente.

### A função bind

A função `bind()` recebe:

```c
#include <sys/socket.h>

int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

Normalmente, passamos outra estrutura convertida com um *cast*.

### Ponto de compatibilidade

As estruturas são desenhadas para começarem de forma compatível:

#### `struct sockaddr`

```c
struct sockaddr {
    sa_family_t sa_family;
    char        sa_data[];
};
```

#### `sockaddr_un`

```c
struct sockaddr_un {
    sa_family_t sun_family;  // Address family (AF_UNIX)
    char        sun_path[];  // Socket pathname
};
```

Exemplo:

```c
struct sockaddr_un addr;

memset(&addr, 0, sizeof(addr));

addr.sun_family = AF_UNIX;
strncpy(addr.sun_path, "/tmp/app.sock", sizeof(addr.sun_path) -1);
```

O exemplo acima representa o **endereço** `"/tmp/app.sock"` usado por um soquete de domínio Unix.

> `strncpy()` copia uma quantidade máxima de caracteres de uma *string* de origem para outra de destino. Se a origem for menor que o limite especificado, a função preencherá o restante com `'\0'`. Se não for, o resultado poderá não ter um terminador nulo.

##### Exemplo: preparando um endereço Unix

```c
#include <stdio.h>
#include <string.h>
#include <sys/un.h>

int main(void) {
    struct sockaddr_un addr;
    
    memset(&addr, 0, sizeof(addr));

    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, "/tmp/app.sock", sizeof(addr.sun_path) -1);

    printf("Família: %d\n", addr.sun_family);
    printf("Caminho: %s\n", addr.sun_path);
    
    return 0;
}
```

#### `sockaddr_in`

```c
struct sockaddr_in {
    sa_family_t    sin_family; // AF_INET
    in_port_t      sin_port;   // Port number
    struct in_addr sin_addr;   // IPv4 address
};
```

Seria representado com:

```c
addr.sin_family = AF_INET;
addr.sin_port = htons(8080);
addr.sin_addr = ...
```

A função `htons()` (*host to network short*) converte um inteiro de 16 bits da ordem de bytes nativa do processador (*host*) para a ordem padronizada utilizada nas redes TCP/IP (*network byte order*), que é **big-endian**.

Diferentes processadores podem armazenar números de formas distintas. Por exemplo, os processadores x86 são ***little-endian***: o byte menos significativo vem primeiro. Já os protocolos de rede usam o formato ***big-endian***.

Portanto, `htons()` inverte a ordem dos bytes apenas se o sistema for *little-endian*. Em sistemas nativamente *big-endian*, ela preserva a ordem.

#### `sockaddr_in6`

```c
struct sockaddr_in6 {
    sa_family_t     sin6_family;    // AF_INET6 
    in_port_t       sin6_port;      // Port number
    uint32_t        sin6_flowinfo;  // IPv6 flow info
    struct in6_addr sin6_addr;      // IPv6 address 
    uint32_t        sin6_scope_id;  // Set of interfaces for a scope
};
```

#### `sockaddr_storage`

Imagine que precisamos receber um endereço cuja família ainda não conhecemos. Nesse caso, precisamos de um espaço grande o suficiente para qualquer endereço. Para isso, usamos `sockaddr_storage`, que pode armazenar os diferentes tipos de endereço. Exemplo:

```c
struct sockaddr_storage addr;
socklen_t len = sizeof(addr);
```

Depois passamos para uma função que preenche o endereço:

```c
accept(fd, (struct sockaddr *) &addr, &len);
```

Depois podemos verificar a família:

```c
struct sockaddr *sa = (struct sockaddr *) &addr;

if (sa->sa_family == AF_UNIX) {
    // Endereço Unix
} else if (sa->sa_family == AF_INET) {
    // Endereço IPv4
} else if (sa->sa_family == AF_INET6) {
    // Endereço IPv6
}
```

### Tabela de resumo

| Estrutura | Família | Uso | Guarda |
| :-------: | :-----: | :-: | :----: |
| `struct sockaddr` | Genérica | Base das APIs | Família + bytes genéricos |
| `struct sockaddr_un` | `AF_UNIX` | IPC local | Caminho, como `/tmp/app.sock` |
| `struct sockaddr_in` | `AF_INET` | IPv4 | Porta + IP de 32 bits |
| `struct sockaddr_in6` | `AF_INET6` | IPv6 | Porta + IP de 128 bits |
| `struct sockaddr_storage` | Qualquer | Código genérico | Espaço grande o suficiente |

### Por que passar o tamanho no `bind()`?

Usamos o `sizeof(addr)` no `bind()` porque o ponteiro geralmente é convertido para o tipo genérico `(struct sockaddr *)`. Depois do cast, a função não sabe sozinha qual era o tamanho real da estrutura original. Exemplo:

```c
struct sockaddr_un addr;

bind(fd, (struct sockaddr *) &addr, sizeof(addr));
```

Aqui estamos dizendo que o endereço é genérico para a API, mas o tamanho real dele é `sizeof(struct sockaddr_un)`.

### Por que usamos `memset()`?

Algumas estruturas possuem campos extras ou preenchimento (*padding*). Zerar toda a estrutura evita valores indeterminados nos campos não preenchidos:

```c
struct sockaddr_un addr;

memset(&addr, 0, sizeof(addr));

addr.sun_family = AF_UNIX;
strncpy(addr.sun_path, "/tmp/app.sock", sizeof(addr.sun_path) -1);
```

### Por que usar `strncpy()` em `sun_path`?

Porque `sun_path` tem tamanho limitado, normalmente definido como algo semelhante a `char sun_path[108]`. Se fizermos:

```c
strcpy(addr.sun_path, path);
```

Se `path` for grande demais, poderemos ultrapassar o limite do *buffer*. Por isso, usamos `strncpy()`. Como a estrutura foi zerada com `memset()` e copiamos, no máximo, `sizeof(addr.sun_path) - 1` bytes, o último byte continuará sendo `'\0'`.

## Socketpair

A função `socketpair()` cria dois *sockets* conectados, como se o kernel entregasse dois telefones já em ligação.

```c
int sv[2];
socketpair(AF_UNIX, SOCK_STREAM, 0, sv);
```

Depois disso, temos:

- `sv[0]`: uma extremidade.
- `sv[1]`: outra extremidade.

Exemplo:

```c
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <sys/socket.h>

int main(void) {
    int sv[2];

    if (socketpair(AF_UNIX, SOCK_STREAM, 0, sv) == -1) {
        perror("socketpair");
        return 1;
    }

    const char *msg = "Olá pelo Unix Domain Socket";
    write(sv[0], msg, strlen(msg));

    char buffer[100];

    ssize_t n = read(sv[1], buffer, sizeof(buffer) - 1);

    if (n == -1) {
        perror("read");
        return 1;
    }

    buffer[n] = '\0';

    printf("Recebido: %s\n", buffer);

    close(sv[0]);
    close(sv[1]);
    
    return 0;
}
```

Agora, veja um exemplo de comunicação entre pai e filho com `fork()`:

```c
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <sys/socket.h>
#include <sys/wait.h>

int main(void) {
    int sv[2];

    if (socketpair(AF_UNIX, SOCK_STREAM, 0, sv) == -1) {
        perror("socketpair");
        return 1;
    }

    pid_t pid = fork();

    if (pid == -1) {
        perror("fork");
        return 1;
    }

    if (pid == 0) {
        close(sv[0]);
        char buffer[100];
        ssize_t n = read(sv[1], buffer, sizeof(buffer) -1);

        if (n == -1) {
            perror("read");
            close(sv[1]);
            return 1;
        }

        buffer[n] = '\0';
        printf("Filho recebeu = %s\n", buffer);

        const char *resp = "Oi pai, tudo bem. Recebi sua mensagem.";
        write(sv[1], resp, strlen(resp));

        close(sv[1]);
        return 0;
    } else {
        close(sv[1]);

        const char *msg = "Oi filho, tudo bem?";
        write(sv[0], msg, strlen(msg));

        char buffer[100];

        ssize_t n = read(sv[0], buffer, sizeof(buffer) - 1);

        if (n == -1) {
            perror("read");
            close(sv[0]);
            return 1;
        }

        buffer[n] = '\0';

        printf("Pai recebeu = %s\n", buffer);
        close(sv[0]);
        wait(NULL);
    }
    return 0;
}
```

`socketpair()` funciona bem quando os processos têm relação, como pai e filho. Para dois processos independentes, precisamos criar um endereço local, que pode ser representado por uma entrada `.sock` no sistema de arquivos.

## Modelo cliente-servidor

Agora, temos um novo modelo: um processo espera conexões (**servidor**), e outro se conecta a ele (**cliente**). O fluxo é:

1. O servidor cria um *socket*.
2. O servidor atribui um endereço local a ele, como `/tmp/app.sock`.
3. O servidor espera por um cliente.
4. O cliente cria um *socket*.
5. O cliente se conecta a `/tmp/app.sock`.
6. O servidor aceita a conexão.
7. Os dois processos se comunicam.

### Funções do servidor

Um servidor com soquete de domínio Unix usa:

- `socket()`: cria uma extremidade de comunicação.
- `bind()`: atribui um endereço local à extremidade, como `/tmp/app.sock`.
- `listen()`: coloca o *socket* no estado de escuta.
- `accept()`: espera a conexão de um cliente e cria um *socket* específico para ele.
- `read()`: lê os dados recebidos.
- `write()`: escreve os dados que serão enviados.
- `close()`: fecha os descritores.
- `unlink()`: remove a entrada `.sock` do sistema de arquivos.

### *Socket* de escuta e *socket* conectado

No servidor, existem dois tipos de descritores de arquivo:

- `server_fd`: *socket* que espera conexões.
- `client_fd`: *socket* conectado a um cliente específico.

O servidor não conversa com o cliente usando `server_fd`. Ele conversa usando o `client_fd`, que vem do `accept()`.

### Servidor Unix Domain Socket

Arquivo: `server.c`

```c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <sys/un.h>

#define SOCKET_PATH "/tmp/my_socket.sock"
#define BUFFER_SIZE 1024

int main(void) {
    int server_fd = socket(AF_UNIX, SOCK_STREAM, 0);

    if (server_fd == -1) {
        perror("socket");
        return 1;
    }

    unlink(SOCKET_PATH);

    struct sockaddr_un addr;
    memset(&addr, 0, sizeof(addr));

    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, SOCKET_PATH, sizeof(addr.sun_path) -1);

    if (bind(server_fd, (struct sockaddr *) &addr, sizeof(addr)) == -1) {
        perror("bind");
        close(server_fd);
        return 1;
    }

    if (listen(server_fd, 5) == -1) {
        perror("listen");
        close(server_fd);
        unlink(SOCKET_PATH);
        return 1;
    }

    printf("Servidor esperando em: %s\n", SOCKET_PATH);

    int client_fd = accept(server_fd, NULL, NULL);

    if (client_fd == -1) {
        perror("accept");
        close(server_fd);
        unlink(SOCKET_PATH);
        return 1;
    }

    char buffer[BUFFER_SIZE];

    ssize_t n = read(client_fd, buffer, sizeof(buffer) -1);

    if (n == -1) {
        perror("read");
        close(server_fd);
        close(client_fd);
        unlink(SOCKET_PATH);
        return 1;
    }

    buffer[n] = '\0';

    printf("Cliente disse = %s\n", buffer);

    const char *resp = "Servidor recebeu sua mensagem\n";

    write(client_fd, resp, strlen(resp));

    close(client_fd);
    close(server_fd);

    unlink(SOCKET_PATH);
    
    return 0;
}
```

### Cliente Unix Domain Socket

Arquivo: `client.c`

```c
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <sys/un.h>

#define SOCKET_PATH "/tmp/my_socket.sock"
#define BUFFER_SIZE 1024

int main(void) {
    int sockfd = socket(AF_UNIX, SOCK_STREAM, 0);

    if (sockfd == -1) {
        perror("socket");
        return 1;
    }

    struct sockaddr_un addr;
    memset(&addr, 0, sizeof(addr));

    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, SOCKET_PATH, sizeof(addr.sun_path) -1);

    if (connect(sockfd, (struct sockaddr *) &addr, sizeof(addr)) == -1) {
        perror("connect");
        close(sockfd);
        return 1;
    }

    const char *msg = "Olá servidor";

    write(sockfd, msg, strlen(msg));

    char buffer[BUFFER_SIZE];

    ssize_t n = read(sockfd, buffer, sizeof(buffer) -1);

    if (n == -1) {
        perror("read");
        close(sockfd);
        return 1;
    }

    buffer[n] = '\0';

    printf("Resposta: %s\n", buffer);

    close(sockfd);
    return 0;
}
```

- `AF_UNIX`: família de endereços locais Unix.
- `SOCK_STREAM`: canal conectado, bidirecional e baseado em um fluxo de bytes.

## Arquivo `.sock`

Quando o servidor acima estiver em execução, poderemos verificar sua entrada no sistema de arquivos:

```bash
ls -l /tmp/my_socket.sock
```

Pode aparecer algo como:

```text
srwxr-xr-x 1 ...
```

Esse `s` no início representa um *socket*. **Não** é um arquivo de texto nem um arquivo binário comum, e os dados não ficam armazenados nele. **É um ponto de encontro.**

## Onde é usado

Exemplos comuns:

- O Docker usa `/var/run/docker.sock`.
- O PostgreSQL pode aceitar conexões locais por um soquete Unix.
- O Nginx pode se comunicar com um *backend* por um soquete Unix.
- *Daemons* locais podem expor comandos por um soquete Unix.

A frase central é:

**Um soquete de domínio Unix funciona como uma "tomada local" na qual um processo servidor fica em escuta e os processos clientes se conectam para trocar dados, tudo dentro da mesma máquina.**
