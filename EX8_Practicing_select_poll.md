# Practicing select() and poll()

這是第一次Klim老師在我們寫作業前給了Sample Code，這代表大事要不妙了，這個作業肯定很難...

事實上也真是如此，首先要了解整個Token Ring是怎樣建立關聯的，這份範例程式碼就已經很難懂了。本來比較簡單建立Token Ring的方式是預先建立好二維陣列，是先建立好`n*2`的pipe放在那邊準備使用。但這份程式碼可不走這浪費資源的道路，能省則省的選擇一邊fork一邊搞定pipe連線，但不得不說是真的很巧妙，但也真的難。

你以為看懂Token Ring就結束了?不，後面步驟還多著。因為這份作業要可以做到順時針和逆時針兩種方式，所以第二關是要讓Token Ring在原本的順時針pipe之外，外掛一個逆時針的pipe。關於這一步晚點筆記會提到。

終於在搞懂整個Token Ring的pipe怎樣建立後，終於要傳資料了吧，但這時就會遇到大問題。到底怎樣讓資料可以順序的傳遞呢?這時候就會需要用上`select()`了，關於這個函數我理解也不深，單純就是知道在這題目中怎樣使用，後面盡可能描述清楚。

## 題目

exercise 8: practicing select() or poll()
Generate a bidirectional ring. Refer to program 7.1. The argv[1] specified the number of processes.

After the ring is setup, the root process then does

        loop
            read from stdin 
            if got 'f' then 
              send 0 1 to the clockwise next process
              read two numbers from clockwise previous process
              write them out
            if got 'p' then 
              write a token to the anti-clockwise next process
              read a token from anti-clockwise previous process
              show pid
        end loop

---

    Each child does
        loop
           monitor two inputs simultaneously 
           (one clockwise and another anti-clockwise)
           if clockwise input is ready then 
             read two numbers, a and b, from clockwise previous process
             show them
             write b (a+b) to clockwise next process
           if anti-clockwise input is ready then
             read a token from anti-clockwise previous process
             show pid
        end loop
---
* For example:

      $ ./a.out 8
      p
      I'am  598
      I'am  596
      I'am  594
      I'am  593
      I'am  591
      I'am  590
      I'am  589
      I'am  588, the root
      f
      I am  589, I got   0   1
      I am  590, I got   1   1
      I am  591, I got   1   2
      I am  593, I got   2   3
      I am  594, I got   3   5
      I am  596, I got   5   8
      I am  598, I got   8  13
      I am root. I got  13  21
      q
      $ ./a.out 3
      p
      I'am  607
      I'am  606
      I'am  600, the root
      f
      I am  606, I got   0   1
      I am  607, I got   1   1
      I am root. I got   1   2
      q
      $

## Token Ring Sample Code

```c
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int main(int argc,  char *argv[ ]) {
   pid_t childpid;             /* indicates process should spawn another     */
   int error;                  /* return value from dup2 call                */
   int fd[2];                  /* file descriptors returned by pipe          */
   int i;                      /* number of this process (starting with 1)   */
   int nprocs;                 /* total number of processes in ring          */ 
           /* check command line for a valid number of processes to generate */
   if ( (argc != 2) || ((nprocs = atoi (argv[1])) <= 0) ) {
       fprintf (stderr, "Usage: %s nprocs\n", argv[0]);
       return 1; 
   }  
   if (pipe (fd) == -1) {      /* connect std input to std output via a pipe */
      perror("Failed to create starting pipe");
      return 1;
   }
   if ((dup2(fd[0], STDIN_FILENO) == -1) ||
       (dup2(fd[1], STDOUT_FILENO) == -1)) {
      perror("Failed to connect pipe");
      return 1;
   }
   if ((close(fd[0]) == -1) || (close(fd[1]) == -1)) {
      perror("Failed to close extra descriptors");
      return 1; 
   }        
   for (i = 1; i < nprocs;  i++) {         /* create the remaining processes */
      if (pipe (fd) == -1) {
         fprintf(stderr, "[%ld]:failed to create pipe %d: %s\n",
                (long)getpid(), i, strerror(errno));
         return 1; 
      } 
      if ((childpid = fork()) == -1) {
         fprintf(stderr, "[%ld]:failed to create child %d: %s\n",
                 (long)getpid(), i, strerror(errno));
         return 1; 
      } 
      if (childpid > 0)               /* for parent process, reassign stdout */
          error = dup2(fd[1], STDOUT_FILENO);
      else                              /* for child process, reassign stdin */
          error = dup2(fd[0], STDIN_FILENO);
      if (error == -1) {
         fprintf(stderr, "[%ld]:failed to dup pipes for iteration %d: %s\n",
                 (long)getpid(), i, strerror(errno));
         return 1; 
      } 
      if ((close(fd[0]) == -1) || (close(fd[1]) == -1)) {
         fprintf(stderr, "[%ld]:failed to close extra descriptors %d: %s\n",
                (long)getpid(), i, strerror(errno));
         return 1; 
      } 
      if (childpid)
         break;
   }                                               /* say hello to the world */
   fprintf(stderr, "This is process %d with ID %ld and parent id %ld\n",
           i, (long)getpid(), (long)getppid());
   return 0; 
}     
```

