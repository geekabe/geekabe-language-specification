# Geekabe Macro Language

> Geekabe 宏语言

## 概述

Geekabe Language 的源码中，通过 `###` 可以开启宏语言代码，在宏语言代码中，又可以通过 `` ``` `` 开启 Geekabe 语言源代码。通过这个模式，二者可以递归嵌套。这一点，很类似于 `jsx` 或 `php` 语言。例如：

```
###
$a = 'hello'
echo `import ./${$a}`
` ``
import ./${$a}
###
$b = 'world'
###
import ./${$b}
` ``
###
```

可以看出，`` ``` `` 包裹的代码，本质上相当于在宏语言中使用 `echo` 关键字输出代表源码的字符串，但比 `echo` 能力更强的是可以进一步地嵌套宏语言。

`#{}` 是使用宏语言的一种便捷模式，相当于使用 `###` 开启宏语言并使用 `echo` 输出。示例如下： 

```
###
$a = 'hel'
$b = 'lo'
###
// 以下四行等等价于 import ./hello
import ./### ```${$a + $b}``` ###
import ./### echo `${$a}${$b}` ###
import ./#{$a + $b}
import ./#{`${$a}${$b}`}
```

## 约定

为了避免 geekabe 宏语言和 geekabe 语言混淆（比如偶然删除了 `###` 或 `#{` 后，宏语言代码可能混入源码），需要尽可能显式地将二者的语法设计来容易区别；但同时又要尽可能保持 geekabe 宏语言和 geekabe 语言在大方向上语法的一致性，以降低认知、学习和使用成本。为此引入以下约定：

* 宏语言相当于就是一个解释型的脚本语言，用于输出强类型的静态语言的源码。
* 所有变量以 `$` 打头，且变量不需要指明类型，也不需要使用 `var` 来声明。
* 除此外的其它语法二者保持一致。

## 词法

词符（token）使用 `javascript` 语言的正则表达式表达，且文档里从上到下先定义的词法有更高的优先级。使用带状态转移的词法描述。初始状态为 macro-default mode，即源码文件中通过 `###` 或 `#{` 开启宏语言模式后的初始状态。

#### macro-default mode（包含 macro-expression mode)

```
SOURCE_BEGIN : ```  // 跳入源语言 default mode

KEYWORD :（完整的关键字列表见下文）

...(macro-expression mode)
```

#### 源语言 default mode

```
进入 geekabe 源语言的词法，使用 Geekabe Language 文档定义的词法。
在源语言中，遇到 ### 或 #{ 后退出源语言 default mode，重新回到 macro-default mode
```


#### macro-expression mode

```
TEMPLATE_STRING_BEGIN : `  // 跳转到 macro-template-string mode
OPERATOR :（完整的符号列表见下文）
ID : \$[a-zA-Z]([a-zA-Z0-9_])*
NUMBER : INT | FLOAT
INT : [+-]?[0-9]+
FLOAT : [+-]?(([0-9]+(\.[0-9]*)?) | (\.[0-9]+))(e[+-]?[0-9]+)?
BOOL : true | false

STRING : SINGLE_QUOTE_STRING | DOUBLE_QUOTE_STRING 
SINGLE_QUOTE_STRING : '[^']*'
DOUBLE_QUOTE_STRING : "[^"]*"

COMMENT : SINGLE_LINE_COMMENT | MULTI_LINE_COMMENT
SINGLE_LINE_COMMENT : \/\/[^\n]*
MULTI_LINE_COMMENT : \/\*.*\*\/
```

#### macro-template-string mode

```
EXPRESSION_BEGIN : ${      // 跳转到 macro-template-string-expression mode
TEMPLATE_STRING_END : `   // 跳转到 macro-default mode
TEMPLATE_STRING_CHARS : .*
```

#### macro-template-string-expression mode

```
...(macro-expression mode)

EXPRESSION_END : }   // 跳转到 macro-template-string mode
```


### 关键字 & 符号

关键字：

`if` `loop` `import` `as` `export` `proto` `type` `fn` `return` `echo`

符号：

`>=` `<=` `==` `>>` `>` `<<` `<` `=` `^` `+` `-` `*` `/` `%` `[` `]`

## 文法

文法由且只由以下元素构成：

* 大陀峰命名单词，代表文法规则（及子规则），例如 `Body`, `FunctionDecl`。
* 全大写字母（中间可加下划线），代表词符（token），例如 `ID`。
* 全小写字母，代表关键字，例如 `if`, `loop`。
* 单引号包裹的字符，代表文法中的字符串符号，例如 `'='`, `'>='`。
* 不被单引号包裹的字符，用于描述文法本身的逻辑关系，例如 `=`, `;`。

其中，用于描述文法本身的有效字符包括：

```
=   为（表明接下来是当前规则的构成细节）
|   或（多个子规则中 N 选 1）
()  分组（用于表达关系）
?   可选（出现 1 次或 0 次）
+   重复（出现至少 1 次）
*   重复（出现 0 到 n 次)
;   结束（表示当前规则的描述结束）
#   注释（用于解释文法规则，仅支持单行注释）
```