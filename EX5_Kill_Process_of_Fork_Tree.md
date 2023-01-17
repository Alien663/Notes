# EX5 : Kill Process of Fork Tree

[![hackmd-github-sync-badge](https://hackmd.io/7t5iuKYoQ8OC2eeC_TOrjQ/badge)](https://hackmd.io/7t5iuKYoQ8OC2eeC_TOrjQ)


###### tags: `Note`

[題目出處:Klim的網站](http://erdos.csie.ncnu.edu.tw/~klim/unix-p/usp-1072.html)

紀錄一下大學時修Unix Programing這堂課的程式碼

## 題目

Refer to the description of exercise 2 (usp-982) for explanation of the code .
Remove the wait(NULL) in the loop. After the line of printing a message to stderr, you should insert code like:
    
    If the process is a leaf,
            while(1) pause();

    else (the process is an internal node)
            while there are children alive do 
              waiting..
            exit(...);   
      
The internal node should wait for all children, and for each child, when waited, reports the signal number if it was terminated by a signal, or the value returned if it exited though exit(). In either case, a number is gained from the child. Those numbers from children are summed and the sum is used as the argument of exit(). This is to say the sum is the return value of this process.
Refer to example 3.22 for the usage of wait().
To test your program, you have to open another window through which you can kill a process with the command `kill`.
For example, to kill a process which pid is 12011 with signal 15, you can type 'kill -15 12011'.

* For example:

      $ ./a.out ddduuduu abcd                      a
      I'm c, my pid=3590, and my ppid=3589        / \
      I'm b, my pid=3589, and my ppid=3588       b   d
      I'm d, my pid=3591, and my ppid=3588       |
      I'm a, my pid=3588, and my ppid=3295       c
      Child 3590 is terminated with signal 3.       $ kill -3 3590
      Child 3589 exits with value 3.
      Child 3591 is terminated with signal 2.       $ kill -2 3591
      $ echo $?                            # Check the return value of root.
      5                                    # It is the sum of 2 and 3.
      $ ./a.out ddduuduu abcd              # Run it agagin.
      I'm c, my pid=3594, and my ppid=3593 # This time we try to kill the 
      I'm b, my pid=3593, and my ppid=3592 # internal node. The leaf node will
      I'm d, my pid=3595, and my ppid=3592 # be an orphan and be adopted by init.
      I'm a, my pid=3592, and my ppid=3295 
      Child 3593 is terminated with signal 4.       $ kill -4 3593
      Child 3595 is terminated with signal 15.      $ kill -15 3595
      $ echo $?
      19
      $ ps -ef | grep a.out                # Find the orphan. Check its ppid.
      klim      3594     1  0 13:39 pts/0    00:00:00 ./a.out ddduuduu abcd
      klim      3597  3295  0 13:40 pts/0    00:00:00 grep a.out
      $ 


## Sample Code(USP-982)
```cpp=
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/wait.h>

int get_treesize(char *s) {
    int i, level;

    for(level=0, i=0; s[i]; i++){
        if(s[i] == 'd') level += 1;
        if(s[i] == 'u') level -= 1;
        if(level == 0) return i+1;
    }

    return i;
}

void main(int argc, char *argv[]){
    char label, *labels, *t_begin, *t_end;
    int  j;

    t_begin = argv[1]; 
    t_end   = t_begin + strlen(t_begin) - 1;
    labels  = argv[2];

newborn:
    label = *labels;  labels += 1;

    t_begin = t_begin + 1;
    while(t_begin < t_end ) {
        j = get_treesize(t_begin);

        if( fork() == 0 ){
            t_end  = t_begin + j - 1;
            goto newborn;
        } else {
            wait(NULL);
            t_begin += j;
            labels += j/2;
        }
    }
  
    fprintf(stderr, "I'm %c, my pid=%d, and my ppid=%d\n", label, getpid(), getppid());
}
```

## 解題思路
這題sample code都給了，包含程式邏輯也都說明了，主要就是練習語法而已，重點內容如下 :

1. 搞清楚process exit signal規則
2. 如何取得child process的exit signal
3. unix雙視窗使用，kill process語法

## Answer
```cpp=
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/wait.h>

int get_treesize(char *s)
{
  int i, level;

  for(level=0, i=0; s[i]; i++){
    if(s[i] == 'd') level += 1;
    if(s[i] == 'u') level -= 1;
    if(level == 0) return i+1;
  }

  return i;
}

int main(int argc, char *argv[]) {
    char label, *labels, *t_begin, *t_end;
    int  j;
    pid_t pid;
    t_begin = argv[1]; 
    t_end   = t_begin + strlen(t_begin) - 1;
    labels  = argv[2];

newborn:
    label = *labels;  labels += 1;

    t_begin = t_begin + 1;
    while(t_begin < t_end ) {
        j = get_treesize(t_begin);
        pid = fork();
        if( pid == 0 ){
            t_end  = t_begin + j - 1;
            goto newborn;
        }
        else {
            t_begin += j;
            labels  += j/2;
        }
    }
    fprintf(stderr, "I'm %c, my pid=%d, and my ppid=%d\n", label, getpid(), getppid());

    int sum = 0;
    if(pid == 0){
        while(1) pause();
    }
    else{
        while(1){
            int status, temp_sig=0;
            pid_t my_children = wait(&status);
            if(my_children == -1) break;

            if(WIFEXITED(status)){ // node's life go to the end
                sum += WEXITSTATUS(status);
                printf("Child %d exits with value %d.\n", getpid(), sum);
            }
            else if(WIFSIGNALED(status)){ // leaf or node is killed with signal
                temp_sig = WTERMSIG(status);
                sum += temp_sig;
                printf("Child %d is terminated with signal : %d.\n", my_children, temp_sig);
            }
        }
    }
    return(sum);
}
```