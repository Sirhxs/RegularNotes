## 信号 初级异步
Linux环境的并发可以分为多进程+信号和多线程两种，信号属于初级异步，多线程属于强烈异步。<br>
信号与信号量不是一回事，信号量是指在同一个资源下，信号量通过P/V原子操作，使得线程之间能够按顺序执行，也就是同步，信号量是用计数的整数和一个队列来完成<br>
实际项目中，信号和多线程基本不会一块使用，要么使用多进程+信号，要么采用多线程的形式。<br>
**同步程序**的特点就是程序的执行流程、分支都是很明确，但是 **异步事件**就是事件到来的时间无法确定，到来之后的结果也是不确定的。比如俄罗斯方块游戏中，需要异步接收用户的方向控制输入。这就需要信号的支持。<br>
异步事件的获取方式通常有两种，一种是查询法，一种是通知法。<br>
异步事件到来的频率比较高的情况考虑用 **查询法**，因为撞到异步事件到来的概率比较高。<br>
异步事件到来的频率比较稀疏的情况考虑通知法，因为经济实惠。<br>
**所有的通知法都需要配合一个监听机制❕** 例如🎣钓鱼，放一个鱼竿就你就走了，就算🐟上钓你也不知道。<br>
#### 时间片调度
时间片调度就是通过中断打断程序的执行，把事件片耗尽的进程移动到队列中等待。所以任何进程在执行的过程中都是磕磕绊绊的不断被打断的，程序在任何地方都有可能被打断，唯独一条机器指令是无法被打断(原子操作)。<br>
### 信号概述
**信号不是中断**，中断只能由硬件产生， **信号是模拟硬件中断⏸️的原理**，在软件层面上进行的。<br>
`kill -l`，向其他进程查看或发送信号
```c
 1) SIGHUP       2) SIGINT       3) SIGQUIT      4) SIGILL
 5) SIGTRAP      6) SIGABRT      7) SIGEMT       8) SIGFPE
 9) SIGKILL     10) SIGBUS      11) SIGSEGV     12) SIGSYS
13) SIGPIPE     14) SIGALRM     15) SIGTERM     16) SIGURG
17) SIGSTOP     18) SIGTSTP     19) SIGCONT     20) SIGCHLD
21) SIGTTIN     22) SIGTTOU     23) SIGIO       24) SIGXCPU
25) SIGXFSZ     26) SIGVTALRM   27) SIGPROF     28) SIGWINCH
29) SIGINFO     30) SIGUSR1     31) SIGUSR2
```
1-31标准信号，这里都是标准信号，其实下面讨论的内容没有特殊标记出来就是针对标准信号的。信号名都被定义为正整数常量<br>
信号有五种不同的默认行为:  **终止，终止+core，忽略，停止进程🤚，继续🏃。**<br>
core文件就是程序在崩溃时由操作系统为它生成的内存现场映像📷和调试信息，只要用来调试程序的，可以使用ulimit命令设置允许生成core文件的最大大小。<br>
* 终止🔚：使程序异常结束🔚。被信号杀死就是异常终止
* 终止+core：杀死进程，并为其产生一个core dump文件，可以使用这个core dump文件获得程序被🔪杀死的原因。
* 忽略：程序会忽略该信号，不作出任何响应。
* 停止🛑进程：将运行中的程序中断。被停止的进程就像被下了一个断点一样，停止运行并不会再被调度，直到收到继续运行的信号。当按下Ctrl+Z时就会将一个正在运行的前台进程停止，其实就是向这个进程发送了一个SIGTSTP信号(STOP🛑)。
* 继续🏃：使被停止的进程继续运行▶️。只有SIGCONT(continue)具有这个功能。
<br>
以下介绍几种常用的标准信号<br>

