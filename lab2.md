# 操作系统实验报告

# 实验1：进程的软中断通信

## 实验内容

（ 1）使用 man 命令查看 fork 、 kill 、 signal、 sleep、 exit 系统调用的帮助手册。

（ 2）根据流程图（如图 2.1 所示） 编制实现软中断通信的程序： 使用系统调用 fork()创建两个子进程，再用系统调用 signal()让父进程捕捉键盘上发出的中断信号（即 5s 内按下delete 键或 quit 键），当父进程接收到这两个软中断的某一个后，父进程用系统调用 kill()向两个子进程分别发出整数值为 16 和 17 软中断信号，子进程获得对应软中断信号，然后分别输出下列信息后终止：

Child process 1 is killed by parent !! Child process 2 is killed by parent !!

父进程调用 wait()函数等待两个子进程终止后，输出以下信息，结束进程执行： Parent process is killed!!

注： delete 会向进程发送 SIGINT 信号， quit 会向进程发送 SIGQUIT 信号。 ctrl+c 为delete， ctrl+\为 quit 。

（ 3）多次运行所写程序，比较 5s 内按下 Ctrl+\或 Ctrl+Delete 发送中断，或 5s 内不进行任何操作发送中断， 分别会出现什么结果？分析原因。

（ 4）将本实验中通信产生的中断通过 14 号信号值进行闹钟中断，体会不同中断的执行样式，从而对软中断机制有一个更好的理解
```c
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/wait.h>
#include <unistd.h>// 定义fork、kill、alarm、pause、sleep等系统调用


void signal_handler(int signal_num) {// 父进程的信号处理函数：接收到SIGINT/Ctrl+C或SIGQUIT/Ctrl+\时执行
  // 注释掉的打印语句：用于调试，显示收到的信号编号
  // printf("Received interrupt signal: %d\n", signal_num);
}


void child_signal_handler(int signal_num) {// 子进程的信号处理函数：接收到16/17号自定义信号时执行终止逻辑
 
  if (signal_num == 16) { // 如果子进程收到16号信号
    printf("Child process 1 is killed by parent !!\n");
  }
  
  if (signal_num == 17) {// 如果子进程收到17号信号
    printf("Child process 2 is killed by parent !!\n");
  }
  exit(0);// 调用exit(0)终止当前子进程，0表示正常退出
}


int main() {
  pid_t child1, child2;// 子进程1和子进程2的PID
  signal(SIGINT, signal_handler);   // 父进程收到SIGINT（Ctrl+C）信号时，执行signal_handler函数
  signal(SIGQUIT, signal_handler);  // 父进程收到SIGQUIT（Ctrl+\）信号时，执行signal_handler函数

 
  // fork()返回值：父进程中返回子进程的PID（大于0），子进程中返回0，失败返回-1
  child1 = fork(); // 创建子进程1
  
  if (child1 == 0) {// 子进程1成功
    // 子进程1注册信号处理规则：收到16号信号时，执行child_signal_handler函数
    signal(16, child_signal_handler); 
    // 子进程1进入无限循环，一直等待信号（不做任何工作，直到收到终止信号）
    while (1) {
    }
  }

  
  child2 = fork();// 创建子进程2
  if (child2 == 0) {
    signal(17, child_signal_handler); 
    while (1) {
    }
  }

  // 父进程的执行逻辑：只有父进程中child1和child2都大于0（两个子进程都创建成功）
  if (child1 > 0 && child2 > 0) {
    alarm(5);// 调用alarm(5)设置5秒超时：5秒后父进程会收到SIGALRM（闹钟）信号（ 14 号闹钟信号）

    pause();// 调用pause()让父进程暂停执行：直到收到任意一个信号后，才会继续执行后续代码
    // 触发条件：1.5秒超时收到SIGALRM；2.手动按Ctrl+C收到SIGINT；3.手动按Ctrl+\收到SIGQUIT
    kill(child1, 16);
    kill(child2, 17);

    wait(NULL); // 调用wait(NULL)等待任意一个子进程退出，防止子进程变成“僵尸进程” 
    wait(NULL); // 再次调用wait(NULL)等待另一个子进程退出 

    printf("Parent process is killed!!\n");// 两个子进程都退出
  }

  return 0;
}
```
### 运行结果截图
![Untitled](picture/2-1(1).png)
![Untitled](picture/2-1(2).png)

