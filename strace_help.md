strace in c tutorial
===================

So you are wondering how to do an strace like program in c ?
Well I can help you !

First of all for this tutorial you will need :
+ Concentration
+ Determination
+ Some research

If you have all of that welcome aboard !

Getting started
==============

The goal is to create a program able to print all *syscall* of a given *binary* or *pid* like this :
```
./strace [-s] [-p <pid>|<command>]
```
with that in mind we need a parser for arg, since you are in tech 2 i'll asume that you know how to do it.
When this is done we can go deeper !

Binary or pid ?
==============

If we pass a binary in args we should excute it with [fork()](http://manpagesfr.free.fr/man/man2/fork.2.html) and [exec family function](http://manpagesfr.free.fr/man/man3/exec.3.html), but is it's an pid that we recive we don't have to execute it beacause he is aleready in execution.
Here is a sample code using fork() function:

```C
int main (int ac, const char **av)
{
    // I don't take care of error in this code
    pid_t pid;
    if (ac < 2)
        return (84);
    pid = fork();
    if (pid == 0) { // if pid == 0 this is the child
        execvp(av[1], av + 1); // av[1] = name of the command && av + 1 args after without the name of the binary
    }
    else { // this is the parent

    }
    return (0);
}
```
The pid is going to be really important, he gonna help us *trace* the program with the help of [ptrace()](http://manpagesfr.free.fr/man/man2/ptrace.2.html) !
For the pid you don't need to excute it you just need to trace it with the pid passed as parameter of your main function.

Ptrace
======

When you become familiar with exec and fork it's time to use [ptrace()](http://manpagesfr.free.fr/man/man2/ptrace.2.html) !
Some of its flag are going to be useful for us since we can't use *PTRACE_SYSCALL*.
here is a list :
1. PTRACE_TRACEME
2. PTRACE_GETREGS
3. PTRACE_PEEKTEXT
4. PTRACE_SINGLESTEP

*PTRACE_TRACEME* is going to help us trace our child like a gps on a child phone to know where he is.
*PTRACE_GETREGS* get a register structure used to know which syscall where on and argument passed in.
*PTRACE_PEEKTEXT* is for analysing the adress of the instruction currently stopped on, with that return value we can know if the current insctruction is a syscall or not. It's like checking were the child phone is located on the map and see is is at the adress 35 street valley.
*PTRACE_SINGLESETP* used to stop at each instruction of the program. This is going to send signal to the child to stop him at each instruction.

we that in mind I know that you can see were we are going !

The child
========

Reusing the code from the above fork sample code we now have to trace our child with...*PTRACE_TRACEME* :

```C
int main (int ac, const char **av)
{
    pid_t pid;
    if (ac < 2)
        return (84);
    pid = fork();
    if (pid == 0) {
        ptrace(PTRACE_TRACEME, NULL, NULL, NULL); // we place a tracer on the child
        execvp(av[1], av + 1);
    }
    else {
        //analyse child here
    }
    return (0);
}
```

this is all we have to do for the child !

The Parent
==========

Let's get right into it guy's !
Now that we have a tracer on th child it's time to use it.
With what we know about ptrace it's almost easy as making pasta !
First *PTRACE_GETREGS* is going to take a pointer to a *struct user_regs_struct* and store the register of the syscall in it.
once we have the register we need to check if the instruction is a syscall using *PTRACE_PEEKTEXT* (i'll talk about it's return value later)
and at the end we need to stop the program at the next instructions using *PTRACE_SIGNLESTEP*
With the return value of *PTRACE_PEEKTEXT* we can know if the instruction is a syscall the two different value are 0x80CD and 0x50F.
If it is a syscall we have to print it using the registers saved previously.

here's the code (not tested) :

```C
int main (int ac, const char **av)
{
    //again no error management
    pid_t pid;
    struct user_regs_strcut regs;
    unsinged short type = 0;
    if (ac < 2)
        return (84);
    pid = fork();
    if (pid == 0) {
        ptrace(PTRACE_TRACEME, NULL, NULL, NULL);
        execvp(av[1], av + 1);
    }
    else {
        ptrace(PTRACE_GETREGS, pid, NULL, &regs);
        type = ptrace(PTRACE_PEEKTEXT, pid, regs.rip/* this what is used to know if it's a syscall*/, NULL);
        ptrace(PTRACE_SINGLESTEP, pid, NULL, NULL);
        if (type == 0x80CD ||type == 0x50F)
            //display syscall
    }
    return (0);
}
```
But wait this work with only one line ! why ? beacause we need to loop for each instruction in the child program.
To do this you will need [waitpid()](http://manpagesfr.free.fr/man/man2/wait.2.html) and some macros.


```C
int main (int ac, const char **av)
{
    //again no error management
    pid_t pid;
    struct user_regs_strcut regs;
    int status = 0;
    unsinged short type = 0;
    if (ac < 2)
        return (84);
    pid = fork();
    if (pid == 0) {
        ptrace(PTRACE_TRACEME, NULL, NULL, NULL);
        execvp(av[1], av + 1);
    }
    else {
        waitpid(pid, &status, 0);
        // this loop check if the child still is stopped if it is the case we coninue to loop all instructions
        while (WIFSTOPPED(status) && (WSTOPSIG(status) == SIGTRAP|| WSTOPSIG(status) == SIGSTOP)) {
            ptrace(PTRACE_GETREGS, pid, NULL, &regs);
            type = ptrace(PTRACE_PEEKTEXT, pid, regs.rip/* this what is used to know if it's a syscall*/, NULL);
            ptrace(PTRACE_SINGLESTEP, pid, NULL, NULL);
            waitpid(pid, &status, 0);
            if (type == 0x80CD ||type == 0x50F)
                //display syscall
        }
    }
    return (0);
}
```

when it's done WP you are tracing all the syscall of the program the last thing to do is to display them...

Displaying syscall
=================

Ok for this part i'm gonna do everything since it's easy, in the last part we got the register using *PTRACE_GETREGS*.
We are going to use it to display all syscall to write, but first I need to learn you thing about this structure field :

*struct user_regs_struct regs filed that we gonna use*

| syscall id   | return value | arg 1        | arg 2        | arg 3        | arg 4        | arg 5        |
| :----------: | :----------: | :----------: | :----------: | :----------: | :----------: | :----------: |
| rax          | rdi          | rsi          | rdx          | r8           | r9           | r10          |

We that in mind let's display the call of write :

```C
int main (int ac, const char **av)
{
    //again no error management
    pid_t pid;
    struct user_regs_strcut regs;
    int status = 0;
    unsinged short type = 0;
    if (ac < 2)
        return (84);
    pid = fork();
    if (pid == 0) {
        ptrace(PTRACE_TRACEME, NULL, NULL, NULL);
        execvp(av[1], av + 1);
    }
    else {
        waitpid(pid, &status, 0);
        // this loop check if the child still is stopped if it is the case we coninue to loop all instructions
        while (WIFSTOPPED(status) && (WSTOPSIG(status) == SIGTRAP|| WSTOPSIG(status) == SIGSTOP)) {
            ptrace(PTRACE_GETREGS, pid, NULL, &regs);
            type = ptrace(PTRACE_PEEKTEXT, pid, regs.rip/* this what is used to know if it's a syscall*/, NULL);
            ptrace(PTRACE_SINGLESTEP, pid, NULL, NULL);
            waitpid(pid, &status, 0);
            if (type == 0x80CD ||type == 0x50F)
                if (regs.rax == 1) // write id is 1
                    printf("write (0x%llx, 0x%llx, 0x%llx) = 0x%llx", regs.rsi, regs.r8, regs.rdi); // 3 argument for write
        }
    }
    return (0);
}
```

Done ! now we trace all syscall to write and print it to stdout all you need to do is to find a way to generalyse this with all syscall.
[Here](https://filippo.io/linux-syscall-table/) is a list of all possible syscall with their id and arguments Godd Luck !

Out !