信号|默认动作|说明
|--|--|:--|
SIGABRT|终止+core|调用abort函数会向自己发送该信号使程序异常终止，通常在程序自杀或夭折时使用
SIGALRM|终止|调用alarm或setitimer定时器超时时⌚️向自身发送信号。setitimere设置which参数的值为ITMER_REAL时，超时后会发送此信号
SIGCHLD|忽略|当子进程状态改变系统会将该信号发送给其父进程。状态改变是🈯️由运行▶️状态改变为⏸️暂停状态、由暂停状态⏸️改变为运行▶️状态、由运行状态▶️改变为终止🛑状态等等
SIGHUP|终止|如果终端💻接口检测到链接断开则将此信号发送给该终端的控制进程，通常会话首进程就是该终端的控制进程
SIGINT|终止|当用户按下中断键➖(Ctrl+C)时，终端驱动程序产生此信号并发送给前台进程组中的每一个进程。
SIGPROF|终止|setitimer设置which参数的值为ITIMER_PROF时，超时后会发送此信号
SIGCONT|继续/忽略|接收信号的进程处于停止🤚信号则继续执行，否则忽略
SIGQUIT|终止+core|当用户在终端按下退出键➖`(Ctrl+\)`时，终端驱动程序产生此信号并发送给前台进程组中的所有进程，该信号与SIGINT的区别时，在终止进程的同时为它生成core dump文件
SIGTERM|终止|使用kill发送信号，若不置顶具体信号，则默认发送该信号
SIGUSR1|终止|用户自定义的信号。系统不赋予特殊意义，想拿来干嘛就干嘛
SIGUSR2|终止|同上
SIGVTALRM|终止|setitimer设置which参数的值为ITIMER_VIRTUAL时，超时后发送该信号

<br>

### 函数signal
```c
// signal - ANSI C signal handling
// 查了一下man手册，signal函数如下
#include <signal.h>
void (*signal(int sig, void (*func)(int)))(int);
// 参数时整型sig和指向参数类型时int，返回类型是void的函数，signal是一个函数指针，指向的函数具有以上两个参数并返回一个指针，指向参数是int型的函数。
// or in the equivalent but easier to read typedef'd version:
typedef void (*sig_t) (int);
sig_t signal(int sig, sig_t func);
```
sig是1-31标准信号，func是收到信号时的处理行为，也就是信号处理函数；也可以使用SIG_DEF和SIG_IGN两个宏来替代。SIG_DEF表示使用信号的默认处理行为。SIG_IGN(ignore)。<br>
返回值，返回原来的信号处理函数。有时候我们在定义自己的信号📶处理函数之前会把原来的信号处理函数保存下来，这样当我们的库使用完之后需要还原原来注册的信号处理函数，避免因为我调用了我们的库而导致别人的库失效的问题。
```c
#include <stdio.h>
#include <signal.h>
#include <unistd.h>

static void handler(int s){
    write(1,"!",1);
}

int main(){
    int i = 0;
    signal(SIGINT,handler);
    for(i = 0;i<10;i++){
        write(1,"*",1);
        sleep(1);
    }
    return 0;
}
```
接下来每秒钟会打印一个(*)，当按下Ctrl+C就打印一个❕。
```c
transcheung$ time ./a.out
******^C!***^C!*
real    0m8.246s
user    0m0.002s
sys     0m0.003s
```
这就改变了传送信号的行为。通过time命令可以测试到，程序并没有持续10秒钟才结束，这是因为信号打断阻塞的系统调用，也就是说，SIGINT打断了sleep。所以这里的信号是随时接收信息的，也不会改变i的值，所以每次打断之后就会重新再跑一遍for循环。<br>

### 竞争
当学习了信号之后，我们的程序中就出现异步的情况了，只要是异步的程序就可能会出现竞争。那什么是竞争呢？<br>
竞争：一个十字路口没有红绿灯🚥，两辆来自不同方向的驶来的车🚗都可能发生碰撞💥，而且碰撞可能严重也可能轻微。当安装上红绿灯🚥之后，就相当于增加了一个协议，如果没有这个下一的限制，大家就可以随意地使用公共资源了，你在十字路口中间打飞机✈️都可以。所以为了避免竞争带来的后果，我们可以使用一些协议来避免竞争的发生。<br>

