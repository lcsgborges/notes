# TCP Sockets

Processos se encontram por IP + Porta, exemplo: `127.0.0.1:8080`. O socket continua sendo um `file descriptor` criado pelo kernel. A diferença é que agora a comunicação pode ser:

- Na mesma máquina
- Entre máquinas diferentes da mesma rede
- Pela internet

## O que é TCP Socket

Um **TCP Socket** é uma ponta de comunicação usando o protocolo **TCP** (Transmission Control Protocol). Ele oferece uma comunicação:

- Conectada
- Bidirecional
- Confiável
- Ordenada
- Em fluxo de bytes (stream)

Ou seja:

```mermaid
    flowchart LR
    A[Processo A]
    B[TCP]
    C[Processo B]
    A <-.-> B <-.-> C
```

O TCP tenta garantir que os bytes enviados de um lado cheguem do outro lado na mesma ordem.

## Endereço TCP: IP + Porta

No Unix Domain Socket, o endereço era uma arquivo, exemplo: `/tmp/app.sock`. No TCP, o endereço é `IP + Porta`. Exemplos:

- `127.0.0.1:3000`
- `192.168.22.46:8000`

### IP

Identifica uma **máquina/interface** na rede. Exemplo: `192.168.22.40`.

### Porta

Identifica o **serviço** na máquina. Exemplos:

| Porta | Serviço |
| :---: | :-----: |
| 22 | `SSH` |
| 80 | `HTTP` |
| 443 | `HTTPS` |
| 5432 | `PostgreSQL` |
| 6379 | `Redis` |
| 8000 | Muito usada em desenvolvimento |

Então, `127.0.0.1:8000`, significa:

> Conectar na própria máquina (localhost), no serviço que está escutando na porta 8000.


## Unix Domain Socket vs TCP Socket

| Característica | Unix Domain Socket | TCP Socket |
| :------------: | :----------------: | :--------: |
| Comunicação | Mesma máquina | Mesma máquina ou rede |
| Endereço | Caminho, ex: `/tmp/app.sock` | IP + Porta |
| Família | `AF_UNIX` | `AF_INET` ou `AF_INET6` |
| Estrutura | `sockaddr_un` | `sockaddr_in` ou `sockaddr_in6` |
| Uso comum | daemon local, Docker, PostgreSQL local | Servidores web, APIs, banco remoto |
| Acesso remoto | Não | Sim |

No código, a principal mudança é essa:

```c
// Unix Domain Socket
int unix_sockfd = socket(AF_UNIX, SOCK_STREAM, 0);

// TCP Socket
int tcp_sockfd = socket(AF_INET, SOCK_STREAM, 0);
```

## Fluxo de um servidor TCP

Um servidor TCP faz:

1. `socket()`
2. `bind()`
3. `listen()`
4. `accept()`
5. `read()`/`write()`
6. `close()`

Mesma ideia do Unix Domain Socket, mas agora o `bind()` associa o socket a uma porta e IP local.

Fluxo:

1. `socket()`: cria o socket
2. `bind()`: associa o socket a um IP e Porta
3. `listen()`: coloca o socket em modo servidor
4. `accept()`: espera um cliente conectar
5. `read()`/`write()`: troca dados com o cliente
6. `close()`: fecha descritores

## Fluxo de um cliente TCP

1. `socket()`
2. `connect()`
3. `write()`/`read()`
4. `close()`

Fluxo:

1. `socket`: cria o socket
2. `connect()`: conecta no IP e porta do servidor
3. `write()`: envia dados
4. `read()`: recebe resposta
5. `close()`: fecha a conexão

## A estrutura `sockaddr_in`

Para **IPv4** usamos:

```c
struct sockaddr_in
```

Ela representa `AF_INET + porta + IP`. Exemplo:

```c
struct sockaddr_in addr;

addr.sin_family = AF_INET;
addr.sin_port = htons(8000);
addr.sin_addr.s_addr = INADDR_ANY;
```

### `sin_family`

Representa a família do endereço: `AF_INET`

### `sin_port`

Representa a porta. Usamos `htons()` para deixar no formato requerido (*big-endian*): `htons(8000)`.

### `sin_addr`

Representa o endereço IPv4, no servidor podemos usar `INADDR_ANY`, que significa: **Aceite conexões em qualquer interface de máquina."

## 127.0.0.1 vs 0.0.0.0

O `127.0.0.1` escuta só na própria máquina, ou seja, **somente programas locais acessam** (útil para teste).

O `0.0.0.0` escuta em todas as interfaces disponíveis, exemplo:

- localhost
- IP da rede local
- Interfaces disponíveis

Se sua máquina tem IP `192.168.0.10`, um servidor em `0.0.0.0:8000` pode aceitar conexões de outra máquina de rede, se firewall permitir. No *C*, para servidor:

```c
addr.sin_addr.s_addr = INADDR_ANY;
```

Isso é equivalente à ideia de `0.0.0.0`.

## Servidor TCP em C