编写代码要考虑的问题：

(1)父进程向子进程发送信号时，如何确保子进程已经准备好接收信号？
(2)如何阻塞住子进程，让子进程等待父进程发来信号？
新代码：
```c
#include <stdio.h>
#include <unistd.h>
#include <sys/wait.h>
#include <stdlib.h>
#include <signal.h>

int flag=0;
int is_timeout=0;
int child1_isready=0;
int child2_isready=0;

void inter_handler(int sig) {
 // TODO
    if(sig==SIGINT || sig==SIGQUIT){
        flag++;
    }
}

void alarm_handler(int sig) {
 // TODO
    if(sig==SIGALRM){
        is_timeout=1;
        flag=1;
    }
}

void child1_isready_handler(int sig) {
    if(sig==16){
        child1_isready=1;
    }
}

void child2_isready_handler(int sig) {
    if(sig==17){
        child2_isready=1;
    }
}

void waiting() {
 // TODO
    while(flag<1){
        pause();
    }
}

void child_handler(int sig) {
    if(sig==16){
        printf("\nChild process1 is killed by parent!!\n");
        exit(0);
    }
    if(sig==17){
        printf("\nChild process2 is killed by parent!!\n");
        exit(0);
    }
}

int main() {
    signal(SIGQUIT, inter_handler);
    signal(SIGINT, inter_handler);
    signal(SIGALRM, alarm_handler);
    signal(16, child1_isready_handler);
    signal(17, child2_isready_handler);

    // TODO: 五秒之后或接收到两个信号
    pid_t pid1=-1, pid2=-1;
    while (pid1 == -1)pid1=fork();
    if (pid1 > 0) {
        while (pid2 == -1)pid2=fork();
        if (pid2 > 0) {
            // TODO: 父进程
            alarm(5);
            waiting();
            alarm(0); 
            if(is_timeout){
                //printf("\nThe signal is timeout proc stopped auto!!\n");
            }
            while(!child1_isready){
                pause();
            }
            kill(pid1, 16);
            while(!child2_isready){
                pause();
            }
            kill(pid2, 17);
            wait(NULL);
            wait(NULL);
            
            printf("\nParent process is killed!!\n");
        } else {
            // TODO: 子进程 2
            kill(getppid(), 17);
            signal(17, child_handler);
            while(1){
                pause();
            }
        }
    } else {
        // TODO:子进程 1
        kill(getppid(), 16);
        signal(16, child_handler);
        while(1){
            pause();
        }
        printf("\nChild process1 is killed by parent!!\n");
        return 0;
    }
    return 0;
}
```
对于第一个问题，我设置了两个变量child1_isready和child2_isready，分别表示子进程1和子进程2是否已经准备好接收信号。子进程在启动时会向父进程发送一个信号，通知父进程它已经准备好了。在父进程中，将信号与signal函数绑定在一起，当他接受到某个子进程的信号后，就将相应进程的ready位设置为1.父进程在发送信号之前会检查这两个变量，确保子进程已经准备好。

对于第二个问题，我使用了pause函数来阻塞子进程。pause函数会使进程进入睡眠状态，直到接收到一个信号为止。子进程在启动后会调用pause函数，等待父进程发送信号。当父进程发送信号后，子进程会被唤醒并执行相应的信号处理函数。

### 代码解释
这份代码是有时钟中断的，如果要使程序没有时钟中断，只需要注释掉signal(SIGALRM, alarm_handler);这一行即可。

flag变量用于记录父进程接收到的信号数量。当父进程接收到SIGINT或SIGQUIT信号时，inter_handler函数会将flag加1。当父进程接收到SIGALRM信号时，alarm_handler函数会将is_timeout设置为1，并将flag加1。

waiting函数会使父进程进入睡眠状态，直到flag的值大于等于1，即接收到至少一个信号为止。

在父进程中设置5秒的闹钟，如果在5秒内没有接收到任何信号，闹钟会触发SIGALRM信号，调用alarm_handler函数，将is_timeout设置为1，并将flag加1，从而使父进程退出等待状态。