### 不可靠的信号 行为不可靠
不可靠信号的行为有可能会丢失，所以在编写信号处理时，应该将信号到来时，在程序里确定该信号已经接收过，这样就不会使信号可能被丢失。<br>
信号的处理就好比我正在玩王者，突然一个电话☎️来了，于是打断了玩游戏，先挂机去接电话。<br>
信号处理函数的执行现场不是程序员布置的，而是 **内核布置的**，因为程序中不会有调用信号处理函数的地方。<br>
同一个信号处理函数的执行线程会被布置在同一个地方，所以当一次信号处理函数未执行完成时再次出发了相同的信号，信号处理函数发生了第二次调用就会覆盖第一次调用的执行现场。<br>
### 可重入函数
函数重入一看有点像是递归，但是，递归调用是程序猿布置的，而 **重入是在一个函数执行未结束时再次发生了调用并且进入了同一个函数现场。**<br>
重入时函数会发生错误的函数称为“不可🙅重入函数”，重入不会出现错误的函数叫做“可👌重入函数”。<br>
所有的系统调用都是可重入函数，所以信号处理函数中可以放心的使用系统调用。但并不是说所有的非系统调用都是不可重入的。<br>
man手册中所有函数如果有一个同名的带_r后缀的函数，是可重入函数，而不带_r后缀的函数是不可重入的函数。
```c
strerror,strerror_r - return string decribing error number
#include <string.h>
char *strerror(int errnum);

int strerror_r(int errnum,char *buf,size_t buflen);
char *strerror_t(int errnum,char *buf, size_t buflen);
```
可重入函数上面讲了一个大概，接下来具体讲讲这个概念。<br>
#### 不可重入函数，使任务调度改变了其他任务调度
如果多个任务调用同一个函数的情况，如果一个函数设计成，不同任务调度这个函数时，都有可能修改其他任务调度这个函数的数据，从而导致不可预料的后果。这就使线程不安全了。因为线程是共用资源的。<br>
因为也导致了不可重入函数在调用的时候不可以被中断，因为一旦中断了，可能资源改变了，就会出现不可预料的问题，因而，它也不能运行在多任务的环境下<br>
#### 可重入函数 一个安全的函数，不担心数据会出错
可重入函数是指一个可以被多个任务调度的过程中，任务在调度的时候不必担心数据是否会出错。简单来说，就是一个 **可以被中断的函数**。<br>
所以说，可重入函数使可以重复进入的，这也是为什么不能在一个函数中随意申请堆内存，这样使得堆内存可能又被重复申请造成资源的浪费，堆栈溢出。一个函数可以被中断意味着它除了使用自己栈上的变量以外，不依赖任何环境(包括static，当然要是预料之内要改变static除外)，这样就允许了该函数可以拥有多个副本，由于使用分离的栈，所以不会互相干扰。如果确实需要访问全局变量，那么就要实施互斥手段，就是锁🔒资源mutex。可重入函数在并行环境中很重要，但是一般要为访问全局变量付出一点性能上的代价。<br>
总结一下，编写可重入函数时，若使用全局变量，则应通过关中断，信号量(P/V操作)等手段加以保护。<br>
来看一个简单的函数
```c
int Exam = 0;
unsigned int noinagain(int para){
    unsigned int temp;
    Exam = para;
    temp = ...;
    return temp;
}
```
这个就是不可重入了，因为使用了全局变量Exam。如果使多进程就会变得不可知Exam的状态。那么加锁呢？<br>
```c
int Exam = 0;
unsigned int noinagain(int para){
    unsigned int temp;
    // down(&mutex)
    Exam = para;
    temp = ...;
    // up(&mutex);
    return temp;
}
```
加锁就会使得该函数可以重入也没问题，这样资源就不会因为会被改变而变得不可知。<br>
因为保证可重入性可以使用以下方法。
* 函数设计精良使用局部变量(寄存器和堆栈)
* 对使用的全局变量加以保护，关中断，信号量，互斥锁等<br>

不可重入就是
* 使用了static
* 申请了堆内存malloc或free
* 调用了标准IO函数<br>
改写就按可重入的规则改写咯。<br>
忠告，保证线程安全，多写可重入函数。<br>

