## lab1实验报告  

### 练习1  

1. ucore.img生成过程：

   生成ucore.img需要生成/bin/bootblock、/bin/kernel

   （1）生成kernel需要生成：

   ​		obj/kern/init/init.o 

   ​		obj/kern/libs/stdio.o 

   ​		obj/kern/libs/readline.o 

   ​		obj/kern/debug/panic.o 

   ​		obj/kern/debug/kdebug.o 

   ​		obj/kern/debug/kmonitor.o 

   ​		obj/kern/driver/clock.o 

   ​		obj/kern/driver/console.o 

   ​		obj/kern/driver/picirq.o 

   ​		obj/kern/driver/intr.o 

   ​		obj/kern/trap/trap.o 

   ​		obj/kern/trap/vectors.o 

   ​		obj/kern/trap/trapentry.o 

   ​		obj/kern/mm/pmm.o  

   ​		obj/libs/string.o 

   ​		obj/libs/printfmt.o

   （2）生成bootblock需要生成：

   ​		obj/bootblock.o

   ​		bin/sign

   ​	并执行bin/sign

2. 一个合法的主引导扇区需要的条件：

   最后一个字节为0x55

   倒数第二个字节为0xAA