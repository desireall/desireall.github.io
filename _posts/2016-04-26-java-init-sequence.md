---
layout:     post
title : "java init sequence"
subtitle:   "\" \""
date:       2016-04-26 12:00:00
author:     "YuHuo"
header-img: "img/post-bg-digital-native.jpg"
catalog: true
tags:
    - java
---
Java中类成员初始化顺序，十分基础但同时也是容易出错的地方。现根据《Thinking in Java》相关章节及自己的理解举例来说明类初始化顺序。    



## 单一对象的初始化  

废话先不说，先上一段代码：

~~~ java  

class Student {
    int age = defaultAge();
    static int height = defaultHeight();
    public Student() {
        System.out.println("construct " + this.age);
    }
    private int defaultAge() {
        System.out.println(age + " defaultAge");
        return 10;
    }
    static {
        System.out.println("static block");
    }
    {
        System.out.println("non-static block");
    }
    static int defaultHeight() {
        System.out.println(height + " defaultHeight");
        return 170;
    }
}
public class InitOrder {
    public static void main(String[] args) {
        new Student();
    }
}  

~~~


上面这一小段程序的输出顺序为：

~~~ 
0 defaultHeight
static block
0 defaultAge
non-static block
construct 10
~~~


从输出的第一行可以看出，static的height变量在被defaultHeight()赋值前已经有了初始值，我这里称这一步为**默认初始化**这一点非常重要，所有的成员变量（不论是不是static的）在声明时如果显式进行了赋值，那么这个成员变量至少会被初始化两次，一次是编译器自动的初始化（基本类型像int、false初始化为0、false，对象初始化为null）也就是默认初始化，然后是**显式初始化**。



对于输出的其他行，大家应该没什么大问题，两大基本原则就是：  

**1.有static先初始化static，然后是非static的**  
**2.显式初始化（如果有的话），构造块初始化（如果有的话），最后调用构造函数进行初始化**

《Thinking in java》中总结创建对象时成员的初始化过程（以一个Dog.java类为例）：  

1.首先需要明确的是，虽然构造函数没有显式用static声明，但所以类的构造函数都是static的[^1]。所以当第一次创建Dog对象或者访问Dog类中的static方法/成员，JVM必须通过classpath定位Dog.class文件。  

2.当在classpath中找到Dog.class文件时JVM会进行加载，这时Dog类会对static相关部分（包括static成员变量与static块）进行默认初始化，而且也`仅会执行这一次`。  

3.当我们用new Dog()创建Dog对象时，JVM会在heap上为其分配空间（由此可以看出类的static成员并不再这个类对象的heap中，我理解的应该在一个全局区的heap上）。  

4.在为heap上的非static成员分配空间的同时按照其类型进行默认初始化（我这里这么理解的，在为成员变量分配空间时会按照其类型进行第一次初始化，如果不进行默认初始化，分配的空间的值是随机的，有点类型c语言中的野指针问题）。  

5.如果成员变量被显式赋值过，现在进行显式初始化。  

6.执行构造函数。





上面这6步大家务必好好理解，下面以阿里巴巴2014校招笔试题来练手，看看大家是否真正理解：

~~~ java  


public class Alibaba {
    public static int k = 0;
    public static Alibaba t1 = new Alibaba("t1");
    public static Alibaba t2 = new Alibaba("t2");
    public static int i = print("i");
    public static int n = 99;
    private int a = 0;
    public int j = print("j");
    {
        print("构造块");
    }
    static {
        print("静态块");
    }
    public Alibaba(String str) {
        System.out.println((++k) + ":" + str + "   i=" + i + "    n=" + n);
        ++i;
        ++n;
    }
    public static int print(String str) {
        System.out.println((++k) + ":" + str + "   i=" + i + "    n=" + n);
        ++n;
        return ++i;
    }
    public static void main(String args[]) {
        Alibaba t = new Alibaba("init");
    }
}

~~~

我当时看到这个题，就觉得与阿里无缘了。 囧

~~~
1:j   i=0    n=0
2:构造块   i=1    n=1
3:t1   i=2    n=2
4:j   i=3    n=3
5:构造块   i=4    n=4
6:t2   i=5    n=5
7:i   i=6    n=6
8:静态块   i=7    n=99
9:j   i=8    n=100
10:构造块   i=9    n=101
11:init   i=10    n=102
~~~

上面是程序的输出结果，下面我来一行行分析之。

1.我们要调用Alibaba类的static方法main，所以JVM会在classpath中加载Alibaba.class，然后对所有static的成员变量进行默认初始化。这时k、i、n被赋值为0，t1、t2被赋值为null。