### 可靠信号属于和语义
![标准信号的处理过程](./img/signal_stand_deal.png "标准信号的处理过程")<br>
mask和padding位图是一一对应的，它们用于反映当前进程信号的状态。每一位代表了一个标准信号，例如SIGINT打断。<br>
mask位图用于记录📝哪些信号可以响应。1表示该信号📶可以响应，0表示该信号不可响应(会被忽略)。<br>
padding位图用于记录收到了哪些信号。1表示收到了该信号，0表示没有收到该信号。<br>
程序在执行的过程中会被打断无数次，也就是说程序被打断吼要停止手头的工作，进入一个队列排队等待再次被调度才能继续工作。<br>
当 **进程获得调度机会后** ，从内核态返回到用户态之前要做很多事情，其中一件事就是 **将mask位图和padding位图进行&运算**，当 **计算结果不为0时** 就需要调用相应的信号处理函数或执行信号的默认动作。<br>
这就是Linux的信号处理机制，从这个机制中，可以总结出几个信号的特点。
* 如果想要屏蔽某个信号，只需要将对应的mask位置为0即可。这样当程序从内核态返回用户态进行mask&padding时，该信号位的计算结果一定为0.
* 信号📶从收到到响应是存在延迟的，一般最长的延迟时间10ms。因为 **只有程序被打断并且重新被调度的时候才有机会发现收到了信号** ，所以当我们向一个程序按下Ctrl+C时，程序并没有立即挂掉，只不过这个时间非常短暂我们一般情况下感觉不到而已，我们自己会以为程序是立即挂掉。写一个死循环就能验证
* 当一个信号没有被处理时，无论再次接受到多少个相同的信号都只能保留一个，因为padding是一个位图，位图的特点就是只能保留最后一次的状态。这一点说得就是标准信号会丢失的特点，如果想要不丢失信号就只能使用 **实时信号**。
* 信号处理函数不允许使用longjmp进行跨过函数跳转。因为处理信号之前系统会把mask对应的位图设置为0来避免信号处理函数重入，当信号📶处理完成之后系统会把对应的mask位设置为1恢复进程对该信号的响应能力。 **(所以其实内核在处理函数的时候，信号一旦收到了就会就mask设置为0，避免其他信号的干扰，等处理完这个信号再置为1)。** 如果进行了长跳转系统就不会恢复mask位图了，也就再也无法收到该信号了。 实际上，信号是县城级别的，即使mask位图在处理前被置为0，依然有可能出现重入的现象，因为其他兄弟线程其实也有可能mask值没改变。<br>
* 信号处理函数的执行时间越短越好，因为信号处理函数是在用户态执行的，在它的执行过程中也会不停的被内核打断，所以如果信号处理函数执行的时间过长会使情况变得复杂。<br>
* 信号的响应是嵌套执行的。就是说假设进行先收到了SIGINT信号，当它的信号处理函数还没有执行完毕时又收到了另一个信号SIGQUIT，那么当进程从内核态返回到用户态时会优先执行SIGQUIT的信号处理函数，SIGQUIT处理好->SIGINT继续处理(回到上次被打断的地方继续执行)，这就给人一种错觉，像是在SIGINT的信号处理函数中调用了SIGQUIT的信号处理函数一样。但其实是执行了SIGQUIT，再从SIGINT打断的地方继续执行。<br>
* 如果同时到来多个优先级差不多的信号，无法保证优先响应哪个信号，它们的响应没有严格意义上的顺序(没有队列的概念)。除非是收到了优先级较高的信号，系统会保证高优先级的先被处理。<br>
### kill 🔪杀死一个进程?不不不，作为一个杀手，我还有很多信号📶
```c
kill - send a signal to a process or a group of process
#include <signal.h>
int kill(pid_t pid,int sig);
// 成功 0， 失败-1 并设置errno
```
kill函数的作用就是将指定的信号sig发送给指定的进程pid<br>
kill其实负责给进程发送各种信号。<br>

值|说明
|--|:--|
`>0`|接收信号的进程ID
`==0`|发送信号给当前进程所在进程组的所有进程
`==-1`|发送信号给当前进程有权向它们发送信号的所有进程，1号init进程除外。相当于📢广播信号，发送这种信号只有1号init会做，比如关机📴，1号就会广播信号📢大家，结束了。
`<-1`|将pid的绝对值作为组ID，给这个组中所有的进程发送信号

<br>
sig:要发送的信号，可以使用`kill -l`列出可以发送的信号。这里要说一下0这个信号，0会执行所有的错误检查，并不发送信号，0只是检查一下这个进程是不是依然存在，如果该进程不存在则返回-1并将errno设置为ESRCH。⚠️，这种检查不是原子，当kill返回测试结果的时候，也许被测试的进程也就终止了。当然也可以测试当前进程是否对目标进程有权限发送信号，如果errno为EPERM表示被测试的进程存在但当前进程无权限访问。<br>

