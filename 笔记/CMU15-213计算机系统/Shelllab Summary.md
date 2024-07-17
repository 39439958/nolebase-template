## Shelllab Summary

ps：对全局变量的访问要屏蔽所有信号，我没加。

### 1、eval

```c
/* 
 * eval - Evaluate the command line that the user has just typed in
 * 
 * If the user has requested a built-in command (quit, jobs, bg or fg)
 * then execute it immediately. Otherwise, fork a child process and
 * run the job in the context of the child. If the job is running in
 * the foreground, wait for it to terminate and then return.  Note:
 * each child process must have a unique process group ID so that our
 * background children don't receive SIGINT (SIGTSTP) from the kernel
 * when we type ctrl-c (ctrl-z) at the keyboard.  
*/
void eval(char *cmdline) 
{
    char *argv[MAXARGS];	
    int bg;//如果是需要在后台运行的程序，则bg=1
    pid_t pid;     		    
    sigset_t mask_child;          

    bg = parseline(cmdline, argv);

    if (!builtin_cmd(argv)) {
        sigemptyset(&mask_child);
        sigaddset(&mask_child, SIGCHLD);
        sigprocmask(SIG_BLOCK, &mask_child, NULL);//屏蔽SIGCHLD信号

        if ((pid = fork()) == 0) {
            sigprocmask(SIG_UNBLOCK, &mask_child, NULL);//解除子进程中的SIGCHILD信号屏蔽，因为子进程可能也有自己的子进程
            setpgid(0, 0);//将进程组号设为进程号,作用是将子进程变为一个独立的进程组，防止关闭它时同时把父进程也关了
            if (execvp(argv[0], argv) < 0) {
                printf("%s: Command not found\n", argv[0]);
                exit(1);
            }
        } else {
            if (!bg) {
                addjob(jobs, pid, FG, cmdline);
            } else {
                addjob(jobs, pid, BG, cmdline);
            }
            if (!bg) {
                waitfg(pid);
            } else {
                printf("[%d] (%d) %s", pid2jid(pid), pid, cmdline);
            }
            sigprocmask(SIG_UNBLOCK, &mask_child, NULL);//解除父进程对SIGHILD信号的屏蔽
        }
    }
    return;
}
```

### 2、builtin_cmd

```c
/* 
 * builtin_cmd - If the user has typed a built-in command then execute
 *    it immediately.  
 */
int builtin_cmd(char **argv) 
{
    if (!strcmp(argv[0], "quit")){
        exit(0);
    } else if (!strcmp(argv[0], "jobs")){
        listjobs(jobs);
        return 1;
    } else if (!strcmp(argv[0], "bg") || !strcmp(argv[0], "fg")) {
        do_bgfg(argv);
        return 1;
    } else if (!strcmp(argv[0], "&")) {
        return 1;
    }
    return 0;     /* not a builtin command */
}
```

### 3、do_bgfg

```c
/* 
 * do_bgfg - Execute the builtin bg and fg commands
 */
void do_bgfg(char **argv) 
{
    struct job_t* job;
    char *tmp;
    int jid;
    pid_t pid;
    sigset_t mask_child;
    
    tmp = argv[1];
    if (tmp == NULL) {
        printf("%s command requires PID or %%jobid argument\n", argv[0]);
        return;
    }
    //获取job的pid
    if (tmp[0] == '%') {
        jid = atoi(&tmp[1]);
        job = getjobjid(jobs, jid);
        if (job == NULL){  
            printf("%s: No such job\n", tmp);  
            return;  
        } else {
            pid = job->pid;
        }
    } else if (isdigit(tmp[0])) {
        pid = atoi(tmp);
        job = getjobpid(jobs, pid);
        if(job == NULL){  
            printf("(%d): No such process\n", pid);  
            return;  
        }
    } else {
        printf("%s: argument must be a PID or %%jobid\n", argv[0]);
        return;
    }
	//给pid组（pid进程及其子进程）发信号
    kill(-pid, SIGCONT);
	//处理job数据结构中的状态位，如果时前台程序，则要等他运行
    if (!strcmp("fg", argv[0])) {
        job -> state = FG;
        sigemptyset(&mask_child);
        sigaddset(&mask_child, SIGCHLD);
        sigprocmask(SIG_BLOCK, &mask_child, NULL);//屏蔽SIGCHLD信号
        waitfg(job -> pid);
        sigprocmask(SIG_UNBLOCK, &mask_child, NULL);//解除对SIGHILD信号的屏蔽
    } else {
        printf("[%d] (%d) %s", job->jid, job->pid, job->cmdline);
        job->state = BG;
    }
}

```

### 4、waitfg

```c
/* 
 * waitfg - Block until process pid is no longer the foreground process
 */
void waitfg(pid_t pid)
{
    sigset_t mask_empty;
	sigemptyset(&mask_empty);
	while (fgpid(jobs) > 0){
		sigsuspend(&mask_empty);
	}
	return;
}
```

另外一个版本的waitfg（但是比较浪费）

```c
/* 
 * waitfg - Block until process pid is no longer the foreground process
 */
void waitfg(pid_t pid)
{
    struct job_t* job;
    job = getjobpid(jobs, pid);
    if (job != NULL) {
        while (pid == fgpid(jobs)) {

        }
    }
    return;
}

```

使用这个函数的话，前面的eval和do_bgfg函数要修改，主要是在执行waitfg前需要把SIGCHLD信号屏蔽，返回后再取消屏蔽。

### 5、信号处理函数

```c
/* 
 * sigchld_handler - The kernel sends a SIGCHLD to the shell whenever
 *     a child job terminates (becomes a zombie), or stops because it
 *     received a SIGSTOP or SIGTSTP signal. The handler reaps all
 *     available zombie children, but doesn't wait for any other
 *     currently running children to terminate.  
 */
void sigchld_handler(int sig) 
{
    int oldErrno = errno;
    pid_t pid;
    int status;

    while ((pid = waitpid(-1, &status, WNOHANG | WUNTRACED)) > 0) {
        if (WIFEXITED(status)) {
            deletejob(jobs, pid);
        } else if (WIFSIGNALED(status)) {
            int jid = pid2jid(pid);
            printf("Job [%d] (%d) terminated by signal %d\n", jid, pid, WTERMSIG(status));
            deletejob(jobs, pid);

        } else if (WIFSTOPPED(status)) {
            struct job_t* job = getjobpid(jobs, pid);
            job -> state = ST;
            printf("Job [%d] (%d) stopped by signal %d\n", job->jid, job->pid, WSTOPSIG(status));
        }
    }

    errno = oldErrno;
    return;
}

/* 
 * sigint_handler - The kernel sends a SIGINT to the shell whenver the
 *    user types ctrl-c at the keyboard.  Catch it and send it along
 *    to the foreground job.  
 */
void sigint_handler(int sig) 
{
    pid_t pid = fgpid(jobs);
    if (pid == 0) {
        return;
    }
    kill(-pid, sig);
}

/*
 * sigtstp_handler - The kernel sends a SIGTSTP to the shell whenever
 *     the user types ctrl-z at the keyboard. Catch it and suspend the
 *     foreground job by sending it a SIGTSTP.  
 */
void sigtstp_handler(int sig) 
{
    pid_t pid = fgpid(jobs);
    if (pid == 0) {
        return;
    }
    kill(-pid, sig);
}

```

