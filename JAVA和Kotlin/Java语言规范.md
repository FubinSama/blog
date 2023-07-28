# Java语言规范

## 语法

### 上下文无关文法

上下文无关文法由一堆产生式组成。每个产生式的左边都是一个抽象的符号（非终态），右边是一个或多个符号（有终态、也有非终态的）组成的序列。所谓终结符（终态的符号）取自某个特定的字母表。
从一个由单个可区分的非终结符组成的句子（目标符号）开始，给定的上下文无关文法指定了一种语言（即可能的终结符号序列的集合，该序列可以通过用右重复替换序列中的任何非终结符而产生- 非终结符在左侧的产生式的左侧）。

### 词法分析简介

由Input（Unicode字符组成的序列）转化为一系列的input元素（即tokens：它会丢弃掉input中的空白符和注释）。
tokens：Java语言中的identifiers（标识，如变量名）、keywords（关键字，如while）、literals（字面量，如`5`）、separators（分隔符，如`;`）、operators（操作符，如`+`）

### 句法分析简介

句法分析由一系列产生式组成。这些产生式的终态标识来自于词法分析的tokens，其目标叫做CompilationUnit。这些产生式描述了如何使用token组成语法正确的程序。

### 语法符号简介

非终结符使用斜体：

```TXT
_IfThenStatement_: 
    if ( _Expression_ ) _Statement_
```

{x}表示x出现0到多次：

```xml
_ArgumentList_: 
    _Argument_ {, _Argument_}
```

[x]表示x可选：

```TXT
_BreakStatement_: 
    break [_Identifier_];
```

很长的产生式右边可以通过明显缩进在第二行继续。
产生式右边的短语 (one of) 表示每个以下一行或多行的终端符号是另一种定义：

```TXT
_ZeroToThree_:
    (one of)
    0 1 2 3
```

当产生式中的替代项似乎是一个记号时，它代表序列组成这样一个标记的字符:

```TXT
_BooleanLiteral_:
(one of)
true false
```

产生式的右边可以指定某些扩展不是允许通过使用短语“but not”指示排除:

```TXT
_Identifier_:
    IdentifierChars but not a Keyword or BooleanLiteral or NullLiteral
```

最后，一些非终结符由罗马类型的叙述短语定义，因为列出所有备选方案是不切实际的：

```TXT
_RawInputCharacter_:
    any Unicode character
```

## 词法分析

Java程序是由Unicode编码的字符组成的，词法分析提供了Unicode逃逸字符到Unicode字符的映射。所以程序可以只使用ASCII编码来引入Unicode字符。

程序定义了行终结符，以在现有主机系统的不同约定下，保持一致的行号。

词法分析会将Unicode字符分成一个个的输入元素（包括：空白符，注释，tokens）。其中token又被分为：identifiers、keywords、literals、separators 和 operators。

### Unicode字符

