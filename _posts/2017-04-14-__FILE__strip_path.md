---
layout: post  
title:  "去掉宏__FILE__的路径"  
date:   2017-04-14  
categories: C/C++  
tags: __FILE__ 路径  
author: sigusr1  
mathjax: true  
---

* content
{:toc}

本文介绍如何去掉宏__FILE__的路径，只显示文件名。





  
宏\__FILE\__展开后会带有路径信息，比如下面的代码：

```
#include<stdio.h>
#include<stdlib.h>

int main()
{
	printf("file_name:%s\n", __FILE__);
	return 0;
}
```

如果Makefile内容如下:

```
CFALG = -Wall

all: /home/helloworld/test.c
	gcc $(CFALG) $< -o test
```

编译运行，程序输出为：

```
file_name:/home/helloworld/test.c
```

为了不让宏\__FILE\__带有路径信息，可以在Makefile中重定义宏\__FILE\__：

```
CFALG = -Wall
CFALG += -U__FILE__ -D__FILE__='"$(subst $(dir $<),,$<)"'

all: /home/helloworld/test.c
	gcc $(CFALG) $< -o test
```

编译运行，程序输出为：

```
file_name:test.c
```

取消宏\__FILE\__会产生编译警告，如果不想产生警告，可以考虑新建一个宏， 比如\__FILENAME\__。