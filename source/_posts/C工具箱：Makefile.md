title: C工具箱：Makefile 
date: 2015-12-20 09:31:54
tags:
- 工具箱
- C

categories: 程序员工具箱

---

关于Makefile有几个知识点：

$(wildcard .cpp)：展开获得路径下所有以.cpp结尾的文件名

OBJS = /$(SRCS:.c=.o)：把SRCS中的.c换成.o，赋值给OBJS

$@表示目标

$^表示所有依赖

$<表示第一个依赖文件


```
CC=gcc
CXX=g++
INCLUDE=-I./
LDFLAGS=-lcurl ./Decoder/release/Decoder.a 

MAKER_SRCS=$(wildcard *.cpp)
MAKER_OBJS=$(MAKER_SRCS:.cpp=.o)
TARGET=ProgramName

.PHONY:all
all: $(TARGET)

$(TARGET):$(MAKER_OBJS)
        @echo 'Building target: $@'
        $(CXX) -o $@ $^  $(LDFLAGS)

%.o:%.cpp
        $(CXX) $(INCLUDE) -o $@ -c $<

.PHONY:clean
clean:
        rm -f $(TARGET) $(MAKER_OBJS)
```