2.又因为static变量被显式赋值了，所以进行显式初始化，当对t1进行显式赋值时，用new的方法调用了Alibaba的构造函数，所以这次需要对t1对象进行初始化。因为Alibaba所有static部分已经在第一步中初始化过了（虽然第一步中还没有执行static块，但在初始化t1时也不会去执行static块，因为JVM认为这次是第二次加载Alibaba这个类了，所有的static部分都会被忽略掉），所以这次直接初始化非static部分。于是有了输出

~~~
1:j   i=0    n=0
2:构造块   i=1    n=1
3:t1   i=2    n=2
~~~

接着对t2进行赋值，过程与t1相同

~~~
4:j   i=3    n=3
5:构造块   i=4    n=4
6:t2   i=5    n=5
~~~

t1、t2这两个static成员变量赋值完后到了static的i与n，于是有了下面的输出：

~~~
7:i   i=6    n=6
~~~

到现在为止，所有的static的成员变量已经被第二次赋值过了，下面到static块了

~~~
8:静态块   i=7    n=99
~~~

至此，所有的static部分赋值完毕了，下面开始执行main方法里面的内容，因为main方法的第一行就又用new的方式调用了Alibaba的构造函数，所以其过程与t1、t2类似

~~~
9:j   i=8    n=100
10:构造块   i=9    n=101
11:init   i=10    n=102
~~~

经过上面这6步，Alibaba这个类的初始化过程就完全结束了。
真心觉得一些基础概念掌握的太不牢固了，阿里巴巴这个面试题出的太好了，让我意识到了自己的技术的肤浅。以后需要恶补。






## 带有继承关系的对象初始化

上面介绍了单个对象的初始化顺序，有了这些基础下面在看个带有继承关系的类的初始化过程：


~~~ java 

class Insect {
    private int i = 9;
    protected int j;
    Insect() {
        System.out.println("i = " + i + " j = " + j);
        j = 39;
    }
    private static int x1 = printInit("static Insect.x1 initialized");;
    static int printInit(String s) {
        System.out.println(s);
        return 47;
    }
}
public class Beetle extends Insect {
    private int k = printInit("Beetle.k initialized");
    public Beetle() {
        System.out.println("k = " + k);
        System.out.println("j = " + j);
    }
    private static int x2 = printInit("static Beetle.x2 initialized");
    public static void main(String[] args) {
        System.out.println("Beetle constructor");
        Beetle b = new Beetle();
    }
}

~~~

当初始化的对象具有继承关系时，也是遵循单一对象初始化时的两大原则的。只不过在考虑static部分，非static部分时需要把子类与父类结合起来整体来看。

~~~
static Insect.x1 initialized
static Beetle.x2 initialized
Beetle constructor
i = 9 j = 0
Beetle.k initialized
k = 47
j = 39
~~~

上面是程序的输出.
**执行main方法时，jvm发现main所在类没有在方法区，于是开始进行classload**。在main执行之前，必须先对类进行初始化。初始化类的变量，还有静态代码块。
在初始化子类时，需要把父类的相关成分先初始化好，比如要想初始化子类的static部分，必须先初始化父类的static部分，其他的也没什么特别的了。输出的具体过程我这里也就不再赘述了。


---
[^1]: 这里好像和我上面说的两大基本原则矛盾，不是说最后执行构造函数，请注意这里的构造函数必须与new来搭配使用，也就是说我们不能在程序中像普通函数调用那样直接使用 Dog()  的这种方式来调用Dog的构造函数，编译器会报找不到Dog()这个方法的错误，这里可能是java中对new进行了特殊处理，使得身为static的构造函数，在初始化阶段的最后执行。


 - 转载自 [keep writing code](http://liujiacai.net/blog/2014/07/12/order-of-initialization-in-java/)




~~~ java 

public class Dervied extends Base {  
    private  String name = "dervied";  
//    private static  String name = "dervied";  
    
    public ClassInit() {  
        tellName();  
        printName();  
    }  

    public void tellName() {  
        System.out.println("Dervied tell name: " + name);  
    }  
      
    public void printName() {  
        System.out.println("Dervied print name: " + name);  
    }  
  
    public static void main(String[] args){  
          
        new Dervied();      
    }  
}  
  
class Base {  
      
    private String name = "base";  
  
    public Base() {  
        tellName();  
        printName();  
    }  
      
    public void tellName() {  
        System.out.println("Base tell name: " + name);  
    }  
      
    public void printName() {  
        System.out.println("Base print name: " + name);  
    }  
}
~~~

上面例子的运行结果

~~~ java 
Dervied tell name: null
Dervied print name: null
Dervied tell name: dervied
Dervied print name: dervied
~~~


new Dervied();这行代码执行的时候，先初始化父类但是执行到父类的tellName()的时候，子类把这个方法覆盖了，所以执行的是子类的tellName()代码，但是执行子类的方法的时候子类对象还没有被创建，所以子类的dname是空，可以换static试试