#### pause
```c
pause - suspend the thread until a signal is received
#include <unistd.h>
int pause(void);
// 专门用于阻塞当前进程，等待一个信号来打断
```
#### alarm
```c
alarm - schedule an alarm signal

#include <unistd.h>

unsigned alarm(unsigned seconds);
```
指定seconds秒，发送一个SIGALARM给自己。为0时，表示取消这个定时器， **并且新设置的值会覆盖上次设置的值**。所以当程序中出现了多个对alarm的调用，⌛️计时是不准确的。<br>
SIGALARM默认动作时杀死进程
```c
#include "../include/apue.h"
#include <pwd.h>

static void my_alarm(int sig){
    struct passwd *rootptr;
    printf("in signal handler\n");
    if((rootptr = getpwnam("root"))==NULL)
        err_sys("getpwnam(root) error");
    alarm(1); // 重新设置alarm 不断重入
}

int main(void){
    struct passwd *ptr;
    signal(SIGALRM,my_alarm);// 等待信号
    alarm(1); // 发出信号 执行my_alarm
    for(;;){
        if((ptr = getpwnam("transcheung"))==NULL)
            err_sys("getpwnam error");
        if(strcmp(ptr->pw_name,"transcheung")!=0)
            printf("return value corrupted!,pw_name=%s\n",ptr->pw_name);
    }
}
```
上面就打到了不断重入的效果，搞笑的...<br>
来看看alarm与time的执行效率对比
```c
#include "../include/apue.h"
#include <signal.h>

long long count = 0;
static volatile int flag = 1;

void alarm_handler(int a){
    flag = 0;
}

int main(void){
    signal(SIGALRM,alarm_handler);
    alarm(5);
    flag = 1;
    while(flag){
        count++;
    }
    printf("%lld\n",count);
    return 0;
}
```
time
```c
#include <stdio.h>
#include <time.h>

int main(void){
    long long count = 0;
    time_t t;
    t = time(NULL)+5; // 延迟5秒
    while(time(NULL)<t){
        count++;
    }
    printf("%lld\n",count);
    return 0;
}
```
执行效率相差1000多倍！！！。<br>
volatile关键字，表示这个变量是随时变化的，所以告诉编译器不用优化。<br>
### 流量控制 是Linux内核提供的流量限速、整形和策略控制机制
播放音乐和电影的时候都要 **按照播放速率读取文件**，而不能像cat命令一样，直接将交给它的文件用最快的速度读取出来，否则你听到的音乐就转瞬即逝了。<br>
流量控制是什么，为什么，怎么用？
流量控制由qdisc、fitler和class三部分组成
* qdisc通过队列将数据包缓存起来，用来控制网络首发速度
* class用来表示控制策略
* filter用来将数据包划分到具体的控制策略中<br>
在这里我只是简单的介绍一下最简单的音乐速率控制，内核对流量的速率控制。<br>
先来看一下最简单的🌰
```c
#include "../include/apue.h"
#include <signal.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>

#include <sys/types.h>
#include <sys/stat.h>

#define BUFSIZE 10
#define MAXTOKEN 1024 // 令牌桶，令牌上限 保持速度的保证

// static volatile int loop = 0;
static volatile int token = 0; // 积攒令牌数量
static void alarm_handler(int a){
    alarm(1); // 其实这里只是计时了1秒 每次实现自调用
    if(token < MAXTOKEN){
        token++; // 每秒钟增加令牌 积攒后按速度播放
    }
}

int main(int argc,char **argv){
    int fd = -1;
    char buf[BUFSIZE] = "";
    ssize_t readsize = -1;
    ssize_t writesize = -1;
    size_t off = 0;

    if(argc < 2){
        fprintf(stderr,"Usage %s <filepath>\n",argv[0]);
        return 1;
    }
    do{ // 这里要open文件，一直要open文件
        fd = open(argv[1],O_RDONLY);
        if(fd<0){
            if(EINTR != errno){
                err_sys("open()");
            }
        }
    }while(fd<0);

    // loop = 1; // 设置开始循环♻️
    signal(SIGALRM,alarm_handler);// 捕捉信号
    alarm(1); //    发送信号 这是启动signal的alarm
    while(1){
        // while loop; 忙等
        // 非忙等
        while(token <=0){ // 如果令牌数量不足则等待添加令牌
            pause(); // 因为添加令牌是通过信号实现的，所以可以使用pause实现非忙等
            // 使调用进程挂起直到捕捉到一个信号
            // SIGALRM打算pause函数，然后继续执行，每次token都会被加一减一
        }
        token--; // 每次读取BUFSIZE个字节的数据就扣减令牌
        printf("token : %d\n",token);
        while((readsize = read(fd,buf,BUFSIZE))<0){ // 开始读 每秒读10个
            if(readsize < 0){
                if(EINTR == errno){
                    continue;
                }
                err_sys("error read()");
                goto e_read;
            }
        }
        if(!readsize){ // 读到了数据
            break;
        }
        off = 0;//  偏移量设为0
        do{
            writesize = write(1,buf+off,readsize); // 开始写到控制台
            off+=writesize;
            readsize-=writesize; // 全写完
        }while(readsize>0);
    }
    close(fd);
    return 0;

    e_read:
        close(fd);
}
```
注释解析得非常详细，这就是令牌桶的实现，每次令牌桶用完就归还，就像token--一样，要是令牌桶没了，就等着。这就是令牌桶的工作原理。<br>
**令牌桶三要素:令牌、令牌上限(代码中的MAXSIZE)、流量速率(CPS)(代码中的BUFSIZE)**<br>
设计令牌上限是为了防止令牌桶溢出，通常没有必要让令牌无限制的上涨。<br>
### getitimer和setitimer函数
```c
getitimer,setitimer - get or set value of an interval timer

#include <sys/time.h>

int getitimer(int which,struct itimerval *curr_value);
int setitimer(int which,const struct itimerval *new_value,struct itimerval *old_value);
// set函数可以替代alarm
```
set函数有两点好
* 精度高，微秒为计时⌛️单位
* 从it_interval赋给it_value是采用原子操作的<br>
setitimer直接可以构成一个类似alarm链的执行结构。也就是说，当it_value被递减为0时会发送一个信号给当前进程。并且自动将it_interval的值赋给it_value使计时从新开始。<br>
which使用不同时间，发送不同信号📶<br>

