---
layout: post
title: methods to avoid zombie process
date: 2016-12-6 10:59:31
category: 技术
tags: Linux Perl
excerpt: three methods to avoid zombie process……
---

## what is zombie process?

From [Wikipedia](https://en.wikipedia.org/wiki/Zombie_process):

>On Unix and Unix-like computer operating systems, a zombie process or defunct process is a process that has completed execution (via the exit system call) but still has an entry in the process table: it is a process in the "Terminated state". This occurs for child processes, where the entry is still needed to allow the parent process to read its child's exit status: once the exit status is read via the wait system call, the zombie's entry is removed from the process table and it is said to be "reaped". A child process always first becomes a zombie before being removed from the resource table. In most cases, under normal system operation zombies are immediately waited on by their parent and then reaped by the system – processes that stay zombies for a long time are generally an error and cause a resource leak.

>The term zombie process derives from the common definition of zombie — an undead person. In the term's metaphor, the child process has "died" but has not yet been "reaped". Also, unlike normal processes, the kill command has no effect on a zombie process.

## what's the difference between zombie process and orphan process?

>Zombie processes should not be confused with orphan processes: an orphan process is a process that is still executing, but whose parent has died. These do not remain as zombie processes; instead, (like all orphaned processes) they are adopted by init (process ID 1), which waits on its children. The result is that a process that is both a zombie and an orphan will be reaped automatically.

## zombie process example

```perl
use POSIX ":sys_wait_h";

my $pid = fork;
die "Unable to fork: $!." unless defined $pid;
unless ( $pid ) {
        sleep 10;
        print 'child exit'."\n";
        exit 0;
}
sleep 20;
print 'parent exit';
```

immediately,there are two normal process as below:

![](/public/img/zombie_process/1.png)

after 10s,the child process exit and become zombie process as below:

![](/public/img/zombie_process/2.png)

*in the next 10s,the child process remains 'zombie' status and parent process is still running...*

after 20s,both the process and child process disappear! 

## ways to avoid zombie process

There are normal three methods to avoid zombie process as below(Perl language examples):

### method 1

Execute two forks to make the child process become orphan process.it will be adopted by init process(pid=1),and accordingly,be "reaped" by init process.

```perl
use POSIX ":sys_wait_h";

my $pid = fork;
die "Unable to fork: $!." unless defined $pid;
unless ( $pid ) {
        #in child
        my $pid = fork;
        die "Unable to fork: $!." unless defined $pid;
        if ($pid) {
                #in parent
                exit(0);
        }
        #in child
        sleep 10;#ensure that parent exit before execute child code
        print 'child exit'."\n";
        exit 0;
}
#in parent
if ( waitpid($pid,0)!=$pid ) { #waitpid for child process
        print "waitpid error: $!\n";
}
sleep 20;
print 'parent exit';
```

immediately,there are two normal process as below(child process's parent is `init`):

![](/public/img/zombie_process/3.png)

after 10s,the child disappear as below("reaped" by init process):

![](/public/img/zombie_process/4.png)

after 20s,the parent process disappear.

### method 2

When a child exits, the parent process will receive a SIGCHLD signal to indicate that one of its children has finished executing; the parent process will typically call the wait() system call at this point. That call will provide the parent with the child’s exit status, and will cause the child to be reaped, or removed from the process table.

So,we can avoid zombie process by define a handler for SIGCHLD that calls waitpid.

```perl
use POSIX ":sys_wait_h";# for nonblocking read

sub REAPER {
        # don't change $! and $? outside handler
        local ($!, $?);
        while ( (my $pid = waitpid(-1, WNOHANG)) > 0 ) {
                if ( WIFEXITED($?) ) {
                        my $ret_code = WEXITSTATUS($?);
                        print "child process:$pid exit with code:$ret_code\n";
                }
        }
}


$SIG{CHLD} = \&REAPER;
my $pid = fork;
die "Unable to fork: $!." unless defined $pid;
unless ( $pid ) {
        #in child
        for (my $i=0;$i<10;$i++){
                sleep 1;
        }
        print 'child exit'."\n";
        exit(1);
}
#in parent
for (my $i=0;$i<20;$i++){
        sleep 1;
}
print 'parent exit';
```

```bash
child exit
child process:20669 exit with code:1
parent exit
```

immediately,there are two normal process as below:

![](/public/img/zombie_process/5.png)

after 10s,the child disappear as below("reaped" by parent process:pid=20668):

![](/public/img/zombie_process/6.png)

after 20s,the parent process disappear.

### method 3(recommended method)

Explicitly set the SIGCHLD handler to SIG_IGN.

If (as in the example above) the signal handler does nothing beyond calling waitpid then an alternative is available. Setting the SIGCHLD handler to SIG_IGN will cause zombie processes to be reaped automatically.

```perl
use POSIX ":sys_wait_h";

$SIG{CHLD}='IGNORE';

my $pid = fork;
die "Unable to fork: $!." unless defined $pid;
unless ( $pid ) {
        #in child
        sleep 10;
        print 'child exit'."\n";
        exit 0;
}
#in parent
sleep 20;
print 'parent exit';
```

immediately,there are two normal process as below:

![](/public/img/zombie_process/7.png)

after 10s,the child disappear as below("reaped" by child process itself):

![](/public/img/zombie_process/8.png)

after 20s,the parent process disappear.

Note that it is not sufficient for SIGCHLD to have a disposition that causes it to be ignored (as the default, SIG_DFL, would do): it is only by setting it to SIG_IGN that this behaviour is obtained.

**<font color="#8B0000">Compared to waitpid and two-forks methods,this method is more efficient and simple.So,it is the recommended method to avoid zombie process.</font>**

**<font color="#8B0000">But,one drawback of this method is that it is slightly less portable than explicitly calling waitpid: the behaviour it depends on is required by POSIX.1-2001, and previously by the Single Unix Specification, but not by POSIX.1-1990.</font>**

## Attention

* 1.signal inheritance
* 2.signal handler setting(durable)
* 3.variable save(signal handler)
* 4.loop waitpid in handler(signal processing mechanism)

## Refs

* [Reap zombie processes using a SIGCHLD handler](http://www.microhowto.info/howto/reap_zombie_processes_using_a_sigchld_handler.html#idp16032)
* [perldoc.perl.org](http://perldoc.perl.org/perlipc.html)
* [waitpid](https://linux.die.net/man/2/waitpid)
* [perl waitpid](http://perldoc.perl.org/functions/waitpid.html)
* [example1](http://www.cnblogs.com/wuchanming/p/4020463.html)
* [example2](https://www.coder4.com/archives/151)
* [example3](http://www.cnblogs.com/baoguo/archive/2009/12/09/1619956.html)