Arquivo `tcp_server.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>

#define PORT 8000
#define BUFFER_SIZE 1024

int main(void) {
    int server_fd = socket(AF_INET, SOCK_STREAM, 0);

    if (server_fd == -1) {
        perror("socket");
        return 1;
    }

    int opt = 1;

    if (setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt)) == -1) {
        perror("setsocketopt");
        close(server_fd);
        return 1;
    }

    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(addr));

    addr.sin_family = AF_INET;
    addr.sin_port = htons(PORT);
    addr.sin_addr.s_addr = INADDR_ANY;

    if (bind(server_fd, (struct sockaddr *) &addr, sizeof(addr)) == -1) {
        perror("bind");
        close(server_fd);
        return 1;
    }

    if (listen(server_fd, 10) == -1) {
        perror("listen");
        close(server_fd);
        return 1;
    }

    printf("Server TCP escutando na porta %d...\n", PORT);

    int client_fd = accept(server_fd, NULL, NULL);

    if (client_fd == -1) {
        perror("accept");
        close(server_fd);
        return 1;
    }

    printf("Client conectado\n");

    char buffer[BUFFER_SIZE];

    ssize_t n = read(client_fd, buffer, sizeof(buffer) -1);

    if (n == -1) {
        perror("read");
        close(client_fd);
        close(server_fd);
        return 1;
    }

    buffer[n] = '\0';

    printf("Client enviou: %s\n", buffer);

    const char *resp = "Mensagem recebida pelo server TCP\n";

    if (write(client_fd, resp, strlen(resp)) == -1) {
        perror("write");
    }

    close(client_fd);
    close(server_fd);
    return 0;
}
```

## Cliente TCP em C

Arquivo `tcp_client.c`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>

#define PORT 8000
#define BUFFER_SIZE 1024

int main() {
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);

    if (sockfd == -1) {
        perror("socket");
        return 1;
    }

    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));

    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(PORT);

    if (inet_pton(AF_INET, "127.0.0.1", &server_addr.sin_addr) <= 0) {
        perror("inet_pton");
        close(sockfd);
        return 1;
    }

    if (connect(sockfd, (struct sockaddr *) &server_addr, sizeof(server_addr)) == -1) {
        perror("connect");
        close(sockfd);
        return 1;
    }

    const char *msg = "Olá server TCP";

    if (write(sockfd, msg, strlen(msg)) == -1) {
        perror("write");
        close(sockfd);
        return 1;
    }

    char buffer[BUFFER_SIZE];

    ssize_t n = read(sockfd, buffer, sizeof(buffer) -1);

    if (n == -1) {
        perror("read");
        close(sockfd);
        return 1;
    }

    buffer[n] = '\0';

    printf("Server respondeu: %s\n", buffer);

    close(sockfd);
    return 0;
}
```

## Explicação das funções principais

### `socket(AF_INET, SOCK_STREAM, 0)`

```c
int fd = socket(AF_INET, SOCK_STREAM, 0);
```

Significa:

```text
AF_INET     -> IPv4
SOCK_STREAM -> fluxo conectado, ou seja, TCP
0           -> protocolo padrão para essa combinação
```

### `bind()`

No servidor:

```c
bind(server_fd, (struct sockaddr *) &addr, sizeof(addr));
```

Associa o socket a: `0.0.0.0:8000`.

### `listen()`

```c
listen(server_fd, 10);
```

Coloca o socket em modo passivo, esperando conexões. O número `10` é o **backlog**, uma fila de conexões pendentes. Significa algo mais próximo de: "Quantas conexões podem ficar aguardando até o servidor chamar `accept()`?".

### `accept()`

```c
int client_fd = accept(server_fd, NULL, NULL);
```

Espera um cliente se conectar. Quando um cliente conecta, `accept()` retorna um **novo file descriptor**.

- `server_fd`: socket de escuta
- `client_fd`: socket conectado com um cliente (comunicação)

O servidor troca dados com o cliente usando o `client_fd` apenas.

### `connect()`

No cliente:

```c
connect(sockfd, (struct sockaddr *) &server_addr, sizeof(server_addr));
```

Pede ao kernel: "Conecte este socket ao servidor `127.0.0.1:8000`. Se o servidor estiver escutando, a conexão é estabelecida.

## TCP é fluxo de bytes

TCP não preserva mensagens. Se o cliente faz:

```c
write(fd, "abc", 3);
write(fd, "def", 3);
```

O servidor pode ler `abcdef`. TCP garante ordem dos bytes, mas não garante "uma mensagem por `read()`". O correto nesses casos é criar um protocolo.

## `inet_pton()`

No cliente usamos:

```c
inet_pton(AF_INET, "127.0.0.1", &server_addr.sin_addr);
```

Isso converte uma string de IP: `"127.0.0.1"` para a forma binária que o kernel usa em `struct sockaddr_in`.

`inet_pton` significa `presentation to network`. Ou seja: `forma textual -> forma de rede`.

## `send()` e `recv()`

Além do `read()` e `write()`, podemos usar:

```c
recv(fd, buffer, tamanho, flags);
send(fd, buffer, tamanho, flags);
```

A diferença principal é:

- `read()` e `write()` são genéricos para file descriptors.
- `recv()` e `send()` são príprios para sockets e aceitam **flags** específicas.