如果接收到了SIGINT或SIGQUIT信号，父进程会退出waiting，并执行后续代码清除定时器，准备向子进程发送信号。

### 问题回答

（1）你最初认为运行结果会怎么样？写出你猜测的结果。

我最初认为两个进程会按照代码顺序那样，先杀死子进程1，再杀死子进程2。

（2）实际的结果什么样？有什么特点？在接收不同中断前后有什么差别？请将 5 秒内中断
和 5 秒后中断的运行结果截图，试对产生该现象的原因进行分析。

实际上，两个子进程的杀死顺序并不确定。并且如果5秒内有任何操作，父进程会向子进程发送信号，而如果五秒内没有任何操作，终端会输出"Alarm clock"。

原因分析：对于杀死顺序不确定这个问题，可能是因为编译代码时，两个kill的顺序并没有严格保证执行顺序，不过这个可能性比较低。
更可能的原因是父进程只负责发送信号，在发送信号后，子进程的执行顺序是由操作系统调度决定的，因此子进程1和子进程2被杀死的顺序是不确定的。

对于5s内中断和5秒后中断的区别：由于alarm信号没有被处理，因此当5秒内没有任何操作时，闹钟会触发SIGALRM信号，导致父进程退出等待状态，并输出"Alarm clock"。如果5秒内有任何操作，父进程会接收到SIGINT或SIGQUIT信号，正常向子进程发送信号。

（3）改为闹钟中断后，程序运行的结果是什么样子？与之前有什么不同？

将signal(SIGALRM, alarm_handler);取消注释后，程序运行结果并不因为5s内有无操作而改变。因为此时改变了对SIGALRM信号的处理方式，父进程在5秒后会接收到SIGALRM信号，并调用alarm_handler函数，将is_timeout设置为1，并将flag加1，使父进程退出waiting状态。
这个状态的退出5s内有操作和没有操作都是一样的。

（4）kill 命令在程序中使用了几次？每次的作用是什么？执行后的现象是什么？

kill命令在程序中使用了两次，分别用于向子进程1和子进程2发送信号。
每次的作用是通知子进程它们应该终止执行。执行后的现象是子进程会输出相应的消息，表示它们被父进程杀死，然后终止执行。

（5）使用 kill 命令可以在进程的外部杀死进程。进程怎样能主动退出？这两种退出方式
哪种更好一些？

进程可以通过调用exit函数来主动退出。exit函数会终止进程的执行，并返回一个状态码给操作系统。
这两种退出方式各有优缺点。使用kill命令可以在进程的外部强制终止进程，适用于需要立即停止进程的情况，但可能会导致资源未被正确释放。而主动调用exit函数可以确保进程有机会进行清理工作，释放资源，但可能需要更多的时间来完成退出过程。总体来说，主动退出更好一些，因为它允许进程有机会进行必要的清理工作。







# 实验2 进程的管道通信

### 实验内容

（ 1）学习 man 命令的用法，通过它查看管道创建、同步互斥系统调用的在线帮助，并阅读参考资料。

（ 2）根据流程图（如图 2.2 所示）和所给管道通信程序，按照注释里的要求把代码补充完整，运行程序，体会互斥锁的作用，比较有锁和无锁程序的运行结果，分析管道通信是如何实现同步与互斥的。

基本代码

