# #pragma常见用法

`#pragma`是一个预处理命令，通常用于设定编译器的状态或指示编译器完成一些特定的动作。

语法：

```c++
#pragma para
```

其中para为参数，下面是一些常见的参数。

## #pragma once

`#pragma once`用于指定该文件在编译源代码文件时仅由编译器包含（打开）一次。

使用 `#pragma once` 可减少生成次数，和使用预处理宏定义来避免多次包含文件的内容的效果是一样的，但是需要键入的代码少，可减少错误率。

```c++
//方式1
#pragma once
// Code

//方式2
ifndef HEADER_H_
define HEADER_H_
// Code

endif // HEADER_H_
```

## #pragma message

`#pragma message`用于在不中断编译的情况下，发送一个字符串文字量到标准输出，`message` 编译指示的典型运用是在编译时显示信息。

```c++
#if _M_IX86 >= 500  		//查看定义的宏有没有大于500，或者有时用作检查有没有定义宏
#pragma message("_M_IX86 >= 500")  
#endif 

//扩展字符串
#pragma message( "Compiling " __FILE__ )   		//显示被编译的文件
#pragma message( "Last modified on " __TIMESTAMP__ ) 	//文件最后一次修改的日期和时间
```

## #pragma waring

`pragma waring`用于启用编译器警告消息的行为和选择性修改。

```c++
#pragma warning(
 warning-specifier : warning-number-list
 [; warning-specifier : warning-number-list ... ] )
#pragma warning( push [ , n ] )
#pragma warning( pop )
```

warning-specifier可选参数

| 警告说明符          | 含义                                                         |
| ------------------- | ------------------------------------------------------------ |
| `1`、`2`、 `3`、`4` | 将给定级别应用于指定的警告。 也会启用默认情况下处于关闭状态的指定警告。 |
| `default`           | 将警告行为重置为其默认值。 也会启用默认情况下处于关闭状态的指定警告。 警告将在其默认存档级别生成。 |
| `disable`           | 不发出指定的警告消息。                                       |
| `error`             | 将指定警告报告为错误。                                       |
| `once`              | 只显示指定消息一次。                                         |
| `suppress`          | 将 pragma 的当前状态推送到堆栈上，禁用下一行的指定警告，然后弹出警告堆栈，从而重置 pragma 状态。 |

```c++
#pragma warning( disable : 4507 4034; once : 4385; error : 164 )
```

上述代码等效于

```c++
// Disable warning messages 4507 and 4034.
#pragma warning( disable : 4507 4034 )

// Issue warning C4385 only once.
#pragma warning( once : 4385 )

// Report warning C4164 as an error.
#pragma warning( error : 4164 )
```

`#pragma warning( push )` 存储每个警告的当前警告状态。 `#pragma warning( push, n )` 存储每个警告的当前状态并将全局警告级别设置为 n。

`#pragma warning( pop )` 弹出推送到堆栈上的最后一个警告状态。 在 `push` 和 `pop` 之间对警告状态所做的任何更改都将被撤消。 请看以下示例：

```c++
#pragma warning( push )
#pragma warning( disable : 4705 )
#pragma warning( disable : 4706 )
#pragma warning( disable : 4707 )
// Some code
#pragma warning( pop )
```

在此代码的末尾，`pop` 会将每个警告（包括 4705、4706 和 4707）的状态还原为其在代码开头的状态。

编写头文件时，可以使用 `push` 和 `pop` 来确保用户所做的警告状态更改不会阻止标头进行正确的编译。 请在标头的开头使用 `push`，并在标头的末尾使用 `pop`。

## #pragma comment

该指令将一个注释记录放入一个对象文件或可执行文件中。

```c++
#pragma comment（comment-type [,“commentstring”]）
```

comment-type是一个预定义的标识符，指定注释的类型，应该是compiler，exestr，lib，linker之一。

注释类型

compiler

放置编译器的版本或者名字到一个对象文件，该选项是被linker忽略的。

lib

对当前工程配置一个链接库。

```c++
#pragma  comment(lib, "comctl32.lib")
#pragma comment( lib, "mysql.lib" )
```

linker

将链接器选项置于对象文件中。 可以使用注释类型来指定链接器选项，而不是将其传递到命令行或在开发环境中指定它。 例如，可以指定 /include 选项来强制包含符号：

```c++
#pragma comment(linker, "/include:__mySymbol") 
```

## #pragma pack

改变编译器对结构体、联合体、类成员的封装对齐方式

```c++
#pragma pack(show)     //以警告信息的形式显示当前字节对齐的值.
#pragma pack(n)        //将当前字节对齐值设为 n .
#pragma pack()         //将当前字节对齐值设为默认值(通常是8) .
#pragma pack(push)     //将当前字节对齐值压入编译栈栈顶.
#pragma pack(pop)      //将编译栈栈顶的字节对齐值弹出并设为当前值.
#pragma pack(push, n)  //先将当前字节对齐值压入编译栈栈顶, 然后再将 n 设为当前值.
#pragma pack(pop, n)   //将编译栈栈顶的字节对齐值弹出, 然后丢弃, 再将 n 设为当前值.

pragma pack(push, identifier)        //将当前字节对齐值压入编译栈栈顶, 然后将栈中保存该值的位置标识为 identifier .

pragma pack(pop, identifier)        //将编译栈栈中标识为 identifier 位置的值弹出, 并将其设为当前值. 注意, 如果栈中所标识的位置之上还有值, 那会先被弹出并丢弃.

pragma pack(push, identifier, n)    //将当前字节对齐值压入编译栈栈顶, 然后将栈中保存该值的位置标识为 identifier, 再将 n 设为当前值.

pragma pack(pop, identifier, n)     //将编译栈栈中标识为 identifier 位置的值弹出, 然后丢弃, 再将 n 设为当前值. 注意, 如果栈中所标识的位置之上还有值, 那会先被弹出并丢弃.

//注意: 如果在栈中没有找到 pop 中的标识符, 则编译器忽略该指令, 而且不会弹出任何值.
```

在使用`#pragma pack`时，最好成对出现，以免引起错误。

## #pragma code_seg

指定对象 (.obj) 文件中存储函数的代码段。

```c++
#pragma code_seg( [ "section-name" [ , "section-class" ] ] )

#pragma code_seg( { push | pop } [ , 标识符 ] [ , “section-name” [ , “section-class” ] ] )
```

