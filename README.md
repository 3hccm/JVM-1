# JVM

## 一、介绍

本项目将尝试性的一步一步实现jvm的主线内容，并对每一次代码的提交进行讲解，每一次讲解都尽可能的假设你一无所知。考虑到来此学习的大部分是Java程序员，所以本项目的C++代码将充满着Java的味道。

 * 适合人群：面向java虚拟机的初学者、想要自实现jvm的学习者。
 如果想自己动手，那么你需要有一定的C\C++基础（可以理解指针即可）。如果仅仅是学习，那么相信强大的你不需要学过C\C++也能看懂。
 
 * 参考资料：《Java虚拟机规范》
 
---
   
## 二、目录（不定时更新……）

### (一)类加载模块
#### 0.[一切的开始](#一切的开始)
#### 0.5[初识字节码文件](#初识字节码文件)
#### 1.[字节码文件读取到内存](#字节码文件读取到内存)
#### 2.[解析内存中的字节码文件](#解析内存中的字节码文件)
##### 2.1[解析“魔数”](#魔数)
##### 2.2[解析“版本号”](#版本号)
##### 2.3[解析“Class常量池”](#常量池)
##### 2.4[解析“访问权限”](#访问权限)
##### 2.5[解析“类名 && 父类名”](#类名)
##### 2.6[解析"接口"](#接口)
##### 2.7[解析“字段”](#字段)
##### 2.8[解析“方法”](#方法)
##### 2.9[解析“属性”](#属性)
#### 3.[类加载器](#类加载器)
### (二)字节码执行器
#### 1.[cpu是如何执行指令的](#cpu是如何执行指令的)
#### 2.[cpu是如何执行方法的](#cpu是如何执行方法的)
#### 3.[JVM如何执行方法的](#JVM如何执行方法的)
##### 3.1[初始字节码指令](#初始字节码指令)
##### 3.2[方法执行理论](#方法执行理论)
#### 4.字节码解释器
#### 5.模版解释器 
### (三)内存池
#### 1.Java进程总内存
#### 2.堆区
#### 3.方法区
### (四)GC
#### 1.标记 && 清除 && 整理
#### 2.新生代 && 老年代 
### (五)即时编译
### (六)扩展内容
#### 1.[cpu如何执行指令与方法](#cpu如何执行指令与方法)

---
      
## 三、猛男讲解
<br/><br/>
PS：每一章节都会进行单元测试，为了简洁，文章部分地方可能只给出方法的声明，但会给出此次提交的commit,里面包含了方法的实现。
<br/><br/>

### (一)类加载模块


**<p id="一切的开始">0.一切的开始：</p>**

---

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


**<p id="初识字节码文件">0.5初识字节码文件：</p>**

---

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
因此即使类加载模块你看的迷迷糊糊也没有关系，它只是整个Java虚拟机的一小部分，这一部分都是规范，没有太多技巧，就是把你写的Java代码用C++存储起来而已，一定不要放弃，假如这是你第一次接触此类内容，你只需要能够跟着我过一遍就很了不起了，你很幸运，在未来的章节中我将时不时地这么鼓励你。
<br/><br/>
围绕着这句总结，现在我们只有一个问题要讨论：**读取与存储的规范是什么？**
<br/><br/>
我将逐步讲解.class中的每一个字节的含义，小伙伴亦可参考：《Java虚拟机规范》
<br/><br/>
不妨先来认识前4个字节，“cafe babe”，这4个字节将先被Java虚拟机读取，它的含义在于指明这是一个.class文件，你可以继续读取了，否则没有读下去的必要，因为Java虚拟机只能解析.class文件。另外它被取了一个听起来更牛*一点的名字叫做“魔数”，英文名“magic"。
<br/><br/>


**<p id="字节码文件读取到内存">1.字节码文件读取到内存：</p>**

---

   **本次commit :** b898c3ba733fb44c06fec6b0928cd9a173f5cd0f 
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

    unsigned char readByOneByte();//往后读取一个字节
    unsigned short  readByTwoByte();//往后读取两个字节
    unsigned int readByFourByte();//往后读取四个字节

};
```
重申：方法的具体实现请移步代码commit:f4f61fcfe7f461d69a58d01b2e71bb6efe4e5387
<br/><br/>
我们来做一个测试
```c++
int main() {
    ClassRead *classRead = ClassRead::readByPath("/Users/e/Documents/github/JDK/out/production/JDK/HelloJVM.class");//换成你自己的path
    printf("%X\n", classRead->readByFourByte());
    return 0;
}
```
输出：CAFEBABE 

小总结：到这你已经成功的完成了字节码文件解析的第一步：验证这个文件是否是一个字节码文件！你真厉害，奖励自己一局”亚索“吧！


**<p id="解析内存中的字节码文件">2.解析内存中的字节码文件：</p>**

---


* **<p id="魔数">2.1解析“魔数”：</p>**

   **本次commit :** b51a7f6ab598a13b1e56b6f8cf9686944c10084c

在前文我们已经成功的将.class文件读取到了内存，创建了按1、2、4个字节往后读取的方法，并且读取了前4个字节为CAFEBABE。
<br/><br/>
在解析剩下的字节之前，我们来思考一个问题，在解析这些字节的时候我们应该将他们存储在哪，才能在需要的时候快速定位到？
通常的做法，也是Hotspot的做法，用C++新建一个名为Klass的类，来对应一个.class文件，我们直接上代码。
```c++
class InstanceKlass {
    int magic; //魔数，CAFEBABE:用来校验是否是.class文件
    
};
```
我们只解析了前四个字节，所以InstanceKlass目前只定义了一个属性，随着解析的继续，InstanceKlass中将包含所有.class文件的内容。
我们来简短的讨论一下代码结构，前面我们定义了ClassRead来读取磁盘上的.class文件到内存中，很自然的我们还需要有一个专门解析内存中的字节的c++类，于是我们新建类ClassFileParser
```c++
class ClassFileParser {
public:
    static InstanceKlass* Parser(ClassRead *classRead);
    static void checkAndPutMagic(ClassRead *classRead, InstanceKlass *klass);
};
```
它目前只有两个方法：<br/>
1.Parser，就是用来解析被读到内存中的.class文件，也就是在上一章节生成的ClassRead对象。然后返回一个InstanceKlass对象，是不是很像一个InstanceKlass工厂。
<br/>
2.checkAndPutMagic，供Parser调用。<br/><br/>
我们来把前四个字节写入InstanceKlass对象中。
```c++
InstanceKlass *ClassFileParser::Parser(ClassRead *classRead) {
    InstanceKlass *klass = new InstanceKlass//创建klass对象用来存储.class文件中的信息;
    checkAndPutMagic(classRead, klass);//校验是否是class文件
    return klass;
}

