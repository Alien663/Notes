# Sigaction

第10個作業回到了Token Rign，不過這次是簡易版本，不像之前那麼複雜，基本上可以拿第8次作業的答案改一改，比較簡單就可以改出來了。

這次需求需要改成

1. 單圈pipe，把anti-clockwise拿掉
2. 輸入改成用signal，而非root process的STDIN
3. Token Ring的各個Process都會收到Signal，收到的Process要當此次傳資料的Root

其實仔細看題目沒有很複雜，並沒有要求處理同時多個Process都收到Signal的狀況，如果要把這部分解決，就得要多加一個Queue去解決，而這個Queue又得是所有Process可以共同存取的，這就會很頭痛了。

不過加Queue的想法也可以思考看看，在Linux中我應該會利用檔案來解決就是了。

照慣例先上題目

## 題目

exercise 10: practicing sigaction

Similar to exercise 8, but the ring is uni-directional. So just refer to program 7.1 for instruction.

The argv[1] specified the number of processes.

After the ring is setup, each process will wait for three signals. One is for sending fibonacci numbers, one is for printing my pid, and the last one is for `quit'. These operations are just the same as those in exercise 8 but there are two differences.

First, these operations are initiated from any process which receives a signal.
Second, there is only ring. So each process may receive messages for 'p' or 'q' or 'f' from the the same neighbour.
Due: 2019/May/22, 00:05

* For example:
```bash
    $ ./a.out 5
    I am 18149
    I am 18150
    I am 18151
    I am 18152                          another window
    I am 18153                   +-----------------------------------
    I am 18152, I got   0   1    |   $ kill -USR1 18151
    I am 18153, I got   1   1    |
    I am 18149, I got   1   2    |
    I am 18150, I got   2   3    |
    I am 18151, I got   3   5    |
    I am 18150                   |   $ kill -USR2 18149
    I am 18151                   |
    I am 18152                   |
    I am 18153                   |
    I am 18149                   |
    I am 18149, I got   0   1    |   $ kill -USR1 18150; kill -USR1 18153
    I am 18151, I got   0   1    |
    I am 18152, I got   1   1    |
    I am 18153, I got   1   2    |
    I am 18149, I got   2   3    |
    I am 18150, I got   1   1    |
    I am 18150, I got   3   5    |
    I am 18151, I got   1   2    |
    I am 18152, I got   2   3    |
    I am 18153, I got   3   5    |
    $                            |   $ kill -INT 18152
```

## Step 1 : Remove anti-clockwise and add signal handler

**移除anti-clockwise的部分就不講解了**

首先要了解，Signal是個OS層級的非同步中斷，是由kernel送給Process的，本質上更接近硬體。

而Handler就類似這個Signal的Try Catch，不過要注意兩者差異，是OS丟出中斷還是程式自己丟出中斷(當然差異不只如此)。

我們在這一步驟要做的，是為我們ex8的程式碼加上signal handler，並保留變數讓Process知道哪個Signal被觸發了

```c
volatile sig_atomic_t flag_usr1 = 0;
volatile sig_atomic_t flag_usr2 = 0;
volatile sig_atomic_t flag_quit = 0;

void handler_usr1(int sig) {
    (void)sig;
    flag_usr1 = 1;
}

void handler_usr2(int sig) {
    (void)sig;
    flag_usr2 = 1;
}

void handler_quit(int sig) {
    (void)sig;
    flag_quit = 1;
}
```

宣告好handler function之後，還要將Signal和handler做綁定，每一個Process都要自己跟自己的handler function綁，比較好的位置就會是在Ring建立好，準備讀取pipe資料之前的wile loop外面。

```c
struct sigaction sa;
sigemptyset(&sa.sa_mask);
sa.sa_flags = 0;

sa.sa_handler = handler_usr1;
if (sigaction(SIGUSR1, &sa, NULL) == -1) {
    perror("sigaction(SIGUSR1)");
    return 1;
}

sa.sa_handler = handler_usr2;
if (sigaction(SIGUSR2, &sa, NULL) == -1) {
    perror("sigaction(SIGUSR2)");
    return 1;
}

sa.sa_handler = handler_quit;
if (sigaction(SIGINT, &sa, NULL) == -1) {
    perror("sigaction(SIGINT)");
    return 1;
}
```

而原本的while loop一開始我們是root process從STDIN讀取資料，這邊要改成判斷有沒有收到Signal訊息

```c
if(flag_usr1){
    // start to do fibonacci, same as ex8 get 'f'
}

if(flag_usr2){
    // say my name, same as ex8 get 'p'
}

if(flag_quit){
    // shutdown process, same as ex8 get 'q'
}
```

## Step 2 : Read data from pipe

第二步驟就簡單了，根據題目要求來寫就好。

要記得根據是否為收到Signal的行為會有差異，這邊可以簡單利用之前的is_root概念來用，讓收到Signal的Process為root，做完事情後解除root身分。

```c
if (flag_usr1) {
    is_root = 1;                           // get signal, become root
    flag_usr1 = 0;                         // usr1 flag on
    int a = 0;
    int b = 1;
    write(cw_out, &a, sizeof(a));
    write(cw_out, &b, sizeof(b));
}

if (flag_usr2) {
    is_root = 1;
    flag_usr2 = 0;
    int token = -2;                        // use -2 to report number
    write(cw_out, &token, sizeof(token));
}

