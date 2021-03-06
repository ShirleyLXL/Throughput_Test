# 测ram盘和磁盘在不同文件粒度下的读写速度
## 一. 安装和挂载ram盘
### 1. 修改  “/usr/src/minix/drivers/storage/memory/memory.c”
```
...
#define RAMDISKS     7
...
```
### 2. 查看所有设备并根据设备号创建设备
```
ls –l /dev/
```
可重定向到文件（方便查看）
```
ls –l /dev/ > devices.txt
```
根据主设备号为1查找次设备号的最大值，这里是12，故设置新设备的次设备号为13
```
mknod /dev/myram b 1 13
```
## 二. 实现buildmyram初始化工具用于给myram分配容量
### 1. 复制ramdisk的目录，重命名buildmyram
```
cp ramdisk buildmyram
```
### 2. 修改buildmyram.c
```
#include <minix/paths.h>

#include <sys/ioc_memory.h>
#include <stdio.h>
#include <fcntl.h>
#include <stdlib.h>

int
main(int argc, char *argv[])
{
	int fd;
	signed long size;
	char *d;

	if(argc < 2 || argc > 3) {
		fprintf(stderr, "usage: %s <size in kB> [device]\n",
			argv[0]);
		return 1;
	}

	d = argc == 2 ? "/dev/myram" : argv[2];
	if((fd=open(d, O_RDONLY)) < 0) {
		perror(d);
		return 1;
	}

#define KFACTOR 1048576
	size = atol(argv[1])*KFACTOR;

	if(size < 0) {
		fprintf(stderr, "size should be non-negative.\n");
		return 1;
	}

	if(ioctl(fd, MIOCRAMSIZE, &size) < 0) {
		perror("MIOCRAMSIZE");
		return 1;
	}

	fprintf(stderr, "size on %s set to %ldmB\n", d, size/KFACTOR);

	return 0;
}


```
### 3. 运行buildmyram
```
buildmyram 400
```
## 三. 测试磁盘/ram盘和磁盘的吞吐量
### 1. 在ram盘上创建内存文件系统
```
mkfs.mfs /dev/myram/ 
mount /dev/myram/ /usr/mnt
```
查看目录挂载 df /usr/mnt
### 2. 测试函数
（a） 顺序/随机写
```C
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>
#include <string.h>
#include <sys/time.h>
#include <sys/wait.h>
#include <fcntl.h>

#define CONCURRENCY 10
#define BUFFER_SIZE 16384	
#define WRITE_SIZE 31457280

long long start_time;
long long end_time;

int main()
{
  int i, j, k;
  int fd;
	char buffer[BUFFER_SIZE]; 
	struct timeval start, end;
	memset(buffer, '&', BUFFER_SIZE);
  srand(time(NULL));
for(k=64; k<16384; k*=2){
  fd = open("writefile", O_WRONLY | O_APPEND | O_CREAT | O_DIRECT | O_SYNC);
  gettimeofday(&start,NULL);
  start_time = start.tv_sec*1000000 + start.tv_usec;
  
  for(i=0; i<CONCURRENCY; i++){
    if(fork()==0){
      for(j=0; j<WRITE_SIZE/k; j++){
        // lseek(fd, rand()%314572800, SEEK_SET);	//把注释去掉即是随机写
        write(fd, buffer, k);
      }
      exit(0);
    }
  }
  for(i=0; i<CONCURRENCY; i++){
      waitpid(-1, NULL, 0);
  }
  close(fd);
  gettimeofday(&end,NULL);
  end_time = end.tv_sec*1000000 + end.tv_usec;
  printf("sequence write:\n");
  printf("time: %f\n", (end_time - start_time)/1000000.0);
  printf("throughput: %f\n", (WRITE_SIZE*CONCURRENCY)/1024.0/1024.0/((end_time - start_time)/1000000.0));
}
	return 0;
}

```
顺序/随机读
```C
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>
#include <string.h>
#include <sys/time.h>
#include <sys/wait.h>
#include <fcntl.h>

#define CONCURRENCY 10
#define BUFFER_SIZE 16384
#define WRITE_SIZE 31457280

long long start_time;
long long end_time;

int main()
{
  int i, j, k;
  int fd;
	char buffer[BUFFER_SIZE]; 
	struct timeval start, end;

  srand(time(NULL));

  for(k=64; k<=16384; k*=2){
  // k = 16384;
    fd = open("writefile", O_RDONLY | O_SYNC | O_DIRECT);
    gettimeofday(&start,NULL);
    start_time = start.tv_sec*1000000 + start.tv_usec;
    for(i=0; i<CONCURRENCY; i++){
      if(fork()==0){
        for(j=0; j<WRITE_SIZE/k; j++){
          // lseek(fd, rand()%31457280, SEEK_SET);
          read(fd, buffer, k);
        }
        exit(0);
      }
    }
    for(i=0; i<CONCURRENCY; i++){
        waitpid(-1, NULL, 0);
    }
    close(fd);
    gettimeofday(&end,NULL);
    end_time = end.tv_sec*1000000 + end.tv_usec;
    printf("sequence read:\n");
    printf("time: %f\n", (end_time - start_time)/1000000.0);
    printf("throughput: %f\n", (WRITE_SIZE*CONCURRENCY)/1024.0/1024.0/((end_time - start_time)/1000000.0));
}
	return 0;
}

```
## 结果
|    	|    64b    	|    128b   	|    256b   	|    512b    	| 1kb        	| 2kb        	| 4kb        	| 8kb        	| 16kb       	|
|:-----------:	|:---------:	|:---------:	|:---------:	|:----------:	|------------	|------------	|------------	|------------	|------------	|
|  磁盘顺序写 	|  10.27984 	| 15.584416 	| 23.622047 	|  28.481011 	| 34.285714  	| 37.57829   	| 42.553191  	| 41.37931   	| 42.452834  	|
|  磁盘随机写 	|  6.147541 	| 11.635424 	| 18.018018 	|  25.210084 	| 31.971582  	| 33.333333  	| 40.089089  	| 41.09589   	| 40.540541  	|
|  磁盘顺序读 	| 12.931034 	| 24.965327 	| 47.619048 	|  88.235294 	| 155.17236  	| 250        	| 375        	| 486.487012 	| 545.454545 	|
| 磁盘随机读  	| 8.097166  	| 14.61039  	| 28.753992 	| 54.878055  	| 101.694915 	| 174.757248 	| 281.249912 	| 409.091095 	| 498.411453 	|
| ram盘顺序写 	| 11.635424 	| 24.423339 	| 49.861501 	| 90.452252  	| 152.542425 	| 243.243309 	| 352.941176 	| 400        	| 450        	|
| ram盘随机写 	| 12.295082 	| 22.988506 	| 44.334973 	| 90.452252  	| 162.162162 	| 272.727273 	| 418.605041 	| 529.411453 	| 500        	|
| ram盘顺序读 	| 24.593164 	| 47.368424 	| 88.669933 	| 155.172441 	| 260.839565 	| 360.869565 	| 391.304178 	| 514.286008 	| 600        	|