void ClassFileParser::checkAndPutMagic(ClassRead *classRead, InstanceKlass *klass) {
    klass->setMagic(classRead->readByFourByte());
    if (klass->getMagic() == MAGIC) { //MAGIC : CAFEBABE
        printf("class文件校验正确\n");
        return;
    }
    printf("class文件校验错误，%X\n", klass->getMagic());
}
```
在Parser中我们new了一个InstanceKlass对象，然后调用checkAndPutMagic将前四个字节赋值给InstanceKlass的magic属性。
<br/><br/>
我们来做一个测试<br/><br/>
```c++
int main() {
    ClassRead *classRead = ClassRead::readByPath("/Users/eba/Documents/github/JDK/out/production/JDK/HelloJVM.class");//换成你自己的path
    InstanceKlass *klass = ClassFileParser::Parser(classRead);
    printf("%X\n", klass->getMagic());
    return 0;
}
```
输出：<br/>
class文件校验正确<br/>
CAFEBABE

小总结：本节我们新建了InstanceKlass用来存储.class文件中的字节信息，并正确的将CAFEBABE属性正确地写入了InstanceKlass对象中，顺便确定了解析流程的代码结构，每次新解析一块内容，只需要新建解析方法，然后Parser方法中调用即可，示例：checkAndPutMagic 方法。

---

* **<p id="版本号">2.2解析“版本号”：</p>**

  **本次commit :** 36344db2cf2e8be84980307cf45d7b1378d4a746

在前文中我们已经成功地将"CAFEBABE"存入到了我们新创建的klass对象，下面我们再来读四个字节，"0000 0034"是用来校验JDK版本号的,其中"0000" 表示JDK的次版本号"minorVersion"，"0034"表示主版本号"majorVersion"，次版本号一般为0，主版本号与JDK版本的对应关系如下：
<br/>

|  jdk版本   | major version  |
|  ----  | ----  |
| 1.1  | 45 |
| 1.2  | 46 |
| 1.3  | 47 |
| 1.4  | 48 |
| 5  | 49 |
| 6  | 50 |
| 7  | 51 |
| 8  | 52 |

<br/>
十六进制的"0034"，即为十进制的52，没错，我用的JDK8。于是我们给InstanceKlass新增两个属性如下：

```c++
class InstanceKlass {
    int magic; //魔数，CAFEBABE:用来校验是否是.class文件
    short minorVersion; //JDK次版本号
    short majorVersion; //JDK主版本号
}
```
然后在ClassFileParser中新增一个解析版本号的方法，然后在Parser方法中调用它，于是我们的代码变成了这样：

```c++
InstanceKlass *ClassFileParser::Parser(ClassRead *classRead) {
    InstanceKlass *klass = new InstanceKlass;
    checkAndPutMagic(classRead, klass);//校验是否是class文件
    checkAndPutVersion(classRead, klass);//校验版本号
    return klass;
}

void ClassFileParser::checkAndPutVersion(ClassRead *classRead, InstanceKlass *klass) {
    klass->setMinorVersion(classRead->readByTwoByte());//读取两个字节作为次版本号set进InstanceKlass属性
    klass->setMajorVersion(classRead->readByTwoByte());//读取两个字节作为主版本号set进InstanceKlass属性
    printf("次版本号：%d\n", klass->getMinorVersion());
    printf("主版本号：%d\n", klass->getMajorVersion());
    switch (klass->getMajorVersion()) {//校验版本号是否在允许范围内
        case 46:
        case 47:
        case 48:
        case 49:
        case 50:
        case 51:
        case 52: {
            if (klass->getMinorVersion() == 0) {
                printf("版本校验正确\n");
                return;
            }
        }
    }
    printf("版本校验错误\n");
};
```
我们来做一个测试：

```c++
int main() {
    ClassRead *classRead = ClassRead::readByPath("/Users/e/Documents/github/JDK/out/production/JDK/HelloJVM.class");//换成你自己的path
    InstanceKlass *klass = ClassFileParser::Parser(classRead);
    return 0;
}
```

输出：<br/><br/>
class文件校验正确<br/>
次版本号：0<br/>
主版本号：52<br/>
版本校验正确<br/>
<br/>
小总结：本章节我们新建了属性与方法用于解析"0000 0034"版本号，事实上，接下来解析剩余字节的流程与此类似。

---

* **<p id="常量池">2.3解析“Class常量池”：</p>**

  **本次commit :** 9a212900d255cda698780d96dafff43dc1152677

“常量池”顾名思义，一堆常量的集合，诸如"main"、"HelloJVM.java"等单词都存储在这里面，或许在这之前你有听说过"方法区"以及它的实现"元空间"、"永久代"，没错，从狭义上，InstanceKlass里面存的所有的内容在"方法区"，而"方法区"就是一块内存空间，我们在GC章节会讲解。
若是你亦听说过"方法区"里存储着"元数据"，那么这一章节你将更加清晰的认识到"元数据"是什么，当然，元数据包含整个InstanceKlass里面的内容，而不局限于本章节的Class常量池。
<br/><br/>
PS：后面我们还会讲到"运行时常量池" 和 "字符串常量池"
<br/><br/>
动手！
<br/><br/>
我们会遇到一个问题，Java类是自已定义的，里面究竟有多少个方法，多少个属性是不一定的，意味着"常量池"的大小是不一定的，那么我们如何确定我们应该读取多少个字节停下呢。
显然我们需要预知长度，这里用了两个字节表示长度。"CAFEBABE 0000 0034 0022 ……" 其中0x0022是十进制的34，这里要特别注意，按照约定，真正有效的常量池数量是所给出的长度n-1，意味着我们只有33个常量池字段需要读取，那么用一个for循环33次是比较合理的。
每一次循环即每一次读取常量池字段的时候，我们还需要知道我们读到的数据是什么类型才能知道接下来读取几个字节能完全取到我们想要的数据，这里给出14中类型如下：

|  类型   | 映射值  |
|  ----  | ----  |
|   CONSTANT_Class               |    7 |
|   CONSTANT_Fieldref            |    9 |
|  CONSTANT_Methodref            |   10 |
|  CONSTANT_InterfaceMethodref   |   11 |
|   CONSTANT_String              |    8 |
|   CONSTANT_Integer             |    3 |
|   CONSTANT_Float               |    4 |
|   CONSTANT_Long                |    5 |
|   CONSTANT_Double              |    6 |
|  CONSTANT_NameAndType          |   12 |
|   CONSTANT_Utf8                |    1 |
|  CONSTANT_MethodHandle         |   15 |
|  CONSTANT_MethodType           |   16 |
|  CONSTANT_InvokeDynamic        |   18 |

<br/><br/>
那么解析常量池的流程基本明了了，先读取两个字节获取到常量池的长度count，然后循环count次，每次先读取一个字节用来匹配类型，这里显然用一个switch case比较合理，匹配到了类型，做不同的处理，然后存在InstanceKlass中。

我们先来在创建常量池类ConstantPool:

```c++
class ConstantPool {
public:
    char *tag;
    std::map<int, char*> data;
};

```
用tag来表示数据类型，用map来存储数据，key为索引，也就是它是第几个常量（后面的章节执行指令时需要用到索引），value即为真正的常量池数据。
<br/><br/>
接下来在InstanceKlass中增加常量池数量和常量池两个字段：
```c++
class InstanceKlass {
    int magic; 
    short minorVersion; 
    short majorVersion;
    short constantPoolCount;//常量数量（新增）
    ConstantPool *constantPool;//常量池数据（新增）
};

```
ClassFileParser中新增两个方法分别解析"常量池数量"和"常量池"：

```c++
InstanceKlass *ClassFileParser::Parser(ClassRead *classRead) {
    InstanceKlass *klass = new InstanceKlass;
    checkAndPutMagic(classRead, klass);//校验是否是class文件
    checkAndPutVersion(classRead, klass);//校验版本号
    parserConstantPoolCount(classRead, klass);//解析出常量池数量
    parserConstantPool(classRead, klass);//解析常量池
    return klass;
}