直接看code很容易模糊，下面透過示意圖表示中間是怎麼變化的

**1. P1 create pipe and dup2 STDIN and STDOUT**
```c
if (pipe (fd) == -1) {      /* connect std input to std output via a pipe */
    perror("Failed to create starting pipe");
    return 1;
}
if ((dup2(fd[0], STDIN_FILENO) == -1) ||
    (dup2(fd[1], STDOUT_FILENO) == -1)) {
    perror("Failed to connect pipe");
    return 1;
}
if ((close(fd[0]) == -1) || (close(fd[1]) == -1)) {
    perror("Failed to close extra descriptors");
    return 1; 
}
/*
   O---|
  /   |-|
P1    |f|
  \   |d|
   I--|_|
*/    
```

**2. create new pipe and fork P2**
```c
for (i = 1; i < nprocs;  i++) {         /* create the remaining processes */
    if (pipe (fd) == -1) {
        fprintf(stderr, "[%ld]:failed to create pipe %d: %s\n",
            (long)getpid(), i, strerror(errno));
        return 1; 
    } 
    if ((childpid = fork()) == -1) {
        fprintf(stderr, "[%ld]:failed to create child %d: %s\n",
                (long)getpid(), i, strerror(errno));
        return 1; 
    } 
/*
   O---|----O       
  /   |-|    \     |-|
P1    |f|     P2   |f|
  \   |d|    /     |d|
   I--|_|---I      |_|
*/
```

**3. change P1.STDOUT and P2.STDIN to new pipe**
```c
if (childpid > 0)               /* for parent process, reassign stdout */
    error = dup2(fd[1], STDOUT_FILENO);
else                              /* for child process, reassign stdin */
    error = dup2(fd[0], STDIN_FILENO);
/*
    __________________ 
   |                  |
   O   |----O         |
  /   |-|    \       |-|
P1    |f|     P2     |f|
  \   |d|       \    |d|
   I--|_|        I---|_|

整理一下變成下面這樣:
          ____
     O---|_fd_|---I
    /              \
(P1)                (P2)
    \     ____     /
     I---|_fd_|---O

新的P3加進來，則重複2, 3步驟，但記得此時的Parent要變成P2，P3是由P2產生出來的:

2. 多一個pipe沒幹嘛，然後複製一個P3，STDIN和STDOUT都跟P2要一樣
          ____
     O---|_fd_|---I---------         _
    /              \        \       |f|
(P1)                (P2)     (P3)   |d|
    \     ____     /        /       |_|
     I---|_fd_|---O---------


3. 改Parent的STDOUT寫到新的pipe，Child的STDIN改從新的pipe讀
                          ______________
          ____           |              |  
     O---|_fd_|---I      O              |
    /              \    /              |f|
(P1)                (P2)     (P3)      |d|
    \     ____              /    \     |_|
     I---|_fd_|------------O     I------|



在整理一下圖:
          ____           
     O---|_fd_|---I---(P2)---O
    /                         \ ____
(P1)                           |_fd_|
    \     ____                /
     I---|_fd_|---O---(P3)---I 
*/
```

## Step 1: Add anti-clockwise pipe

在開始加入順逆時針的pipe之前，需要先了解幾件事情:

1. STDIN, STDOUT都是獨有的，兩種pipe存在的情況下無法直接使用
2. 順逆時針屬於不同pipe，要分別獨立建立與使用

因此我們要稍微改一下範例程式碼一開始的變數宣告:
```c
pid_t childpid;
int cw[2], acw[2];                  //順逆時針各自的pipe
int cw_in, cw_out, acw_in, acw_out; //新增4個變數取代STDIN, STDOUT，又因為順逆時針兩個方向，所以共有4個
int i;
int nprocs;
```