> [官网地址](http://www.unicode.org/)

Java语言的版本会跟踪Unicode标准的变更。可以在类`Character`中看到当前Java版本使用的Unicode的版本。

现在Unicode的范围是：`U+0000` 到 `U+10FFFF`，即 $17 * 16 ^ 4$个，Unicode官方将其称为17个平面，其中`U+0000`到`U+FFFF`（排除`U+D800`到`U+DFFF`）被称作基本多文种平面。其它的被称为辅助平面。

将基本多文种平面未使用的`U+D800`到`U+DFFF`分成两组，即lead（`U+D800`到`U+BFFF`）和tail（`U+DC00`到`U+DFFF`）。

16个辅助平面一共 $16*16^4$ 种，而使用两位`UTF-16`，即：`head`+`tail`，可以表示 $(4*16*16)^2=16*16^4$ 种，所以刚好用来表示辅助平面，具体算法可以参考 [维基百科](https://zh.m.wikipedia.org/zh-hans/UTF-16)

术语`code point`用来表示一个完整的Unicode字符，而`code unit`用来表示一个UTF-16的head或tail或基本多文种平面中的字符。

Java程序中只有注释、标识符和字符串字面量中可以直接使用Unicode字符，其它必须使用ASCII字符（或者用ASCII字符表示的Unicode逃逸符）。

### 词法转换

一个原始的Unicode字符流将被转换为一系列的token。它的基本工作步骤如下：

1. 将原始的Unicode字符流中的Unicode逃逸字用对应的Unicode字符代替。所谓的Unicode逃逸字符即使一个个用`\uxxxx`表示的`UTF-16 code unit`。因为有这一步转换，所以程序可以只使用ASCII字符来表示。
2. 由第一步产出的Unicode字符流，将被区分为输入字符和行终结符。
3. 上一步产出的输入字符和行终结符将更近一步划分为一系列输入元素：被忽略的空白符和注释，各种`token`（这是下一步句法分析的终结符）。

上述转化步骤遵循最长匹配原则。不过，有个特例（为了解析范型）：对于在类型上下文中出现的字面量转换，遇到多个`>`时，不会遵循最长匹配原则，将其认为是右移运算符（`>>`或`>>>`），而是认为它是一个简单的范型声明的尾巴（`>`）

### Unicode逃逸符

Java编译器在它的输入代码中如果发现了`Unicode`逃逸符（`\uxxxx`），就会将其转化为一个`UTF-16 code unit`，这意味着一个辅助平面的字符实际上由两个逃逸字符（或转换后的`UTF-16 code unit`）表示。对应的产生式如下：

```TXT
_UnicodeInputCharacter_:
    _UnicodeEscape_
    _RawInputCharacter_

_UnicodeEscape_:
    \_UnicodeMarker_ _HexDigit_ _HexDigit_ _HexDigit_ _HexDigit_

_UnicodeMarker_:
    u{u}

_HexDigit_:
    (one of)
    0 1 2 3 4 5 6 7 8 9 a b c d e f A B C D E F

_RawInputCharacter_:
    any Unicode character
```

注意：

1. `UnicodeMarker`允许一到多个`u`，这意味着`String x = "\uuuuD834\uuuDD1E\uD834\uDD1E";`这个写法没有任何问题，这行语句是两个辅助平面的字符`𝄞`。
2. 另外`\\`表示实际上的一个`\`，如：`String x = "\\\uD834\uDD1E";`，这里`x`的值为`\𝄞`。
3. 如果`\`后面没有跟`u`，那么这个就是一个`RawInputCharacter`，最终会被当作一个正常的`Unicode`逃逸字符(注意这个不同于上文说的`Unicode逃逸符`，这里所谓的`Unicode`逃逸字符是`Unicode`字符集编码中定义的，如`\n`表示回车)。
4. 如果`\u`后面跟的不足四个16进制字符，则会抛出编译期异常。
5. 另外，`\u005c`是`\`的Unicode逃逸字，但是`\u005cu005a`会导致编译期错误，因为Java编译器不会把解析成`RawInputCharacter`的字符再拿来当作`UnicodeEscape`的开头。

Java 编程语言指定了一种将用 Unicode 编写的程序转换为 ASCII 的标准方法，该方法将程序更改为可以由基于 ASCII 的工具处理的形式。转换包括通过添加额外的 u 将程序源文本中的任何 Unicode 转义符转换为 ASCII - 例如，\uxxxx 变为 \uuxxxx - 同时将源文本中的非 ASCII 字符转换为每个包含单个 u 的 Unicode 转义符.
这个转换后的版本同样可以被 Java 编译器接受，并且代表完全相同的程序。稍后可以通过将存在多个 u 的每个转义序列转换为具有少一个 u 的 Unicode 字符序列，同时将具有单个 u 的每个转义序列转换为相应的单个 Unicode 字符，可以从这种 ASCII 形式恢复确切的 Unicode 源。

### 行终结符

Java编译器的下一步是通过识别行终结符将Unicode输入流划分行。

```TXT
_LineTerminator_:
    the ASCII LF character, also known as "newline"
    the ASCII CR character, also known as "return"
    the ASCII CR character followed by the ASCII LF character

_InputCharacter_:
    _UnicodeInputCharacter_ but not CR or LF
```

Java编译器通过行终结符确认每一行的行号，同时识别`//`对应的单行注释。

这一步的输入是一系列的行终结符和输入符号。

### 输入元素和token

上一步产生的行终结符和输入符号会在这一步被简化为一系列输入元素。

```TXT
_Input_:
    {_InputElement_} [_Sub_]

_InputElement_:
    _WhiteSpace_
    _Comment_
    _Token_

_Token_:
    _Identifier_
    _Keyword_
    _Literal_
    _Separator_
    _Operator_

_Sub_:
    the ASCII SUB character, also known as "control-Z"
```
