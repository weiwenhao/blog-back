---
title: 算法第四版环境搭建(管道重定向)
date: 2017-05-29 22:25:45
tags:
---


>  踩了很多乱七八糟的坑,今天碰巧建好了算法第四版中可运行的环境就分享一下方法。
>  终于可以愉快的看书了 \手动滑稽
>  java环境搭建和eclipse安装还请自行百度。
>  本文适合不会java的新手观看,比如在下

<!-- more -->

### 采用eclipse编译, 命令行执行的方法。


- 搭建一个可以执行的eclipse环境(配置javahome,classpath等环境变量), 需要引入官方提供的algs4.jar. [点击下载 algs4.jar][1]

- myEclipse2017(其他的包括eclipse应该也是类似的)
![](http://omjq5ny0e.bkt.clouddn.com/17-5-29/18116468.jpg)

![](http://omjq5ny0e.bkt.clouddn.com/17-5-29/27315029.jpg)

- 引入完毕后的project目录结构

![](http://omjq5ny0e.bkt.clouddn.com/17-5-29/75569768.jpg)

- 代码示例(可测试)
RandomSeq.java 输出随机数

```java
package day1;   //包名 我是先右键创建了day1这个包,然后在该包名下创建的java文件,至于为什么要使用包名~ 我也不造

import edu.princeton.cs.algs4.*; //引入algs包下的所有类, 当然也可以把*换成特定的类名

/******************************************************************************
 *  Compilation:  javac RandomSeq.java
 *  Execution:    java RandomSeq n lo hi
 *  Dependencies: StdOut.java
 *
 *  Prints N numbers between lo and hi.
 *
 *  % java RandomSeq 5 100.0 200.0
 *  123.43
 *  153.13
 *  144.38
 *  155.18
 *  104.02
 *
 ******************************************************************************/


public class RandomSeq { 

    // this class should not be instantiated
    private RandomSeq() { }


    /**
     * Reads in two command-line arguments lo and hi and prints n uniformly
     * random real numbers in [lo, hi) to standard output.
     *
     * @param args the command-line arguments
     */
    public static void main(String[] args) {

        // command-line arguments
        int n = Integer.parseInt(args[0]);

        // for backward compatibility with Intro to Programming in Java version of RandomSeq
        if (args.length == 1) {
            // generate and print n numbers between 0.0 and 1.0
            for (int i = 0; i < n; i++) {
                double x = StdRandom.uniform();
                StdOut.println(x);
            }
        }

        else if (args.length == 3) {
            double lo = Double.parseDouble(args[1]);
            double hi = Double.parseDouble(args[2]);

            // generate and print n numbers between lo and hi
            for (int i = 0; i < n; i++) {
                double x = StdRandom.uniform(lo, hi);
                StdOut.printf("%.2f\n", x);
            }
        }

        else {
            throw new IllegalArgumentException("Invalid number of arguments");
        }
    }
}

```
- 使用eclipse编译, ctrl+s(保存操作)后即已经编译
- 在你创建的project目录下有src和bin两个目录,src为java文件目录, bin为编译过后的class文件目录


- 执行时在bin目录下已带包名的形式执行.
    例: class文件在 `D:\java\Code\Algs\bin\day1\Average.clas

    该类的包名如上面的代码为`package day1`;
    则我们在 bin目录下打开命令行,然后执行 
    `java day1.RandomSeq 10 1 10`  //输出1~10之间的10个double类型的值

---
###  标准输入流(命令行中的输入)在windows下采用 `enter    ctrl+z  enter`结束
例:

```bash
d:\java\Code\Algs\bin
λ java day1.Average
1.0 2.0 3.0
^Z // ctrl+z

Average is 2.0 //输出
```
---
### 管道重定向

测试输入或者输出最好是先测试输出到文件.不会有路径问题,且能够知道in和out类采取的路径策略(既:是采用绝对路径呢,还是相对路径)

**输出到文件:**
```
//这里我使用RandomSeq类(输出随机数)来进行测试标准输入
d:\java\Code\Algs\bin
λ java day1.RandomSeq 10 1 10

2.30
3.36
6.77
8.47
4.67
9.62
6.44
6.83
7.37
5.51

//重定向到文件
d:\java\Code\Algs\bin
λ java day1.RandomSeq 10 1 10 > test.txt
//在当前目录(既bin目录下)得到了一个test.txt  在notepad编辑器打开格式如下(不要再使用记事本啦)
5.20
4.55
8.73
3.47
5.85
5.54
2.92
7.28
6.93
5.64

```
**输入文件到程序:**
```
//采用Average()计算平均数

d:\java\Code\Algs\bin
λ java day1.Average < test.txt 
Average is 5.611

//路径方面采用绝对路径,相对路径(./test.txt)都没有问题,文件的后缀并不会影响输入流的读取
```
---


  [1]: http://algs4.cs.princeton.edu/code/algs4.jar