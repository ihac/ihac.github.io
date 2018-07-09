---
title: Network Programming Cheet Sheet
date: 2018-07-09 23:21:43
tags: [Cheet Sheet, Network Programming, C]
---

Mainly from *Unix Network Programming*

----------

## Data Types, Functions and Macros

>**IP Address**

``` c
// IPv4
struct in_addr {
    in_addr_t s_addr;
};
struct sockaddr_in {
    uint8_t sin_len;
    sa_family_t sin_family;
    in_port_t sin_port;
    struct in_addr sin_addr;
    char sin_zero[8];
}
```

``` c
// IPv6
struct in6_addr {
    uint8_t s6_addr[16];
};
#define SIN6_LEN /* required for compile-time tests */
struct sockadd_in6 {
    uint8_t sin6_len;
    sa_family_t sin6_family;
    in_port_t sin6_port;
    uint32_t sin6_flowinfo;
    struct in6_addr sin6_addr;
    uint32_t sin6_scope_id;
};
```
``` c
// general
struct sockaddr {
    uint8_t sa_len;
    sa_family_t sa_family;
    char sa_data[14];
};
```
<br>
>**MACRO**

``` c
#define INET_ADDRSTRLEN 16
#define INET6_ADDRSTRLEN 46

// family
#define AF_INET // IPv4
#define AF_INET6 // IPv6
#define AF_LOCAL // Unix域协议
#define AF_ROUTE // 路由套接字
#define AF_KEY // 密钥套接字

// type
#define SOCK_STREAM // 字节流套接字
#define SOCK_DGRAM // 数据包套接字
#define SOCK_SEQPACKET // 有序分组套接字
#define SOCK_RAW // 原始套接字

// protocol
#define IPPROTO_TCP // tcp
#define IPPROTO_UDP // udp
#define IPPROTO_SCTP //sctp
```
<br>
>**family + type**

|\    |AF_INET|AF_INET6|AF_LOCAL|AF_ROUTE|AF_KEY|
|:---:|:---:|:---:|:---:|:---:|:---:|
|SOCK_STREAM|TCP\|SCTP|TCP\|SCTP|Yes|
|SOCK_DGRAM|UDP|UDP|Yes|-|-|
|SOCK_SEQPACKET|SCTP|SCTP|Yes|-|-|
|SOCK_RAW|IPv4|IPv6|-|Yes|Yes|

<br>
>**main-Functions**

``` c
#include "sys/socket.h"
int socket(int family, int type, int protocol); // return non-negative sockfd if succeed; -1 if error
int connect(int sockfd, const struct sockaddr *servaddr, socklen_t addrlen);
    // return 0 if succeed; -1 if error
int bind(int sockfd, const struct sockaddr *myaddr, socklen_t addrlen);
    // return 0 if succeed; -1 if error
int listen(int sockfd, int backlog); // return 0 if succeed; -1 if error
int accept(int sockfd, struct sockaddr *cliaddr, socklen_t *addrlen);
    // return non-negative sockfd if succeed; -1 if error
int close(sockfd);

// sock
int getsockname(int sockfd, struct sockaddr *localaddr, socklen_t *addrlen);
int getpeername(int sockfd, struct sockaddr *peeraddr, socklen_t *addrlen);
    // both return 0 if succeed; -1 if error


// fork() and exec()
#include "unistd.h"
pid_t fork(void);
int execl(const char *pathname, const char *arg0, ... /* (char *) 0 */);
int execv(const char *pathname,  char *const *argv[]);
int execle(const char *pathname, const char *arg0, ... /* (char *) 0, char *const envp[] */ );
int execve(const char *pathname, char *const *argv[], char *const envp[]);
int execlp(const char *filename, const char *arg0, ... /* (char *) 0 */ );
int execvp(const char *filename, char *const argv[]);
    // for all exec(), return -1 if error
```

>**co-Functions**

``` c
// byte-ordering
#include "netinet/in.h"
uint16_t htons(uint16_t host16bitvalue);
uint32_t htonl(uint32_t host32bitvalue);
uint16_t ntohs(uint16_t net16bitvalue);
uint32_t ntohl(uint32_t net32bitvalue);

// address transform
#include "arpa/inet.h"
int inet_aton(const char *strptr, struct in_addr *addrptr); // return 1 if valid; non-0 if unvalid
in_addr_t inet_addr(const char *strptr); // return IPv4 addr if valid; INADDR_NONE if unvalid
char *inet_ntoa(struct in_addr inaddr); // return a pointer which points to a decimal-dot string
int inet_pton(int family, const char *strptr, void *addrptr); 
    // return 1 if succeed; 0 if unvalid expression; -1 if error
const char *inet_ntop(int family, const void *addrptr, char *strptr, size_t len);
    // return pointer if succeed; NULL if error
```
``` c
// byte-operating
#include "string.h"
void bzero(void *dest, size_t nbytes);
void bcopy(const void *src, void *dest, size_t nbytes);
int bcmp(const void *ptrl, const void *ptr2, size_t nbytes); // return 0 if equal; non-0 if not equal
```