which可选宏值|对应信号
|--|--|
ITIMER_PROF|SIGPROF
ITIMER_REAL|SIGALRM
ITIMER_VIRTUAL|SIGVTALRM

* new_value: 新的定时器周期
* old_value: 由该函数回填以前设定的定时器周期，不需要保存可以设置为NULL<br>

结构
```c
struct itimerval{
    struct timeval it_interval; // next value
    struct timeval it_value; // current value
};

struct timeval{
    time_t tv_sec; // seconds
    suseconds_t tv_usec; // microseconds
}
```
递减的是it_value的值，当it_value被递减为0的时候，将it_interval的值原子化的赋给it_value。tv_sec表示以秒为单位；tv_usec表示以微秒为单位。<br>

### 信号集 一种能表示一组信号的数据类型，一般都是用在批量设置信号掩码时使用
使用sigset_t类型表示，有一组函数可以操作它。<br>
```c
sigemptyset, sigfillset, sigaddset, sigdelset, sigismember - POSIX signal set operations

#include <signal.h>

int sigemptyset(sigset_t *set);

int sigfillset(sigset_t *set);

int sigaddset(sigset_t *set, int signum);

int sigdelset(sigset_t *set, int signum);

int sigismember(const sigset_t *set, int signum);
```
就是对信号集中的信号进行增删查改。

### sigprocmask 人为干扰信号mask位图
```c
sigrocmask - examine and change blocked signals
#include <signal.h>

int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);
```
padding图是无法人为干扰的，能干扰还得了！<br>
不能保证信号什么时候到来，但是这个函数目的就是为了决定什么时候响应信号<br>
* how 指定如何干扰mask位图，可以使用下表中三个宏中的任何一个来指定<br>

宏|含义
|--|:--|
SIG_BLOCK|将当前进程的信号屏蔽字和set信号集中的信号全部屏蔽，也就是它们的mask位设置为0
SIG_UNBLOCK|将set信号集中与当前信号屏蔽字重叠的信号解除屏蔽，也就是将它们的mask位设置为1
SIG_SETMASK|将set信号集中的信号mask位设置为0，其他信号全部恢复为1