void ClassFileParser::parserConstantPoolCount(ClassRead *classRead, InstanceKlass *klass) {
    klass->setConstantPoolCount(classRead->readByTwoByte());
};
void ClassFileParser::parserConstantPool(ClassRead *classRead, InstanceKlass *klass) {
    klass->setConstantPool(new ConstantPool);
    ConstantPool *constantPool = klass->getConstantPool();
    constantPool->tag = new char[klass->getConstantPoolCount()];
    for (int i = 1; i < klass->getConstantPoolCount(); i++) {
        unsigned char tag = classRead->readByOneByte();
        *(constantPool->tag + i) = tag;
        switch (tag) {
        }
    }
};

```
   * 处理方式：
     * utf-8:长度是可变的，规定用2个字节指明了长度，于是先读取两个字节获取长度length，随后读取length个字节的内容，将其存储在ConstantPool中的data中。
     * Integer、Float:读取4个字节将其存储在ConstantPool中的data中。
     * Long、Double:占用两个ConstantPool->data索引，暂不处理。
     * Class、String:读取2个字节将其存储在ConstantPool中的data中。
     * Fieldref、Methodref、InterfaceMethodref、NameAndType: 申请4个字节的内存i，先读取2个字节存储在i的左16位，再读取两个字节存储在i的右16位。
<br/><br/>
     
于是我们的parserConstantPool方法变成了这样：

```c++
void ClassFileParser::parserConstantPool(ClassRead *classRead, InstanceKlass *klass) {
    klass->setConstantPool(new ConstantPool);//为常量池初始化内存空间
    ConstantPool *constantPool = klass->getConstantPool();
    constantPool->tag = new char[klass->getConstantPoolCount()];//根据常量池数量初始化常量池数据区的内存空间

    for (int i = 1; i < klass->getConstantPoolCount(); i++) {
        unsigned char tag = classRead->readByOneByte();//读取一个字节获取到类型的映射值
        *(constantPool->tag + i) = tag;//存储类型
        switch (tag) {//根据不同的类型，有不同的处理方式，主要是读取的字节数不同
            case CONSTANT_Utf8: {
                unsigned short len = classRead->readByTwoByte();//读两个字节作为utf-8的字节长度
                char *target = new char[len + 1]; //申请len+1长度的字节内存，c语言最后一位需要用'\0'填充
                classRead->readByFreeByte(len, target);//读取len个字节
                (constantPool->data[i]) = target;//存储在常量池
                printf("第%d个，类型utf-8，值%s\n", i, constantPool->data[i]);
                break;
            }
            case CONSTANT_Integer: {
                char *temp = new char;
                *temp = classRead->readByFourByte(); //读取四个字节
                constantPool->data[i] = temp;//存储在常量池
                printf("第%d个，类型Integer，值%d\n", i, *constantPool->data[i]);
                break;
            }
            case CONSTANT_Float: {
                char *temp = new char;
                *temp = classRead->readByFourByte();//读取四个字节
                constantPool->data[i] = temp;//存储在常量池
                printf("第%d个，类型Float，值%d\n", i, *constantPool->data[i]);
                break;
            }
            case CONSTANT_Long: {
                printf("暂不处理\n");
                break;
            }
            case CONSTANT_Double: {
                printf("暂不处理\n");
                break;
            }
            case CONSTANT_Class: {
                char *temp = new char;
                *temp = classRead->readByTwoByte();//读取两个字节
                constantPool->data[i] = temp;//存储在常量池
                printf("第%d个，类型Class，值%d\n", i, *constantPool->data[i]);
                break;
            }
            case CONSTANT_String: {
                char *temp = new char[3];
                *temp = classRead->readByTwoByte();//读取两个字节
                temp[2] = '\0';
                constantPool->data[i] = temp;//存储在常量池
                printf("第%d个，类型String，值%d\n", i, *constantPool->data[i]);
                break;
            }
            case CONSTANT_Fieldref:
            case CONSTANT_Methodref:
            case CONSTANT_InterfaceMethodref: {
                int *temp = new int;//申请四个字节的内存空间
                short classIndex = classRead->readByTwoByte();//读取两个字节
                short nameAndTypeIndex = classRead->readByTwoByte();//读取两个字节
                *temp = htonl(classIndex << 16 | nameAndTypeIndex);//左16位存储classIndex 右16为存储nameAndTypeIndex
                (constantPool->data[i]) = (char *) temp;//存储在常量池
                printf("第%d个，类型file、method、Interface Methodref，值%X\n", i, htonl(*(int *) constantPool->data[i]));
                break;
            }
            case CONSTANT_NameAndType: {
                int *temp = new int;
                short nameIndex = classRead->readByTwoByte();//读取两个字节
                short descriptorIndex = classRead->readByTwoByte();//读取两个字节
                *temp = htonl(nameIndex << 16 | descriptorIndex);//左16位存储classIndex 右16为存储nameAndTypeIndex
                (constantPool->data[i]) = (char *) temp;//存储在常量池
                printf("第%d个，类型NameAndType，值%X\n", i, htonl(*(int *) constantPool->data[i]));
                break;
            }
            default:
                break;
        }

    }
};

```
我们来做一个测试：

```c++
int main() {
    ClassRead *classRead = ClassRead::readByPath("/Users/e/Documents/github/JDK/out/production/JDK/HelloJVM.class");//换成你自己的path
    InstanceKlass *klass = ClassFileParser::Parser(classRead);
    return 0;
}

