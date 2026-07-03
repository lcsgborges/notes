# Soquetes TCP

Os processos se comunicam por meio de um endereço IP e de uma porta, como em `127.0.0.1:8080`. O *socket* continua sendo representado por um descritor de arquivo criado pelo kernel. A diferença é que, agora, a comunicação pode ocorrer:

- Na mesma máquina.
- Entre máquinas diferentes da mesma rede.
- Pela internet.

## O que é um *socket* TCP?

Um ***socket* TCP** é uma extremidade de comunicação que usa o **TCP** (*Transmission Control Protocol*). Ele oferece uma comunicação:

- Conectada.
- Bidirecional.
- Confiável.
- Ordenada.
- Baseada em um fluxo de bytes (*stream*).

Ou seja:

```mermaid
    flowchart LR
    A[Processo A]
    B[TCP]
    C[Processo B]
    A <-.-> B <-.-> C
```

O TCP garante que os bytes entregues ao destino sejam apresentados na mesma ordem em que foram enviados.

## Endereço TCP: IP e porta

Em um soquete de domínio Unix, o endereço pode ser um caminho, como `/tmp/app.sock`. No TCP, o endereço é composto por um IP e uma porta. Exemplos:

- `127.0.0.1:3000`.
- `192.168.22.46:8000`.

### IP

Identifica uma **interface de rede**. Exemplo: `192.168.22.40`.

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

Portanto, `127.0.0.1:8000` significa:

> Conectar-se à própria máquina (*localhost*), no serviço que está em escuta na porta 8000.


## Soquete de domínio Unix e soquete TCP

| Característica | Soquete de domínio Unix | Soquete TCP |
| :------------: | :----------------: | :--------: |
| Comunicação | Mesma máquina | Mesma máquina ou rede |
| Endereço | Caminho, como `/tmp/app.sock` | IP e porta |
| Família | `AF_UNIX` | `AF_INET` ou `AF_INET6` |
| Estrutura | `sockaddr_un` | `sockaddr_in` ou `sockaddr_in6` |
| Uso comum | *Daemon* local, Docker e PostgreSQL local | Servidores *web*, APIs e banco remoto |
| Acesso remoto | Não | Sim |

No código, a principal mudança é esta:

```c
// Soquete de domínio Unix.
int unix_sockfd = socket(AF_UNIX, SOCK_STREAM, 0);

// Soquete TCP.
int tcp_sockfd = socket(AF_INET, SOCK_STREAM, 0);
```

## Fluxo de um servidor TCP

Um servidor TCP executa as seguintes operações:

1. `socket()`.
2. `bind()`.
3. `listen()`.
4. `accept()`.
5. `read()` e `write()`.
6. `close()`.

A ideia é a mesma de um soquete de domínio Unix, mas, agora, `bind()` associa o *socket* a um endereço IP local e a uma porta.

Fluxo:

1. `socket()`: cria o *socket*.
2. `bind()`: associa o *socket* a um IP e a uma porta.
3. `listen()`: coloca o *socket* no modo de escuta.
4. `accept()`: espera que um cliente se conecte.
5. `read()` e `write()`: trocam dados com o cliente.
6. `close()`: fecha os descritores.

## Fluxo de um cliente TCP

1. `socket()`.
2. `connect()`.
3. `write()`.
4. `read()`.
5. `close()`.

Fluxo:

1. `socket()`: cria o *socket*.
2. `connect()`: conecta-se ao IP e à porta do servidor.
3. `write()`: envia dados.
4. `read()`: recebe a resposta.
5. `close()`: fecha a conexão.

## A estrutura `sockaddr_in`

Para **IPv4**, usamos:

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

Representa a família do endereço: `AF_INET`.

### `sin_port`

Representa a porta. Usamos `htons()` para deixar no formato requerido (*big-endian*): `htons(8000)`.

### `sin_addr`

Representa o endereço IPv4. No servidor, podemos usar `INADDR_ANY`, que significa: **aceite conexões em qualquer interface da máquina**.

## `127.0.0.1` e `0.0.0.0`

Quando um servidor usa `127.0.0.1`, ele aceita conexões apenas da própria máquina, ou seja, **somente programas locais podem acessá-lo**. Isso é útil para testes.

Quando um servidor usa `0.0.0.0`, ele aceita conexões por todas as interfaces disponíveis. Por exemplo:

- *Loopback*.
- Interface da rede local.
- Outras interfaces disponíveis.

Se uma máquina tiver o IP `192.168.0.10`, um servidor em `0.0.0.0:8000` poderá aceitar conexões de outra máquina da rede, se o *firewall* permitir. Em C, usamos no servidor:

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
        perror("setsockopt");
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

    printf("Servidor TCP escutando na porta %d...\n", PORT);

    int client_fd = accept(server_fd, NULL, NULL);

    if (client_fd == -1) {
        perror("accept");
        close(server_fd);
        return 1;
    }

    printf("Cliente conectado.\n");

    char buffer[BUFFER_SIZE];

    ssize_t n = read(client_fd, buffer, sizeof(buffer) -1);

    if (n == -1) {
        perror("read");
        close(client_fd);
        close(server_fd);
        return 1;
    }

    buffer[n] = '\0';

    printf("Cliente enviou: %s\n", buffer);

    const char *resp = "Mensagem recebida pelo servidor TCP.\n";

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

int main(void) {
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

    const char *msg = "Olá, servidor TCP!";

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

    printf("O servidor respondeu: %s\n", buffer);

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

Coloca o *socket* no modo passivo, à espera de conexões. O número `10` define o ***backlog***, relacionado à fila de conexões pendentes. Em termos simples, ele limita quantas conexões podem aguardar enquanto o servidor ainda não chamou `accept()`.

### `accept()`

```c
int client_fd = accept(server_fd, NULL, NULL);
```

Espera que um cliente se conecte. Quando isso acontece, `accept()` retorna um **novo descritor de arquivo**.

- `server_fd`: *socket* de escuta.
- `client_fd`: *socket* conectado a um cliente.

O servidor troca dados com o cliente usando apenas `client_fd`.

### `connect()`

No cliente:

```c
connect(sockfd, (struct sockaddr *) &server_addr, sizeof(server_addr));
```

Solicita ao kernel: "Conecte este *socket* ao servidor `127.0.0.1:8000`." Se o servidor estiver em escuta, a conexão será estabelecida.

## TCP é fluxo de bytes

TCP não preserva mensagens. Se o cliente faz:

```c
write(fd, "abc", 3);
write(fd, "def", 3);
```

O servidor pode ler `abcdef`. TCP garante ordem dos bytes, mas não garante "uma mensagem por `read()`". O correto nesses casos é criar um protocolo.

## `inet_pton()`

No cliente, usamos:

```c
inet_pton(AF_INET, "127.0.0.1", &server_addr.sin_addr);
```

Isso converte uma *string* de IP, `"127.0.0.1"`, para a forma binária que o kernel usa em `struct sockaddr_in`.

`inet_pton` significa *presentation to network*, ou seja: `forma textual -> forma de rede`.

## `send()` e `recv()`

Além de `read()` e `write()`, podemos usar:

```c
recv(fd, buffer, tamanho, flags);
send(fd, buffer, tamanho, flags);
```

A diferença principal é:

- `read()` e `write()` são genéricos para descritores de arquivo.
- `recv()` e `send()` são próprios para *sockets* e aceitam opções (*flags*) específicas.