再來是for loop前的pipe建立，要注意這邊需要保留初始pipe的順時針IN給最後一個process串接，逆時針則是OUT
```c
pipe(cw);
pipe(acw);
cw_in = cw[0];
cw_out = cw[1];
acw_in = acw[0];
acw_out = acw[1];
```

for迴圈裡面要做的事情就比較簡單了，先是加上建立逆時針的pipe
```c
if (pipe(cw) == -1 || pipe(acw) == -1) {
    fprintf(stderr, "[%ld]:failed to create pipe %d: %s\n", (long)getpid(), i, strerror(errno));
    return 1; 
} 
```

最後是設定pipe通道，由於不是用STDIN/STDOUT，直接assign變數既可，要注意順逆時針的通道設定是相反的
```c
if (childpid > 0){    // parent process reassign clockwsie out and anti-clockwise in
    cw_out = cw[1];   // clockwise write to child
    acw_in = acw[0];  // anti-clockwise read from child
    close(cw[0]);     // clockwise close read from child
    close(acw[1]);    // anti-clockwise close write to child
}
else{                 // child process assign clockwise in and anti-clockwise out
    cw_in = cw[0];    // clockwise read from parent
    acw_out = acw[1]; // anti-clockwise write to parent
    close(cw[1]);     // clockwise close write to parent
    close(acw[0]);    // anti-clockwise close read from parent
}
```

至此，其實整個雙向的Token Ring就已經建立好了，但只是這樣沒有辦法知道是否已經建立好pipe，所以需要再跳出for loop後建立select去監聽整個Token Ring是否有按照順序來。

## Step 2: Add select()

題目要求的Token Ring需要循環往復，所以會需要一個while迴圈不停跑，然後再用select把整個流程擋住，直到監聽的fd有值才要繼續做下去。

整體設定需要做幾件事情:

1. 設定空間儲存監聽項目(有哪些fd)
2. 監聽必要項目: STDIN, cw_in, acw_in
3. 設定select等待監聽項目有異動
4. 看看是哪個監聽項目有異動，哪個有資料動哪個

由於這部分都是屬於設定為主，沒有太多複雜邏輯，直接上code

```c
fd_set readfds; // 宣告一個集合，等等監聽事件會被丟進來
int maxfd = (STDIN_FILENO > cw_in ? STDIN_FILENO : cw_in); // STDIN, cw_in, acw_in三個裡面找一個最大的，select要檢查readfds裡面的[0, maxfd]
maxfd = (maxfd > acw_in ? maxfd : acw_in) + 1;
int is_root = (i == 1);
while(1){
    FD_ZERO(&readfds);                     // reset readfds空間
    if(is_root)                            // root要多監聽STDIN的輸入
        FD_SET(STDIN_FILENO, &readfds);  
    FD_SET(cw_in, &readfds);               // 監聽cw_in pipe傳來的值
    FD_SET(acw_in, &readfds);              // 監聽acw_in pipe傳來的值

    if(select(maxfd, &readfds, NULL, NULL, NULL) < 0){ // 等，等readfds至少有個fd可以讀取
        fprintf(stderr, "[%ld]:failed to create select: %s\n", (long)getpid(), strerror(errno));
        break;
    }

    if(is_root && FD_ISSET(STDIN_FILENO, &readfds)){
        // 根據題目要求，從STDIN讀取輸入，並以此輸入決定要傳資料給cw_out或是acw_out開始Token Ring傳資料
    }

    if(D_ISSET(cw_in, &readfds)){
        // todo: clockwise
    }

    if(D_ISSET(acw_in, &readfds)){
        // todo: anti-clockwise
    }
}
```

然而現在只是可以正常跑，但還沒有完成最後收尾。

這種跟fork有關的程式每次要關閉都很麻煩，為了避免產生孤兒，要很計較關閉的順序。

所以習慣上這個Token Ring要從最末代開始關閉，自然就會走anti-clockwise的順序去關閉，當然你要是心大一點，也可以嘗試兩邊同時跑。

但除了fork的程式關閉順序要考慮之外，還有建立好的pipe要關閉。試想，要是P3傳資料告訴P2要關閉前就已經把自己給關了，就會導致P2僵在那邊，沒有child可以互傳資料，也沒辦法正常停止。

所以要手動控制好關閉的順序，我這邊設定成token為-1時會自己關閉。

