---
title: "c++_exception_table"
category: 编程语言
---

g++默认异常处理的机制是Dwarf2 table-based unwinding (dw2) exception handling

正常函数执行时无开销,编译时根据try-catch生成.gcc_exception_table,  
throw时再unwind各层调用查表来匹配异常处理

throw Obj时将Obj分配在heap上(\_\_cxa_allocate_exception), 加上参数point to type_info of Obj,调用\_\_cxa_throw

参考  
<http://gcc.gnu.org/ml/gcc-help/2010-09/msg00116.html>
