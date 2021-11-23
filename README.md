# JVM

#### 一、介绍

本项目将尝试性的一步一步实现jvm的主线内容，并对每一次代码的提交进行讲解，每一次讲解都尽可能的假设你一无所知。考虑到来此学习的大部分是Java程序员，所以本项目的C++代码将充满着Java的味道。

 * 适合人群：面向java虚拟机的初学者、想要自实现jvm的学习者。
 如果想自己动手，那么你需要有一定的C\C++基础（可以理解指针即可）。如果仅仅是学习，那么相信强大的你不需要学过C\C++也能看懂。
 
 * 参考资料：《Java虚拟机规范》
 
---
   
#### 二、目录（不定时更新……）

##### (一)[类加载模块](#类加载模块)
###### 0.[一切的开始](#一切的开始)
###### 0.5[初识字节码文件](#初识字节码文件)
###### 1.[字节码文件读取到内存](#字节码文件读取到内存)
###### 2.[解析内存中的字节码文件](#解析内存中的字节码文件)
##### (二)字节码执行器
###### 1.字节码解释器 
###### 2.模版解释器 
##### (三)内存池
###### 1.Java进程总内存
###### 2.堆区
###### 3.方法区
##### （四）GC
###### 1.标记 && 清除 && 整理
###### 2.新生代 && 老年代 
##### （五）即时编译

---
      
#### 三、猛男讲解
<br/><br/>
PS：每一章节都会进行单元测试，为了简洁，文章此处只给出方法的声明，但会给出此次提交的commit,里面包含了方法的实现。
<br/><br/>
##### 类加载模块：

---

###### 一切的开始:
<br/>
   
     首先用你的IDEA，新建一个javaClass，输出 “Hello JVM”，并将其运行，相信这对于聪明的你易如反掌，就犹如探囊取物。
```java
       
public class HelloJVM {
    public static void main(String[] args) {
        System.out.println("Hello JVM");
    }
}
       
```
随后你会看到有一个 out 文件生成，里面有一个HelloJVM.class 文件。

<img src="https://github.com/ArosyW/picture/blob/master/outClass.png" width = "450" height = "300" />

没错，这就是IDEA帮你编译后的文件，java虚拟机最终执行的就是这个.class文件，我们先来打开看看它里面是啥玩意。<br/>
<br/>
打开方式：<br/>
1.直接拖入subLime Text (某文本编辑器)<br/>
2.命令行(mac)：
```
vim /Users/e/Documents/github/JDK/out/production/JDK/HelloJVM.class
:%!xxd
```
当然你也可以以任何你喜欢的其他方法打开它，最终会看到其内容如下：

<img src="https://github.com/ArosyW/picture/blob/master/class16.jpeg" width = "400" height = "500" />

---

###### 初识字节码文件:
<br/>
<br/>
既然我们自定义的HelloJVM类最终被编译成了这个16进制文件，并被Java虚拟机读取解析执行，那一定要满足两个最基本的条件：
<br/><br/>
1.这个16进制文件一定包含了HelloJVM类所有内容<br/>
2.这个16进制文件的内容顺序一定有规范，只有这样Java虚拟机才能按照约定的规律正确读取。
<br/><br/>


我们将通过一边讲解一边写代码解析的方式来知晓它的全貌，整个类加载模块要做的，可以用一句话总结：<br/><br/>
**读取硬盘上的.class文件，并把里面的内容用C++的方式（主要指c++的类文件）按照一定规范存储在内存中，以供需要的时候可以立即检索到。**
<br/><br/>

围绕着这句话，现在我们只有一个问题要讨论：**读取与存储的规范是什么？**
<br/>
我将逐步讲解.class中的每一个字节的含义，小伙伴亦可参考：《Java虚拟机规范》
<br/><br/>
不妨先来认识前4个字节，“cafe babe”，这4个字节将先被Java虚拟机读取，它的含义在于指明这是一个.class文件，你可以继续读取了，否则没有读下去的必要，因为Java虚拟机只能解析.class文件。
<br/><br/>

---

###### 字节码文件读取到内存
   **本次commit :** f4f61fcfe7f461d69a58d01b2e71bb6efe4e5387 
   * 读到内存
   新建 ClassRead.h && ClassRead.cpp 
```c++
class ClassRead {
    char *data;
    int cur = 0;
public:
    //构造方法
    ClassRead(char *data);

    //根据路径读取.class文件
    static ClassRead* readByPath(string path);

    unsigned char readByOneByte();//往前读取一个字节
    unsigned short  readByTwoByte();//往前读取两个字节
    unsigned int readByFourByte();//往前读取四个字节

};
```
重申：方法的具体实现请看移步代码commit:f4f61fcfe7f461d69a58d01b2e71bb6efe4e5387
<br/><br/>
单元测试一下
```c++
int main() {
    ClassRead *classRead = ClassRead::readByPath("/Users/e/Documents/github/JDK/out/production/JDK/HelloJVM.class");//换成你自己的path
    printf("%X\n", classRead->readByFourByte());
    return 0;
}
```
输出：CAFEBABE 

到这你已经成功的完成了字节码文件解析的第一步：验证这个文件是否是一个字节码文件！你真厉害，奖励自己一局”亚索“吧！

---

###### 解析内存中的字节码文件


   
   

















