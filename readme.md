This is my computer technology blogs.
async/await关键字，基本事实
1、async关键字标记的函数为异步函数，返回值是Promise对象（会隐式转换，例如return 1，其实返回的是Promise.resolve(1)）。
2、async函数内可以有0个或多个await表达式，具有await表达式的async函数才是真正的异步执行。如果没有await表达式，代码还是同步执行。
3、async函数内有await表达式时，从第一行到await表达式前，代码同步执行，await表达式异步执行。执行到await表达式时，async函数会让出线程执行的控制权，让其他代码继续执行(即异步的实现）。直到await表达式有返回值，再继续执行await表达式后面的代码。
4、await关键字只能在async函数中是使用，在其他地方编译器会报错。


第一次提交
第二次提交