```c
if (read(acw_in, &token, sizeof(token)) > 0) {
    if(token == -1){
        if(!is_root)
            write(acw_out, &token, sizeof(token)); // 傳遞結束信號
        running = 0;
    }
......

// close fd
close(cw_in);
close(cw_out);
close(acw_in);
close(acw_out);

return 0; 
```

## Final result

```c
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/select.h>

int main(int argc,  char *argv[ ]) {
   pid_t root_pid = getpid();
   pid_t childpid;
   int cw[2], acw[2];
   int cw_in, cw_out, acw_in, acw_out;
   int i;
   int nprocs;
   
   /* check command line for a valid number of processes to generate */
   if ( (argc != 2) || ((nprocs = atoi (argv[1])) <= 0) ) {
      fprintf (stderr, "Usage: %s nprocs\n", argv[0]);
      return 1; 
   }
   
   if (pipe(cw) == -1 || pipe(acw) == -1) {
      perror("Failed to create starting pipe");
      return 1;
   }

   cw_in = cw[0];
   cw_out = cw[1];
   acw_in = acw[0];
   acw_out = acw[1]; 

   /* create the remaining processes */
   for (i = 1; i < nprocs;  i++) {
      if (pipe(cw) == -1 || pipe(acw) == -1) {
         fprintf(stderr, "[%ld]:failed to create pipe %d: %s\n", (long)getpid(), i, strerror(errno));
         return 1; 
      } 
      if ((childpid = fork()) == -1) {
         fprintf(stderr, "[%ld]:failed to create child %d: %s\n", (long)getpid(), i, strerror(errno));
         return 1; 
      }
      if (childpid > 0){
         cw_out = cw[1];
         acw_in = acw[0];
         close(cw[0]);
         close(acw[1]);
      }
      else{
         cw_in = cw[0];
         acw_out = acw[1];
         close(cw[1]);
         close(acw[0]);
      }
      if (childpid)
         break;
   }

   fd_set readfds;
   int maxfd = (STDIN_FILENO > cw_in ? STDIN_FILENO : cw_in);
   maxfd = (maxfd > acw_in ? maxfd : acw_in) + 1;
   int running = 1;
   int is_root = (root_pid == getpid());
   while(running){
      FD_ZERO(&readfds);
      if(is_root) FD_SET(STDIN_FILENO, &readfds);
      FD_SET(cw_in, &readfds);
      FD_SET(acw_in, &readfds);

      int ret = select(maxfd, &readfds, NULL, NULL, NULL);
      if (ret < 0) {
         perror("select");
         break;
      }

      if( is_root && FD_ISSET(STDIN_FILENO, &readfds)) {
         char ch;
         int token = 1;
         if(read(STDIN_FILENO, &ch, 1) > 0) {
            if(ch == 'q') {
               token = -1; // 結束信號
               write(acw_out, &token, sizeof(token));
            }
            else if (ch == 'p')
            {
               write(acw_out, &token, sizeof(token));
            }
            else if (ch == 'f'){
               int a=0, b=1;      
               write(cw_out, &a, sizeof(a));
               write(cw_out, &b, sizeof(b));
            }
         }
      }

      if (FD_ISSET(cw_in, &readfds)) {
         // CW pipe 有資料
         int a, b;
         if (read(cw_in, &a, sizeof(a)) > 0 && read(cw_in, &b, sizeof(b)) > 0) {
            printf("I am P%d, %ld, got : %d %d\n", i, (long)getpid(), a, b);
            // 計算下一步
            if(!is_root){
               int sum = a + b;
               write(cw_out, &b, sizeof(b));
               write(cw_out, &sum, sizeof(sum));
            }
         }
      }

      if (FD_ISSET(acw_in, &readfds)) {
         // ACW pipe 有 token
         int token;
         if (read(acw_in, &token, sizeof(token)) > 0) {
            if(token == -1){ // when receive termination signal
               if(!is_root)
                  write(acw_out, &token, sizeof(token)); // 傳遞結束信號
               running = 0;
            }
            else{
               if(is_root)
                  printf("I'm P%d: %ld, the root\n", i,(long)getpid());
               else{
                  printf("I'm P%d: %ld\n", i, (long)getpid());
                  write(acw_out, &token, sizeof(token)); // 繼續傳
               }
            }
         }
      }
   }

   if ((close(cw_in) == -1) || (close(cw_out) == -1) || (close(acw_in) == -1) || (close(acw_out) == -1)) {
      perror("Failed to close final descriptors");
      return 1; 
   }   

   return 0; 
}     
```