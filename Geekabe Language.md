

# Geekabe Language

> Geekabe 语言


## 词法

词符（token）使用 `javascript` 语言的正则表达式表达，且文档里从上到下先定义的词法有更高的优先级。使用带状态转移的词法描述。初始状态为 default mode，即一个源码文件第 0 个字符的初始状态。

#### default mode（包含 expression mode）

```
MACRO_BEGIN : ###  // 跳转 Macro Mode
MACRO_ECHO : #{    // 跳转 Macro Mode

KEYWORD :（完整的关键字列表见下文）

...(expression mode)
```

#### expression mode

```
TEMPLATE_STRING_BEGIN : ` // 跳转 template string mode
OPERATOR :（完整的符号列表见下文）
ID : [a-zA-Z]([a-zA-Z0-9_])*
NUMBER : INT | FLOAT
INT : [+-]?[0-9]+
FLOAT: [+-]?(([0-9]+(\.[0-9]*)?) | (\.[0-9]+))(e[+-]?[0-9]+)?
BOOL : true | false

STRING : SINGLE_QUOTE_STRING | DOUBLE_QUOTE_STRING 
SINGLE_QUOTE_STRING : '[^']*'
DOUBLE_QUOTE_STRING : "[^"]*"

COMMENT : SINGLE_LINE_COMMENT | MULTI_LINE_COMMENT
SINGLE_LINE_COMMENT : \/\/[^\n]*
MULTI_LINE_COMMENT : \/\*.*\*\/
```

#### template-string mode

```
EXPRESSION_BEGIN : ${      // 跳转到 template string expression mode
TEMPLATE_STRING_END : `    // 跳转到 default mode
TEMPLATE_STRING_CHARS : .*
```

#### template-string-expression mode

```
...(expression mode)

EXPRESSION_END : }   // 跳转到 template string mode
```

#### macro mode

```
进入 macro mode 后，开启宏语言代码，使用 Geekabe Macro Language 的词法
从宏语言模式退出后，重新进入 default mode。
```

### 关键字 & 符号

关键字：

`if` `loop` `var` `import` `as` `export` `proto` `fn` `return` `u8` `u16` `u32` `u64` `uint` `int` `i8` `i16` `i32` `i64` `bool` `str`

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

### Body

代码的存储单元是文件，每一个文件内容的整体，定义为 `Body`。
```
# imports 语句必须整体放在文件顶部。
Body = ImportStmt* (Block | StmtList | ExportStmt) ;
ExportStmt = export DeclStmt ;
Block = '{' StmtList '}' ;
StmtList = (Stmt | Block)*;
```
### ImportStmt

```
ImportStmt = import IMPORT (as ID)? ';'?
```

### Stmt

```
Stmt = DeclStmt | ProtoStmt | IfStmt | SwitchStmt | LoopStmt | Expr;
```

* `DeclStmt` 声明语句，包括类型声明、变量声明、函数声明。
* `ProtoStmt` 原型定义语句。
* `IfStmt` 和 `SwitchStmt` 条件语句。
* `LoopStmt` 循环语句。
* `Expr` 表达式语句。

### IfStmt & SwitchStmt

```
IfStmt = if Expr Block (else if Expr Block)* (else Block)? ;

# 和 go 语言相同，switch case 默认不穿透（默认 break），除非使用 continue 关键字
SwitchStmt = switch Expr '{' (case Expr ':' Block (continue)? )* (default ':' Block)? '}' ;
```

### LoopStmt

```
LoopStmt = loop (LoopRange | LoopIter | LoopIf )? Block ;
LoopCondRange = ID ':' NUM ( (',' NUM)) | (',' NUM ',' NUM) );
LoopCondIter = ID (',' ID)? in Expr ;
LoopCondIf = if Expr ;
```

循环语句涵盖了传统语句的 `for`, `while`, `do` 等表达式。

示例：

```
// 无限循环，loop 后不跟任何语句。
// 等价于 c 等语言的 do while true，可通过 break 退出
loop {}

// 范围循环，等价于 c 等语言的 for
// loop 后面是参数名，紧跟冒号代表范围循环
// 冒号后面 3 个参数，分别代表终值、初始值（默认为 0）、步进值（默认为 1）
loop i: 10 {}
loop i: 10, 1 {}
loop i: 10, 2, 2 {}

// 迭代循环，等价于 js 语言的 for of，注意不是 js 的 for in
// loop 后是参数名，紧跟 in 关键字，其后是待遍历的数组
var arr = #{array_util.fill(10)};
loop n in arr {
  log(n);
}
// 参数后还可以再跟一个索引参数
loop n, i in arr {
  log(`${n} ${i}`);
}

// 条件循环，loop 后紧跟 while 关键字，再后面必须是一个 bool 表达式。
var i = 0
loop if i < arr.length {
  log(arr[i])
  i += 1
}
```

### DeclStmt & ProtoStmt

```
DeclStmt = TypeDeclStmt | VarDeclStmt | FuncDeclStmt ;

VarDeclStmt = var ID (':' Type)? '=' Expr ';'? ; # 变量的类型可以省略

FuncType = '(' FuncArgs ')' '=>' (Type | void) ;
FuncArgs = (ID ':' Type ','?)* ;
FuncDeclStmt = func ID '(' FuncArgs ')' Block ;

ProtoStmt = proto ID '{' (AtProto | ID) '(' FuncArgs ')' Block)* '}' ;
AtProto = '@' (plus | minus | multiply | divide | to);
```

* 函数参数允许重载。
* `@` 符号打头的原型用于运算符重载和类型转换。

示例：

```
type Person {
  name: str
  age: u8
}
proto Person {
  new(age: u8, name: str) {
  }
  @from(age: u8) {
    return Person(name: 'unknown', age)
  }
  @to(u8) {
    return this.age
  }
  @to(str) {
    return #{strigify('name', 'age')}
  }
}
type I32 = int32
proto I32 {
  @to(Person) {
    return Person(name: 'unknown', age: this)
  }
}

var age: I32 = 18;
var person = Person(age);
var person = Person.from(18);
console.log(person); // "{ name: 'unknown', age: 18}"
console.log(u8(person)); // "18"
```

#### AssignStmt 

```
AssignStmt = ID '=' Expr ';'? ;
```

#### TypeDeclStmt

```
TypeDeclStmt = type ID '=' Type ';'? ;

Type = (PrimaryType | EnumType | StructType | FuncType | ID) ('[' ']')* ;

PrimaryType =
		'int' | 'i8' | 'i16' | 'i32'
	| 'uint' | 'u8' | 'u16' | 'u32' | 'bool'
	| 'str' 
	;

EnumType = '{' EnumField + '}' ;
EnumField = ID (':' NUMBER)? ','? ;
StructType = '{' StructField + '}' ; # 不支持 {} 这种空结构
StructField = ID ':' (PrimaryType | ID) ','? ;
```

### Expr

```
Expr =  ID | '-'? NUMBER | 
	AssignExpr | FuncCallExpr | BoolExpr | TypeConsExpr |
	'(' Expr ')' |
	Expr ('&' | '|') Expr |
	Expr ('>>' | '<<') Expr |
	Expr ('*' | '/') Expr |
	Expr ('+' | '-') Expr
	;
BoolExpr = Expr ((('>' | '<') '='?) | '==' | '&&' | '||') Expr ;
AssignExpr = ID ('+' | '-' | '*' | '/' | '&' | '|' | '>>' | '<<')? '=' Expr ;
FuncCallExpr = ID '(' (Expr ',')* ')' ;     # 函数调用表达式
TypeConsExpr = ID '(' (ID (':' Expr)?)* ')' ;    # object type 的构造表达式
```