```c
#include <unistd.h> 
#include <signal.h> 
#include <stdio.h> 
#include<sys/wait.h>
#include<stdlib.h>
int pid1,pid2; // 定义两个进程变量
int main() {
    int fd[2]; 
    char InPipe[1000]; // 定义读缓冲区
    char c1='1', c2='2'; 
    pipe(fd); // 创建管道
    while((pid1 = fork( )) == -1); // 如果进程 1 创建不成功,则空循环
    if(pid1 == 0) { // 如果子进程 1 创建成功,pid1 为进程号
        lockf(fd[1],1,0);// 锁定管道
        
        for(int i=0;i<2000;i++)
            write(fd[1],&c1,1);// 分 2000 次每次向管道写入字符’1’ 
    sleep(5); // 等待读进程读出数据
    lockf(fd[1],0,0);// 解除管道的锁定
    exit(0); // 结束进程 1 
    } 
    else { 
    while((pid2 = fork()) == -1); // 若进程 2 创建不成功,则空循环
    if(pid2 == 0) { 
    lockf(fd[1],1,0); 
        for(int i=0;i<2000;i++)
            write(fd[1],&c2,1);// 分 2000 次每次向管道写入字符’2’ 
    sleep(5); 
    lockf(fd[1],0,0); 
    exit(0); 
    } 
    else { 
    wait(0);// 等待子进程 1 结束
    wait(0); // 等待子进程 2 结束
    read(fd[0],InPipe,4000);// 从管道中读出 4000 个字符
    InPipe[4000] = '\0';// 加字符串结束符
    printf("%s\n",InPipe); // 显示读出的数据
    exit(0); // 父进程结束
    }
}
```
#### 上述原代码的执行结果
![Untitled](picture/2.21.png)
![Untitled](picture/2.22.png)
在ls命令打印结果中可以看到有5个文件描述符，0，1，2分别对应标准输入，标准输出，标准错误输出，3和4分别对应管道的读端和写端。符合我们的预期，并且由于lockf函数的使用，管道通信是互斥的，子进程1和子进程2不会同时向管道写入数据，因此读出的数据是2000个'1'后跟2000个'2'。

在这个程序中，我认为sleep是没有用处的，注释掉之后，输出结果与上述相同。

并且如果不考虑僵尸进程的风险，wait也是没有必要出现在while读取之前的，因为while循环在子进程写入数据的过程中是不会跳出的，注释掉之后，运行结果仍然相同。

上述三种情况是有lockf的情况，输出1，2是有序的。

#### 注释掉lockf
##### 第一种情况仍然保留了sleep和wait
可以看到输出结果是交错的，说明两个子进程在没有锁的情况下同时向管道写入数据，导致读出的数据是混乱的。
##### 注释掉sleep和wait，运行结果相同


### 问题回答
(1)你最初认为运行结果会怎么样？

首先system命令会在终端打印当前进程的文件描述符情况，应该会有5个文件描述符，分别对应标准输入，标准输出，标准错误输出，管道的读端和写端。在有lockf情况下，我认为输出结果应该是2000个'1'后跟2000个'2'，因为lockf函数会确保两个子进程不会同时向管道写入数据。在没有lockf情况下，我认为输出结果可能是交错的'1'和'2'，因为两个子进程可能会同时向管道写入数据，导致读出的数据是混乱的。

(2)实际的结果什么样？有什么特点？试对产生该现象的原因进行分析。

实际运行结果与我的预期相符。在有lockf的情况下，输出结果是2000个'1'后跟2000个'2'，说明lockf函数成功地确保了两个子进程的互斥访问管道。在没有lockf的情况下，输出结果是交错的'1'和'2'，说明两个子进程确实同时向管道写入数据，导致读出的数据是混乱的。

(3)实验中管道通信是怎样实现同步与互斥的？如果不控制同步与互斥会发生什么后果？
管道通信通过lockf函数实现了同步与互斥。lockf函数用于锁定文件描述符，确保在一个进程写入数据时，其他进程无法访问该文件描述符，从而实现了互斥访问管道。如果不控制同步与互斥，多个进程可能会同时向管道写入数据，导致读出的数据是混乱的，无法正确地反映各个进程写入的数据内容。

# 实验4 页面的置换

###2.4.1 实验目的
通过模拟实现页面置换算法（FIFO、LRU），理解请求分页系统中，页面置换的实现思路，
理解命中率和缺页率的概念，理解程序的局部性原理，理解虚拟存储的原理。
###2.4.2 实验内容
（1）理解页面置换算法 FIFO、LRU 的思想及实现的思路。
（2）参考给出的代码思路，定义相应的数据结构，在一个程序中实现上述 2 种算法，运行
时可以选择算法。算法的页面引用序列要至少能够支持随机数自动生成、手动输入两种生成
方式；算法要输出页面置换的过程和最终的缺页率。
（3）运行所实现的算法，并通过对比，分析 2 种算法的优劣。
（4）设计测试数据，观察 FIFO 算法的 BLEADY 现象；设计具有局部性特点的测试数据，分
别运行实现的 2 种算法，比较缺页率，并进行分析。


基本代码（FIFO）