set: 需要被干扰mask位图的信号集<br>
oldset: 由该函数回填之前被打扰的信号集<br>
使用这个函数，修改一下mask位图，尝试重写打印✨和❕的程序<br>
每次打印5个✨，然后停止，收到SIGINT信号不会立即响应，而是等待本行打印结束后再响应，并且在收到信号之后再打印下一行
```c
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>

static void int_handler(int s){
    write(1,"!",1);
}

int main(){
    sigset_t set,oset,saveset;
    int i,j;

    signal(SIGINT,int_handler); 
    // 这里要是改为实时信号，那么收到多少个信号就打印多少行星号
    printf("signal_print\n");
    sigemptyset(&set); // 置空信号
    sigaddset(&set,SIGINT); // 赋值信号

    sigprocmask(SIG_UNBLOCK,&set,&saveset); // 解除屏蔽 mask设为1 这里保证mask值一定为1

    sigprocmask(SIG_BLOCK,&set,&oset); // 屏蔽信号 mask设为0

    for(j = 0;j<10000;j++){
        for(i = 0;i<5;i++){
            write(1,"*",1);
            sleep(1); // 这里实现每一秒打印一次
        }
        write(1,"\n",1);
        sigsuspend(&oset); //   等待被信号打断，再重新屏蔽信号 这里的mask置为1
    }

    sigprocmask(SIG_SETMASK,&saveset,NULL); // mask恢复0，其他信号恢复为1
    exit(0);
}
```
打印每行✳️之前先屏蔽信号，当打印完成之后再恢复信号，然后等待被信号打断，再重新屏蔽信号，打印信号，但是为什么按下Ctrl+C就可以打印下一行还有❕，因为打印✳️之前我们屏蔽了mask位，就是设为0，收到信号时，与padding求&，计算得出0，所以不响应信号。当✳️打印完了，就pause，suspend这里有个pause等待被信号打断，所以打印一个❕继续打印✳️，这就是sigsuspend函数的原子化功劳。<br>

### sigpending 获取当前收到但是没有响应的信号集
系统调用，所以当它从内核中返回的时候需要对信号位图做&操作，相应的信号已经被处理了，所以当它返回用户态的时候，它带回来的结果可能不准确了。除非调用它之前先把所有信号都block,`sigprocmask(SIG_BLOCK,&set,&oset); `像这样，然后再调用它，返回的结果才准确。emmm，开发中没啥用途<br>
```c
sigpending - examine pending signals
#include <signal.h>

int sigpending(sigset_t *set);
```
###  sigaction 用来替换signal 多使用这个，丢弃signal
```c
sigaction - examine and change a signal action

#include <signal.h>

int sigaction(int signum,const struct sigaction *act, struct sigaction *oldact);
```
signum: 要设定信号处理函数的信号<br>
act: 对信号处理函数的设定<br>
oldact: 由函数回填之前的信号处理函数设定，备份用，如果不需要可以填NULL<br>
```c
struct sigaction{
    // 前两个是信号处理函数，二选一，在某些平台伤是一个共用体
    void (*sa_handler)(int); // 为了兼容signal函数
    void (*sa_sigaction)(int,siginnfo_t*,void *);
    // 第二个参数可以获得信号的来源和属性
    sigset_t sa_mask; // 信号集位图，指定要处理的信号集，并且信号集中的任何一个信号被触发时，信号集中的其他成员同时会被block，避免像signal的信号处理函数一样当多个信号同时到来时发生重入
    int sa_flags; // 特殊要求，如果使用三参的信号处理函数，需要指定为SA_SIGINFO
    void (*sa_restorer)(void); // 基本被废弃了
};
```
其实就是要让信号重入的时候有个好的处理，排队处理。其他成员被block，防止重入。也是一个原子操作了。<br>
一般一个参数的信号处理函数和三个参数的信号处理函数，使用哪个都行，一般一个参数就够用了。<br>
给个sigaction的🌰，怼一下不靠谱的signal<br>
```c
#include "../include/apue.h"
#include <fcntl.h>
#include <string.h>
#include <syslog.h>

#define Fname "/tmp/out"

static FILE *fp;

static int Daemonize(void){
    pid_t pid;
    int fd;

    pid = fork();
    if(pid<0){
       // err_sys("fork error");
        return -1;
    }
    if(pid>0)
        exit(0);
    
    fd = open("/dev/null",O_RDWR);
    if(fd<0)
        return -2;
    
    dup2(fd,0);
    dup2(fd,1);
    dup2(fd,2);

    if(fd > 2)
        close(fd);
    
    setsid();
    chdir("/");
    umask(0); // 设置文件模式创建掩码
    return 0;
}

static void daemon_exit(int s){
    fclose(fp);
    closelog();
    syslog(LOG_INFO,"daemonixe exited");
    exit(0);
}

int main(){
    int i;
    struct sigaction sa;

    /***
     * 如果使用 signal(2) 函数则是这样注册信号处理函数
     * signal(SIGINT,daemon_exit);
     * signal(SIGTERM,daemon_exit);
     * signal(SIGQUIT,daemon_exit);  ***/
    // 改用sigaction
    sa.sa_handler = daemon_exit;
    sigemptyset(&sa.sa_mask);
    sigaddset(&sa.sa_mask,SIGQUIT);// 信号集加入了三个不同的信号
    sigaddset(&sa.sa_mask,SIGTERM);
    sigaddset(&sa.sa_mask,SIGINT);
    // 任意信号想要杀死进程，把资源释放掉再结束即可
    sa.sa_flags = 0;
    sigaction(SIGINT,&sa,NULL); // 不需要区分那个信号，只要信号来了就可以傻死进程，然后释放资源
    sigaction(SIGTERM,&sa,NULL);
    sigaction(SIGQUIT,&sa,NULL);

    openlog("mydaemon",LOG_PID,LOG_DAEMON);

    if(Daemonize()){
        syslog(LOG_ERR,"daemonize failed");
        exit(1);
    }else{
        syslog(LOG_INFO,"daemonize succssed");
    }
    fp = fopen(Fname,"w");
    if(fp == NULL){
        syslog(LOG_ERR,"fopen():%s",strerror(errno));
        exit(1);
    }

    for(i = 0;;i++){
        fprintf(fp,"%d\n",i); // 每秒钟写一个序列
        fflush(fp);
        syslog(LOG_DEBUG,"%d was printed",i);
        sleep(1);
    }
    exit(0);
}
```
当多个信号同时到来的时候，一定会发生内存泄漏。因为signal函数在一个信号到来的时候，不会把其他注册了同一个信号处理函数的信号屏蔽掉，这就是要怼signal的地方，这样信号处理函数会发生重入咯。<br>
所以使用sigaction还是比较安全的，因为它屏蔽掉了其他信号。<br>
### setjmp和sigsetjmp 信号处理函数不能使用跨函数长跳转！但是sigsetjmp来打救你
```c
sigsetjmp - save stack context for nonlocl goto

#include <setjmp.h>

int sigsetjmp(sigjmp_buf env, int savesigs);
```
如果savesigs为真，跳转的时候保存信号掩码，为假就不保存信号掩码。<br>
```c
#include <stdio.h>
#include <setjmp.h>
#include <signal.h>
#include <unistd.h>

static sigjmp_buf env;

static void fun(void){
    long long i = 0;
    sigsetjmp(env,1); // jmp的位置
    printf("before %s\n",__FUNCTION__);
    for(i = 0;i<1000000000;i++);
    printf("end%s\n",__FUNCTION__);
}

static void a_handler(int s){
    printf("before %s\n",__FUNCTION__);
    siglongjmp(env,1); // 这里恢复了信号
    printf("end%s\n",__FUNCTION__);
}

int main(void){
    long long i=0;
    struct sigaction sa;
    // 改用sigaction
    sa.sa_handler = a_handler;
    sigemptyset(&sa.sa_mask);
    sigaddset(&sa.sa_mask,SIGINT);
    // 任意信号想要杀死进程，把资源释放掉再结束即可
    sa.sa_flags = 0;
    sigaction(SIGINT,&sa,NULL);
    fun();

for(i = 0;;i++){
    printf("%lld\n",i);
    pause(); // 等待信号打断等待 
}
}
```
### abort 发送一个SIGABRT，终止+产生coredump文件 自杀的时候用
```c
abort - cause abnormal process termination

#include <stdlib.h>
void abort(void);
```
避免缺陷扩散，自杀的时候使用

