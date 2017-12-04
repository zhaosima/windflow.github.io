---
title: how to resolve can not find stddef.h
date: 2017-09-20 20:05:47
tags:
---
## how to resolve can not find stddef.h

When I write a simple helloworld code and try to compile it. The gcc report a error "stddef.h: No such file or directory".

I update the gcc but it still has the problem. So I google and find a way.

    > cpp hello.c > hello.i
    > gcc -c hello.i
    > gcc -o hello hello.o


It works but too troublesome.After a meaningless search, suddently I come out an idea.

    > sudo find / -name stddef.h
    > cp stddef-path-from-last /usr/include
    > gcc -c hello.c
Great! I do not know why the head file is not there, but fine, I solve it. 