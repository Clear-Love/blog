---
title: "Linux System  V IPC 进程间通信"
date: 2024-06-07T09:32:35+08:00
draft: false
hidden: false
series:
  - linux开发
categories:
  - 嵌入式linux
tags:
  - linux
  - 进程间通信
  - IPC
  - 共享内存
  - 消息队列
  - 信号量
featuredImage: https://pan.lmio.xyz/pic/39aa4431069272f1b8551e33dc2b3d30.png
toc: true
---

## IPC对象

![](https://pan.lmio.xyz/pic/39aa4431069272f1b8551e33dc2b3d30.png)

## 消息队列

相关API

```c
#include <sys/msg.h>
// 获取消息队列
int msgget(key_t key, int msgflg);
// 发送消息
int msgsnd(int msqid, const void *msgp, size_t msgsz, int msgflg);
// 接收消息
ssize_t msgrcv(int msqid, void *msgp, size_t msgsz, long msgtyp, int msgflg);
// 删除消息队列
int msgctl(int msqid, int cmd, struct msqid_ds *buf);

// 消息结构
struct msgbuf {
    long mtype;
    char mtext[1];
};
```

## 共享内存

相关API

```c
// 创建共享内存
int shmget(key_t key, size_t size, int shmflg);
// 共享内存映射
void *shmat(int shmid, const void *shmaddr, int shmflg);
// 断开共享内存
int shmdt(const void *shmaddr);
// 删除共享内存
int shmctl(int shmid, int cmd, struct shmid_ds *buf);
```

## 信号量

linux中信号量的定义
```c
#include <sys/sem.h>
 
struct semid_ds {
    struct ipc_perm sem_perm;    //指向与信号量集相对应的ipc_perm结构的指针
    struct sem *sem_base;        //指向这个集合中第一个信号量的指针
    ushort sem_nsems;            //集合中信号量的数量
    time_t sem_otime;            //最近一次调用semop函数的时间
    time_t sem_ctime;            //最近一次改变的时间
};

struct sem {
    ushort semval;    //信号量的值
    pid_t sempid;     //最近一次执行操作的进程的进程号
    ushort semncnt;   //等待信号值增长，即等待可利用资源出现的进程数
    ushort semzcnt;   //等待信号值减少，即等待全部资源可被独占的进程数
};
```


```c
#include <sys/sem.h>
#include <sys/ipc.h>
#include <sys/types.h>

struct sembuf {
    unsigned short sem_num;
    short sem_op;
    short sem_flg;
};

int semget(key_t key, int nsems, int semflg);
int semop(int semid, struct sembuf *sops, unsigned nsops);
int semctl(int semid, int semmum, int cmd, ...);
```

|参数值|说明|
|---|---|
|SETVAL|设置单个信号量的值|
|GETALL|返回信号量集中所有信号量的值|
|SETALL|设置信号量集中所有信号量的值|
|IPC_STAT|放置与信号量集相连的semid_ds结构当前值于arg.buf指定的缓冲区|
|IPC_SET|用arg.buf指定结构值替代与信号量集合相连的semid_ds结构的值|
|GETVAL|返回单个信号量的值|
|GETPID|返回最后一个操作该信号量集的进程ID|
|GETNCNT|返回semncnt的值|
|GETZCNT|返回semzcnt的值|
|IPC_RMID|删除指定的信号量集|
执行IPC_SET、IPC_RMID命令的进程只能是信号量集的创建进程、拥有者进程或特权进程，执行其他命令的进程必须拥有信号量集的读取或更新权限。

根据实际cmd参数的具体内容，semctl可能拥有第4个参数arg，这是一个共用体/联合体，其中的val用于semctl中cmd值为SETVAL时，指明要设置的信号量值；buf用于IPC_STAT/IPC_SET，表示存放信号量集合数据结构的缓冲区；array用于GETALL/SETALL，存放所获得的或要设置的信号量集中所有信号量的值。结构定义如下：
```c
union semum {
    int val;
    struct semid_ds *buf;
    unsigned short array;
};
```

## 使用信号量，共享内存和循环队列实现一个生产者消费者模型

循环队列

```c
// deque.h
#ifndef __DEQUE_H__
#define __DEQUE_H__

#include <stdbool.h>
#include <stdlib.h>

typedef struct {
    int head;
    int tail;
    int size;
} deque;

static inline deque* deque_init(size_t size){
    deque *q = malloc(sizeof(deque));
    q->head = 0;
    q->tail = 0;
    q->size = size;
    return q;
}

static int deque_popleft(deque *q){
    int x = q->head;
    q->head = (q->head + 1) % q->size;
    return x;
}

static int deque_push(deque *q) {
    int ret = q->tail;
    q->tail = (q->tail+1) % q->size;
    return ret;
}

static inline int deque_destory(deque *q){
    free(q);
}

#endif // __DEQUE_H__
```
生产者消费者模型

```c
#include <semaphore.h>
#include <sys/shm.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <stdio.h>
#include <pthread.h>
#include <unistd.h>
#include "deque.h"

#define BUFSIZE 1024
#define BUFCNT 10

sem_t* sem_r;
sem_t* sem_w;
pthread_mutex_t w_mutex;
pthread_mutex_t r_mutex;

int cnt = 0;

int shmid;
char (*shmaddr)[BUFSIZE];

deque* q;

void shm_read() {
    pthread_mutex_lock(&r_mutex);
    int ret = deque_popleft(q);
    printf("tid: %ld, read: %s\n", pthread_self(), shmaddr[ret]);
    pthread_mutex_unlock(&r_mutex);
}

void shm_write() {
    pthread_mutex_lock(&w_mutex);
    int ret = deque_push(q);
    sprintf(shmaddr[ret], "%d", cnt);
    printf("%ld write: %d\n", pthread_self(), cnt);
    cnt++;
    pthread_mutex_unlock(&w_mutex);
    
}

void* writer(void* arg) {
    while (1) {
        // 等待写空间
        sem_wait(sem_w);
        shm_write();
        // 读信号量+1
        sem_post(sem_r);
        usleep(200000);
    }
}

void* reader(void* arg) {
    while (1) {
        // 等待读空间
        sem_wait(sem_r);
        shm_read();
        // 写信号量+1
        sem_post(sem_w);
        sleep(1);
    }
}

int main(int argc, char const *argv[]) {
    q = deque_init(BUFCNT);
    pthread_mutex_init(&w_mutex, NULL);
    pthread_mutex_init(&r_mutex, NULL);

    // 创建一个读信号量，不存在则创建，存在就清空内容
    sem_r = sem_open("sem_r", O_CREAT, 0666, 0);
    sem_init(sem_r, 0, 0);
    if (sem_r == SEM_FAILED) {
        perror("sem_r error");
    }
    
    // 创建一个写信号量, 循环队列实际容量减一
    sem_w = sem_open("sem_w", O_CREAT, 0666, BUFCNT);
    sem_init(sem_w, 0, BUFCNT);
    if (sem_w == SEM_FAILED) {
        perror("sem_w error");
    }

    //1 申请key值
    key_t key = ftok(".", 100);
    if(key == -1)
    {
        perror("ftok");
        return -1;
    }
    //2 根据key获取共享内存ID号， 如果该共享内存不存在，则创建
    shmid = shmget(key, BUFSIZE*BUFCNT, IPC_CREAT | 0666);
    if(shmid == -1) {
        perror("shmget");
        return -1;
    }
    //3 将共享内存 映射到用户的虚拟内存空间中
    shmaddr = shmat(shmid, NULL, 0);
    
    if(shmaddr == (void*)-1) {
        perror("shmat");
        return -1;
    }

    // 创建一个线程用来写
    pthread_t tidw_1;
    pthread_create(&tidw_1, NULL, writer, NULL);

    pthread_t tidw_2;
    pthread_create(&tidw_2, NULL, writer, NULL);

    // 创建一个线程用来读
    pthread_t tidr_1;
    pthread_create(&tidr_1, NULL, reader, NULL);

    pthread_t tidr_2;
    pthread_create(&tidr_2, NULL, reader, NULL);

    pthread_t tidr_3;
    pthread_create(&tidr_3, NULL, reader, NULL);

    pthread_join(tidw_1, NULL);
    pthread_join(tidw_2, NULL);
    pthread_join(tidr_1, NULL);
    pthread_join(tidr_2, NULL);
    pthread_join(tidr_3, NULL);

    // 4 解除映射
    shmdt(shmaddr);
    // 5 删除共享内存
    shmctl(shmid, IPC_RMID, NULL);
    // 6 删除信号量
    sem_unlink("sem_r");
    sem_unlink("sem_w");
    deque_destroy(q);
    return 0;
}

```