### system (自建系统命令)

### select
```c
select - synchronous I/O multiplexing

#include <sys/select.h>

int select(int nfds,fd_set *readfds,fd_set *writefds,fd_set *exceptfds,struct timeval *timeout);
```
安全的定时器🔐。看🌰
```c
#include "../include/apue.h"
#include <sys/select.h>

int main(void){
    int i = 0;
    struct timeval timeout;

    for(i = 0;i<5;i++){
        timeout.tv_sec = 1;
        timeout.tv_usec = 0;
// 定时器只需要给定时间就可可以了
        if(select(0,0,0,0,&timeout)<0){
            err_sys("select()");
            exit(1);
        }
        printf("hehe\n");// 不写\n就会全缓冲
        // 写了\n就是行缓冲
    }
    return 0;
}
```
一个定时器的功能。select作用还是蛮大的，在第14章会持续讲。<br>
### sigsuspend 原子操作，等待信号打断
```c
sigsuspend - wait for a signal

#include <signal.h>

int sigsuspend(const sigset_t *mask);
```
🌰上面sigrocmask已经展示了。<br>
### 实时信号 32-64
实时信号按队列来处理，先到先响应。信号是否会排队或丢失，取决于使用那种信号，而且实时信号不采用位图来实现，而是采用链式结构来实现。