```

输出：<br/><br/>

```
class文件校验正确 	
次版本号：0 	
主版本号：52 	
版本校验正确 	
第1个，类型file、method、Interface Methodref，值60014 	
第2个，类型file、method、Interface Methodref，值150016 	
第3个，类型String，值23 	
第4个，类型file、method、Interface Methodref，值180019 	
第5个，类型Class，值26 	
第6个，类型Class，值27 	
第7个，类型utf-8，值<init> 	
第8个，类型utf-8，值()V 	
第9个，类型utf-8，值Code 	
第10个，类型utf-8，值LineNumberTable 	
第11个，类型utf-8，值LocalVariableTable 	
第12个，类型utf-8，值this 	
第13个，类型utf-8，值LHelloJVM; 	
第14个，类型utf-8，值main 	
第15个，类型utf-8，值([Ljava/lang/String;)V 	
第16个，类型utf-8，值args 	
第17个，类型utf-8，值[Ljava/lang/String; 	
第18个，类型utf-8，值SourceFile 	
第19个，类型utf-8，值HelloJVM.java 	
第20个，类型NameAndType，值70008 	
第21个，类型Class，值28 	
第22个，类型NameAndType，值1D001E 	
第23个，类型utf-8，值Hello JVM 	
第24个，类型Class，值31 	
第25个，类型NameAndType，值200021 	
第26个，类型utf-8，值HelloJVM 	
第27个，类型utf-8，值java/lang/Object 	
第28个，类型utf-8，值java/lang/System 	
第29个，类型utf-8，值out 	
第30个，类型utf-8，值Ljava/io/PrintStream; 	
第31个，类型utf-8，值java/io/PrintStream 	
第32个，类型utf-8，值println 	
第33个，类型utf-8，值(Ljava/lang/String;)V 	

```

小总结：本章节相信聪明的你已经深切地感受到了我们这一模块的工作要领：按照约定把Java类拆解用C++存起来。假如你对于代码理解很困难，没关系，看一下小测试的输出，只需要知道我们是在存储类信息就好。

---

* **<p id="访问权限">2.4解析“访问权限”：</p>**

  **本次commit :** a340887eaa2ef20cb609c3dd90dce7af54c8bb67

本章节较为简单，只有两个字节，用来表示Java类的访问权限，下面给出一张权限枚举表：

|  类型   | 映射值  |
|  ----  | ----  |
|   PUBLIC               |    0x0001 |
|   FINAL            |    0x0010 |
|  SUPER            |   0x0020 |
|  INTERFACE   |   0x0200 |
|   ABSTRACT              |    0x0300 |
|   SYNTHETIC             |    0x1000 |
|   ANNOTATION             |    0x2000 |
|   ENUM             |    0x4000 |

于是我们在InstanceKlass中新增accessFlags属性：
```c++
class InstanceKlass {
    int magic; //魔数，CAFEBABE:用来校验是否是.class文件
    short minorVersion; //JDK次版本号
    short majorVersion; //JDK主版本号
    short constantPoolCount;//常量数量
    ConstantPool *constantPool;//常量池数据
    short accessFlags;//类的访问权限 （新增）
};
```
在ClassFileParser中新建parserAccessFlags方法用来解析类的访问权限：

```c++
void ClassFileParser::parserAccessFlags(ClassRead *classRead, InstanceKlass *klass) {
    klass->setAccessFlags(classRead->readByTwoByte());//读取两个字节，set进InstanceKlass属性
    printf("访问权限：%d\n", klass->getAccessFlags());
};

```
记得在Parser方法中调用parserAccessFlags。

我们来做一个测试：

```c++
int main() {
    ClassRead *classRead = ClassRead::readByPath("/Users/e/Documents/github/JDK/out/production/JDK/HelloJVM.class");//换成你自己的path
    InstanceKlass *klass = ClassFileParser::Parser(classRead);
    return 0;
}

```

输出：<br/><br/>
访问权限：33 （PUBLIC SUPER）

小总结：本章节太简单不想总结了，就是顽皮。另外，本次提交修复了上一章节在解析常量池时留下的bug,多循环了一次。

---


* **<p id="类名">2.5解析“类名 && 父类名”（全限定名）：</p>**

  **本次commit :** 6ef6e3a6c4d41d86404fd26079b1468f88332f72

先来看一张图，我们已经解析到这里了：

<img src="https://github.com/ArosyW/picture/blob/master/serp.jpeg" width = "400" height = "500" />

<br/><br/>
“类名 && 父类名”也很简单，各自占用两个字节。需要注意的是，这两个字节表示的并不是真正的String类型的类名，而是一个Class常量池（前面章节解析的ConstantPool）索引，例如在此处，往下读两个字节是"00 05"，表示类名的常量池索引为5，
而常量池索引为5的值如下，是26，常量池索引为26的值即为我们的类名"HelloJVM"。另外这个名字是带有路径的，也就是全限定名，在项目中是唯一的，比如其父类名为"java/lang/Object"。

```

……
第4个，类型file、method、Interface Methodref，值180019
第5个，类型Class，值26
第6个，类型Class，值27
……
第26个，类型utf-8，值HelloJVM
……
```
老套路，InstanceKlass新增属性"类名"和"父类名"：

```c++

class InstanceKlass {
int magic; //魔数，CAFEBABE:用来校验是否是.class文件
……
short thisClass;//类名
short superClass;//父类名
}

```
ClassFileParser中新增parserThisClass、parserSuperClass方法分别解析类名与父类名：

```c++

void ClassFileParser::parserThisClass(ClassRead *classRead, InstanceKlass *klass) {
    klass->setThisClass(classRead->readByTwoByte());//读取两个字节，set进InstanceKlass属性
    printf("类名：%X\n", klass->getThisClass());
};

void ClassFileParser::parserSuperClass(ClassRead *classRead, InstanceKlass *klass) {
    klass->setSuperClass(classRead->readByTwoByte());//读取两个字节，set进InstanceKlass属性
    printf("父类名：%X\n", klass->getSuperClass());
};

```
保姆级提示：要记得在总的解析流程Parser方法中调用这两个新增的方法。

我们来做一个测试：

```c++
int main() {
    ClassRead *classRead = ClassRead::readByPath("/Users/e/Documents/github/JDK/out/production/JDK/HelloJVM.class");//换成你自己的path
    InstanceKlass *klass = ClassFileParser::Parser(classRead);
    int indexClass = *(klass->getConstantPool()->data[klass->getThisClass()]);//获取常量池索引
    int indexSuperClass = *(klass->getConstantPool()->data[klass->getSuperClass()]);//获取常量池索引
    printf("类名：%s\n", klass->getConstantPool()->data[indexClass]);//再次索引获取name
    printf("父类名：%s\n", klass->getConstantPool()->data[indexSuperClass]);//再次索引获取name
    return 0;
}

```
输出：<br/><br/>
类名：HelloJVM<br/>
父类名：java/lang/Object

小总结：把四个字节set进InstanceKlass对象，总结完了。

---

* **<p id="接口">2.6解析"接口"：</p>**


  **本次commit :** 8c4245d1fb835ddf1803b916477cd9c2df8a1641


由于Java可实现多个接口，意味着接口的数量是可变的，所以先用两个字节指明接口数量count，接下来才是接口。那么我们只需要for循环count次读取接口信息就可以完成接口的解析,所谓的解析，就是往后读取两个字节获得常量池索引，存储到InstanceKlass属性中
,与前文中的解析"类名"一样，这个常量池索引的值又是一个常量池索引，意味着需要在常量池索引两次，就可以获取到接口的全限定名。
<br/>
于是创建C++类InterfacesInfo用来描述接口的常量池索引和名字:
```c++
class InterfacesInfo {
    short constantPoolIndex;
    string interfacesName;
};
```
在InstanceKlass中新增接口数量interfacesCount和接口属性interfaces:

```c++

class InstanceKlass {
    int magic; //魔数，CAFEBABE:用来校验是否是.class文件
……
    short interfacesCount;//接口数量
    InterfacesInfo *interfaces;//接口
}

```
ClassFileParser中新建解析接口数量和解析接口的方法parserInterfacesCount、parserInterfaces：

```c++

void ClassFileParser::parserInterfacesCount(ClassRead *classRead, InstanceKlass *klass) {
    klass->setInterfacesCount(classRead->readByTwoByte());
    printf("接口数量：%d\n", klass->getInterfacesCount());
}

void ClassFileParser::parserInterfaces(ClassRead *classRead, InstanceKlass *klass) {
    if (klass->getInterfacesCount() == 0) {
        return;//未实现接口就不解析了
    }
    klass->setInterfaces(new InterfacesInfo[klass->getInterfacesCount()]);
    InterfacesInfo *interfaces = klass->getInterfaces();
    for (int i = 0; i < klass->getInterfacesCount(); i++) {
        unsigned short constantPoolIndex = classRead->readByTwoByte();//往后读取两个字节作为常量池索引
        int index = *(klass->getConstantPool()->data[constantPoolIndex]);//获取新的常量池索引
        string name = klass->getConstantPool()->data[index];//第二次索引取得全限定名
        printf("第%d个接口，name:%s,索引位置：%d", i + 1, name.c_str(), constantPoolIndex);
        *(interfaces + i) = *(new InterfacesInfo(constantPoolIndex, name));//将接口set进InstanceKlass属性中
    }
};
```
我们来做一个测试：

```c++
int main() {
    ClassRead *classRead = ClassRead::readByPath("/Users/e/Documents/github/JDK/out/production/JDK/HelloJVM.class");//换成你自己的path
    InstanceKlass *klass = ClassFileParser::Parser(classRead);
    return 0;
}
```
输出：<br/><br/>
接口数量：0

<br/>是的，我们的HelloJVM没有实现接口。
<br/><br/>

小总结：本章节共读了4个字节，分别表示接口数量、接口名字的常量池索引，然后将他们存储在InstanceKlass属性中。

---


* **<p id="字段">2.7解析"类字段"：</p>**


  **本次commit :** 0ccbeddf7e67d6bc166e26f6271581dcd2890df7

"类字段"指的是成员变量，为了模拟这种情况，我们修改一下我们的Java代码，新定义一个"out"属性，然后编译，如下：
```java
public class HelloJVM {
    static int out = 9;
    public static void main(String[] args) {
        System.out.println("Hello JVM");
    }
}
```
由于字段的数量也是可变的，所以先由两个字节指明字段数量count，然后每一个字段给出8个字节，按顺序这8个字节表示的含义分别为：
访问权限、name的常量池索引、类型描述的常量池索引、附加属性数量length，最后解析length个附加属性。对这些含义做个解释，
<br/><br/>
访问权限给出权限映射表：

|  类型   | 映射值  | 
|  ----  | ----  |
|   PUBLIC               |    0x0001 |
|   FINAL            |    0x0010 |
|  PRIVATE            |   0x002 |
|  PROTECTED   |   0x0004 |
|   STATIC              |    0x0008 |
|   VOLATILE             |    0x0040 |
|   TRANSIENT             |    0x0080 |
|   SYNTHETIC             |    0x1000 |
|   ENUM             |    0x4000 |

name的常量池索引：用该索引查到常量池ConstantPool可以获得字段的名字。
<br/><br/>
类型描述的常量池索引：用该索引查到常量池ConstantPool可以获得字段的类型描述符号，下面给出类型与描述符的对应关系：

|  常量池类型描述符   | 类型  |
|  ----  | ----  |
|   B               |    byte |
|   C            |    char |
|  D            |   double |
|  F   |   float |
|   I              |    int |
|   J             |    long |
|   S             |    short |
|   Z             |    boolean |
|   V             |    void |
|   L             |    对象类型，如Ljava/lang/Object |

附加属性：字段可以在附加属性有更多的描述，此处我们为0，属性的详细内容放到后面讲。


我们新建FieldsInfo类用来描述字段：
```c++
class FieldsInfo {
    short accessFlag;
    short nameIndex;
    short descriptorIndex;
    short attributesCount;
};
```
InstanceKlass中新增字段数量fieldsCount和字段fieldsInfo：

```c++

class InstanceKlass {
    int magic; //魔数，CAFEBABE:用来校验是否是.class文件
……
    short fieldsCount;//字段数量
    FieldsInfo *fieldsInfo;//字段
}

```
ClassFileParser中新建解析接口数量和解析接口的方法parserInterfacesCount、parserInterfaces：
```c++
void ClassFileParser::parserFieldsCount(ClassRead *classRead, InstanceKlass *klass) {
    klass->setFieldsCount(classRead->readByTwoByte());//往后读取两个字节set进InstanceKlass中的字段数量属性
    printf("字段数量：%d\n", klass->getFieldsCount());
}

void ClassFileParser::parserFieldsInfo(ClassRead *classRead, InstanceKlass *klass) {
    klass->setFieldsInfo(new FieldsInfo[klass->getFieldsCount()]);//按照字段数量初始化字段内存空间
    FieldsInfo *fieldsInfo = klass->getFieldsInfo();
    for (int i = 0; i < klass->getFieldsCount(); i++) {//循环字段数量count次，解析字段
        FieldsInfo *fields = (fieldsInfo + i);
        fields->setAccessFlag(classRead->readByTwoByte());//往后读取两个字节set进FieldsInfo的访问权限
        fields->setNameIndex(classRead->readByTwoByte());//往后读取两个字节set进FieldsInfo的name
        fields->setDescriptorIndex(classRead->readByTwoByte());//往后读取两个字节set进FieldsInfo的类型描述
        fields->setAttributesCount(classRead->readByTwoByte());//往后读取两个字节set进FieldsInfo的附加属性数量
        printf("field 解析：\n access:%X,\n nameIndex:%X\n descriptorIndex:%X \n attributesCount:%X\n",
               fields->getAccessFlag(), fields->getNameIndex(), fields->getDescriptorIndex(),
               fields->getAttributesCount());
    }
};
```
我们来做一个测试：

```c++
int main() {
    ClassRead *classRead = ClassRead::readByPath("/Users/qsc/Documents/github/JDK/out/production/JDK/HelloJVM.class");//换成你自己的path
    InstanceKlass *klass = ClassFileParser::Parser(classRead);
    printf("字段访问权限：%d\n", klass->getFieldsInfo()->getAccessFlag());
    printf("字段名字：%s\n",klass->getConstantPool()->data[klass->getFieldsInfo()->getNameIndex()]);
    printf("字段类型：%s\n",klass->getConstantPool()->data[klass->getFieldsInfo()->getDescriptorIndex()]);
    printf("字段附加属性数量：%d\n", klass->getFieldsInfo()->getAttributesCount());
    return 0;
}
```
输出：<br/><br/>
字段访问权限：8<br/>
字段名字：out<br/>
字段类型：I<br/>
字段附加属性数量：0<br/>

小总结：按照约定的字段结构成功存储进了InstanceKlass

|  字段数量   | 访问权限  | 名字索引  | 类型索引  |  附加属性数量  | 附加属性  |
|  ----  | ----  | ----  | ----  | ----  | ----  |
|   2字节 |    2字节 |  2字节 |  2字节 | 2字节 | *字节 |

---

* **<p id="方法">2.7解析"方法"：</p>**

  **本次commit :** 0a96cbf9fe2199bdb7cd19d55b29a7d4edcd39e7

方法的数量是不确定的，因此先用两个字节指明方法数量count。下面是每一个方法所包含的属性：

  | 访问权限  | 名字索引  | 方法描述索引  |  附加属性数量  | 附加属性  |
 | ----  | ----  | ----  | ----  | ----  |
 |    2字节 |  2字节 |  2字节 | 2字节 | *字节 |

其中附加属性是用来修饰这个方法的一些额外的信息，例如方法一般包括一个称为Code的属性，是JVM提供的指令的集合，用来描述这个方法如何执行的，"指令"后面在执行器篇将会详细讲解，此处我们仅仅需要把它存储在InstanceKlass即可。

其他的附加属性过多，不列在这里了，可以查阅《Java虚拟机规范》4.7章节。

下面给出附加属性Code的结构：

 | 名字索引  | 当前属性长度  |  最大栈深度  | 当前方法的局部变量表个数  | Code（指令）长度length  | 指令  | exception_table_length  |  exception_table |  附加属性数量  | 附加属性  |
 | ----  | ----  | ----  | ----  | ----  | ----  |  ----  |  ----  | ----  |  ----  | 
|    2字节 |  4字节 |  2字节 | 2字节 | 4字节  | length字节 | 2字节  | *字节 | 2字节  | *字节


想必看完这么长的表格你已经疯掉了吧，稍微解释一下。

* 名字索引：Class常量池索引
* 当前属性长度：当前属性即名为"Code"的属性，指明了这个属性此次占用多少字节。
* 最大栈深度：方法在执行的时候会在栈中创建"栈帧"，也称为"操作数栈"，最大栈深度即指方法在执行时操作数栈中最大容纳的操作数的数量。
* 当前方法的局部变量表个数：方法中声明的局部变量的数量
* Code（指令）长度length：当前方法的JVM指令的个数
* 指令：当前方法的JVM指令
* exception_table_length：异常处理器的个数，一般方法中有try、catch才会有exception_table。
* exception_table：包含4个属性，我们暂不处理异常，可查阅《Java虚拟机规范》4.7.3
* 附加属性：一般为LineNumberTable、LocalVariableTable，属于调试信息，不是运行时必须的，不必太关注，按照约定存储起来就好。
  * LineNumberTable：

    | name的常量池索引  | 属性长度  | tableLength  |  table 
    | ----  | ----  | ----  | ----  | 
    |    2字节 |  4字节 |  2字节 |  *字节 | 

      给出table的结构：

      | start_pc  | 行号  | 
    | ----  | ----  | 
    |    2字节 |  2字节 |  

  * LocalVariableTable：
  
    | name的常量池索引  | 属性长度  | 局部变量Length   |  table
    | ----  | ----  | ----  | ----  
    |    2字节 |  4字节 |  2字节 |  *字节

    给出局部变量table的结构：

    | start_pc  | length  | name常量池索引  | 类型的常量池索引  | 局部变量表索引
    | ----  | ----  | ----  | ----  | ----  | 
    |    2字节 |  2字节 |   2字节 |   2字节 |  2字节 |  

<br/>
方法的解析东西略微多一些，显然我们需要嵌套for循环去解析，上面已经列出了我们解析所需要的所有约定，大部分与我们初步实现JVM无关，按约定解析就好，无需纠结他们是做什么用的。需要重点关注的是"指令"，这是我们后面执行方法所依赖的。
<br/>

于是我们需要新建C++类来映射这些属性，以达到存储的目的：
MethodInfo、CodeAttributeInfo、AttributeInfo、LineNumberTable、LocalVariableTable、BytecodeStream。

新增方法parserMethodCount、parserMethod、parserLineNumberTable、parseLocalVariableTable分别用来解析方法数量、解析方法、解析LineNumberTable、解析LocalVariableTable。

重点看一下parserMethod方法：
```c++
void ClassFileParser::parserMethodInfo(ClassRead *classRead, InstanceKlass *klass) {
    klass->setMethodInfo(new MethodInfo[klass->getMethodCount()]);//初始化InstanceKlass中的methodInfo的内存空间
    for (int i = 0; i < klass->getMethodCount(); i++) {
        MethodInfo *method = new MethodInfo;
        method->setBelongKlass(klass);//将方法与所属InstanceKlass关联
        method->setAccessFlags(classRead->readByTwoByte());//存储方法的访问权限
        method->setNameIndex(classRead->readByTwoByte());//存储方法的访问权限
        method->setMethodName((string) klass->getConstantPool()->data[method->getNameIndex()]);//存储方法的名字
        printf("解析方法%s\n", method->getMethodName().c_str());
        method->setDescriptorIndex(classRead->readByTwoByte());//存储方法的描述（包含了参数、返回值）
        method->setAttributesCount(classRead->readByTwoByte());//存储属性数量
        method->initCodeAttributeInfo();
        *(klass->getMethodInfo() + i) = *method;
        for (int j = 0; j < method->getAttributesCount(); j++) {
            CodeAttributeInfo *codeAttributeInfo = new CodeAttributeInfo;
            codeAttributeInfo->setAttrNameIndex(classRead->readByTwoByte());//存储属性名字的常量池索引
            codeAttributeInfo->setAttrLength(classRead->readByFourByte());//存储属性长度
            codeAttributeInfo->setMaxStack(classRead->readByTwoByte());//存储最大栈深度
            codeAttributeInfo->setMaxLocals(classRead->readByTwoByte());//存储局部变量表数量
            codeAttributeInfo->setCodeLength(classRead->readByFourByte());//存储指令数量
            BytecodeStream *bytecodeStream = new BytecodeStream(method, codeAttributeInfo,
                                                                codeAttributeInfo->getCodeLength(), 0,
                                                                new char[codeAttributeInfo->getCodeLength()]);
            classRead->readByFreeByte(codeAttributeInfo->getCodeLength(), bytecodeStream->getCodes()); //重点： 将方法的JVM指令存储在bytecodeStream
            codeAttributeInfo->setCode(bytecodeStream);
            printf("\t第%d个属性，access flag:%X name index : %X  stack:%X container:%X  code length:%X \n", j,
                   method->getAccessFlags(), codeAttributeInfo->getAttrNameIndex(), codeAttributeInfo->getMaxStack(),
                   codeAttributeInfo->getMaxLocals(), codeAttributeInfo->getCodeLength());
            codeAttributeInfo->setExceptionTableLength(classRead->readByTwoByte());//存储Exception表长度，此处暂时为0，因为我们没有任何异常需要处理
            codeAttributeInfo->setAttributesCount(classRead->readByTwoByte());//存储属性长度
            method->setAttributeInfo(codeAttributeInfo, j);
            for (int k = 0; k < codeAttributeInfo->getAttributesCount(); k++) {//循环解析属性
                int nameIndex = classRead->readByTwoByte();
                string attrName = klass->getConstantPool()->data[nameIndex];
                if ("LineNumberTable" == attrName) {//解析LineNumberTable
                    parserLineNumberTable(classRead, codeAttributeInfo, attrName, nameIndex, klass);
                } else if ("LocalVariableTable" == attrName) {//解析LocalVariableTable
                    parseLocalVariableTable(classRead, codeAttributeInfo, attrName, nameIndex, klass);
                }
            }
        }
    }
};

```

更详细的解析请看本次commit: 0a96cbf9fe2199bdb7cd19d55b29a7d4edcd39e7
<br/><br/>
我们来做一个测试：

```c++
int main() {
    ClassRead *classRead = ClassRead::readByPath("/Users/qsc/Documents/github/JDK/out/production/JDK/HelloJVM.class");//换成你自己的path
    InstanceKlass *klass = ClassFileParser::Parser(classRead);
    return 0;
}
```
<br/>

输出：<br/>

```
方法数量：3
解析方法<init>
第0个属性，access flag:1 name index : B  stack:1 container:1  code length:5
lineNumberTable: name index:12,attr len:6, table len:1
第0个属性，start pc : 0,line numnber:1
LocalVariableTable:第0个属性，start pc:0,length:5,name index:14,descrip:15,index:0
解析方法main
第0个属性，access flag:9 name index : B  stack:2 container:1  code length:8
lineNumberTable: name index:12,attr len:10, table len:2
第0个属性，start pc : 0,line numnber:4
lineNumberTable: name index:12,attr len:10, table len:2
第1个属性，start pc : 7,line numnber:5
LocalVariableTable:第0个属性，start pc:0,length:8,name index:18,descrip:19,index:0
解析方法<clinit>
第0个属性，access flag:8 name index : B  stack:1 container:0  code length:6
lineNumberTable: name index:12,attr len:6, table len:1
第0个属性，start pc : 0,line numnber:2
```
小总结：如果这对你来说过于陌生，你只需要知道我们把方法的JVM指令存起来了就可以跳过本部分了。另外.class的解析我们已经走了99%了。

---

* **<p id="属性">2.7解析"属性"：</p>**

  **本次commit :** 4a6fbf0dda391153ea2eb8f706cff67bfb7655ba

<br/>
有个好消息，解析"属性"是.class文件解析的最后一章节。

此处的属性与我们在解析字段和方法时遇到的"附加属性"是相同的结构，由于属性的枚举数量过多，此处不给出了，详细的枚举解析可查阅《Java虚拟机规范》4.7章节。
我们的Java类"HelloJVM"是包含一个称为"SourceFile"的属性。同LineNumberTable、LocalVariableTable一样，属于调试信息，不是运行时必须的，不必过于关注。
按照下面给出的约定存储在InstanceKlass中就好。

|  属性name的常量池索引  | 属性长度length   | 文件名的常量池索引  | 
|  ----   |  ----   | ----   | 
|  2字节  |  4字节  |   2字节  | 

于是我们在InstanceKlass中新增attributeCount属性数量、attributeInfo属性：
```c++

class InstanceKlass {
    int magic; //魔数，CAFEBABE:用来校验是否是.class文件
……
    short attributeCount;
    AttributeInfo *attributeInfo;
}

```
在ClassFileParser中新增parserAttributeCount、parserAttribute方法用来解析属性数量和属性：

```c++

void ClassFileParser::parserAttributeCount(ClassRead *classRead, InstanceKlass *klass) {
    klass->setAttributeCount(classRead->readByTwoByte());//读取两个字节将属性数量存储在InstanceKlass中
    printf("解析属性，数量%d\n", klass->getAttributeCount());
};

void ClassFileParser::parserAttribute(ClassRead *classRead, InstanceKlass *klass) {
    for (int i = 0; i < klass->getAttributeCount(); i++) {
        short nameIndex = classRead->readByTwoByte();
        string attrName = klass->getConstantPool()->data[nameIndex];
        if ("SourceFile" == attrName) {
        printf("\tSourceFile\n");
        parseSourceFile(classRead, klass, nameIndex, i);//解析SourceFile属性
        } else {
            printf("\t无法识别的属性:%X\n", attrName.c_str());
        }
    }
};

void ClassFileParser::parseSourceFile(ClassRead *classRead, InstanceKlass *klass, short nameIndex, short index) {
    klass->setAttributeInfo(new AttributeInfo[klass->getAttributeCount()]);//初始化内存空间来存储属性
    AttributeInfo *attributeInfo= klass->getAttributeInfo();
    *(klass->getAttributeInfo() + index) = *attributeInfo;
    attributeInfo->setAttributeNameIndex(nameIndex);//存储属性name的常量池索引
    attributeInfo->setAttributeLength(classRead->readByFourByte());//存储属性长度length
    attributeInfo->initContainer();
    *(attributeInfo->getContainer()) = classRead->readByTwoByte();//存储文件名的常量池索引
    printf("\t\t第%d个属性，%s:nameIndex:%d,length:%d,data:%d,(%s)\n", index, klass->getConstantPool()->data[nameIndex],
    attributeInfo->getAttributeNameIndex(), attributeInfo->getAttributeLength(), *attributeInfo->getContainer(),
    klass->getConstantPool()->data[*attributeInfo->getContainer()]);
};


```
<br/><br/>
我们来做一个测试：

```c++
int main() {
    ClassRead *classRead = ClassRead::readByPath("/Users/e/Documents/github/JDK/out/production/JDK/HelloJVM.class");//换成你自己的path
    InstanceKlass *klass = ClassFileParser::Parser(classRead);
    int attrIndex = klass->getAttributeInfo()->getAttributeNameIndex();
    int length = klass->getAttributeInfo()->getAttributeLength();
    short fileIndex = *(short *)(klass->getAttributeInfo()->getContainer());
    printf("解析属性完成，属性名字：%s,长度：%d，文件名字：%s\n", klass->getConstantPool()->data[attrIndex],
           length,
           klass->getConstantPool()->data[fileIndex]);
    return 0;
}
```
<br/>

输出：<br/>

解析属性完成，属性名字：SourceFile,长度：2，文件名字：HelloJVM.java

小总结：由于我们的Java类HelloJVM只有一个SourceFile属性，为了快速的实现我们的简陋的JVM，我们此处仅仅写了这一种属性的解析逻辑。
再次指路，其他属性的结构请移步《Java虚拟机规范》4.7章节。


**<p id="类加载器">3.类加载器：</p>**

---
* 新增类加载器
  
  **本次commit :** e6906988a7e5e1b89c6197d2acefb7c623a726d8

<br/>
在前文中我们实现了.class文件的解析，
但一个类的加载不仅仅有.class文件的解析，后续我们来会有其他操作，例如执行’clinit‘
方法，因此我们继续对ClassFileParser进行封装，直到一个Java类能够真正的被初始化完成，然后才能对外使用。
于是我们新建BootClassLoader:

```c++
class BootClassLoader {
public:
    static string prePath;//路径前缀 ：项目地址
    static map<string, InstanceKlass *> allClass;//所有加载完成可以使用的Java类
    static InstanceKlass* loadKlass(string& path);//根据路径（全限定名）加载Java类
};
```
 
用loadKlass方法加载Java类，用allClass存储所有被加载完成的Java类：
 
```c++
InstanceKlass *BootClassLoader::loadKlass(string &p) {
    string path = p.append(".class");
    string tempPath = prePath;
    InstanceKlass *klass = ClassFileParser::Parser(ClassRead::readByPath(tempPath.append(path)));
    allClass[p] = klass;
    return klass;
}
```

为了避免重复加载，我们在方法的开始加一个逻辑，先通过allClass判断这个类是否已经加载过了：
 
```c++
InstanceKlass *BootClassLoader::loadKlass(string &p) {
    if (allClass[p] != 0 ) {//先判断此类是否加载过
        return allClass[p];
    }
    string path = p.append(".class");
    string tempPath = prePath;
    InstanceKlass *klass = ClassFileParser::Parser(ClassRead::readByPath(tempPath.append(path)));//读取.class到内存，然后解析它
    allClass[p] = klass;//将解析完的klass存起来
    return klass;
}
```
 
即使这样，我们的loadKlass仍然是不完善的，在后面的章节我们会对它进行加锁以及介绍并执行‘clinit‘方法。

<br/>
我们来做一个测试：

```c++
int main() {
    string path = "HelloJVM";
    InstanceKlass *klass = BootClassLoader::loadKlass(path);
    int index = *klass->getConstantPool()->data[klass->getThisClass()];//获取该类的类名索引
    printf("%s\n", klass->getConstantPool()->data[index]);//取出类名
    return 0;
}
```
<br/>

输出：<br/>

HelloJVM

小总结：为了后面完整的加载Java类，我们新增了类加载器，并用map将加载完成的类存起来，当然为了更快的实现这个简陋的JVM，我们不再考虑双亲委派等加载实现。当然这里缺少线程安全的逻辑，下次一定。
<br/><br/>
 
### (二)字节码执行器

---

Java方法的执行就是字节码执行器执行指令的过程。有必要先讲一下cpu是如何执行指令的。
 >PS：为什么需要讲一下cpu是如何执行的，此段启发自《揭秘Java虚拟机》 作者：封亚飞，特意贴出，以示尊敬。这本书里面讲的过于细致了，我担心你看得丧失信心。所以此处用我自己的逻辑简写一段。

<br/><br/>
 **<p id="cpu是如何执行指令的">1.cpu是如何执行指令的：</p>**
 
 ---

  计算机所能执行的只有机器码，例如x86指令集中的：
  <br/>  <br/>
    FNOP       无操作        机器码 ： DA E9
  <br/>  <br/>
 cpu会从一段连续的内存空间的起始位置开始一个字节一个字节的读取，通过译码器去匹配"指令集"，如果匹配不到就继续取一个字节，直到匹配到正确的指令（匹配这件事情是译码器的工作，我们无需过于关注）。每一条指令都有特定的"操作数"，其实就是这条指令的"参数"，下面给出一个例子：
<br/>  <br/>
  MOV AL,1
  <br/>  <br/>
这条指令的意思是将AL寄存器的值设置为 1 ，它的机器码是1011 0000 0000 0001 （二进制），当cpu读取到前八位"操作码"：1011 0000时就可以通过译码器知道操作的是：为AL寄存器赋值。赋值为多少呢？后面紧
跟一个字节的"操作数"就是要赋的值，那么如何知道后面跟的参数的长度呢？其实已经包含在了前八位的操作码里，当cpu读到这个操作码时就默认后面跟一个8位的操作数。
那么cpu只需要再往后读8位得到0000 0001就可以完整地完成这条指令的执行。可以说"操作码"与"操作数"的长度是约定好的。
  <br/>  <br/>
 
 **<p id="cpu是如何执行方法的">2.cpu是如何执行方法的：</p>**
 
 ---

**而所谓的方法，就是多个指令的集合，且在一段连续的内存空间内**，在上一段cpu执行完一条指令时，会继续往下读取下一条指令，当译码器无法匹配到合法的指令时将会发生异常。既然所有的方法都会被编译成机器码在一段连续的内存空间中存放着，那么想要执行这个方法，只需要让cpu跳转到这段内存空间的首地址即可，那就需要一个能实现跳转的指令：jmp，它是无条件跳转指令。
<br/><br/>
但它自己并不能完成方法的调用，因为它无法"返回"调用者继续执行调用者的代码。因此我们需要的是:call指令配合ret指令，它可以保存调用者的下一条指令，当执行ret完时自动执行调用者的下一条指令。
<br/><br/>
还有一个问题没有解决，就是参数与返回值如何传递。来一段代码：

```c++
int main() {
    func(2);   
    return 0;
}
int func(int arg){
    return arg;
}
```

编译后：

```c++
main: 
 pushq  %rbp  //保存调用者栈基地址
 movq   %rsp, %rbp //保存调用者栈基地址
 subq   $0x10, %rsp //为当前方法分配16个字节栈空间 
 movl   $0x0, -0x4(%rbp) // 将0存入到 rbp寄存器往下四个字节的位置
 movl   $0x2, %edi //将 2 存入edi寄存器
 callq  func //存储下一条指令（xorl） 然后 调用 func方法
 xorl   %ecx, %ecx
 movl   %eax, -0x8(%rbp) //将eax寄存器中的值赋值给rbp往下8个字节的位置
 movl   %ecx, %eax
 addq   $0x10, %rsp
 popq   %rbp

func:
 pushq  %rbp //保存调用者栈基地址
 movq   %rsp, %rbp //保存调用者栈基地址
 movl   %edi, -0x4(%rbp) //将edi寄存器的值存入到 rbp寄存器往下四个字节的位置
 movl   -0x4(%rbp), %eax //rbp寄存器往下四个字节的位置 存入到 eax寄存器
 popq   %rbp 
 retq  
```

  * 参数传递过程：
  
     * 我们的参数是2。
    
     * 在main方法的第五条命令（movl   $0x2, %edi），将2 放入了edi寄存器。
     
     * 然后在func方法第三条命令（movl   %edi, -0x4(%rbp)）将edi寄存器中的数据取出放进了func方法自己的栈内存当中。
     
     * 显然计算机执行方法时参数的传递是通过寄存器中转实现的。
     
     * 但当一个方法参数足够多的时候，寄存器的数量显然是不够用的。此时可以使用栈内存来进行传递。

    
  * 返回值传递过程：
 
    * func方法第四条命令（movl   -0x4(%rbp), %eax），将rbp寄存器往下四个字节的位置的值赋给了eax寄存器。
    而-0x4(%rbp)这个位置的值就是在func第三条指令中放入的，其实就是func方法的返回值。
    
    * 还可以看到在main方法当中第八条指令（movl   %eax, -0x8(%rbp)），将eax寄存器中的值赋值给了栈内存，这就是取出func方法返回值的过程，所以，返回值的传递也是通过寄存器中转实现的，如果返回值过多或不能使用寄存器时，也会使用栈内存就行传递。
    


执行main方法，还没执行func方法时的内存图：
<br/><br/>
<img src="https://github.com/ArosyW/picture/blob/master/main.png" width = "600" height = "500" />

执行func方法时的内存图：
<br/><br/>
<img src="https://github.com/ArosyW/picture/blob/master/func.png" width = "600" height = "500" />
<br/><br/>
可以看出rbp寄存器 与 rsp寄存器 始终是指向栈底与栈顶的，而且调用者与被调用者的栈之间隔着16个字节的内存地址用于存放调用者的下一条指令，那么其实被调用者的栈底地址加16字节就是调用者的栈顶地址，这样
就很容易得到被调用者栈内的任何数据了，这也是为什么能够使用栈内存来进行参数\返回值传递的原因。
<br/> <br/> 
当func方法执行完成后，rbp寄存器 与 rsp寄存器会重新指向main方法的栈底与栈顶。
<br/> <br/>
小总结：**计算机执行方法的方式就是通过跳转指令，然后通过寄存器与栈内存进行参数与返回值的传递。而指令的格式则是：操作码+（多个）操作数**。为实现我们简陋的JVM，理解到这一句话就足够了。如果想要看更详细的cpu执行指令、方法流程的，不巧，鄙人也给你准备了，点这<a href="#cpu如何执行指令与方法">`cpu如何执行指令与方法`</a>
 <br/>

  **<p id="JVM如何执行方法的">3.JVM如何执行方法的：</p>**

**<p id="初始字节码指令">3.1 初始字节码指令：</p>**

JVM全称Java虚拟机，所谓虚拟机，即是虚拟的计算机，再换句话说，它模拟了计算机执行指令的过程。

  * JVM指令集：

既然是模拟了真实的计算机，那么它必然有自己的指令集，类似于cpu所能执行的 **操作码+（多个）操作数** 形式的指令，JVM也定义了自己的指令集，但是cpu能够执行指令是因为这是焊死在硬件里的，通过"译码器"可以识别。而JVM自己自定义的指令值是不能被cpu所能识别的，因此JVM需要
自己实现逻辑去解释这些指令到底是啥意思，最终还是会被编译成cpu所能指别的指令集才能被cpu正确执行。

  * 参数/返回值的传递：
  
在前文我们看到cpu执行指令与方法，无论是参数的传递还是局部变量的存储，无非就是在寄存器与栈内存之间跳来跳去，可以说**真实的计算机的指令集是基于寄存器&&栈的**。而JVM既然是虚拟的计算机，那它也必然需要实现参数的传递，才能实现方法的调用，才能像真实的计算机一样去执行方法。
有一点不同，**JVM的指令是基于栈**，它的指令实现只涉及对栈的操作。

  * 举一个凶猛的栗子🌰：<br/><br/>
    ACONST_NULL  1
    <br/><br/>
  这条指令的意思是将NULL推向栈顶。ACONST_NULL 这个单词只是帮助我们记忆的助记符，真实值是 1 ，当执行引擎读取到1的时候就会执行将"NULL推向栈顶"的逻辑。后面**章节会再详细介绍字节码指令。

**<p id="方法执行理论">3.2 方法执行理论：</p>**



### (六)扩展内容
 
**<a id="cpu如何执行指令与方法">1.cpu如何执行指令与方法：</a>**




