---
title: "DevOps/SRE interview"
date: 2021-04-14T19:27:18Z
draft: true
---

#### Introduction

During my career, I did a lot of interviews for several roles such as
Developper, Linux System Engineer, DevOps Engineer and Site Reliability
Engineer. Here are some common questions and answers that I have been
interviewed and I hope it can help you if you prepare for interviews.

#### What is a kernel space and a user space?

A kernel space is an area where the processes have a full privileges for
performing any tasks, for instance to allocate some memory and resource on CPU ...,
while a user space is an area where the processes have a restrict privilege to
do that, these processes have to use a system call, thus it prevents to do some
mistakes.<br />
I remind you that a kernel provides 3 mains purpose:

* ???
* ???
* providing an interface named system calls

#### What does the system when you perform the "ls" command in a bash shell?

A syscall that consists to create a new process doesn't exist, when you perform
this command, it will perform the syscall "fork()", that means the bash
process will be duplicated instead of creating a new process, then it executes
the syscall "exec()".
