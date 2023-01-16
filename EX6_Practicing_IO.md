# EX6 : practicing I/O

[![hackmd-github-sync-badge](https://hackmd.io/jSFo9hJiRdi4uRLi7xKmBA/badge)](https://hackmd.io/jSFo9hJiRdi4uRLi7xKmBA)

###### tags: `Note`


[題目出處:Klim的網站](http://erdos.csie.ncnu.edu.tw/~klim/unix-p/usp-1072.html)

紀錄一下大學時修Unix Programing這堂課的程式碼

## 題目
Write a program which is invoked as
./a.out str-a str-b [f1 f2 ....]
The program will read input from stdin if no files are provided or will read input from the specified files. For each line read, each occurrence of str-a will be replaced by str-b. The processed line is written to stdout. For example,
```bash
$ cat k1
is oak still OK? 
or oak is still oak?
$ gcc ex6-substr.c 
$ ./a.out oak klim < k1             #  no file are provided
is klim still OK? 
or klim is still klim?
$ ./a.out oak klim k1 k1 k1         #  3 files to handle
is klim still OK? 
or klim is still klim?
is klim still OK? 
or klim is still klim?
is klim still OK? 
or klim is still klim?
$ ./a.out oak YG k1                 # test again with another string
is YG still OK? 
or YG is still YG?
$ 
```

## Answer
```cpp
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include <fcntl.h>
#define BLKSIZE 1024

char* replaceWord(const char* s, const char* oldW, const char* newW) { 
    char* result; 
    int i, cnt = 0; 
    int newWlen = strlen(newW); 
    int oldWlen = strlen(oldW); 
  
    // Counting the number of times old word 
    // occur in the string 
    for (i = 0; s[i] != '\0'; i++) { 
        if (strstr(&s[i], oldW) == &s[i]) { 
            cnt++; 
  
            // Jumping to index after the old word. 
            i += oldWlen - 1; 
        } 
    } 
  
    // Making new string of enough length 
    result = (char*)malloc(i + cnt * (newWlen - oldWlen) + 1); 
  
    i = 0; 
    while (*s) { 
        // compare the substring with the result 
        if (strstr(s, oldW) == s) { 
            strcpy(&result[i], newW); 
            i += newWlen; 
            s += oldWlen; 
        } 
        else
            result[i++] = *s++; 
    } 
  
    result[i] = '\0'; 
    return result; 
} 

void main(int argc, char *argv[]){
    // process stdin
    char *buff;
    if(argc<4) { 
        buff = (char *)malloc(1024 * sizeof(char));
        char *out_str;
        ssize_t size = read(STDIN_FILENO, buff, BLKSIZE);
        out_str = replaceWord(buff, argv[1], argv[2]);
        write(STDOUT_FILENO, out_str, strlen(out_str));

    }
    else{ // process argv
        for(int i=3; i<argc; i++){
            buff = (char *)malloc(1024 * sizeof(char));
            char *out_str;
            int fd = open(argv[i], O_RDONLY);
            ssize_t size = read(fd, buff, BLKSIZE);
            buff[size] = '\0';
            close(fd);
            out_str = replaceWord(buff, argv[1], argv[2]);
            write(STDOUT_FILENO, out_str, strlen(out_str));
        }
    }
}
```