```c
#include<stdio.h>
#include<stdlib.h>
#define PAGEMAX 100  // 定义页面序列的最大长度

// 物理内存的一个页框
typedef struct page{
    int num;          // 页面编号
    int status;       // 页框的状态：0=空闲，1=被占用
    struct page *next;// 指向下一个页框节点的指针
}page;

// 页框链表的头尾指针
page *head;
page *tail;

//通过repos变量记录下一个要替换的页框位置，每次替换后repos循环递增
//（repos = (repos+1)%blocknum），实现 “最早进入的页面先被替换”。
int repos=0;		// 记录下一个要替换的页框位置
int pagenum;         // 页面访问序列的长度
int blocknum;        // 物理内存的页框数量
int *page_seq;       // 存储页面访问序列的动态数组
int missingcount=0;  // 缺页次数（初始化为0）

// 初始化函数：创建包含blocknum个页框的单链表
void init(){
    int i;
    head = (page *)malloc(sizeof(page));// 动态分配头节点内存（第一个页框）
    tail=head;        // 尾指针初始指向头节点
    head->num = 0;    // 初始页面编号为0（未分配）
    head->status = 0; // 初始状态为空闲

   
    page *Pp[blocknum]; // 存储后续页框节点的地址（简化链表连接）
    for(i =0; i < blocknum-1;i++){// 循环创建blocknum-1个页框节点
        Pp[i] = (page *)malloc(sizeof(page));
        Pp[i]->num = 0;       // 初始页面编号为0
        Pp[i]->status = 0;    // 初始是空闲
    }
    
    for(i =0;i<blocknum-2;i++){// 连接后续页框节点，形成单链表
        Pp[i]->next = Pp[i+1];
    }
    
    if(Pp[0])// 如果创建了后续节点，将头节点的next指向第一个后续节点
        head->next =Pp[0];
    // 如果页框数≥2，尾指针指向最后一个后续节点
    if(blocknum>=2)
        tail = Pp[blocknum-2];
    tail->next=NULL;  // 链表尾节点的next置空
}

// 打印函数：输出当前所有页框的页面编号、头尾节点的页面编号
void print(){
    page *p = head;
    printf("Page ：");
    // 遍历链表，打印每个页框的页面编号
    while (p != NULL) {
        printf("%d ", p->num);
        p = p->next;
    }
    printf("\n");
    printf("The poisition of head：%d\n", head->num); // 打印头节点页面编号
    printf("The poisition of tail：%d\n", tail->num); // 打印尾节点页面编号
    printf("\n");
}

// 查找函数：判断指定页面是否已在物理内存的页框中（命中返回1，缺页返回0）
int search(int p) {
    page *q = head;
    while (q != NULL) {
        if (q->num == p) {
            return 1; // 页面存在，命中
        }
        q = q->next;
    }
    return 0; // 页面不存在，缺页
}

// 查找空闲页框函数：返回第一个空闲（status=0）的页框节点指针，无空闲返回NULL
page *find_free() {
    page *q = head;
    while (q != NULL) {
        if (q->status == 0) {
            return q; // 找到空闲页框
        }
        q = q->next;
    }
    return NULL; // 无空闲页框
}

// 替换函数：实现FIFO置换规则，将指定页面替换到当前repos指向的页框
void replace(int p) {
    int i;
    page *P =(page *)malloc(sizeof(page)); // 注意：此处malloc会造成内存泄漏，后续优化
    P =head; // P指向头节点，开始遍历到repos位置的页框
    // 循环遍历到第repos个页框节点
    for(i=0;i<repos;i++){
        P=P->next;
    }
    P->num = p; // 将新页面编号写入该页框
    repos = (repos+1)%blocknum; // repos循环递增，指向下一个要替换的页框（FIFO核心）
}

// 模拟函数：遍历页面访问序列，模拟页面访问、缺页、置换的全过程
void simulate(){
    int i;
    // 遍历每个页面访问请求
    for(i=0;i<pagenum;i++){
        printf("serch page: %d\n",page_seq[i]); // 打印当前访问的页面
        if(search(page_seq[i])){ // 查找页面是否在内存中
            printf("page hit! \n"); // 页面命中
        }
        else{ // 页面缺页
            printf("page missing \n");
            missingcount++; // 缺页次数加1
            page *fb = find_free(); // 查找空闲页框
            if(fb){ // 有空闲页框
                printf("There is free block, fill it in\n");
                fb->num = page_seq[i]; // 将页面写入空闲页框
                fb->status = 1; // 标记页框为占用状态
            }
            else{ // 无空闲页框，需要置换
                printf("No free block, replace\n");
                replace(page_seq[i]); // 调用FIFO置换函数
            }
            print(); // 打印置换后的页框状态
        }
    }
}

// 主函数：实验入口，处理输入、初始化、模拟、结果输出
int main(){
    int i;int random=0;
    printf("input page number：\n");
    scanf("%d", &pagenum); // 输入页面访问序列的长度
    printf("input block number：\n");
    scanf("%d", &blocknum); // 输入物理内存的页框数量
    // 动态分配内存存储页面访问序列
    page_seq = (int *)malloc(sizeof(int) * pagenum);
    printf("choose the way of creating page sequence\n");
    printf("input 1 for random sequence, 0 for your own sequence\n");
    scanf("%d",&random); // 选择序列生成方式：1=随机，0=手动输入
    if(random){ // 随机生成页面序列
        printf("\nrandom page sequence:");
        for ( i = 0; i < pagenum; i++) {
            // 随机生成页面编号：范围[1, blocknum+2]，确保有足够的缺页
            page_seq[i] = rand() % ((blocknum) + 2)+1;
            printf("%d ", page_seq[i]);
        }
        printf("\n");
    }
    else{ // 手动输入页面序列
        printf("input page sequence: ");
        for ( i = 0; i < pagenum; i++) {
            scanf("%d", &page_seq[i]);
        }
    }
    init(); // 初始化页框链表
    simulate(); // 模拟页面访问过程
    // 输出实验结果：缺页次数和缺页率（保留2位小数）
    printf("page missing number：%d\n", missingcount);
    printf("page missing rate：%.2f%%\n", (double)missingcount / pagenum * 100);
    return 0;
}
```
![Untitled](picture/2-3fifo.png)
![Untitled](picture/2-3fifo1.png)
基本代码（LRU）
```c
#include <stdio.h>
#include <stdlib.h>

// 单链表，物理内存的页框
typedef struct Node {
    int data;          // 页面编号
    struct Node* next; // 指向下一个页框节点的指针
} Node;

int cpn=0;//当前内存中的页面数量
int pageMissCount = 0;//缺页次数

// 创建新节点
Node* createNode(int data) {
    Node* newNode = (Node*)malloc(sizeof(Node));
    newNode->data = data;
    newNode->next = NULL;
    return newNode;
}

// 头插法插入节点：将新页面插入到链表头部（标记为最近使用）
void insertAtHead(Node** head, int data) {
    Node* newNode = createNode(data);
    newNode->next = *head; // 新节点的next指向原链表头
    *head = newNode;       // 链表头指针指向新节点
}

// 查找节点：判断指定页面是否在内存（链表）中，存在返回节点指针，否则返回NULL
Node* findNode(Node* head, int data) {
    Node* current = head;
    while (current != NULL) {
        if (current->data == data) {
            return current;
        }
        current = current->next;
    }
    return NULL;
}

// 删除指定节点：从链表中删除指定页面的节点（用于命中时将页面移到头部）
void deleteNode(Node** head, int data) {
    Node* current = *head;
    Node* prev = NULL;

    // 遍历链表，找到要删除的节点及其前驱节点
    while (current != NULL && current->data != data) {
        prev = current;
        current = current->next;
    }

    // 如果找到节点，执行删除操作
    if (current != NULL) {
        if (prev == NULL) { // 要删除的是头节点
            *head = current->next;
        } else { // 要删除的是中间/尾部节点
            prev->next = current->next;
        }
        free(current); // 释放节点内存，避免泄漏
    }
}

// 删除尾节点：删除链表最后一个节点（淘汰最久未使用的页面）
void deleteTail(Node** head) {
    Node* current = *head;
    Node* prev = NULL;

    // 遍历到链表尾部
    while (current != NULL && current->next != NULL) {
        prev = current;
        current = current->next;
    }

    // 执行尾节点删除
    if (current != NULL) {
        if (prev == NULL) { // 链表只有一个节点
            *head = NULL;
        } else { // 链表有多个节点
            prev->next = NULL;
        }
        free(current); // 释放尾节点内存
    }
}

// 打印链表：输出当前内存中的所有页面（从最近使用到最久未使用）
void printList(Node* head) {
    Node* current = head;
    while (current != NULL) {
        printf("%d ", current->data);
        current = current->next;
    }
    printf("\n");
}

// 主函数：实验入口，处理输入、模拟LRU置换、输出结果
int main() {
    int i;
    int generate=0; // 标记是否生成随机页面序列：1=随机，0=手动输入
    int memorySize, pageSize; // memorySize=物理内存页框数，pageSize=页面序列长度

    // 输入物理内存的页框数量
    printf("input block number: ");
    scanf("%d", &memorySize);

    // 输入页面访问序列的长度
    printf("input page number: ");
    scanf("%d", &pageSize);

    // 动态分配内存，存储页面访问序列
    int* pages = (int*)malloc(sizeof(int) * pageSize);

    // 选择页面序列的生成方式
    printf("choose the way of creating page sequence\n");
    printf("input 1 for random sequence, 0 for your own sequence\n");
    scanf("%d",&generate);

    if(generate){ // 随机生成页面序列
        printf("\nrandom page sequence:");
        for ( i = 0; i < pageSize; i++) {
            // 随机生成页面编号：范围[1, memorySize+2]，确保有足够的缺页
            pages[i] = rand() % ((memorySize) + 2)+1;
            printf("%d ", pages[i]);
        }
        printf("\n");
    }
    else{ // 手动输入页面序列
        printf("input page sequence: ");
        for ( i = 0; i < pageSize; i++) {
            scanf("%d", &pages[i]);
        }
    }
    printf("\n");

    // 初始化链表：memory为物理内存的链表头指针，pageList未实际使用（可忽略）
    Node* memory = NULL;
    Node* pageList = NULL;

    // 遍历页面访问序列，模拟LRU置换过程
    for (i = 0; i < pageSize; i++) {
        int currentPage = pages[i]; // 当前访问的页面编号
        printf("serch for page %d\n", currentPage); // 打印当前访问的页面（笔误：search）

        // 第一步：判断页面是否在内存中（命中）
        if (findNode(memory, currentPage) != NULL) {
            printf("page hit\n"); // 页面命中
            // 命中处理：将该页面从原位置删除，重新插入到链表头部（标记为最近使用）
            deleteNode(&memory, currentPage);
            insertAtHead(&memory, currentPage);
        } 
        // 第二步：页面缺页，执行置换逻辑
        else {
            printf("page missing\n"); // 打印缺页提示
            // 如果内存已满（当前页面数≥页框数），淘汰尾部的最久未使用页面
            if (cpn>=memorySize) {
                printf("page evict\n"); // 打印页面淘汰提示
                deleteTail(&memory); // 删除尾节点（淘汰最久未使用页面）
                cpn--; // 内存页面数减1
            }
            // 将新页面插入到链表头部（标记为最近使用）
            insertAtHead(&memory, currentPage);
            cpn++; // 内存页面数加1
            pageMissCount++; // 缺页次数加1
        }

        // 打印当前内存中的页面状态（从最近使用到最久未使用）
        printf("current page situation：");
        printList(memory);
        printf("\n");
    }

    // 输出实验结果：缺页次数和缺页率（保留2位小数）
    printf("missing number: %d\n", pageMissCount);
    printf("missing rate: %.2f%%\n", (float)pageMissCount / pageSize * 100);

    // 释放动态分配的内存，避免内存泄漏
    free(pages); // 释放页面序列的内存
    // 释放物理内存链表的所有节点
    Node* current = memory;
    while (current != NULL) {
        Node* next = current->next;
        free(current);
        current = next;
    }
    // 释放pageList链表（未实际使用，可忽略）
    current = pageList;
    while (current != NULL) {
        Node* next = current->next;
        free(current);
        current = next;
    }

    return 0;
}
```
![Untitled](picture/2-3lru.png)
![Untitled](picture/2-3lru1.png)
