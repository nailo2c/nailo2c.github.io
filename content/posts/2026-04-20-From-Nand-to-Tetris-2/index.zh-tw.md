---
title: "From Nand to Tetris (Part 2)"
date: "2026-04-20T00:00:00-00:00"
draft: true
description: ""
featuredImage: ""

tags: ["Study"]
categories: ["Study"]

math:
  enable: true
---

# Unit 1 - VM - Stack Arithmetric

VM是Stack Machine，主因是這樣VM跟Compiler都會變得簡潔，我認為是課堂需要。

Memory Segment將Memory切成數個區塊存放不同類別的資料例如 argument, local, static, constant

這堂課有八個virtual memory segment： local / argument / this / that / constant / static / pointer / temp

要注意 push xxx 1 是指複製 xxx 的數值到 stack，並不會消滅它 (xxx 不包含 constant)

pop xxx 2 是指從 stack 的頂層拿出數值後assign到 xxx 的 index 2 中 (xxx 不包含 constant)

## Pointer

D = *p 等價於下列三行:  
@p  
A=M  
D=M  

p-- 等價於 p = p - 1  
如果 RAM index p 的 value 是 257，那 p-- 後，該 value 會變 256

如果 RAM index q 的 value 是 1024，那 *q = 9 則是將 RAM index 1024 的 value 改成 9  
如果下一行執行 q++，則代表 RAM index q 的 value +1，也就是 1025

但題目又有點把我搞混，題目如下:

RAM[256] = 22
RAM[257] = 31
RAM[258] = 200
RAM[259] = 28

求:  
1  p1 = 256
2  p1 = p1 + 3
3  *p1 = *p1 + 3
4  p2 = p1 - 2
5  p1--
6  *(p2 + 1) = *p1 + *p2

第2行 -> p1 = 259  
第3行 -> *p1 = RAM[259] + 3 = 31 -> 代表寫31回RAM[259]  
第4行 -> p2 = 257  
第5行 -> p1 = 256  
第6行 -> *(p2+1) = *258 = *p1 + *p2 = 31 + 200 = 231 -> 代表寫231回RAM[258]  

看起來*p的意思就是要把資料寫進該address

*SP = 17  
SP++  

等價於下列:  
@17  // D=17  
D=A  
@SP  // *SP=D  
A=M  
M=D  
@SP  // SP++  
M=M+1  

## Memory Segment

1. Local - LCL
    1. LCL只記得base address，接著依序存放資料
    2. local有三個資料7, -91, 3，假設base address是1015，則RAM[1015]=7, RAM[1016]=-91, RAM[1017]=3
2. local / argument / this / that
    1. object -> this
    2. array -> that
    3. 上述四個都是有 base address，然後再依序取資料
    4. push 實作: addr = segmentPointer + i, *SP = *addr, SP++ (基本上push就是放到stack)
    5. pop 實作: addr = segmentPointer + i, SP--, *SP = *addr
3. constant
    1. 只有 push，沒有 pop
    2. *SP = i, SP++
4. static
    1. 他是全域變數，因此無法像上述segment可以dynamic的放在RAM裡
    2. 語法: static 5 -> @Foo.5
    3. 這堂課裡會把static放在RAM[16] ~ RAM[255]
5. temp
    1. Hack VM提供8個temp variables，RAM[5] ~ RAM[12]
    2. push 實作: addr = 5 + i, *SP = *addr, SP++
    3. pop 實作: addr = 5 + i, SP--, *SP = *addr
6. pointer
    1. 持續追蹤 this 跟 that 用，在compiler中比較能體現它的用處
    2. pointer 0 <=> THIS
    3. pointer 1 <=> THAT
    4. push 實作: *SP = THIS/THAT, SP++
    5. pop 實作: SP--, THIS/THAT = *SP
    6. "that7" = RAM[THAT + i]

## VM translator

VM translator目的是把VM code轉譯成Assembly Code

Unit 1的project先實作以下部分:

Arithmetic / Logical commands:  
`add`  
`sub`  
`neg`  
`eq`  
`gt`  
`lt`  
`and`  
`or`  
`not`  

Memory access commands:  
`pop segment i`  
`push segment i`  

