---
layout: post
title: 轻松理解Y结合子——Javascript推导
category: 函数式编程
date: 2014-12-08
---

Y结合子（Y Combinator，也译作Y组合子），是在原生不支持递归的编程语言中利用lambda演算实现递归的一种方式，Y结合子在支撑递归的语言中没有什么实际的用途，更多是为了锻炼大家的程序逻辑思维，通过推演充分理解lambda和闭包。下面我们利用Javascript来一步步推导Y结合子。

##原生递归

先从一个经典的递归算法——斐波那契数列讨论，以下是Javascript原生支撑的递归版本：

    var fibonacci = function (n) {
        if (n == 0) return 0;
        if (n == 1) return 1;
        return fibonacci(n - 1) + fibonacci(n - 2); // 1
    }
    console.log(fibonacci(2));
    
或者

    var fibonacci = function (n) {
        if (n == 0) return 0;
        if (n == 1) return 1;
        return arguments.callee(n - 1) + arguments.callee(n - 2);     // 2
    }
    console.log(fibonacci(2));
    
简单解释一下，注释1标注的代码行中，fibonacci是通过Javascript的闭包访问到的，是其父作用域中的fibonacci，也就是目标函数自己本身，从而实现了递归。注释2标注的代码行中，利用arguments.callee访问到函数本身。

##第一步
假设，我们不想通过父作用域的（想纯粹使用lambda演算）或者arguments.callee调用自己，定义一个fibonacci的高阶函数——f，同时f接收一个function参数（其形式等同于f本身），其返回值就是fibonacci函数（目标函数）。换句话说就是f(f)展开以后就是fibonacci函数，而f的函数体中可以通过参数f，调用f(f)展开，得到函数本身，实现递归，代码如下：

    var f = function (f) {
        var fibonacci = function (n) {
            if (n == 0) return 0;
            if (n == 1) return 1;
            return f(f)(n - 1) + f(f)(n - 2);     // 3
        };
        return fibonacci;
    }
    var fibonacci = f(f);
    console.log(fibonacci(3));
    
##第二步
注释3标注的代码行中，目标函数需要通过f(f)得到，形式不够简洁，可以利用lambda演算变换一下，首先构造一个函数g，把”f(f)“抽象出来。

    var f = function (f) {
        var g = function (n) {
            return f(f)(n);
        }
        var fibonacci = function (n) {
            if (n == 0) return 0;
            if (n == 1) return 1;
            return g(n - 1) + g(n - 2);     // 4
        };
        return fibonacci;
    }
    var fibonacci = f(f);
    console.log(fibonacci(4));
    
##第三步
现在fibonacci 函数体的形式一样和期望的一样了，但是我们最后要把fibonacci 抽象成任意函数，现在代码行4中的函数g是通过闭包传递的，所以要把它变成参数传递。定义一个新的函数fun（fibonacci 的高阶函数），把fibonacci 包装起来，传递参数g函数，并返回fibonacci。

    var f = function (f) {
        var g = function (n) {
            return f(f)(n);
        }
        var fun = function (g) {     // 5
            var fibonacci = function (n) {
                if (n == 0) return 0;
                if (n == 1) return 1;
                return g(n - 1) + g(n - 2);
            };
            return fibonacci;
        }
        return fun(g);
    }
    var fibonacci = f(f);
    console.log(fibonacci(5));
    
##第四步
代码行5中，fun应该从外部传进来，所以最后把fun抽象出来，同样上一步一样的技巧，把fun也变成参数传递

    var Y = function (fun) {
        var f = function (f) {     // 6
            var g = function (n) {     // 7
                return f(f)(n);
            }
            return fun(g);
        }
        return f(f);     // 8
    }

    var fun = function (g) {
        var fibonacci = function (n) {
            if (n == 0) return 0;
            if (n == 1) return 1;
            return g(n - 1) + g(n - 2);
        };
        return fibonacci;
    }
    var fibonacci = Y(fun);
    console.log(fibonacci(6));
    
##第五步
现在需要把6、7行f，g的定义处内联到引用处，就能得到Y结合子最后的形式，不过在内联之前注意注释的第8行，f引用了两次，内联的时候就会出现重复代码，再次运用lambda演算技巧，”包装传参“。
把f(f)抽象成函数recur。

    var Y = function (fun) {
        var f = function (f) {
            var g = function (n) {
                return f(f)(n);     // 9
            }
            return fun(g);
        }
        var recur = function (f){
            return f(f);
        }
        return recur(f);
    }

    var fun = function (g) {
        var fibonacci = function (n) {
            if (n == 0) return 0;
            if (n == 1) return 1;
            return g(n - 1) + g(n - 2);
        };
        return fibonacci;
    }
    var fibonacci = Y(fun);
    console.log(fibonacci(7));
    
##第六步
现在再看g、f、recur的引用处，发现各只有一个（注意：不包括参数传递的g、f），现在内联就不会有重复代码了。在内联之前把注射第9行代码修改一下，让Y结合子可以接受任意的形参个数的函数，如下：

    return f(f).apply(null, arguments);
    
最后内联g、f、recur三个函数，得到著名的Y结合子：

    var Y = function (fun) {
        return (function (f) {
            return f(f);
        })(function (f) {
            return fun(function () {
                return f(f).apply(null, arguments);
            });
        });
    }

    var fibonacci = Y(function (g) {
        var fibonacci = function (n) {
            if (n == 0) return 0;
            if (n == 1) return 1;
            return g(n - 1) + g(n - 2);
        };
        return fibonacci;
    });
    console.log(fibonacci(8));
    
注意最后的到的Y结合子中全部都是lambda函数，是一个完全的lambda演算，直接看Y结合子的最终形式比较吃力，读者能看懂第六步没有内联的形式，大致上就能充分理解Y结合子。^_^
