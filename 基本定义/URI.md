# URI(Uniform Resource Identifier)

## RFC2396中的定义（里面没有定义ipv6）

下列定义中：

- [ A ]表示该部分可选
- ( A )用来提高A的运算优先级，使得|等运算只发生在A中
- A | B表示从A和B中只选一个，相当于对A和B求了并集
- "a"表示一个a字符
- AB表示这两部分使用字符串拼接，其中A、B均可以是其它字符串或字符定义
- n*A表示至少n个A进行拼接操作，其中A可以是其它字符串或字符定义。当省略n时，n值为0

```BNF
URI-reference = [ absoluteURI | relativeURI ] [ "#" fragment ] # URI-reference = [URI][#fragment]，其中URI可以为绝对，也可以为相对。这取决于是否指明了scheme
absoluteURI   = scheme ":" ( hier_part | opaque_part ) # absoluteURI，它要由scheme开头
relativeURI   = ( net_path | abs_path | rel_path ) [ "?" query ] # relativeURI，它不会有scheme

hier_part     = ( net_path | abs_path ) [ "?" query ] # 可以层次化的部分，即以/开头，其中的内容是分层的，可以进行更细微的定义
opaque_part   = uric_no_slash *uric # 不透明的部分，即不以/开头

uric_no_slash = unreserved | escaped | ";" | "?" | ":" | "@" | "&" | "=" | "+" | "$" | "," # uric排除掉/

net_path      = "//" authority [ abs_path ] # 以//开头的是网络路径
abs_path      = "/"  path_segments # 以/开头的是绝对路径
rel_path      = rel_segment [ abs_path ] # 相对路径，即开头不是/

rel_segment   = 1*( unreserved | escaped | ";" | "@" | "&" | "=" | "+" | "$" | "," ) rel_segment = uric字符（不可用/?:）组成字符串

scheme        = alpha *( alpha | digit | "+" | "-" | "." ) # scheme（协议） = 字母数字+-.组成的字符串，开头只能是字母

authority     = server | reg_name # authority可以分层的情况下就是server，否则只是uric字符（排除/?，一个是authority的头分隔，一个是尾分隔）组成的字符串

reg_name      = 1*( unreserved | escaped | ";" | ":" | "@" | "&" | "=" | "+" | "$" | "," ) # reg_name = uric字符（不可用/?）组成字符串

server        = [ [ userinfo "@" ] hostport ] # server = 可选的userinfo需用@与hostport相连接（如wfb:123@wfb.com:80）,其本身就可选
userinfo      = *( unreserved | escaped | ";" | ":" | "&" | "=" | "+" | "$" | "," ) # userinfo = uric字符（不可用/?@）组成字符串

hostport      = host [ ":" port ] # hostport = host 后跟 可选的端口号，若有端口号，要用:分隔，否则相当于根据scheme选择默认端口号
host          = hostname | IPv4address # host = 主机名（即域名） 或 ipv4地址
hostname      = *( domainlabel "." ) toplabel [ "." ] # 主机名由多个子域名（可选）和最后的顶级域名组成，之间用.分隔，最后可以加一个.表示根域名，也可省略
domainlabel   = alphanum | alphanum *( alphanum | "-" ) alphanum # 子域名由字母、数字和-组成，-不能做开头和结尾
toplabel      = alpha | alpha *( alphanum | "-" ) alphanum # 顶级域名由字母、数字和-组成，只能由字母做开头，字母或数字作结尾
IPv4address   = 1*digit "." 1*digit "." 1*digit "." 1*digit # ipv4地址=x.x.x.x，其中x为1位以上的数字，更准确的定义应该是0-255，这里没有这样定义
port          = *digit # 端口号是多个数字

path          = [ abs_path | opaque_part ] # path即绝对路径abs_path或者不透明部分opaque_part
path_segments = segment *( "/" segment ) # path_segments = 多个segment（1个以上），之间用/隔开
segment       = *pchar *( ";" param ) # segment = pchar重复多次（包括0次）后可选是否跟参数param，若根要用一个;来隔开
param         = *pchar # param = pchar重复多次（包括0次）
pchar         = unreserved | escaped | ":" | "@" | "&" | "=" | "+" | "$" | "," # param中可以使用的字符，即uric排除掉;/?

query         = *uric # query = 任意字符重复多次（包括0次）

fragment      = *uric # fragment = 任意字符重复多次（包括0次）

--------------------------------下面是所使用到的基本字符的定义------------------------------------------
uric          = reserved | unreserved | escaped # uri所用字符=保留字符+未保留字符+逃逸字符
reserved      = ";" | "/" | "?" | ":" | "@" | "&" | "=" | "+" | "$" | "," # 保留字符，即需要逃逸的字符
unreserved    = alphanum | mark # 未保留字符，即字母数字+标记
mark          = "-" | "_" | "." | "!" | "~" | "*" | "'" | "(" | ")" # 标记，即除保留字符外的符号字符

escaped       = "%" hex hex  # 逃逸字符 = %后跟两个16进制数字，用来在使用保留字符的原始意义时表示该字符
hex           = digit | "A" | "B" | "C" | "D" | "E" | "F" | "a" | "b" | "c" | "d" | "e" | "f"  # 十六进制数字

alphanum      = alpha | digit  # 字母数字alphanum = 字母+数字
alpha         = lowalpha | upalpha  # 字母 = 大写字母+小写字母

lowalpha      = "a" | "b" | "c" | "d" | "e" | "f" | "g" | "h" | "i" | "j" | "k" | "l" | "m" | "n" | "o" | "p" | "q" | "r" | "s" | "t" | "u" | "v" | "w" | "x" | "y" | "z"
upalpha       = "A" | "B" | "C" | "D" | "E" | "F" | "G" | "H" | "I" | "J" | "K" | "L" | "M" | "N" | "O" | "P" | "Q" | "R" | "S" | "T" | "U" | "V" | "W" | "X" | "Y" | "Z"
digit         = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9"
```

