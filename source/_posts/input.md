---
title: Writeup.pwnable.kr系列之input
date: 2018-03-06 20:36:11
categories: ctf
tags:
	- pwn
	- pwnable.kr
---
pwnable.kr是一个pwn学习站点，本系列文章为上面题目的writeup。

<!--more-->
# Writeup.pwnable.kr系列之input

## 审题

题目描述：

![](/images/ctf/pwn/pwnable.kr/input_01.jpg)

看起来又是和input相关，连上去，`cat input.c`看看：

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>

int main(int argc, char* argv[], char* envp[]){
	printf("Welcome to pwnable.kr\n");
	printf("Let's see if you know how to give input to program\n");
	printf("Just give me correct inputs then you will get the flag :)\n");

	// argv
	if(argc != 100) return 0;
	if(strcmp(argv['A'],"\x00")) return 0;
	if(strcmp(argv['B'],"\x20\x0a\x0d")) return 0;
	printf("Stage 1 clear!\n");	

	// stdio
	char buf[4];
	read(0, buf, 4);
	if(memcmp(buf, "\x00\x0a\x00\xff", 4)) return 0;
	read(2, buf, 4);
        if(memcmp(buf, "\x00\x0a\x02\xff", 4)) return 0;
	printf("Stage 2 clear!\n");
	
	// env
	if(strcmp("\xca\xfe\xba\xbe", getenv("\xde\xad\xbe\xef"))) return 0;
	printf("Stage 3 clear!\n");

	// file
	FILE* fp = fopen("\x0a", "r");
	if(!fp) return 0;
	if( fread(buf, 4, 1, fp)!=1 ) return 0;
	if( memcmp(buf, "\x00\x00\x00\x00", 4) ) return 0;
	fclose(fp);
	printf("Stage 4 clear!\n");	

	// network
	int sd, cd;
	struct sockaddr_in saddr, caddr;
	sd = socket(AF_INET, SOCK_STREAM, 0);
	if(sd == -1){
		printf("socket error, tell admin\n");
		return 0;
	}
	saddr.sin_family = AF_INET;
	saddr.sin_addr.s_addr = INADDR_ANY;
	saddr.sin_port = htons( atoi(argv['C']) );
	if(bind(sd, (struct sockaddr*)&saddr, sizeof(saddr)) < 0){
		printf("bind error, use another port\n");
    		return 1;
	}
	listen(sd, 1);
	int c = sizeof(struct sockaddr_in);
	cd = accept(sd, (struct sockaddr *)&caddr, (socklen_t*)&c);
	if(cd < 0){
		printf("accept error, tell admin\n");
		return 0;
	}
	if( recv(cd, buf, 4, 0) != 4 ) return 0;
	if(memcmp(buf, "\xde\xad\xbe\xef", 4)) return 0;
	printf("Stage 5 clear!\n");

	// here's your flag
	system("/bin/cat flag");	
	return 0;
}
```



##分析

题目还是有点意思的，只要过5关就能够`system("/bin/cat flag")`，来看看这5关需要的输入：

1. 100个参数（包含程序名自身），第65个参数应该是"\x00"，第66个参数应该是"\x20\x0a\x0d"。

   因为参数有"\x00"、回车换行等字符，所以不能直接通过python之类的来传参，直接在服务器上写个程序编译执行吧，中间调用API——execve来传参，然后按部就班的布局即可，argv['A']="\x00",argv['B']="\x20\x0a\x0d"。

2. 第一个read中的fd为0，表示stdin，这在fd一题中已经见过了，但是对比的"\x00\x0a\x00\xff"串无法通过命令行输入；而第二个read中的fd为2，表示stderr，对比"\x00\x0a\x02\xff"，stderr更是没有办法从命令行输入。因此，只好用Linux中常见的套路：I/O重定向。

   但是要怎么重定向呢？其实很简单，只需要fork一个子进程，然后通过pipe即可。

3. 传说中的main函数第三个参数，这是个C实现扩展参数，并未在标准中定义。env的值默认是系统环境变量，默认不需要填写，这里可以通过指定env并利用API——execve("input",argv,env)传递。

4. 读文件"\x0a"，前四个字节是"\x00\x00\x00\x00"即可。这个处理就最简单了，直接在程序中如法炮制一个就是了。

5. 这个困难一些，socket编程，但实际上socket说白了也只是进程间通信的一种方式罢了，只是他支持远程。和stage 4类似，我们进行这一关的反操作，也就是建立套接字，连接到server上发送"\xde\xad\xbe\xef"即可。

## Pwn

最终的代码整理一下：

```cpp
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/socket.h>
#include <arpa/inet.h>

