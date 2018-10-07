---
title: 简单的ELF解析器
date: 2018-03-19 00:00:52
tags:  
- ELF
- Linux

---



## 0x00 前言

学习过ELF文件格式之后，为了加深理解就想着自己去解析一遍，文件结构网上有大量的资料，就不多说啦0.0

<!--more-->

## 0x01手动解析

这里编译了带有调试信息的glibc，对于版本的glibc可以去这里下载应版本http://mirrors.ustc.edu.cn/gnu/libc/

编译：

64位：

```
cd glibc && mkdir build && cd build
CFLAGS="-g -g3 -ggdb -gdwarf-4 -Og"
CXXFLAGS="-g -g3 -ggdb -gdwarf-4 -Og"
../configure --prefix=/path/to/install

```

32位：

```
cd glibc && mkdir build32 && cd build32
CC="gcc -m32" CXX="g++ -m32" \
CFLAGS="-g -g3 -ggdb -gdwarf-4 -Og" \
CXXFLAGS="-g -g3 -ggdb -gdwarf-4 -Og" \
../configure --prefix=/path/to/install --host=i686-linux-gnu
```



然后先编写一个简单的HelloWorld

```
# include<stdio.h>

int main(){

    printf("Hello world");
    return 0;

}
```

然后指定glibc版本编译，这里引用的是我编译好的glibc存放的目录

```
hello.o: hello.c

    gcc hello.c -o hello -Wl,--dynamic-linker=/home/ubuntu/debugGlibc/lib/ld-2.23.so

```

然后再在命令行下指定下LD_LIBRARY_PATH路径（这只是临时的，退出这个shell即失效）

```
export LD_LIBRARY_PATH=/home/ubuntu/debugGlibc/lib
```

{%asset_img 0.png%}

然后gdb开始调试运行，这里查看一下进程布局

`i proc mappings  `

{%asset_img 1.png%}

进程起始地址为0x400000

那么来手动解析下elf header

{%asset_img 2.png%}

对比下readelf

{%asset_img 3.png%}

那么再来解析下section header啊

{%asset_img 4.png%}

嘛玩意儿？emmmm，别忘了section header并不会加载进内存（手动滑稽）

那敢情好，解析下program header试试

{%asset_img 5.png%}

那么手动解析到此，别的字段可以自行试试



## 0x02 编写简易的ELF解析器

本来想用python或者java写的，但想想指针这个东西某些时候确实方便（手动滑稽）

代码部分参考自https://github.com/finixbit/elf-parser

自己用CMake重新编译了下，然后稍稍修改了一丢丢

当然本身也就是学习为了熟悉下elf结构，所以自己添加了大量的注释说明



这个简易的解析器只实现了解析section header table功能



首先看头文件