## java.net.URI中定义

```BNF
URI-reference -> [scheme:]scheme-specific-part[#fragment]
```

URI可以分为两类：absoluteURI指定了scheme；否则就是relativeURI。
URI还根据它们是不透明的还是层次化的进行分类：不透明URI是绝对URI，其scheme-specific-part部分不以斜杠字符 ('/') 开头。
如下面这几个都是不透明的绝对URI：

- `mailto:java-net@java.sun.com`
- `news:comp.lang.java`
- `urn:isbn:096139210x`

层次化的URI可以是scheme-specific-part部分以/开头的absoluteURI，也可以是relativeURI。
如下几个都是层次化的URI：

- `http://java.sun.com/j2se/1.3/`
- `docs/guide/collections/designfaq.html#28`
- `../../../demo/jfc/SwingSet2/src/SwingSet2.java`
- `file:///~/calendar`

一个典型的层次化URI在语法上主要有如下形式：

```BNF
[scheme:][//authority][path][?query][#fragment]
```

层次化URI的authority部分可能是基于服务器的(server-based)，也可能是基于注册表的(registry-based)。如果是server-based，那么它的语法结构如下：

```BNF
[user-info@]host[:port]
```

如果一个authority无法以上述语法解析，那么它就是registry-based。

组成absoluteURI的path部分可以是绝对路径（以/开头），也可以是相对路径（不以/开头）。如果是个absoluteURI或者指定了authority的relativeURI，那么其path部分一定是绝对路径。

总而言之，一个 URI 实例有以下九个组成部分：

组件｜类型
---｜---
scheme|String
scheme-specific-part|String
authority|String
user-info|String
host|String
port|int
path|String
query|String
fragment|String

在给定的实例中，任何特定组件要么未定义，要么具有不同的值。未定义的字符串组件由 null 表示，而未定义的整数组件由 -1 表示。可以将字符串组件定义为将空字符串作为其值；这不等于未定义该组件。

特定组件是否在实例中定义取决于所表示的 URI 的类型。一个absoluteURI一定有一个scheme组件，一个scheme-specific-part和可能的fragment组件，且不会有其它的组件。一个层次化的URI总是有一个path部分（尽管它可能为空串）和一个scheme-specific-part（这部分至少会包含着path部分），且可能会有其它的嘴贱。如果一个authority组件被表示为server-based，那么host组件将会被定义，user-info和port组件也可能会定义。

### 对URI实例的操作

该类（java.net.URI）支持的主要操作是：规范化、分解和相对化。

规范化是删除层次化URI的path部分中不必要的"."和".."的过程。每一个"."都可以简单的删除（因为.就表示当前路径）。对于".."，仅当它前面有非".."的部分时时才会删除（因为abs/../x就等于x）。归一化对于不透明URI不会产生任何影响（因为不透明URI中的"."和".."不代表当前路径和父路径，没有层次化的含义）。

解析是指对另一个URI（baseURI）对一个URI进行解析的过程。对于层次化URI，baseURI.resolve(aURI)可以认为结果是baseURI/aURI规范化的值(如果aURI是个absoluteURI的话，则值还是原来的absoluteURI)。比如：

```java
// absoluteURI.resolve(relativeURI)
URI baseURI = URI.create("http://java.sun.com/j2se/1.3/");
URI aURI = URI.create("docs/guide/collections/designfaq.html#28");
baseURI = baseURI.resolve(aURI)
baseURI.toString() // http://java.sun.com/j2se/1.3/docs/guide/collections/designfaq.html#28
// 下面的示例也是absoluteURI.resolve(relativeURI)，不过因为是relavitePath，所以触发了规范化
URI bURI = URI.create("../../../demo/jfc/SwingSet2/src/SwingSet2.java");
baseURI = baseURI.resolve(bURI)
baseURI.toString() // http://java.sun.com/j2se/1.3/demo/jfc/SwingSet2/src/SwingSet2.java
// 下面的示例是baseURI.resolve(absoluteURI)，其结果值还是原来的absoluteURI
URI cURI = aURI.resolve()
cURI.toString() // http://java.sun.com/j2se/1.3/demo/jfc/SwingSet2/src/SwingSet2.java
// 下面的示例是relativeURI.resolve(relativeURI)
URI dURI = aURI.resolve(bURI)
dURI.toString() // demo/jfc/SwingSet2/src/SwingSet2.java
```

相对化是解析的反向操作：对于任何两个规范化的URI u 和 v,

```txt
u.relativize(u.resolve(v)).equals(v)
u.resolve(u.relative(v)).equals(v)
```

```java
var aURI = URI.create("http://java.sun.com/j2se/1.3/docs/guide/index.html")
var baseURI = URI.create("http://java.sun.com/j2se/1.3")
var bURI = baseURI.relativize(aURI)
bURI.toString() // docs/guide/index.html
```
