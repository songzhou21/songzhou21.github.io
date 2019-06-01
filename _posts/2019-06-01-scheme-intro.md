---
layout: post
title: "Scheme 介绍"
---

最近一直在看 [sicp](http://sarabander.github.io/sicp/html/index.xhtml) 这本书，准备写点读书笔记，这篇当作是对 Scheme 语言的介绍，毕竟全书是用这门语言写的。

下面是我对书中使用的 [mit-scheme](https://www.gnu.org/software/mit-scheme/) 的一些理解，我没有正统学习过函数式或类 Lisp 语言，只有一些我自己在做这本书习题时的一些理解，文中表述可能不会太专业，请谅解。

# Expressions

basic

```scheme
(+ 1 2)

; 3
```

compound


```scheme
(* (+ 1 2)
   (+ 3 4))
   
; 21

```

# Variables

```scheme
(define pi 3.14159)
(define radius 10)

(* pi (* radius radius))

; 314.159

```

# Procedure

```scheme
; 定义 square 过程
(define (square x) (* x x))

; 调用 square 过程
(square (square 3))

;81
````

# Condition

## cond

```scheme
(define (abs x)
  (cond ((> x 0) x)
        ((= x 0) 0)
        ((< x 0) (- x))))

(abs -42)
; 42

(define (abs x)
  (cond ((< x 0) (- x))
        (else x)))

(abs -42)
; 42

```
 
比较有趣的是，在 mit-scheme 里，变量，过程等是可以同名的，会选用最后的定义，所以上述代码是完全可以运行的。

## if

```scheme
(define (abs x)
  (if (< x 0)
      (- x)
      x))

(abs -42) 
; 42

```

# Logic

```scheme
;; e Expression

(and <e1> <e2>)

(or <e1> <e2>)

(not <e>)

(and (> x 5) (< x 10))
```

# Lambda

无名字 procedure（其他语言也有叫做匿名函数），可作为 procedure 的参数传递，返回。

```scheme
(define (inc x)
  (+ x 1))

;; 返回一个名字为 double 执行 proc 两次的 lambda
(define (double proc)
  (lambda (x)
    (proc (proc x))))


((double inc) 5)   

; 7
```

# Variable binding

## let

```scheme
(let ((x 10)
      (y 5))
  y)

; 5

;; reference binding
(let ((x 10))
  (let ((y (+ x 6)))
    y)) 
    
; 16

;; 用 let 会报错，因为在一个 let expression 里不能引用前面的 bindings
;; 使用 let*
(let ((x 10)
      (y (+ x 6)))
  y)
  
;Unbound variable: x
```
## let*

```scheme
(let* ((x 10)                                                                                                    
       (y (+ x 6)))
  y)

; 16
````

# Data Structures

## Pairs

```scheme
(define x (cons 1 2))

(car x)
1

(cdr x)
2
```

## List

```scheme
;; 下面两个等价

(list 1 2 3)

;Value: (1 2 3)
 
;; '() 表示空 list，相当于 nil
(cons 1 (cons 2 (cons 3 '())))

;Value: (1 2 3)
```

# Symbol

symbol 是以 `'` 开头的字符，是不带语义的变量。

```scheme
;; define a symbol
'apple

; apple

;; 空 list
'() 

; ()

;; list of symbol
(list 'a 'b 'c) 

; (a b c)

;; 可以简写成
'(a b c)

;(a b c)
```

## equal
```scheme
;; symbol equality
(eq? 'apple 'apple))

; #t

;; return sublist of the list beginning with the first occurrence of the symbol
(memq 'apple '(x (apple sauce) y apple pear)) 

; (apple pear)
```

待续 