if (flag_quit) {
    is_root = 1;
    flag_quit = 0;
    int token = -1;                        // use -1 to quit process
    write(cw_out, &token, sizeof(token));
}
```

緊接著還有題目要求的邏輯，根據pipe來的資料不同做不同事情

```c
if (read(cw_in, &a, sizeof(a)) > 0) {
    if (a >= 0) { // case 1 : report fabonacci
        read(cw_in, &b, sizeof(b));
        printf("I am P%d %ld, got : %d %d\n", i, (long)getpid(), a, b);
        int sum = a + b;
        if (is_root) is_root = 0; // root don't write to next, need to stop the report
        else {
            write(cw_out, &b, sizeof(b));
            write(cw_out, &sum, sizeof(sum));
        }
    } else if (a == -2) { // case 2 : report number
        printf("I am P%d %ld\n", i, (long)getpid());
        if (is_root) is_root = 0; 
        else {
            write(cw_out, &a, sizeof(a));
        }
    } else { // quit process
        if (is_root) is_root = 0; 
        else {
            write(cw_out, &a, sizeof(a));
        }
        running = 0;
    }
}
```

## Answer

```c
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <signal.h>
#include <sys/types.h>
#include <sys/select.h>

volatile sig_atomic_t flag_usr1 = 0;
volatile sig_atomic_t flag_usr2 = 0;
volatile sig_atomic_t flag_quit = 0;

void handler_usr1(int sig) {
    (void)sig;
    flag_usr1 = 1;
}

void handler_usr2(int sig) {
    (void)sig;
    flag_usr2 = 1;
}

void handler_quit(int sig) {
    (void)sig;
    flag_quit = 1;
}

int main(int argc, char *argv[]) {
    pid_t childpid;
    int cw[2];
    int cw_in, cw_out;
    int i;
    int nprocs;

    if ((argc != 2) || ((nprocs = atoi(argv[1])) <= 0)) {
        fprintf(stderr, "Usage: %s nprocs\n", argv[0]);
        return 1;
    }

    if (pipe(cw) == -1) {
        perror("Failed to create starting pipe");
        return 1;
    }

    cw_in = cw[0];
    cw_out = cw[1];

    for (i = 1; i < nprocs; i++) {
        if (pipe(cw) == -1) {
            fprintf(stderr, "[%ld]:failed to create pipe %d: %s\n", (long)getpid(), i, strerror(errno));
            return 1;
        }
        if ((childpid = fork()) == -1) {
            fprintf(stderr, "[%ld]:failed to create child %d: %s\n", (long)getpid(), i, strerror(errno));
            return 1;
        }
        if (childpid > 0) {
            cw_out = cw[1];
            close(cw[0]);
        } else {
            cw_in = cw[0];
            close(cw[1]);
        }
        if (childpid) {
            break;
        }
    }

    struct sigaction sa;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;

    sa.sa_handler = handler_usr1;
    if (sigaction(SIGUSR1, &sa, NULL) == -1) {
        perror("sigaction(SIGUSR1)");
        return 1;
    }

    sa.sa_handler = handler_usr2;
    if (sigaction(SIGUSR2, &sa, NULL) == -1) {
        perror("sigaction(SIGUSR2)");
        return 1;
    }

    sa.sa_handler = handler_quit;
    if (sigaction(SIGINT, &sa, NULL) == -1) {
        perror("sigaction(SIGINT)");
        return 1;
    }

    printf("I am P%d : %ld\n", i, (long)getpid());
    fd_set readfds;
    int running = 1;
    int is_root = 0;

    while (running) {
        if (flag_usr1) {
            is_root = 1;
            flag_usr1 = 0;
            int a = 0;
            int b = 1;
            write(cw_out, &a, sizeof(a));
            write(cw_out, &b, sizeof(b));
        }

        if (flag_usr2) {
            is_root = 1;
            flag_usr2 = 0;
            int token = -2;
            write(cw_out, &token, sizeof(token));
        }

        if (flag_quit) {
            is_root = 1;
            flag_quit = 0;
            int token = -1;
            write(cw_out, &token, sizeof(token));
        }

        FD_ZERO(&readfds);
        FD_SET(cw_in, &readfds);

        int ret = select(cw_in + 1, &readfds, NULL, NULL, NULL);
        if (ret < 0) {
            if (errno == EINTR) {
                continue;
            }
            perror("Failed to create select()");
            break;
        }

        if (FD_ISSET(cw_in, &readfds)) {
            int a, b;
            if (read(cw_in, &a, sizeof(a)) > 0) {
                if (a >= 0) {
                    read(cw_in, &b, sizeof(b));
                    printf("I am P%d %ld, got : %d %d\n", i, (long)getpid(), a, b);
                    int sum = a + b;
                    if (is_root) is_root = 0; 
                    else {
                        write(cw_out, &b, sizeof(b));
                        write(cw_out, &sum, sizeof(sum));
                    }
                } else if (a == -2) {
                    printf("I am P%d %ld\n", i, (long)getpid());
                    if (is_root) is_root = 0; 
                    else {
                        write(cw_out, &a, sizeof(a));
                    }
                } else {
                    if (is_root) is_root = 0; 
                    else {
                        write(cw_out, &a, sizeof(a));
                    }
                    running = 0;
                }
            }
        }
    }

    if ((close(cw_in) == -1) || (close(cw_out) == -1)) {
        perror("Failed to close final descriptors");
        return 1;
    }

    return 0;
}

```