---
title: 过程抽象解一般方程
date: 2018-01-27 17:00:03
tags:
---


<!-- more -->

## 传统解方程


#### 二次函数为例

![img](https://upload.wikimedia.org/wikipedia/commons/thumb/7/7a/Polynomialdeg_2.svg/233px-Polynomialdeg_2.svg.png)

解析式：**f(x) = x² - x -2**

在数学中，**二次函数** `f(x) = ax² + bx + c` (a ≠ 0，且a、b、c 是常数)的多项式函数，其中，x 为自变量，a, b, c 分别是函数解析式的二次项系数、一次项系数和常数项。二次函数的图像是一条主轴平行于![y](https://wikimedia.org/api/rest_v1/media/math/render/svg/b8a6208ec717213d4317e666f1ae872e00620a0d) 轴的抛物线



> 令f(x) = 0 可以将二次函数转换成二次方程,该方程的解称为方程的根或者函数的零点(y = 0是x的值)
>
> 通过判别式 △ = b² - 4ac 来判定方程根的情况

#### 因式分解

例 x² - 3x + 2 = 0 可以将左边的方程使用十字相乘法分解成 (x - 1) (x - 2) = 0, 所以分别有 x - 1 = 0 或者 x - 2 = 0 从而得到方程的两个解

#### 求根公式

对`ax² + bx + c = 0 (a ≠ 0)`它的根可以表示为：

![x_{{1,2}}={\frac  {-b\pm {\sqrt  {b^{2}-4ac\ }}}{2a}}.](https://wikimedia.org/api/rest_v1/media/math/render/svg/ddcdc99b985b5d370851854c27f1e803c29ebd6a)



## 区间折半法求任意函数的零点

f(x) = 0, f是一个连续的函数. 如果给定点a和b有 `f(a) < 0 < f(b)`, 那么f在a和b之间必然有一个零点(f(x) = 0的点).为了确定这个零点. 我们只需要缩小 a和b的区间从而确定零点所在的范围.具体的做法是

令 x是a和b的平均值并计算出 f(x). 如何f(x) > 0 那么 a和x之间必然有一个f的零点. f(x) < 0则反之

实现

```scheme
(define (average a b)
  (/ (+ a b)
     2))

; 更加通用的寻找一个函数的零点
(define (search f lower upper)
  ; 判断是否可以接收误差区间
  (define (close-enough? x y)
    (< (abs (- x y)) 0.001))
  (let ((middle (average lower upper)))
    (if (close-enough? lower upper)
        middle
        (let ((mid-value (f middle))) ; 求解中间值
            (cond ((positive? mid-value) ; 判断大小进行下一次操作
                   (search f lower middle))
                  ((< mid-value 0)
                   (negative? f middle upper))
                  (else middle))))))

(define (half-interval-method f a b)
  (let ((a-value (f a))
        (b-value (f b)))
    (cond ((and (negative? a-value) (positive? b-value))
           (search f a b))
          ((and (negative? b-value) (positive? a-value))
           (search f b a))
          (else
            (display "Values are not of opposite sign")))))


; 折半法解 x³ - 2x - 3 = 0 在1和2之间的根
(half-interval-method (lambda (x) (- (* x x x) (* 2 x) 3))
                      1
                      2)
```



区间折半法给出了一种解方程的通用方法.其是高度抽象的且适用于计算机的可以解任何线性方程的方法.

## 找出函数的不动点

**定义:**f(x) = 0,称为f(x) 的零点. 而f(x) = x. x称为f(x)的不动点, 

例如 `f(x) = -x + 3`令 x = -x + 3 解得 x = - (3/2). 这就是方程f(x)的不动点.

当然并不是所有的函数都有不动点,如 `f(x) = x + 2`,我们找不到一个x使得 x = x + 2 (当然0除外)

🤔 **给定任意函数,如何找到其不动点呢?**

对于某些函数,通过某个初始猜测触发,反复的应用f

`x, f(x), (f(fx)), f(f(f(x))),..., `

直到值的变化不大时,就可以找到它的一个不动点.

这种特殊的不动点称为**吸引不动点** , 其严格定义是: 对于函数的不动点x0, 在足够接近x0的定义域(x相对于f(x)的取值范围称为定义域)的任何x, 通过迭代序列 `x, f(x), f(f(x))…` , x会主键收敛于x0

如何才是 "足够接近"是个微妙的问题, 在计算机中不断的 cos(cos(cos(cos(……R, 它会快速的收敛大雨 0.73908513. 这就是不动点. 这种情况下 足够接近的约束实际上是不存在的

![https://upload.wikimedia.org/wikipedia/commons/e/ea/Cosine_fixed_point.svg](https://upload.wikimedia.org/wikipedia/commons/e/ea/Cosine_fixed_point.svg)

根据上面的定义我们抽象出一个寻找函数不懂点的过程

```scheme
(define tolerance 0.00001)

(define (fixed-point f guess)
  (define (close-enough? a b)
    (< (abs (- a b)) tolerance))
  (define (iter guess)
    (let ((next (f guess)))
      ; 计算出的下一个猜测值和当前猜测值的差在允许误差范围内则表示已经收敛完毕, next既是我们需要寻找的不动点
      (if (close-enough? guess next)
          next
          (iter next))))
  (iter guess))

; 测试 cos的不动点
(fixed-point cos 1)
```





#### 通过寻找不动点的方式求平方根

x² = y.  转换一下可以得到 x = y/x. 现在只要找到一个x使得 x = y/x . 那就这个x就是y的平方根. 且通过上面的描述,我们可以知道这样的x也称为 y/x 的不动点. 我们已经有了一个寻找函数不动点的过程.现在应用看看

```scheme
(define (my-sqrt y)
  (fixed-point (lambda (x) (/ y x)) 1))

(my-sqrt 4) ; 这不ok,我们陷入了一个iter无法自拔了...
```

初始化时 x = x1; x2 = y/x1; x3 =  y/(y/x1) = x1  这里陷入了一个死循环,我们始终在x3 ~ x1之间震荡.

由于实际答案总是在 x 和 y/x 之间

> x * y/x = y  平方根总是在中点部分. 其余被乘数和乘数则一定会有一个大于中间值.比如 16的平方根是4, 那16的其余乘数比如 1 \* 16 / 2 \* 8 / 0.5 \* 32等等 总是有一个乘数会大于4.

既然有了上一次的结论,那我们可以不适用 y/x作为下一次的猜测值, 而使用x和 y / x的平均值作为下一次的猜测值,从而使每一次的猜测值变化都没有那么剧烈

```scheme
(define (my-sqrt y)
  (fixed-point (lambda (x) (average x (/ y x))) 1.0))
```



successful!

将过程作为参数传递,能够显著增强我们的程序设计语言的表达能力.可以表述计算的一般性过程,与其中所涉及的特定函数无关.
