# 1. Pipes, FIFO, Message Queues
## 1.1 What is IPC?
IPC (InterProcess communication) is general term, with 3 function:
* Communication: exchanging data between processes
* Synchronization: synchronizing the actions of processes or threads
* Signals: can be used as a synchronization technique in certain circumstances

1. Communication facilities:

__Data-transfer__: In order to communicate, one process writes data to the IPC facility, and another process reads the data. One transfer from user memory to kernel memory during writing, and another transfer from kernel memory to user memory during reading. Read is __constructive__\
  [![](http://10.0.0.220:9090/uploads/images/gallery/2026-09/scaled-1680-/image-1789092269847.png)](http://10.0.0.220:9090/uploads/images/gallery/2026-09/image-1789092269847.png)

__Share-memory__: exchange information by placing it in a region of memory that is shared between the processes. A process can make data available to other processes by placing it in the shared memory region

With __Data-Transfer__, we have subcategories:
  * Byte stream: The data exchanged via pipes, FIFOs, and datagram sockets **is an undelimited byte stream**. Each read operation may read an arbitrary number of bytes from the IPC facility, regardless of the size of blocks written by the writer.
  * The data exchanged via System V message queues, POSIX message queues, and datagram sockets takes the form of delimited messages
  * Pseudoterminals (specical situations): emulator Terminal Device

With __Shared memory__:
  * shared memory provides fast communication (because dont require system call or exchange by kernel)
  * Data placed in shared memory is visible to all of the processes that share that memory

[![](http://10.0.0.220:9090/uploads/images/gallery/2026-09/scaled-1680-/image-1789093842370.png)](http://10.0.0.220:9090/uploads/images/gallery/2026-09/image-1789093842370.png)

## 1.2 Pipe
Ex: ``` $ ls | wc -l ```\
In order to execute the above command, the shell creates two processes, executing ```ls``` and ```wc``` (done using ```fork()``` and ```exec()```)\
child will do ```ls``` and has its standard output (stdout)\
```|``` is a pipe which connect stdout of ```ls``` with stdin of ```wc -l```\
another child is created will do ```wc -l``` to count the line from output of ```ls```\
[![](http://10.0.0.220:9090/uploads/images/gallery/2026-09/scaled-1680-/image-1789095379681.png)](http://10.0.0.220:9090/uploads/images/gallery/2026-09/image-1789095379681.png)

The example code equal ```ls | wc -l```:
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/wait.h>
int main() {
    int pipefd[2];
    pid_t pid1, pid2;

    if (pipe(pipefd) == -1) {     /*Create pipe*/
        perror("Failure on create pipe");
        exit(1);
    }
    pid1 = fork();
    if (pid1 == 0) {
        dup2(pipefd[1], STDOUT_FILENO); /* Connect stdout of this process to the write of pipe */

        /*Copied success, close head and tail of pipe*/
        close(pipefd[0]);
        close(pipefd[1]);

        execlp("ls", "ls", NULL); /* Overwrite by execlp */
        perror("exec fail");
        exit(1);
    }
    pid2 = fork();
    if (pid2 == 0) {
        dup2(pipefd[0], STDIN_FILENO); /* Connect the read end of pipe to stdin of this process*/

        /*Copied success, close head and tail of pipe*/
        close(pipefd[0]);
        close(pipefd[1]);

        execlp("wc", "wc", "-l", NULL); /* Overwrite by execlp */
        perror("exec fail");
        exit(1);
    }
    close(pipefd[0]);
    close(pipefd[1]);

    waitpid(pid1, NULL, 0);
    waitpid(pid2, NULL, 0);

    return 0;
}
```
Overview feature of __PIPE__:\
__A pipe is a byte stream__:No limit size of the process reading, data pass through pipe sequentially—bytes

__Reading from a pipe__: Read from a pipe that is currently block until at least one byte has been written to the pipe. If the wrtie end of file of a pipe is closed, then __reading__ process from a pipe will see __end-of-file__ (read() return 0)

__Pipes are unidirectional__: One end of pipe using for writing, and the other used for reading

__Writes of up to PIPE_BUF bytes are guaranteed to be atomic__:\
Example about PIPE_BUF:
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/wait.h>
int main() {
    int pipefd[2];
    pid_t pid1, pid2;

    if (pipe(pipefd) == -1) {     /*Create pipe*/
        perror("Failure on create pipe");
        exit(1);
    }
    pid1 = fork();
    if (pid1 == 0) {
        dup2(pipefd[1], STDOUT_FILENO); /* Connect stdout of this process to the write of pipe */
        /*Copied success, close head and tail of pipe*/
        close(pipefd[1]);
        
        char *msg = "Dong1\nDong2\nDong3\n";
        ssize_t n = write(STDOUT_FILENO, msg, strlen(msg));
        fprintf(stderr, "[Write] Wrote %ld bytes into pipe\n",n);
        
        exit(0);
    }
    pid2 = fork();
    if (pid2 == 0) {
        close(pipefd[1]);
        dup2(pipefd[0], STDIN_FILENO); /* Connect the read end of pipe to stdin of this process*/
        /*Copied success, close head and tail of pipe*/
        close(pipefd[0]);

        char buffer[8];   //PIPE_BUF
        ssize_t bytes_read;
        while ((bytes_read = read(STDIN_FILENO, buffer, sizeof(buffer) - 1)) > 0) {
            buffer[bytes_read] = '\0';
            printf("[Read] Received %ld bytes: \"%s\"\n", bytes_read, buffer);
        }
        exit(0);
    }
    close(pipefd[0]);
    close(pipefd[1]);

    waitpid(pid1, NULL, 0);
    waitpid(pid2, NULL, 0);

    return 0;
}
```
Result: 
```
[Write] Wrote 18 bytes into pipe
[Read] Received 7 bytes: "Dong1
D"
[Read] Received 7 bytes: "ong2
Do"
[Read] Received 4 bytes: "ng3
"
```
--> It means that if data larger than PIPE_BUF bytes (__writer__), kernel tranfers data in multiple smaller pieces

But, With the theory: 
>However, if there are multiple writer processes, then
writes of large blocks __may be__ broken into segments of arbitrary size (which may be
smaller than PIPE_BUF bytes) and interleaved with writes by other processes

__May be theory is wrong__ . (Addition: I'm wrong. Buffer size is only __8__ < __PIPE_BUF (4096 bytes)__ --> still remain __atomic__)\
Example (multiple writers):
```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/wait.h>
int main() {
    int pipefd[2];
    pid_t pid1, pid2, pid3;

    if (pipe(pipefd) == -1) {     /*Create pipe*/
        perror("Failure on create pipe");
        exit(1);
    }
    pid1 = fork();
    if (pid1 == 0) {
        dup2(pipefd[1], STDOUT_FILENO); /* Connect stdout of this process to the write of pipe */
        /*Copied success, close head and tail of pipe*/
        close(pipefd[1]);
        
        char *msg = "Dong1\nDong2\nDong3\n";
        ssize_t n = write(STDOUT_FILENO, msg, strlen(msg));
        fprintf(stderr, "[Write1] Wrote %ld bytes into pipe\n",n);
        
        exit(0);
    }
    pid3 = fork();
    if (pid3 == 0) {
        dup2(pipefd[1], STDOUT_FILENO);
        close(pipefd[1]);

        char *msg = "Dong4\nDong5\nDong6\n";
        ssize_t n = write(STDOUT_FILENO, msg, strlen(msg));
        fprintf(stderr, "[Write2] Wrote %ld bytes into pipe\n",n);
        
        exit(0);

    }
    pid2 = fork();
    if (pid2 == 0) {
        close(pipefd[1]);
        dup2(pipefd[0], STDIN_FILENO); /* Connect the read end of pipe to stdin of this process*/
        /*Copied success, close head and tail of pipe*/
        close(pipefd[0]);

        char buffer[8];   //PIPE_BUF
        ssize_t bytes_read;
        while ((bytes_read = read(STDIN_FILENO, buffer, sizeof(buffer) - 1)) > 0) {
            buffer[bytes_read] = '\0';
            printf("[Read] Received %ld bytes: \"%s\"\n", bytes_read, buffer);
        }
        exit(0);
    }
    close(pipefd[0]);
    close(pipefd[1]);

    waitpid(pid1, NULL, 0);
    waitpid(pid2, NULL, 0);

    return 0;
}
```
Result: 
```
[Write1] Wrote 18 bytes into pipe
[Write2] Wrote 18 bytes into pipe
[Read] Received 7 bytes: "Dong1
D"
[Read] Received 7 bytes: "ong2
Do"
[Read] Received 7 bytes: "ng3
Don"
[Read] Received 7 bytes: "g4
Dong"
[Read] Received 7 bytes: "5
Dong6"
[Read] Received 1 bytes: "
"
```
still __transfers as much data as possible__, not broken __into segments of arbitrary size__

__Pipes have a limited capacity__:
```c
int fd[2];
    pipe(fd);
    char c = 'c';
    int count = 0;
    while (1) {
        write(fd[1], &c, 1);
        printf("%d byte\n", ++count);
    }
```
Result: max of pipe size is 65536 bytes (16KB)
### What happen if I write the data bigger than MAX Capacity Pipe Size?
Follow the theory, it must write 64kb (65535 bytes) for each time. But see the example:
```c
#define BLOCK_SIZE (PIPE_BUF * 30)  /* 122880 bytes*/

void writer(int wfd) {
    char *buf = malloc(BLOCK_SIZE); 
    memset(buf, 'A', BLOCK_SIZE);
    buf[BLOCK_SIZE - 1] = '\n';

    ssize_t total = 0;
    while (total < BLOCK_SIZE) {
        ssize_t n = write(wfd, buf + total, BLOCK_SIZE - total);
        if (n <= 0) { perror("write"); exit(1); }
        total += n;
        fprintf(stderr, "[Writer] wrote %ld bytes (total %ld)\n", n, total);
    }
    free(buf);
}

int main() {
    int pipefd[2];
    if (pipe(pipefd) == -1) { perror("pipe"); exit(1); }

    pid_t w1 = fork();
    if (w1 == 0) {
        close(pipefd[0]);
        writer(pipefd[1]);
        close(pipefd[1]);
        exit(0);
    }

    close(pipefd[1]);
    char buf[4096];
    ssize_t n;
    long total = 0;
    while ((n = read(pipefd[0], buf, sizeof(buf))) > 0) {
        total += n;
        fprintf(stderr, "[Reader] read %ld bytes (total %ld), \n",n, total);
    }
    close(pipefd[0]);
    waitpid(w1, NULL, 0);
    return 0;
}
```
Result:
```
[Writer A] wrote 122880 bytes (total 122880)
[Reader] read 4096 bytes (total 4096), 
[Reader] read 4096 bytes (total 8192), 
[Reader] read 4096 bytes (total 12288), 
[Reader] read 4096 bytes (total 16384), 
[Reader] read 4096 bytes (total 20480), 
[Reader] read 4096 bytes (total 24576), 
[Reader] read 4096 bytes (total 28672), 
[Reader] read 4096 bytes (total 32768), 
[Reader] read 4096 bytes (total 36864), 
[Reader] read 4096 bytes (total 40960), 
[Reader] read 4096 bytes (total 45056), 
[Reader] read 4096 bytes (total 49152), 
[Reader] read 4096 bytes (total 53248), 
[Reader] read 4096 bytes (total 57344), 
[Reader] read 4096 bytes (total 61440), 
[Reader] read 4096 bytes (total 65536), 
[Reader] read 4096 bytes (total 69632), 
[Reader] read 4096 bytes (total 73728), 
[Reader] read 4096 bytes (total 77824), 
[Reader] read 4096 bytes (total 81920), 
[Reader] read 4096 bytes (total 86016), 
[Reader] read 4096 bytes (total 90112), 
[Reader] read 4096 bytes (total 94208), 
[Reader] read 4096 bytes (total 98304), 
[Reader] read 4096 bytes (total 102400), 
[Reader] read 4096 bytes (total 106496), 
[Reader] read 4096 bytes (total 110592), 
[Reader] read 4096 bytes (total 114688), 
[Reader] read 4096 bytes (total 118784), 
[Reader] read 4096 bytes (total 122880), 
```
We can see that Writer does not write 65535 bytes per time, It write all 122880 bytes (log)\
__FACT__: __write() pass all the loop into Kernel Space__
1. Linux see that Size of written data is bigger than 65535 bytes --> __non-atomic__
2. OS pour full 65535 bytes into Pipe buffer
3. Instead return User-Space and notify "Full 65535 bytes", Write() go to sleep state inside __Kernel-Space__
4. Reader will read 4096 bytes per time, then Kernel wake up Write() and pour 4096 bytes into Pipe. Loop again
5. This loop is handle inside __Kernel , not User-Space__
## 1.3 popen() pclose()
popen() = pipe() + fork() + dup2() + exec() \
pclose() = waitpid(pid1,...), waitpid(pid2, ...)

We have an example convert Code1 __use__ pipe() + fork() + dup2() + exec() + waitpid()\
to Code 2 __use__ only popen() and pclose()

This example will illustrate ```ls | wc -l``` shell command

Code1:
```c
int pipefd[2];
    pid_t pid1, pid2;

    if (pipe(pipefd) == -1) {     /*Create pipe*/
        perror("Failure on create pipe");
        exit(1);
    }
    pid1 = fork();
    if (pid1 == 0) {
        dup2(pipefd[1], STDOUT_FILENO); /* Connect stdout of this process to the write of pipe */

        /*Copied success, close head and tail of pipe*/
        close(pipefd[0]);
        close(pipefd[1]);

        execlp("ls", "ls", NULL); /* Overwrite by execlp */
        perror("exec fail");
        exit(1);
    }
    pid2 = fork();
    if (pid2 == 0) {
        dup2(pipefd[0], STDIN_FILENO); /* Connect the read end of pipe to stdin of this process*/

        /*Copied success, close head and tail of pipe*/
        close(pipefd[0]);
        close(pipefd[1]);

        execlp("wc", "wc", "-l", NULL); /* Overwrite by execlp */
        perror("exec fail");
        exit(1);
    }
    close(pipefd[0]);
    close(pipefd[1]);

    waitpid(pid1, NULL, 0);
    waitpid(pid2, NULL, 0);
```
Code2:
```c
    FILE *fp;
    char buffer[128];

    /* popen do: pipe() + fork() + dup2() + execlp("/bin/sh","-c","ls | wc -l") */
    fp = popen("ls | wc -l", "r");
    if (fp == NULL) {
        perror("popen failed");
        exit(1);
    }

    /* Get the result from FILE fp (return type of popen) */
    if (fgets(buffer, sizeof(buffer), fp) != NULL) {
        printf("%s", buffer);  
    }

    int status = pclose(fp);
    if (status == -1) {
        perror("pclose failed");
        exit(1);
    }

    return 0;
```
Compare 2 methods:
|Code 1|Code 2|
|-|-|
|pipe(fd) | Shell creates internal pipe between ```ls``` and ```wc -l```|
|1st fork() for ```ls```, 2nd fork() for ```wc```|Shell self-fork() 2 processes for ```ls``` and ```wc -l``` 
|dup2(pipefd[1], STDOUT_FILENO) in child1| Shell self-bind stdout of ```ls``` into pipe|
|dup2(pipefd[0], STDIN_FILENO) in child2 | Shell self-bind stdin of ```wc -l``` into pipe|
|close(pipefd[0]) / close(pipefd[1])| Shell self-manage|
|execlp("ls",...), execlp("wc","-l",...)| Merge in to a string  "ls \ wc -l"|
|waitpid(pid1,...), waitpid(pid2,...)|pclose(fd) to self-wait shell child process terminate|

## 1.3 Coprocesses
```
                    pipe
    A        ─────────────────>    B
(parent)     <───────────────── (Child)
                    pipe
```
```c
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>
int main() {
int to_child[2];    // A write -> B đọc (dup2 into stdin of B)
int from_child[2];  // B write -> A đọc (dup2 into stdout of B)

pipe(to_child);
pipe(from_child);

pid_t pid = fork();
if (pid == 0) {
    // Process B (coprocess)
    dup2(to_child[0], STDIN_FILENO);
    dup2(from_child[1], STDOUT_FILENO);

    close(to_child[0]); close(to_child[1]);
    close(from_child[0]); close(from_child[1]);

    execlp("some_filter", "some_filter", NULL);
    exit(1);
}

// Process A (parent)
close(to_child[0]);      // A does not read this pipe
close(from_child[1]);    // A dose not write on this pipe 

write(to_child[1], "input data\n", 11);   // Assump send data to B
char buf[128];
read(from_child[0], buf, sizeof(buf));    // B received 
}
```

__In summary:__
* Pipe or anonymous Pipe, has no name and no exist by __physical file on disk__. It's temporary buffer locate in __kernel RAM__
  * Pipe use FIFO mechanism
  * Only use with relative relationship process (parent - child; child - child (in 1 process)
## 1.4 FIFO (Named Pipe)
Overview:
* FIFO has a name within file system
* So that, FIFO can communicate between unrelated processes
* As with pipe, when all descriptors reffering  to a FIFO have been closed, any outstanding data is discarded

2 ways to create a FIFO
```
$ mkfifo [ -m mode ] pathname     /* On shell */
------------------------------------------------
#include <sys/stat.h>
int mkfifo(const char* pathname, mode_t mode);       /* Function */
                 /* Return 0 on success, -1 on error */
```
_mode_ = __O_RDONLY, O_WRONLY, O_RDWR__

Synchonization mechanism:
* If process A open FIFO with Write only Mode, ```open()``` block until process B open FIFO with Read Only Mode
* Both Read and Write Mode appear, then ```open()``` is unblock and continue run

Using FIFO to design a Client-Server Application
```c
/* server.c */

#define FIFO_REQUEST  "/tmp/fifo_request"
#define FIFO_RESPONSE "/tmp/fifo_response"

int main() {
    /* Create 2 FIFO */         /* mode: permission mask */
    mkfifo(FIFO_REQUEST, 0666);  // 1st char: 0 (octal) not Decimal, 2nd char: User/Owner (create file)
    mkfifo(FIFO_RESPONSE, 0666); // 3rd char: Group (same level with Owner), Last char: Others (Any user)
                                 // 6 = 4 (read)  + 2 (write)

    printf("[Server] Wait for client connect...\n");

    while (1) {
        int req_fd = open(FIFO_REQUEST, O_RDONLY);
        if (req_fd == -1) { perror("open request"); continue; }

        char buffer[256];
        ssize_t n = read(req_fd, buffer, sizeof(buffer) - 1);
        close(req_fd);   /* close after read successfully 1 request, open in next loop*/

        if (n <= 0) continue;
        buffer[n] = '\0';
        printf("[Server] Received from client: %s\n", buffer);

        /* Convert to Upper */
        for (int i = 0; buffer[i]; i++) 
            buffer[i] = toupper(buffer[i]);

        /* Open FIFO response to send result to client */
        int resp_fd = open(FIFO_RESPONSE, O_WRONLY);
        if (resp_fd == -1) { perror("open response"); continue; }
        write(resp_fd, buffer, strlen(buffer));
        close(resp_fd);
    }
    return 0;
}
```
```c
/* client.c */
#define FIFO_REQUEST  "/tmp/fifo_request"
#define FIFO_RESPONSE "/tmp/fifo_response"

int main() {
    char msg[256];
    printf("Write message to send to server: ");
    fgets(msg, sizeof(msg), stdin);
    msg[strcspn(msg, "\n")] = '\0';   /* dont use newline char */

    /* Send request */
    int req_fd = open(FIFO_REQUEST, O_WRONLY);
    if (req_fd == -1) { perror("open request"); exit(1); }
    write(req_fd, msg, strlen(msg));
    close(req_fd);

    /* Wait và read response */
    int resp_fd = open(FIFO_RESPONSE, O_RDONLY);
    if (resp_fd == -1) { perror("open response"); exit(1); }

    char buffer[256];
    ssize_t n = read(resp_fd, buffer, sizeof(buffer) - 1);
    close(resp_fd);

    if (n > 0) {
        buffer[n] = '\0';
        printf("[Client] Server answer: %s\n", buffer);
    }
    return 0;
}
```
__NOTE__: Although I disconnect client-server but still exits file (pathname) on filesystem\
[![](http://10.0.0.220:9090/uploads/images/gallery/2026-09/scaled-1680-/image-1789456422571.png)](http://10.0.0.220:9090/uploads/images/gallery/2026-09/image-1789456422571.png)\
Although permit 0666 (all read write) but output of ```ls -l``` is ```prw-rw-r```\
p: Named Pipe (FIFO)\
Linux has protect filter, is __umask__. Reality authority is 0664, not 0666.\ 
It mean: ```Reality authority = 0666 AND NOT 0002 = 0664``` (OS will decrease your authority in __umask__)

If we want to terminate this file, use ```unlink(const char* pathname)``` or ```rm``` (remove) command \
Although file is exits but if we close fd, buffer (file descriptor) is not exits too
# 2. IPC Identifiers, Keys, Permission, Limits
## 2.1 IPC Keys and IPC Identifiers
To create or open one object about System V IPC, process have to call ```get```\
like __(msgget(), semget(), or shmget())__
1. Definition and Relationship
* __Key (key_t)__: integer value, have purpose like name of IPC object
* __Identifier__: integer value is returned after __get()__ function, to identify for IPC object is openning at the moment
* __Different__: file descriptor is attribute of each process, but IPC identifier has __system-wide attribute__. Any process know this __identifier__ can access directly into IPC object
2. Create unique key
  * Use const ```IPC_PRIVATE```: example ```shmget(IPC_PRIVATE,...)```
  * Use ```ftok()``` (_file to key_) function: ```key_t ftok(char* pathname, int proj);```
    * Return integer key on success, -1 on error

Example:
```c
int main() {
    key_t key;
    int id;
    key = ftok("/home/hoainam0508/Documents/IPC/ftok.c", 'x');
    if (key == -1)
        exit(1);
    id = msgget(key, IPC_CREAT | S_IRUSR | S_IWUSR);
    if (id == -1)
        exit(1);
    else printf("Create KEY successfully. Key = %d\n", id);
    
    return 0;
}
```
Result: ```Create KEY successfully. Key = 0```\
If I change ```'x'``` to ```'A'```, the result is: ```Create KEY successfully. Key = 1```\
It's a __table__ inside kernel. If we call ```msgget(x)```,it's first queue (Index 0)\
But we call ```msgget(A)``` after, key x haven't deleted. So key A belong Index 1

We can use ```ipcs``` to check:
```
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    
0x780660d3 0          hoainam050 600        0            0           
0x410660d3 1          hoainam050 600        0            0   
```
0x780660d3 : hex of 'x'\
0x410660d3: hex of 'A'

NOTE that: ```pathname``` in ftok() function is __regular file__, not __system file__

### Compare Key (after ftok function) vs Identifier (get function)
If I added ```printf("Key after ftok function is %d\n", key);``` into above code. The result is 
```
Key after ftok function is 2013683923
Create KEY successfully. Key = 0
```
|Characteristic|Key|Identifier|
|-|-|-|
|Meaning| External/Global Name | Internal Identifier |
|Purpose| Dependent Processes can find each other| Process can operate directly to resource|
|Understand| like a name of IPC | Represent to IPC is open |

3. Algorithm to create __identifier__ of Kernel
* Mathematical: ```identifer = index + seq * SEQ_MULTIPLIER``` (index is the position in data page of kernel, SEQ_MULTIPLIER = 32768, Seq = sequence number)
* __PURPOSE__: if server die and restart, old IPC object is deleted and new IPC is created in __same index__ position so _seq_ increase, then new __identifier__ differ old

## 2.2 IPC permission
Kernel manage Permission of each IPC object through ```ipc_perm``` component in kernel (including __uid, gid, cuid, cgid__ and bitmask __mode__)  (user, group, creator user, creator group)
* Only have Read and Write authority
* Check authority: Kernel check by __Effective User ID (EUID), Effective Group ID (EGID)__ and __Supplementary Group IDs__ of calling processes
  Note: Linux check UID/GID; but IPC check Effective UID/GID
* To delete or change attribute of IPC:
  * Read or write, use Read/Write authority
  * To delete __IPC_RMID__ or change structure __IPC_SET__, Effective UID = __Owner UID, Creator UID__. Or process must have authority: __CAP_SYS_ADMIN, CAP_IPC_OWNER__

```
struct ipc_perm {
key_t __key;      /* Key, as supplied to 'get' call */
uid_t uid;        /* Owner's user ID */
gid_t gid;        /* Owner's group ID */
uid_t cuid;       /* Creator's user ID */
gid_t cgid;       /* Creator's group ID */
unsigned short mode;   /* Permissions */
unsigned short __seq;  /* Sequence number */
};
```
## 2.3 IPC Limits
Run ```ipcs -l```
```
------ Messages Limits --------
max queues system wide = 32000
max size of message (bytes) = 8192
default max size of queue (bytes) = 16384

------ Shared Memory Limits --------
max number of segments = 4096
max seg size (kbytes) = 18014398509465599     /* 16 EB (Exabytes) = 16 * 1024 PB (Peta) = 16 * 1024^2 TB */
max total shared memory (kbytes) = 18446744073709551612
min seg size (bytes) = 1

------ Semaphore Limits --------
max number of arrays = 32000
max semaphores per array = 32000
max semaphores system wide = 1024000000
max ops per semop call = 500
semaphore max value = 32767
```
# 3. Message Queue (System V)
Overview: message queue is data exchange method belong __message__ in __data tranfer__ mechanism. The characteristics is __limited data exchange__

The different from pipes and FIFO:
* Handle through _identifier_ returned by msgget()
* Communication is __oriented__ --> reader receive whole messages, as written by writer
  * not possible to read part of message, leaving the remainder in queue, or read multiple message at a time
  * __contrast__ with pipes: provide an undifferentiated stream of bytes
* To contain data, each _message_ has integer ```type```.
* Message can be __retrieved__ from a queue in first-in first-out order or __retrieved__ by ```type```

## 3.1 Create or open message queue
```c
int main() {
    key_t key;
    int id;
    key = ftok("/home/hoainam0508/Documents/IPC/ftok.c", 'B');
    if (key == -1)
        exit(1);
    else printf("Key after ftok function is %d\n", key);
    id = msgget(key, IPC_CREAT | IPC_EXCL);
    if (id == -1) {
        printf("Can not create or open ms\n");
        exit(1);
    }
    else printf("Create KEY successfully. Key = %d\n", id);
    
    return 0;
}
```
I create Key 'x' first, then delete it by ```ipcrm -q``` command. Then create Key 'x' again & Key "A" (both normal) and Key 'B' (Exclusive flag) 
So the result of ```ipcs -q```:
```
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    
0x410660d3 1          hoainam050 600        0            0                 /* Key x */    
0x780660d3 2          hoainam050 600        0            0                 /* Key A */ 
0x420660d3 3          hoainam050 0          0            0                 /* Key B */  exclusive
```
__Why 0?__ \
0 = 000 (no authorize access for __Owner, Group and Others User__)\
--> to have authority, pass authotiry value in to ```msgget()``` function
```c
key = ftok("/home/hoainam0508/Documents/IPC/ftok.c", 'C');     /* Key C */
id = msgget(key, 0666 | IPC_CREAT | IPC_EXCL);
```
So the result:
```
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    
0x410660d3 1          hoainam050 600        0            0           
0x780660d3 2          hoainam050 600        0            0           
0x420660d3 3          hoainam050 0          0            0           
0x430660d3 4          hoainam050 666        0            0         /* Key C */ is just created
```

## 3.2 Exchange Messages
###  Send message
msgflg: __IPC_NOWAIT__ --> instead block when message queue is full, msgsnd() returns immediately with the error __EAGAIN__
Prototype:
```c
int msgsnd(int msqid, const void *msgp, size_t msgsz, int msgflg);
```
Assump we must send message:
```c
struct my_msg {
    long mtype;      /* Must be > 0 */
    char text[100];
} msg;
```
* msgid is identifier of message queue
* msgp: type of void, to allow it to be a pointer to any type of structure
* msgz : number of bytes contained in the _text_ field
* msgflg: 0 or IPC_NOWAIT

Example:
```c
struct my_msg {
    long mtype;
    char text[100];
} msg;
int main () {
    msg.mtype = 1;
    strcpy(msg.text, "Hello Queue");
    int snd = msgsnd(4, &msg, sizeof(msg) - sizeof(long), 0);
    if (snd == -1) {
        printf("Send Message unsuccess\n");
        exit(1);
    }
    else printf("Send successfully\n");
}
```
#### Alignment & Padding (Optimization hidden memory Mechanism)
__NOTE__: Each times We send the message (Example above), the used-byte increase 104 bytes 
```
hoainam0508@hoainam0508-ThinkPad-T14s-Gen-4:~$ ipcs -q
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    
0x410660d3 1          hoainam050 600        0            0           
0x780660d3 2          hoainam050 600        0            0           
0x420660d3 3          hoainam050 0          0            0           
0x430660d3 4          hoainam050 666        912          9           

hoainam0508@hoainam0508-ThinkPad-T14s-Gen-4:~$ ipcs -q      (After send message)

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    
0x410660d3 1          hoainam050 600        0            0           
0x780660d3 2          hoainam050 600        0            0           
0x420660d3 3          hoainam050 0          0            0           
0x430660d3 4          hoainam050 666        1016         10  
```
Allign follow multiples (bội số) of 8 bytes (Max size of type in struct)\
gcc see 100 bytes (__text[100]__)\
but 100 not divisible by 8\
--> add 4 bytes (104 divisible by 8) 

### Received Message
reads (and removes) a message from a message queue, and copies its contents into the buffer pointed to by _msgp_.\
Prototype:
```c
ssize_t msgrcv(int msqid, void *msgp, size_t maxmsgsz, long msgtyp, int msgflg);
                                      /* Returns number of bytes copied into mtext field, or –1 on error */
```
* maxmsgsz: The maximum space available in the _text_ field . IF no left message for removed, _msgrcv()_ return error __E2BIG__. Use MSG_NOERROR flag

We can select message to read according to _mtype_ field. Through:
* ```msgtyp```
  * msgtyp = 0: removed first message from queue & returned to calling process
  * msgtyp > 0: first message from queue whose ```msgtyp``` = ```mtype``` is removed and returned to the calling process
  * msgtyp < 0: treat _waiting message_ as a _priority queue_. The first message of the lowest ```mtype``` less than or equal to the absolute value of ```msgtyp``` is removed and returned to the calling process.
* ```msgflg```: IPC_NOWAIT | MSG_EXCEPT |  MSG_NOERROR

Example: 
```c
struct my_msg {
    long mtype;
    char text[100];
}; 
void send_message () {
    struct my_msg snd_msg;
    snd_msg.mtype = 1;
    strcpy(snd_msg.text, "Hello Queue");

    int snd = msgsnd(2, &snd_msg, sizeof(snd_msg) - sizeof(long), 0);
    if (snd == -1) {
        printf("Send Message unsuccess\n");
        exit(1);
    }
    else printf("Send successfully\n");
}
void receive_message() {
    struct my_msg rcv_msg;
    ssize_t rcv = msgrcv(2, &rcv_msg, sizeof(rcv_msg) - sizeof(long), 1, IPC_NOWAIT);
    if (rcv == -1) {
        printf("cannot receive message\n");
        exit(1);
    }
    else {
        printf("Receive/Removed %ld bytes, message: %s\n",sizeof(rcv_msg) - sizeof(long), rcv_msg.text);
    }
}
int main() {
    // send_message();
    receive_message();
}
```
No message left: 
```
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    
0x410660d3 1          hoainam050 600        0            0           
0x780660d3 2          hoainam050 600        0            0           
0x420660d3 3          hoainam050 0          0            0           
0x430660d3 4          hoainam050 666        0            0  
```
Block when attempt receive/remove message from queue --> use IPC_NOWAIT flag to return immediately error code\
Note: Queue with perms ```0``` cannot read or write anything message
### Control Message
Prototype:
```c
int msgctl(int msqid, int cmd, struct msqid_ds *buf);
```
* cmd:
  * IPC_RMID: Immediately remove the message queue __object__ and its associated msqid_ds data structure
  * IPC_STAT: copy _msqid_ds_ data structure
  * IPC_SET: update selected fields of _msqid_ds_ using value provided in the buffer pointed to by _buf_

Example: message queue id 4 after use ```msgctl``` with IPC_RMID:
```c
------ Message Queues --------                                         (Before)
key        msqid      owner      perms      used-bytes   messages    
0x410660d3 1          hoainam050 600        0            0           
0x780660d3 2          hoainam050 600        0            0           
0x420660d3 3          hoainam050 0          0            0           
0x430660d3 4          hoainam050 666        520          5   

int ctl = msgctl(4, IPC_RMID, NULL); 

------ Message Queues --------                                        (After)
key        msqid      owner      perms      used-bytes   messages    
0x410660d3 1          hoainam050 600        0            0           
0x780660d3 2          hoainam050 600        0            0           
0x420660d3 3          hoainam050 0          0            0           
```
### Message Queue Associated Data Structure & Limits
```c
struct msqid_ds {
struct ipc_perm msg_perm;        /* Ownership and permissions */
time_t            msg_stime;     /* Time of last msgsnd() */   
time_t            msg_rtime;     /* Time of last msgrcv() */
time_t            msg_ctime;     /* Time of last change */
unsigned long   __msg_cbytes;    /* Number of bytes in queue */
msgqnum_t         msg_qnum;      /* Number of messages in queue */
msglen_t          msg_qbytes;    /* Maximum bytes in queue */
pid_t             msg_lspid;     /* PID of last msgsnd() */
pid_t             msg_lrpid;     /* PID of last msgrcv() */
};
```
Limits:
```
/proc/sys/kernel$ cat msgmni      /* Number of message queue identifiers that can be created */
32000             msgget();
/proc/sys/kernel$ cat msgmax      /* Number of (text) bytes can be written is a single message */
8192              msgsnd();
/proc/sys/kernel$ cat msgmnb      /* Max number of (text) bytes can be held in msq at one time */
16384                      /* If limit is reached, msgsnd() blocks or fail with EAGAIN (use WNOWAIT flag) */
```
#### Can change MSQ limits
* temporary:
  * echo (overwrite): ```sudo sh -c "echo 65536 > /proc/sys/kernel/msgmax"```
  * use ```sysctl```: ```sudo sysctl -w kernel.msgmax=65536```
* pernament: (2 steps)
  1. Open configure file by ROOT: ```sudo nano /etc/sysctl.conf```
  2. Add this line into tail of file: ```kernel.msgmax = 65536```
# 4. Shared Memory (System V)
## 4.1 Creating or Opening a Shared Memory Segment
Prototype:
```c
int shmget(key_t key, size_t size, int shmflg);
                             /* Returns shared memory segment identifier on success, or –1 on error */
```
* size (> 0):
  * size of segment, in bytes. __Kernel allocates shared memory in multiples (bội số) of the system page size, so _size_ is effectively rounded up to the next multiple of the system page size__
  * If existing segment, size has no effect. But it must be __less than or equal__ to the size of segment
* shmflg: IPC CREAT | IPC EXCL

Default shared memory segment on my device: ```ipcs -m -p```
```
------ Shared Memory Creator/Last-op PIDs --------
shmid      owner      cpid       lpid      
6          hoainam050 3266       2677    /* MS Teams for Linux */     
7          hoainam050 3266       2677    /* MS Teams for Linux */   
41         hoainam050 41400      2677    /* VS Code */  
42         hoainam050 41400      2677    /* VS Code */    
51         hoainam050 3266       2677    /* MS Teams for Linux */   
52         hoainam050 3266       2677    /* MS Teams for Linux */   
58         hoainam050 41280      57705   /* VS code */  
62         hoainam050 1739       55692   /* chrome */
```
Example:
```c
int main() {
    key_t key;
    int id;
    key = ftok("/home/hoainam0508/Documents/IPC/ftok.c", 'C');
    if (key == -1)
        exit(1);
    else printf("Key after ftok function is %d\n", key);
    int shmid = shmget(key, 1024, 0666 | IPC_CREAT);
    if (shmid == -1) {
        printf("cannot get identifier\n");
        exit(1);
    }
    else printf("Segment is created: %d\n", shmid);
    return 0;
}
```
After create new segment
```
------ Shared Memory Segments --------
key        shmid      owner      perms      bytes      nattch     status
0x00000000 6          hoainam050 606        504000     2          dest         
0x00000000 7          hoainam050 606        504000     2          dest  
....        ...        ...       ...         ...        ...        ... 
0x00000000 62         hoainam050 600        524288     2          dest   
0x430660d3 63         hoainam050 666        1024       0  
```
## 4.2 Using shared memory 

Prototype:
```c
void *shmat(int shmid, const void *shmaddr, int shmflg);
            /* Returns address at which shared memory is attached on success, or (void *) –1 on error */
```
> attaches the __shared memory segment__ identified by _shmid_ to the calling process’s __virtual address space__

* _shmaddr_
  * NULL: segment is attached at a suitable address selected by the kernel. __It's preferred method__
  * not NULL, SHM_RND not set: segment is attached at the address specified by _shmaddr_, which must be __multiple of the system page size__
  * not NULL, SHM_RND is set: segment is mapped at the address provided in _shmaddr_, rounded down to the nearest multiple of the constant SHMLBA (_shared memory low boundary address__) --> improve CPU cache performance on some architectures

Specifying a __non-NULL__ value for _shmaddr is_ __not recommended__, because:
* reduces the portability of an application. An address valid on one UNIX implementation may be invalid on another.
* may be address (specified) in use

Value|Description
|-|-|
SHM_RDONLY|Attach segment read-only
SHM_REMAP|Replace any existing mapping at shmaddr
SHM_RND|Round shmaddr down to multiple of SHMLBA bytes

__When a process no longer needs to access a shared memory segment__
```c
int shmdt(const void *shmaddr);             /* Detach segment from its virtual address space */
                                            /* Returns 0 on success, or –1 on error */
```
NOTE THAT: __Detach is not Delete__ 
* exec() in child after fork() = detach
* terminated a process = detach

|Detach|Remove|
|-|-|
|Return pointer, dont want to see memory|Require Kernel delete memory|
Local (Calling process)| Global (Effect on resource of System-Wide)
|Memory, data are exist on RAM of kernel|Free all data, RAM is clean|

__IN SUMMARY__:
* shmat() create new segment in __page table__ of this calling process, Point to a physical memory where is allocated by kernel through ```shmget()``` function
  * virtual address may be different from processes
  * but __physical memory__ is unique (only one)
  * A write, B read immediately (need no copy to kernel like __message__) because both point to __physical memory__

