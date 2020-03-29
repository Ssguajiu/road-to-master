---
title: epoll-server-test-code
tags: source-code epoll
---
```cpp
//
// Created by sguajiu on 2020/3/29.
//

#include <sys/epoll.h>
#include <cstdlib>
#include <cstdio>
#include <fcntl.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <unistd.h>
int main() {
#define MAX_EVENTS 10
    struct epoll_event ev, events[MAX_EVENTS];
    int listen_sock, conn_sock, nfds, epollfd;

/* Code to set up listening socket, 'listen_sock',
   (socket(), bind(), listen()) omitted */


    listen_sock = socket(AF_INET,SOCK_STREAM,0);
    sockaddr_in saddr;
    saddr.sin_addr.s_addr = INADDR_ANY;
    saddr.sin_port = htons(12000);
    saddr.sin_family = AF_INET;
    socklen_t t = sizeof(saddr);
    if(bind(listen_sock,(sockaddr*)&saddr, t) == -1){
        perror("bind error");
        exit(-1);
    }
    listen(listen_sock,5);
    epollfd = epoll_create1(0);
    if (epollfd == -1) {
        perror("epoll_create1");
        exit(EXIT_FAILURE);
    }

    ev.events = EPOLLIN;
    ev.data.fd = listen_sock;
    if (epoll_ctl(epollfd, EPOLL_CTL_ADD, listen_sock, &ev) == -1) {
        perror("epoll_ctl: listen_sock");
        exit(EXIT_FAILURE);
    }

    for (;;) {
        nfds = epoll_wait(epollfd, events, MAX_EVENTS, -1);
        if (nfds == -1) {
            perror("epoll_wait");
            exit(EXIT_FAILURE);
        }

        for (int n = 0; n < nfds; ++n) {
            if (events[n].data.fd == listen_sock) {
                sockaddr_in addr;
                socklen_t len = sizeof(sockaddr);
                conn_sock = accept(listen_sock,
                                   (struct sockaddr *) &addr, &len);
                if (conn_sock == -1) {
                    perror("accept");
                    exit(EXIT_FAILURE);
                }
                int flags = fcntl(conn_sock, F_GETFL, 0);
                fcntl(conn_sock, F_SETFL, flags | O_NONBLOCK);
                ev.events = EPOLLIN ; //| EPOLLET;
                ev.data.fd = conn_sock;
                if (epoll_ctl(epollfd, EPOLL_CTL_ADD, conn_sock,
                              &ev) == -1) {
                    perror("epoll_ctl: conn_sock");
                    exit(EXIT_FAILURE);
                }
            } else {
                printf("data coming, ");
                char data[10] = {0};
                read(events[n].data.fd,data,5);
                printf("take a little bite... data : %s\n ",data);
            }
        }
    }
}
```
