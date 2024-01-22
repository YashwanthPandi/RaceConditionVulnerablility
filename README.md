Overview:
During this lab, students will be presented with a program featuring a race-condition vulnerability. Their objective is to develop a strategy to exploit this vulnerability and gain root privileges. In addition to executing these attacks, students will be guided through various protective measures designed to thwart race-condition attacks. Students will be required to assess the effectiveness of these protective schemes and provide explanations for their evaluations.
Environment Setup:
Disabling Protective Measures
Ubuntu has a defense mechanism against race condition attacks. It limits permissions for following symbolic links in globally writable sticky directories. For this exercise, deactivate these protective measures with:
sudo chmod u+s /tmp
sudo chmod g-w /tmp


// For Ubuntu 20.04, use the following:
$ sudo sysctl -w fs.protected_symlinks=0
$ sudo sysctl -w fs.protected_regular=0

// For Ubuntu 16.04, utilize the following command:
$ sudo sysctl -w fs.protected_symlinks=0


// For Ubuntu 12.04, employ the subsequent command:
$ sudo sysctl -w kernel.yama.protected_sticky_symlinks=0


Task 1: Choosing Our Target:
We want to take advantage of a weakness in the program, which we call a "race condition vulnerability." Our target is the password file (/etc/passwd), which regular users can't normally change. By exploiting this weakness, we aim to add a new user account to the password file, and we want this new account to have the highest level of access, called "root privilege."The password file contains information about each user, including the root user. This information is separated by colons (:). The root user's information looks like this:
root:x:0:0:root:/root:/bin/bash

The third field, which is "0," gives the root user its special power. It's like a secret code that says, "This user is super important." We want to create a new account with this special code to get the same power.Each user's information also has a password field (the second field). In the example above, it's "x," which means the password is stored somewhere else. To make things simpler, we can just put the password in the password file, so the computer doesn't have to look for it in another place.But wait, the password field doesn't actually hold the real password; it holds a kind of secret code (hash) of the password. To get this secret code for a password, we can do some tricks, like adding a new user on our system and finding the code. Or we can use a known magic code used in Ubuntu live CDs, which means you don't need to type a password.
Here's what we're going to do:
As a superuser (administrator), we'll add a special entry to the end of the password file.
This special entry has the magic code for a passwordless account and the special code for root privilege.
We'll check if we can log in without typing a password and if we have super admin powers (root privilege).
After this test, we'll remove the special entry from the password file.
One important thing to remember: in the past, some students accidentally messed up the password file during this process. If that happens, you won't be able to log in anymore. So, it's a good idea to make a copy of the original password file or take a snapshot of your computer before you start. This way, you can easily fix things if something goes wrong.
Simulating a Slow Machine
To simulate a 10-second delay between access() and fopen(), add a sleep(10) call. The program would look like this:

if (!access(fn, W_OK)) { sleep(10); fp = fopen(fn, "a+"); ...

To add a root account to the system, you can:
Compile the vulp program with the addition that it will pause for 10 seconds.
Create a symbolic link to /dev/null called /tmp/XYZ.
When the program resumes, it will write to /tmp/XYZ, but the actual content will be written to /dev/null.
You will now have a root account.
$ ln -sf /dev/null /tmp/XYZ $ ls -ld /tmp/XYZ
lrwxrwxrwx 1 seed seed 9 Dec 25 22:20 /tmp/XYZ -> /dev/null

Task 2: Launching the Real TIme Race Condition Attack:
In the previous task, we simplified the attack by slowing down the vulnerable program. This is not practical in a real attack. In this task, we will carry out a real attack without slowing down the program.
Race condition attacks involve running an attack program alongside the vulnerable program to exploit a brief window of vulnerability. Precise timing is challenging, so success is probabilistic. With a narrow time window, the success rate may be low, but we can repeatedly launch the attack until we hit the window.
The attack program will utilize the symlink() function to create symbolic links. Since Linux prohibits creating a link if it already exists, we need to remove the existing link first. The provided C code snippet demonstrates how to remove a link and then create a symbolic link from /tmp/XYZ to /etc/passwd.
To automate running the vulnerable program repeatedly, we will write a dedicated program. Instead of manually entering input each time, we will use input redirection, storing the input in a file and instructing vulp to read the input from that file using vulp < inputFile.
To determine whether the attack is successful, we will monitor the timestamp of the file. The provided shell script executes the ls -l command, which outputs information about a file, including the last modified time. By comparing the outputs of the command with previous outputs, we can ascertain if the file has been modified.
The shell script runs the vulnerable program (vulp) in a loop, with the input provided by the echo command via a pipe. You need to determine the appropriate input. If the attack is successful, the script will terminate. You may need to be patient; success should typically be achieved within 5 minutes.

Improved Methods 

In Task 2.B, you may encounter a situation where your attack fails despite following all the instructions correctly. This is due to a race condition in the target program, which allows it to regain ownership of the file /tmp/XYZ before your attack program can complete its task. To resolve this issue, we can utilize the renameat2 system call to atomically swap symbolic links, effectively eliminating the race condition and ensuring the success of your attack.
#define _GNU_SOURCE
#include <stdio.h> #include <unistd.h> int main()
{ unsigned int flags = RENAME_EXCHANGE;
unlink("/tmp/XYZ"); symlink("/dev/null",	"/tmp/XYZ"); unlink("/tmp/ABC"); symlink("/etc/passwd", "/tmp/ABC");
renameat2(0, "/tmp/XYZ", 0, "/tmp/ABC", flags);
return 0;
}

Pictures Attached:


At present, exploiting this vulnerability requires additional time and increased computational resources. The attack is currently focused on the software, but there exists an advanced code that can enable real-time attacks. Below, you'll find a set of commands to achieve this.

