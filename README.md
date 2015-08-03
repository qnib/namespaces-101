# namespaces-101

Basic fooling around with Linux Namespaces inspired be Jean-Tiare Le Bigots' [Blog posts](https://blog.jtlebi.fr/2013/12/22/introduction-to-linux-namespaces-part-1-uts/). 

## Vagrant

If you are not running a Linux flavour (like I do using MacOSX), you need to spin up a virtual machine.
By running `vagrant up` in the repository you will spin up a ubuntu box to do the namespacing stuff.

```
$ vagrant up
Bringing machine 'default' up with 'virtualbox' provider...
==> default: Box 'lucid32' could not be found. Attempting to find and install...
    default: Box Provider: virtualbox
*snip*
==> default: Mounting shared folders...
    default: /vagrant => /Users/ckniep/src/github.com/qnib/namespaces-101
79 13:57:47 rc=0 DOCKER:mbp namespaces-101 {master} $ vagrant ssh
```
Login to the vagrant box:
```
79 13:57:47 rc=0 DOCKER:mbp namespaces-101 {master} $ vagrant ssh
Linux lucid32 2.6.32-38-generic #83-Ubuntu SMP Wed Jan 4 11:13:04 UTC 2012 i686 GNU/Linux
Ubuntu 10.04.4 LTS

Welcome to Ubuntu!
 * Documentation:  https://help.ubuntu.com/
New release 'precise' available.
Run 'do-release-upgrade' to upgrade to it.

Welcome to your Vagrant-built virtual machine.
Last login: Fri Sep 14 07:26:29 2012 from 10.0.2.2
vagrant@lucid32:~$ cd /vagrant/
vagrant@lucid32:/vagrant/$ 
```
Now you can proceed with the tutorial as if you would ran on Linux (since you do:)

## Sceleton
Jeans sceleton looks like this:
```
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>
 
#define STACK_SIZE (1024 * 1024)
 
static char child_stack[STACK_SIZE];
char* const child_args[] = {
  "/bin/bash",
  NULL
};
 
int child_main(void* arg)
{
  printf(" - World !\n");
  execv(child_args[0], child_args);
  printf("Ooops\n");
  return 1;
}
 
int main()
{
  printf(" - Hello ?\n");
  int child_pid = clone(child_main, child_stack+STACK_SIZE, SIGCHLD, NULL);
  waitpid(child_pid, NULL, 0);
  return 0;
}
```

Compiling and running this piece of software is going to clone a child process that prints `World`. As it's the template it's not using namespaces yet. :)

```
vagrant@lucid32:/vagrant$ gcc -Wall -o template src/main-0-template.c && ./template
 - Hello ?
 - World !
vagrant@lucid32:/vagrant$ #now we are inside of the bash that is spawned by the child.
vagrant@lucid32:/vagrant$ exit
exit
vagrant@lucid32:/vagrant$ # back to the parent again
```

## UTS Namespace

Ok, as we have our sceleton setup, let's use the first namespace. The template is changed slightly:
```
$ git diff src/main-0-template.c
diff --git a/src/main-0-template.c b/src/main-0-template.c
index de1a897..2f0876a 100644
--- a/src/main-0-template.c
+++ b/src/main-0-template.c
@@ -17,6 +17,7 @@ char* const child_args[] = {
 int child_main(void* arg)
 {
   printf(" - World !\n");
+  sethostname("In Namespace", 12);
   execv(child_args[0], child_args);
   printf("Ooops\n");
   return 1;
@@ -25,7 +26,8 @@ int child_main(void* arg)
 int main()
 {
   printf(" - Hello ?\n");
-  int child_pid = clone(child_main, child_stack+STACK_SIZE, SIGCHLD, NULL);
+  int child_pid = clone(child_main, child_stack+STACK_SIZE, CLONE_NEWUTS | SIGCHLD, NULL);
   waitpid(child_pid, NULL, 0);
   return 0;
 }
```

If we now compile and run it, we will get another UTS namespace, therefore a different hostname then the parent.
```
vagrant@lucid32:/vagrant$ gcc -Wall -o bin/uts src/main-0-template.c && sudo ./bin/uts
 - Hello ?
 - World !
root@In Namespace:/vagrant# hostname
In Namespace
root@In Namespace:/vagrant# exit
exit
vagrant@lucid32:/vagrant$
```

Similar examples can be found for all the other namespaces involved.

- [Introduction to Linux namespaces – Part 1: UTS](https://blog.jtlebi.fr/2013/12/28/introduction-to-linux-namespaces-part-2-ipc/) (From which this inherits)
- [Introduction to Linux namespaces – Part 2: IPC](https://blog.jtlebi.fr/2013/12/28/introduction-to-linux-namespaces-part-2-ipc/)
- [Introduction to Linux namespaces – Part 3: PID](https://blog.jtlebi.fr/2014/01/05/introduction-to-linux-namespaces-part-3-pid/)
- [Introduction to Linux namespaces – Part 4: NS (FS)](https://blog.jtlebi.fr/2014/01/12/introduction-to-linux-namespaces-part-4-ns-fs/)
- [Introduction to Linux namespaces – Part 5: NET](https://blog.jtlebi.fr/2014/01/19/introduction-to-linux-namespaces-part-5-net/)
- [Introduction to Linux namespaces – Part 6: USER]() This one is missing... AFAIK the namespace is not stable yet :(
