<!--
author: luodx
date: 2015-08-07 
title: 正则和LPEG 入门 
status: publish
--> 
###一. 什么是正则 
Regular Expression ， 中文为正则表达式， 它是指用一个表达式来描述字符串的子字符串的特征， 通常用来进行字符串搜索，匹配， 替换。

相对于用一个固定子字符串来就行搜索， 正则表达式非常的灵活。

这个是一个正则的例子： \d\d\d\d-\d\d-\d\d

\d表示一个数字字符， 那么这个表达式表达的是特征是： 4个数字-2个数字-2个数字， 这个通常用来查找字符串里的日期。
###二. 正则的标准 
正则早期出现于文本处理工具比如awk,sed,后来有了POSIX标准， 作为文本处理神器的perl， 不甘落后， 在perl里进一步扩展了正则的功能，

于是后面python等语言纷纷使用perl正则标准， 所以我们现在通常提到的正则标准就是perl标准。

lua里面也有字符串匹配，那个是阉割版的正则，只有非常简单的功能， 表达上略有不同的地方是它用%来代替\，比如%d表示的其实是perl正则里的\d
###三. 基本正则入门
这里只讲一些最基本的正则知识
######首先是单个字符的表示方法：######
a \d \w [a-z] [0-9] . \s \S \W

a是指a这个字符

\d表示一个数字

\w表示一个英文字母或者一个数字或者下划线（w表示word， 通常用于匹配单词）

[a-z] 表示 a到z里的任意一个字符

[0-0]表示0-9的任意一个数字， 其实等同于 \d

. 一般表示除了\n以外的任意一个字符（除非使用了/s修饰符才可匹配换行）

\s 表示空格，制表符这些可以表示空白的字符

\S 表示\s以外的字符

\W 表示\w以外的字符
######匹配字符出现的次数：
\* + ? {n,m} {n}

\* 表示[0, 无限) 次
+ 表示[1, 无限) 次
? 表示[0,1]次
{n,m} 表示[n,m]次
{n} 表示n次
######捕获你要的子字符串
用小括号(), 匹配到后后， 你数一下左括号在表达式里出现的顺序， 就可以用 $1, $2, $n来取到第n个小括号匹配到的子字符串
######两个简单的正则
日期: (\d{4}-\d{2}-\d{2})

域名: ^http://([\w\-\.]+)

对正则想进一步学习的，可以看一下perl文档 http://perldoc.perl.org/perlre.html
####四. 正则的使用范围和局限性
正则通常用在需要做一些简单匹配字符串的地方， 比如网页抓取下来后的分析， 日志的分析等等，

我认为主要用于局部匹配， 如果是要对目标字符串进行完整的parse， 不建议用正则。

正则也可以写得非常的复杂， 但是通常来说很难记住， 写完几天后回头看基本需要重新分析自己写的正则

正则的贪婪匹配没用对 会影响性能， 这点也要注意
####五. PEG的出现
Bryan Ford 在2004年发表了一篇论文（Parsing Expression Grammars） 从理论上提出了PEG, 你可以定义一组规则(patern), 把这些规则组合起来用来表示一个grammar, 用来解析目标字符串。

Lua作者之一roberto 在lua里实现了 PEG， 也就是我们今天要提到的lpeg
####六. PEG的用途
我们这里不使用编译原理的术语来描述 PEG能干嘛。

我们可以使用PEG来写你知道的大多数编程语言的parser, 或者你也可以自定义一些文件格式， 或者DSL（专用领域语言）, 用PEG可以很容易把这些东西解析出你要的数据结构（通常是你解析出各个token之后按照你要的格式把它们塞到一个lua table里 ）

包括我们可以非常简单的用20行Lua代码（lpeg）就可以写完一个JSON字符串的parser, 云风在pbc里也是使用了lpeg对google protobuffer的 数据文件进行了解析
####七. LPEG入门
同样的，我们来看LPEG里如何定义一个patarn来进行匹配的
######基本的patern定义
local patt = P(“abc”) --可以匹配ab这个字符串
local patt = S(“ \t”) --S的意思是Set， 可以匹配空格或者\t， 类似于正则里的[ \t]
local patt = R(“AZ”) R(“09”) --R的意思是Range, 等同于正则里的[A-Z],[0-9]
local patt = P(1) --表示匹配任意一个字符，有点像正则里的.

patt ^ n

n >= 0 时，表示至少匹配n次
n <0 时, 表示最多匹配n次
######Pattern逻辑组合
patt1 * patt2 先满足patt1 再满足patt2
patt1 + patt2 满足了patt1 或者满足了patt2
patt1 -patt2 满足patt1而且不满足patt2
-patt2 = P(“”) - patt2 不满足patt2

注意lpeg对运算符做了重载， * + / - 这些运算符只要任意一边有一个对象是lpeg patern, 那么另外一边也会被转成pattern
######Pattern 例子
我们重新把上面正则里的例子拿出来用lpeg重写一下：

日期: R(“09”)^4 * ”-“ * R(“09”)^2 *”-“ * R(“09”)^2

域名: “http://“ * (R(az) + R(AZ) + R(09) + S(“.-”)) ^ 1

要注意lpeg里小括号可不是正则里的捕获， 捕获我们下面会提到， 这里的小括号只是普通的运算小括号，用来优先进行一些计算

