# Sort binary file

第9題不算太難的題目，沒有複雜的邏輯沒有難懂的function，甚至測試資料都有具體的檔案可以參考，非常簡單。

不過要注意的是，由於系統在處理是使用little-endian，而教授故意使用big-endian的binary檔案，所以要特別轉成little-endian之後再排序，排完再轉成big-endian輸出。

而題目中限制使用的function也是為了要練習這件事，不然直接使用ntohl就可以轉換了。

## 題目

exercise 9: practicing binary read() and write()

Write a program to sort an integer array. 
For example, `$ ./sort-int array-a array-b`

The two arguments denote files which contains an array. The layout of a file is:

First 4 bytes stores the size. (An integer, denoted as n, stored in binary big-endian)

Next 4*n byte store the integer array. Each integer occupy 4 bytes and is stored in binary, big-endian.

You should invoke qsort() to sort integers.
Only the functions mentioned in textbook are allowed.

Due: 2019/May/15, 00:05

* For example:

```bash
    $ od -t x1 k1      # 6 numbers: 12 5 6 9 3 191
    0000000 00 00 00 06 00 00 00 0c 00 00 00 05 00 00 00 06
    0000020 00 00 00 09 00 00 00 03 00 00 00 bf
    0000034
    $ gcc ex9-sort-int.c 
    $ ./a.out k1 k2
    $ od -t x1 k2
    0000000 00 00 00 06 00 00 00 03 00 00 00 05 00 00 00 06
    0000020 00 00 00 09 00 00 00 0c 00 00 00 bf
    0000034
    $ 
```
For your convenience, here is the test file int-array inside which are 6 12 5 6 9 3 191. The first 6 is the size of array. Others are the contents of the array.

## Step 1 : Build Architecure

雖然前面提到了little-endian的東西，但實際上在做可以更簡單一點，寫兩個function去解決

1. bigEndianBytesToInt()
2. intToBigEndianBytes()

然後就只是單純的讀檔寫檔就好，核心邏輯可以寫成下面這樣，之後再補齊其他function就好。

```c
int main(int argc, char *argv[]){
    int n;
    int *ptr = bigEndianBytesToInt(&n, argv[1]);  // read binary file to int array
    qsort(ptr, n, sizeof(int), comp);             // sort array
    intToBigEndianBytes(n, ptr, argv[2]);         // write int array to binary file
    free(ptr);
}
```

## Step 2 : Programing

先處理Byte和Int互轉問題

```c
int bigEndianBytesToInt(unsigned char* b, unsigned length){
    int val = 0;
    unsigned i;
    for(i=0;i<length;++i){
        val = (val << 8) | (b[i] & 0xff);
    }
    return val;
}

void intToBigEndianBytes(int val, unsigned char* b, unsigned length){
    unsigned i;
    for(i=0;i<length;++i){
        b[length - 1 - i] = (unsigned char)(val & 0xff);
        val >>= 8;
    }
}
```

之後是讀/寫檔的function

```c
int *fromFile(int* n, char* filename){
    int fp, i;
    int *ptr = NULL;
    unsigned char buff[4];
    fp = open(filename, O_RDONLY);
    read(fp, buff, 4);
    *n = bigEndianBytesToInt(buff, 4);
    ptr = (int *)malloc(sizeof(int)*(*n));
    for(i=0;i<*n;i++){
        read(fp, buff, 4);
        ptr[i] = bigEndianBytesToInt(buff, 4);
    }
    close(fp);
    return ptr;
}

void toFile(int n, int *arr, char* Finename){
    int i, fp;
    unsigned char buff[4];
    fp = open(Finename, O_RDWR|O_CREAT, 0666);
    intToBigEndianBytes(n, buff, 4);
    write(fp, buff, 4);
    for(i=0;i<n;i++){
        intToBigEndianBytes(arr[i], buff, 4);
        write(fp, buff, 4);
    }
    close(fp);
}
```

順帶一提qsort需要自己寫個比較函數進去，這個就直接抄網路上別人的寫法就好了

```c
int comp(const void*a,const void*b)
{
    return *(int*)a-*(int*)b;
}
......
qsort(ptr, n, sizeof(int), comp);
```

再補上一些檢查用的程式碼，就完成了

## Answer

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdlib.h>

int bigEndianBytesToInt(unsigned char* b, unsigned length){
    int val = 0;
    unsigned i;
    for(i=0;i<length;++i){
        val = (val << 8) | (b[i] & 0xff);
    }
    return val;
}

void intToBigEndianBytes(int val, unsigned char* b, unsigned length){
    unsigned i;
    for(i=0;i<length;++i){
        b[length - 1 - i] = (unsigned char)(val & 0xff);
        val >>= 8;
    }
}

int comp(const void*a,const void*b)
{
    return *(int*)a-*(int*)b;
}

int *fromFile(int* n, char* filename){
    int fp, i;
    int *ptr = NULL;
    unsigned char buff[4];
    fp = open(filename, O_RDONLY);
    if(fp < 0) return NULL;
    if(read(fp, buff, 4) != 4) {
        close(fp);
        return NULL;
    }
    *n = bigEndianBytesToInt(buff, 4);
    ptr = (int *)malloc(sizeof(int)*(*n));
    for(i=0;i<*n;i++){
        if(read(fp, buff, 4) != 4) {
            free(ptr);
            close(fp);
            return NULL;
        }
        ptr[i] = bigEndianBytesToInt(buff, 4);
    }
    close(fp);
    return ptr;
}

void toFile(int n, int *arr, char* Finename){
    int i, fp;
    unsigned char buff[4];
    fp = open(Finename, O_RDWR|O_CREAT, 0666);
    intToBigEndianBytes(n, buff, 4);
    write(fp, buff, 4);
    for(i=0;i<n;i++){
        intToBigEndianBytes(arr[i], buff, 4);
        write(fp, buff, 4);
    }
    close(fp);
}

int main(int argc, char *argv[]){
    if(argc != 3){
        fprintf(stderr, "Usage: %s <input_file> <output_file>\n", argv[0]);
        return 1;
    }
    int n;
    int *ptr = fromFile(&n, argv[1]);
    if(ptr == NULL){
        fprintf(stderr, "Error reading input file\n");
        return 1;
    }
    qsort(ptr, n, sizeof(int), comp);
    toFile(n, ptr, argv[2]);
    free(ptr);
    return 0;
}
```