```
//
// Created by Mask on 2018/3/14.
//

#ifndef ELF_PARSE_ELF_PARSE_H
#define ELF_PARSE_ELF_PARSE_H


#include <cstdlib>
#include <iostream>
#include <elf.h>
#include <vector>
#include <sys/stat.h>
#include <sys/fcntl.h>
#include <sys/mman.h>

namespace elf_parser{
    typedef struct {
        int section_index = 0;
        std::intptr_t section_offset,section_addr;
        std::string section_name,section_type;
        int section_size,section_ent_size,section_addr_align;
    } section_t;



    class Elf_parser {
    public:
        Elf_parser(std::string &program_path): m_program_path{program_path}{ //构造函数，将elf文件加载进内存
            load_memory_map();
        }
        std::vector<section_t> get_sections();
        u_int8_t *get_memory_map();

    private:
        void load_memory_map();
        std::string get_section_type(int tt);
        std::string m_program_path; //elf文件路径
        u_int8_t *m_mmap_program; //获取elf文件加载进内存的初始地址
    };

    void Elf_parser::load_memory_map() {
        int fd,i;
        struct stat st;

        if((fd = open(m_program_path.c_str(),O_RDONLY))<0){ //打开失败报错
            perror("open");
            exit(-1);
        }
        if(fstat(fd,&st)<0){
            perror("fstat");
            exit(-1);
        }
        m_mmap_program = static_cast<u_int8_t *>(mmap(NULL,st.st_size,PROT_READ,MAP_PRIVATE,fd,0));//映射进内存
        if (m_mmap_program == MAP_FAILED){
            perror("mmap");//映射失败
            exit(-1);
        }

        auto header = (Elf64_Ehdr*)m_mmap_program;
        if (header->e_ident[EI_CLASS]!=ELFCLASS64){ //判断是否为elf64格式
            printf("Only 64-bit files supported\n");
            exit(1);
        }


    }

    std::string Elf_parser::get_section_type(int tt) {
        if(tt < 0)
            return "UNKNOWN";

        switch(tt) {
            case 0: return "SHT_NULL";      /* Section header table entry unused */
            case 1: return "SHT_PROGBITS";  /* Program data */
            case 2: return "SHT_SYMTAB";    /* Symbol table */
            case 3: return "SHT_STRTAB";    /* String table */
            case 4: return "SHT_RELA";      /* Relocation entries with addends */
            case 5: return "SHT_HASH";      /* Symbol hash table */
            case 6: return "SHT_DYNAMIC";   /* Dynamic linking information */
            case 7: return "SHT_NOTE";      /* Notes */
            case 8: return "SHT_NOBITS";    /* Program space with no data (bss) */
            case 9: return "SHT_REL";       /* Relocation entries, no addends */
            case 11: return "SHT_DYNSYM";   /* Dynamic linker symbol table */
            default: return "UNKNOWN";
        }
        return "UNKNOWN";
    }


    std::vector<section_t> Elf_parser::get_sections() {
        Elf64_Ehdr *ehdr = (Elf64_Ehdr*)m_mmap_program; //获取elf_header
        Elf64_Shdr *shdr = (Elf64_Shdr*)(m_mmap_program+ehdr->e_shoff);//获取section header table
        int shnum = ehdr->e_shnum;//section数

        Elf64_Shdr *sh_strtab = &shdr[ehdr->e_shstrndx]; //获取短表字符串表，这个表里有每个section的名称
        const char *const sh_strtab_p = (char*)m_mmap_program+sh_strtab->sh_offset;

        std::vector<section_t> sections;
        for (int i = 0; i < shnum; i++) {
            section_t section;
            section.section_index = i;
            section.section_name = std::string(sh_strtab_p+shdr[i].sh_name);
            section.section_type = get_section_type(shdr[i].sh_type);
            section.section_addr = shdr[i].sh_addr;
            section.section_offset = shdr[i].sh_offset;
            section.section_size = shdr[i].sh_size;
            section.section_ent_size = shdr[i].sh_entsize;
            section.section_addr_align = shdr[i].sh_addralign;

            sections.push_back(section);


        }
        return sections;
    }


}

#endif //ELF_PARSE_ELF_PARSE_H

```



再来看获取section的主控函数

```
#include <iostream>
#include <cinttypes>

#include "elf_parse.h"

void print_sections(std::vector<elf_parser::section_t> &sections);

int main(int argc,char* argv[]) {

    char usage_banner[] = "usage:./section [<excutable>]\n";
    if (argc<2){
        fprintf(stderr,usage_banner);
        return -1;
    }

    std::string program((std::string)argv[1]);
    elf_parser::Elf_parser elf_parser(program);

    std::vector<elf_parser::section_t> secs = elf_parser.get_sections();
    print_sections(secs);
    return 0;
}

void print_sections(std::vector<elf_parser::section_t> &sections) {
    printf("  [Nr] %-16s %-16s %-16s %s", "Name", "Type", "Address", "Offset");
    printf("       %-16s %-16s %5s\n",
           "Size", "EntSize", "Align");

    for (auto &section : sections) {
        printf("  [%2d] %-16s %-16s %016" PRIx64 " %08" PRIx64 "",
               section.section_index,
               section.section_name.c_str(),
               section.section_type.c_str(),
               section.section_addr,
               section.section_offset);

        printf("       %016zx %016" PRIx64 " %5" PRIu64 "\n",
               section.section_size,
               section.section_ent_size,
               section.section_addr_align);
    }
}

```

CMakelist.txt

```
cmake_minimum_required(VERSION 3.9)
project(Elf_parse)

set(CMAKE_CXX_STANDARD 11)

add_executable(Elf_parse sections.cpp elf_parse.h)
```

运行看看效果

{%asset_img 7.png %}



emmm还可以，其他字段的解析类似，只要对着elf的文件结构解析即可

## 0x03 一点小小的学习笔记

有关mmap函数

https://blog.csdn.net/yangle4695/article/details/52139585

C++ 11的一些新特性

http://blog.jobbole.com/44015/