local result = patt:match( text)
如果匹配不上, result为nil
如果可以匹配，result的值取决于patt有没有定义返回值
###### <a name="capture"></a>Pattern 返回值的定义
我们写lpeg并不是只是为了匹配， 我们需要在匹配中捕获我们要的数据， 然后存到我们要的地方去。

捕获在lpeg里， 主要是通过下面的定义：

(下面我们假设patt1, patt2, patt3内部都定义了捕获，分别为C1, C2, C3)

**patt = C(patt1 * patt2 * patt3)**

返回一个捕获，类型为列表

local match_str, C1, C2, C3 = patt:match("...")
match_str为捕获到的整个子字符串

**patt = Ct(patt1 * patt2 * patt3)**

返回一个捕获, 类型为table

local tbl =  patt:match("...")
tbl为一个表格, 默认为数组格式,  {C1,C2,C3}
有一种特殊情况， 比如C2如果是通过Cg定义的命名捕获, 那么
tbl的值为{C1,C3, "命名"=C2}

**patt = Cf(patt1 * patt2 * patt3, f)**

返回一个捕获, 类型取决于f()最后一次返回的值

local fvalue = patt:match("...")
fvalue的值=  f(f( f(C1),C2), C3) 
==常用于对多组数据做统一处理==

**patt = Cg(patt1 * patt2 * patt3)**

返回一个捕获, 类型为列表:

local C1, C2, C3 = patt:match("...")

常跟Cf组合使用, 比如

local patt = Cf( Cg(patt1 * patt2) * Cg(patt3*patt4), f)

local fvalue = patt:match("...")

fvalue的值为  f(f(C1, C2), C3, C4)

==通常用于保存成对的值==, 比如key=value

**patt = Cg(patt1, somename)**

一般不返回任何捕获，只是给C1起了个名字somename， 临时存起来， 后面可以通过Cb(somename)获取， 或者在外面套一个Ct：

local patt = Ct( Cg(patt1, name))
local tbl = patt:match("...")

这样tbl里就有一个key叫somename, value为C1

==通常用于对捕获到的值取一个容易记住的key方便后面读取==


更多的lpeg例子，可以参考

http://lua-users.org/wiki/LpegRecipes
http://www.inf.puc-rio.br/~roberto/lpeg/#ex
####八. 使用lpeg写一个Json parser
下面是我们解决问题的几个步骤
######1. 分析目标， 自上而下就行分解， 写出伪生成式
首先我们要熟悉和分析目标Json数据结构，然后按照我们的理解，把它进行分解。

伪生成式是我们随手写的， 并不是正规编译原理里的生成式。

分解后我们可以得到：

```language-javascript
S <-   hash
hash <-  { str: value, str:value, …  }
value <- number | str  | hash | array
array <- [value,value,value…..]
str <-   2个双引号中间含有0个以上字符
number <-  正负数
```

######2. 根据伪生成式， 使用lpeg实现匹配
注意我们这里不考虑str的双引号中间含有\"这种转义符
```language-perl
local S = lpeg.S; local P = lpeg.P; local R =lpeg.R; local V = lpeg.V
local Space = S(" \n\t")^0
local Number = (P("-")^-1 * R("09")^1) * Space
local Colon = P(':') * Space
local Comma = P(',') * Space
local Str = P('"') * ((1 - P('"'))^0 )*P('"')*Space
local Pair = V('Pair')
local Value = V('Value')
local Array = V('Array')
local Hash = V('Hash')
local gramma = P(
  {
    "S",
    S = Space*Hash,
    Hash = '{' * Pair^0*(Comma*Pair)^0 * '}'*Space,
    Value = Number + Str + Hash + Array,
    Array = '[' * Value^0*(Comma*Value)^0 * ']'*Space,
    Pair = Str * Colon*Value*Space
  }
)
```

######3. 根据我们的需要添加捕获，修改pattern写法
```language-perl
local S = lpeg.S; local P = lpeg.P; local R =lpeg.R; local V = lpeg.V
local Space = S(" \n\t")^0
local Number = lpeg.C(P("-")^-1 * R("09")^1) * Space / tonumber
local Colon = P(':') * Space
local Comma = P(',') * Space
local Str = P('"') * lpeg.C((1 - P('"'))^0 )*P('"')*Space
local Pair = V('Pair')
local Value = V('Value')
local Array = V('Array')
local Hash = V('Hash')
local gramma = P(
    {
        "S",
        S = Space*Hash,
        Hash = lpeg.Cf('{' * lpeg.Ct("")*Pair^0*(Comma*Pair)^0 ,rawset)* '}'*Space,
        Value = Number + Str + Hash + Array,
        Array = '[' * lpeg.Ct(Value^0*(Comma*Value)^0) * ']'*Space,
        Pair = lpeg.Cg(Str * Colon*Value)*Space
    }
)
```

然后来测试一下：
```language-perl
local t = gramma:match('{"a#" : "b","g":{"g1": "gv", "g2":2, "g33":[1,"ggg",3]}}')
```
打印出来这样的结构：
```language-perl
-     "a#" = "b"
-     "g" = {
-         "g1"  = "gv"
-         "g2"  = 2
-         "g33" = {
-             1 = 1
-             2 = "ggg"
-             3 = 3
-         }
-     }
```

到这里大家已经对LEPG有了基本认识，后面可以尝试自己写一些练习， 比如html的解析, linux上各种配置文件的解析等等