int main()
{
  int pipe_stdin[2] = {-1,-1}, pipe_stderr[2] = {-1, -1};
  char *argv[101] = {0};
  /* stage 3 */
  char *envp[2] = {"\xde\xad\xbe\xef=\xca\xfe\xba\xbe", NULL};
  FILE *fp = NULL;
  pid_t pid_child;
  
  /* stage 1 */
  argv[0] = "/home/input2/input";
  for(int i=1;i<100;i++)
  {
    argv[i] = "a";
  }
  argv['A'] = "\x00";
  argv['B'] = "\x20\x0a\x0d";
  argv['C'] = "55555";
  argv[100] = NULL;
  
  /* stage 4 */
  fp = fopen("\x0a","wb");
  if(!fp)
  {
    perror("Cannot open file.");
    exit(-1);
  }
  fwrite("\x00\x00\x00\x00",4,1,fp);
  fclose(fp);
  fp = NULL;
  
  /* stage 2 */
  if(pipe(pipe_stdin) < 0 || pipe(pipe_stderr) < 0)
  {
    perror("Cannot create pipe!");
    exit(-1);
  }
  
  if((pid_child = fork()) < 0)
  {
    perror("Cannot create child process!");
    exit(-1);
  }
  
  if(pid_child == 0)
  {
    //子进程先等待父进程重定向pipe read，然后关闭不使用的pipe read
    //继而向两个pipe write对应的字符串，父进程此时已经把pipe read重定向到stdin和stderr
    //最终由input程序接收
    sleep(1);
    close(pipe_stdin[0]);	
    close(pipe_stderr[0]);
    write(pipe_stdin[1], "\x00\x0a\x00\xff", 4);
    write(pipe_stderr[1], "\x00\x0a\x02\xff", 4);
  }
  else
  {
    //父进程无需pipe write操作，首先close掉，然后把两个pipe read重定向到0和2
    //也就是stdin和stderr
    close(pipe_stdin[1]);
    close(pipe_stderr[1]);
    dup2(pipe_stdin[0], 0);
    dup2(pipe_stderr[0], 2);

    execve("/home/input2/input", argv, envp);
  }
  
  /* stage 5 */
  sleep(5);
  int sockfd;
  struct sockaddr_in saddr;
  sockfd = socket(AF_INET, SOCK_STREAM, 0);
  if(sockfd == -1)
  {
    perror("Cannot create socket!");
    exit(-1);
  }
  saddr.sin_family = AF_INET;
  saddr.sin_addr.s_addr = inet_addr("127.0.0.1");
  saddr.sin_port = htons(55555);
  if(connect(sockfd, (struct sockaddr*)&saddr, sizeof(saddr)) < 0)
  {
    perror("Cannot connect to server!");
    exit(-1);
  }
  write(sockfd, "\xde\xad\xbe\xef", 4);
  close(sockfd);
  
  return 0;
}
```

在/tmp下建一个我们的目录input，编写test.c并编译，注意该目录下没有flag，所以要先通过`ln -s /home/input2/flag flag`来建一个软链接。

之所以新建目录而不是在tmp下操作是因为tmp目录我们本身也没有权限，这会限制最后一步的system("/bin/cat flag")。

![](/images/ctf/pwn/pwnable.kr/input_02.jpg)

**本题实际上考察的就是Linux基本编程功力，需要对Linux的各种基础知识了如指掌，才能游刃有余的解开这道题目。**