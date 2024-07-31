https://go.dev/ref/spec

Go1.22编程语言规范（2024年2月6日）

.. title:: Go编程语言规范

* `源代码表示`_
* `词法元素`_
* `常量`_
* `变量`_
* `类型`_
* `类型属性`_
* `代码块`_
* `声明和作用域`_
* `表达式`_
* `语句`_
* `内置函数`_
* `错误和异常`_
* `包`_
* `程序初始化和执行`_
* `系统底层考量`_
* `附录`_

该文档是Go编程语言规范参考手册。不支持泛型的Go1.18之前的版本，可以看 `这里`_。
更多的信息和文档，见 `go.dev`_。

Go是通用型系统编程语言，强类型垃圾回收显式并发编程支持。程序由包组成，包的属性
可以高效地管理依赖。语法紧凑简单，很容易被自动化工具分析。

语法使用 EBNF（Extended Backus-Naur Form）范式的变种来描述。其中{}表示出现0次
或多次，[]表示可选即出现0次或1次，小括号()用来分组，用|来表示多种选择。小写名称
表示词法终结符，非终结符名称使用首个字母大写形式表示一个语法结构，词法字面量使用
双引号""或者``括起。语法由语法结构和词法元素组成，语法结构递归的由语法结构和词法
元素构成是非终结符，而词法元素可以认为是终结符不再包含其他结构，终结符就是语法树
的叶子结点不再继续派生语法结构，词法元素是注释标识符操作符字面量等等。 ::

    Syntax      = { Production } .
    Production  = production_name "=" [ Expression ] "." .
    Expression  = Term { "|" Term } .
    Term        = Factor { Factor } .
    Factor      = production_name | token [ "…" token ] | Group | Option | Repetition .
    Group       = "(" Expression ")" .
    Option      = "[" Expression "]" .
    Repetition  = "{" Expression "}" .

字符…不是Go语言的词法元素，形如a…b的表示是一个从字符a到b的字符范围。另外[`Go1.xx`_]
形式的链接表示对应语言特性需要的最小版本支持，详情见附录的 `语言版本`_ 部分。

.. _这里: https://go.dev/doc/go1.17_spec.html
.. _go.dev: https://go.dev/
.. _Go1.xx: `语言版本`_

源代码表示
===========

源代码使用UTF-8编码，编码文本是没有规范化的，单个变音字符与对应的音调和字母是
不同的，后者被当作两个字符对待。为简单性，该文档使用字符与Unicode的编码点进行
对应。每一个编码点都是独立区别的，例如小写和大写是不同的字符。为了与其他工具
兼容，编译器允许源代码中包含NUL字符（U+0000）。同样为了兼容其他工具，编译器可
能会忽略UTF-8的编码记号字节（U+FEFF），但它必须是源文件的第一个字符，记号字节
可能不允许在源文件的其他地方出现。

Unicode字符被分类区别::

    newline = /* 换行 U+000A */ .
    unicode_char = /* 除换行外的其他任意Unicode字符 */ .
    unicode_letter = /* 表示字母的Unicode分区中的字符 */ .
    unicode_digit = /* 表示数字的Unicode分区中的字符 */ .

在 `Unicode标准8.0`_ 中的4.5节定义了字符分区。字符分区Lu Ll Lt Lm Lo中的字符都
表示一个字母，而Nd分区中的字符都表示一个数字。另外下划线字符（U+005F）被当作是一个
小写字母。 ::

    letter = unicode_letter | "_" .
    dec_digit = "0" … "9" .
    bin_digit = "0" | "1" .
    oct_digit = "0" … "7" .
    hex_digit = "0" … "9" | "A" … "F" | "a" … "f" .

.. _Unicode标准8.0: https://www.unicode.org/versions/Unicode8.0.0/

词法元素
=========

* `注释`_
* `标识符和关键字`_
* `操作符和标点`_
* `整型字面量`_
* `浮点字面量`_
* `虚数字面量`_
* `字符字面量`_
* `字符串字面量`_

词法元素组成Go语言的词汇表，分成标识符、关键字、操作符标点符、以及字面量。
空白字符 ``' ' '\a' '\r' '\n'`` 会被忽略除非它分隔了词法元素，其中注释会被
解析为空白字符。一个换行符或者文件结束符可能触发插入一个分号。当将输入流分
隔为词法元素时，会找下一个最长的字符序列。

分号在语法中当作很多语法结构的终止符，Go语言使用下面两个规则避免大多数分号的书写：

1. 将输入流分隔成词法元素时，以下情况会在一行最后自动插入分号

  - 最后一个词法元素是一个标识符、整数、浮点、复数、字符、字符串字面量
  - 最后一个词法元素是break continue fallthrough return这几个关键字
  - 最后一个词法元素是一个操作符或者是++ -- ) ] }这几个标点符

2. 为了允许在一行书写复杂的语句，在)和}之前的分号可以忽略

为了反映惯例用法，该文档中的代码示例都使用上面的规则，尽量避免书写分号。

注释
-----

注释包括以//开始的行注释和以 ``/*`` 开始 ``*/`` 结束的块注释，行注释和没有
换行的块注释被解析成一个空格，有换行的块注释被解析成一个换行。注释不能出现在
字符字面量和字符串字面量中，或者另一个注释里面。

标识符和关键字
--------------

标识符用来命名程序中的变量类型函数等。一个标识符必须使用字母开头，然后跟随
一个或多个字母或数字::

    identifier = letter { letter | unicode_digit } .

一些标识符可能是在全局代码块（universe block） [Go1.18]_ [Go1.21]_ 中隐式预声明的::

    类型 any bool byte comparable
         complex64 complex128 error float32 float64
         int int8 int16 int32 int64 rune string
         uint uint8 uint16 uint32 uint64 uintptr
    常量 true false iota
    零值 nil
    函数 append cap clear close complex copy delete
         imag len make max min new panic print println
         real recover

还有一些特殊的标识符是保留给语言使用的，称为关键字::

    break case chan const continue default defer
    else fallthrough for func go goto if import
    interface map package range return select
    struct switch type var

操作符和标点
------------

操作符和标点包括::

    + - * / % += -= *= /= %=
    ~ ! & | ^ << >> &^ &= |= ^= <<= >>= &^=
    && || <- ++ --
    == < > = != <= >= :=
    ( ) [ ] { } , : ; . ...

整型字面量
----------

整型字面量以0到9的数字开头的整数常量，可以是以0b 0B开头的二进制常量，或者以
0 0o 0O开头的八进制常量，或者以0x 0X开头的十六进制常量。单个0表示一个十进制
常量零。为阅读方便，下划线字符可以出现在前缀之后和连续两个数字之间::

    int_lit = dec_lit | bin_lit | oct_lit | hex_lit .
    dec_lit = "0" | ("1" … "9") [["_"] dec_digits] .
    bin_lit = "0" ("b" | "B") ["_"] bin_digits .
    oct_lit = "0" ["o" | "O"] ["_"] oct_digits .
    hex_lit = "0" ("x" | "X") ["_"] hex_digits .
    dec_digits = dec_digit {["_"] dec_digit} .
    bin_digits = bin_digit {["_"] bin_digit} .
    oct_digits = oct_digit {["_"] oct_digit} .
    hex_digits = hex_digit {["_"] hex_digit} .

浮点字面量
----------

浮点字面量是一个十进制或者十六进制表示的浮点常量。十进制浮点字面量包含用十进制
表示的整数部分，一个小数点，用十进制表示的小数部分，以及指数部分（用e或E开始加
一个可选的符号加十进制表示的指数）。整数部分和小数部分的其中一个可以省略，小数点
和指数部分的其中一个可以省略。一个指数值exp将底数（整数加小数部分）放大10\ :sup:`exp` 倍。

十六进制浮点字面量包含一个0x或0X前缀，十六进制表示的整数部分，小数点，十六进制
表示的小数部分，以及指数部分（用p或P开始加一个可选的符号加十进制表示的指数）。
整数部分和小数部分的其中一个可以省略，小数点可以省略，但是指数部分必须存在。这种
十六进制表示的语法匹配IEEE 754-2008 §5.12.3形式中的一种。一个指数值exp将底数
（整数加小数部分）放大2\ :sup:`exp` 倍 [Go1.13]_。

为了可读性，下划线可以出现在前缀之后或者两个连续的数字之间。这些下划线字符不改变
原本字面量的值。 ::

    float_lit = dec_float_lit | hex_float_lit .
    dec_float_lit = dec_digits "." [ dec_digits ] [ dec_exponent ] | dec_digits dec_exponent | "." dec_digits [ dec_exponent ] .
    dec_exponent  = ( "e" | "E" ) [ "+" | "-" ] dec_digits .
    hex_float_lit = "0" ( "x" | "X" ) hex_mantissa hex_exponent .
    hex_mantissa = [ "_" ] hex_digits "." [ hex_digits ] | [ "_" ] hex_digits | "." hex_digits .
    hex_exponent = ( "p" | "P" ) [ "+" | "-" ] dec_digits .

例如::

    0.           // 0.0
    72.40 072.40 // 72.40
    2.71828
    1.e+0        // 1.0 * 10^0
    6.67428e-11  // 6.67428 * 10^(-11)
    1E6          // 1.0 * 10^6
    .25          // 0.25
    .12345E+5    // 0.12345 * 10^5
    1_5.         // 15.0
    0.15e+0_2    // 0.15 * 10^2 = 15.0

    0x1p-2       // 0.25，即1.0 * 2^(-2)
    0x2.p10      // 2048.0，即2.0 * 2^10
    0x1.Fp+0     // 1.9375，即0x1.F，0b1.1111，1+0.5+0.25+0.125+0.0625=1.9375
    0X.8p-0      // 0.5，即0x0.8，0b0.1000，0+0.5=0.5
    0X_1FFFP-16  // 0.1249847412109375，即0x1FFF * 2^(-16)
    0x15e-2      // 不是一个浮点字面量，而是两个整数的减法 0x15e - 2

    0x.p1        // 非法，整数部分和小数部分只能省略一个
    1p-2         // 非法，用p表示的浮点数需要一个十六进制前缀的底数
    0x1.5e-2     // 非法，十六进制浮点字面量需要使用p作为指数的开始而不是e
    1_.5         // 非法，下划线只能在前缀之后或者两个数字之间
    1._5         // 非法，下划线只能在前缀之后或者两个数字之间
    1.5_e1       // 非法，下划线只能在前缀之后或者两个数字之间
    1.5e_1       // 非法，下划线只能在前缀之后或者两个数字之间
    1.5e1_       // 非法，下划线只能在前缀之后或者两个数字之间

虚数字面量
----------

虚数字面量用来表示复数常量的虚数部分。它包含一个整数或浮点字面量和一个小写字符i。
一个虚数字面量的值是对应的整数或浮点字面量的值乘以虚数单位i [Go1.13]_ 。 ::

    imaginary_lit = (dec_digits | int_lit | float_lit) "i" .

为了兼容旧版本，虚数字面量的整数部分如果只由十进制数字表示则都被解析成十进制整数，
即便是以零开始。 ::

    0i
    0123i         // 123i
    0o123i        // 0o123 * 1i，83i
    0xabci        // 0xabc * 1i，2748i
    0.i           // 0.0i
    2.71828i
    1.e+0i        // 1.0 * 10^0 * 1i
    6.67428e-11i  // 6.67428 * 10^(-11) * 1i
    1E6i          // 1 * 10^6 * 1i
    .25i          // 0.25i
    .12345E+5i    // 0.12345 * 10^5 * 1i
    0x1p-2i       // 1 * 2^(-2) * 1i，0.25i

字符字面量
----------

字符字面量是用单引号引起的字符常量，代表一个Unicode编码点。单引号内可以出现除了
换行和未转义的单引号外的任何其他字符。因为Go语言源代码是用UTF-8编码的，因此单个
字符不一定只有一个字节。

单引号引起的字符可以是转义字符， ``\xhh`` 两个十六进制表示的字符， ``\ooo`` 三个
八进制表示的字符不能大于255， ``\uhhhh`` 四个十六进制表示的字符， ``\Uhhhhhhhh`` 
八个十六进制表示的字符对应的Unicode编码点需要是合法的，比如大于0x10FFFF或者surrogate
halves都是非法字符，特殊转义字符 ``\a \b \f \n \r \t \v \\ \' \"``::

    rune_lit = "'" (unicode_value | byte_value) "'"
    unicode_value = unicode_char | little_u_value | big_u_value | escaped_char .
    byte_value = oct_byte_value | hex_byte_value .
    oct_byte_value = `\` oct_digit oct_digit oct_digit .
    hex_byte_value = `\` "x" hex_digit hex_digit .
    little_u_value = `\` "u" hex_digit hex_digit hex_digit hex_digit .
    big_u_value = `\` "U" hex_digit hex_digit hex_digit hex_digit hex_digit hex_digit hex_digit hex_digit .
    escaped_char = `\` ("a" | "b" | "f" | "n" | "r" | "t" | "v" | `\` | "'" | `"`) .

字符串字面量
------------

字符串字面量包括正常字符串字面量和原生字符串字面量，原生字符串字面量不会对其中
的字符进行解析，按原样表示其内容。原生字符串以`开头可以包含除了`字符之外的任何
字符，因为不会被解析，反斜杠字符没有任何特殊意义，并且可以包含换行，另外 ``\r`` 
字符会被删除。

正常的进行解析的字符串使用双引号引起，引号内可以出现除了换行和未转义双引号外的
任何其他字符，但是不允许出现 ``\'`` 转义字符::

    string_lit = raw_string_lit | interpreted_string_lit .
    raw_string_lit = "`" {unicode_char | newline} "`" .
    interpreted_string_lit = `"` { unicode_value | byte_value } `"` .

常量
=====

常量包括布尔常量、字符常量、整型常量、浮点常量、复数常量、字符串常量。其中字符、
整型、浮点、复数常量称为数值常量。因此常量分为布尔常量、数值常量、和字符串常量。

常量值可以是：

1. 数值字面量和字符串字面量
2. 布尔常量true和false
3. 预定义标识符iota表示的整型常量
4. 引用常量值的标识符
5. 一个常量表达式
6. 一个结果是常量值的转换
7. 传入参数是常量的一些内置函数调用比如min和max
8. unsafe.Sizeof对一些值的调用
9. cap和len对一些表达式的调用
10. real和imag对复数常量的调用以及complex对数值常量的调用

数值常量表示一个任意精度没有上溢的确定的值，因此常量不能表示IEEE-754的负零、
无穷大、以及N/A（not-a-number）的值。

常量是有类型的或者无类型的（typed or untyped），字面常量、true、false、iota、
以及只包含无类型常量操作数的常量表达式，都是无类型的。无类型的常量有默认类型
bool rune int float64 complex128 string。

一个常量可以通过常量声明或者转换显式的指定类型，或者在变量声明、赋值语句、作为
表达式操作数中使用时隐式的给定类型。如果当常量不能用对应类型的值表示时，会出现
错误。如果类型是一个类型参数，常量会被转换成对应类型参数实例化类型的非常量值。

实现限制：尽管数值常量有任意精度，但是编译器可能使用一个整数表示，只有有限精度。
为了规范化每个实现都必须：

1. 至少用256位表示一个整型常量
2. 至少用256位表示一个浮点数的尾数，至少用16位表示有符号的指数部分
3. 如果不能精确表示一个整型常量要报错
4. 如果因为上溢而不能表示浮点或复数常量要报错
5. 如果由于精度限制不能表示一个浮点或复数要近似到最近的表示

字面常量和常量表达式产生的结果都需要遵循上面的需求。

变量
=====

变量是一个存储位置用来保存一个值，变量容许的值由变量的类型决定。

变量声明、或者函数声明和函数字面量的签名中的函数参数和返回值，都定义了一个预留了
存储位置的命名变量。通过调用内置函数new或者使用复合结构字面量的地址会在运行时分配
变量的存储空间，这些匿名的变量通过一个指针被间接的引用。

结构化的变量例如数值、切片、结构体有单独地址的内部元素，每个元素都如同是一个变量。

一个变量的静态类型或者简单说类型，是变量定义时给定的类型，如调用new或者使用复合
结构字面量提供的类型，或者复合结构变量的元素或成员类型。接口类型变量还拥有动态类型，
动态类型是一个非接口的在运行时赋给变量的类型，除非一个变量声明成nil，它没有类型。变量
的动态类型可以在运行时改变，但是接口变量中存储的值始终是一个可以赋值给接口变量静态类型
的值。

在表达式中使用变量即引用该变量的值，这个值是该变量最近的一次赋值，如果一个变量还
没有被赋值，这个变量的值是对应类型的零值。 ::

    var x interface{} // 变量x的静态类型是interface{}，当前的值是nil
    var v *T          // 变量v的静态类型是*T，当前的值为nil
    x = 42            // 接口变量x的动态类型变成int，其值是42
    x = v             // 接口变量x的动态类型变成*T，其值是(*T)(nil)

类型
=====

* `布尔类型`_
* `数值类型`_
* `字符串类型`_
* `数组类型`_
* `切片类型`_
* `映射类型`_
* `通道类型`_
* `结构体类型`_
* `指针类型`_
* `函数类型`_
* `接口类型`_

类型定义了一个值的集合，以及在这些值上的一组操作和方法。一个类型如果有指定的类型名称，
可以使用这个类型名称表示该类型，如果类型是一个泛型，类型名称后面必须要指定类型实参。
一个类型还可以通过类型字面量来指定，这些类型相当于是匿名的，这种方式可以从存在的类型
基础上产生一个新的复合类型。

语言预声明了一些类型名称，其他类型可以通过类型声明（包括类型参数声明）引入。复合类型，
包括数组、结构体、指针、函数、接口、切片、映射、通道，还可以使用类型字面量来定义。

预声明的类型、定义的类型（defined type）、以及类型参数都是命名类型（named type）。
通过类型别名声明的类型也是一个命名类型。 ::

    Type = TypeName [TypeArgs] | TypeLit | "(" Type ")" .
    TypeName = identifier | QualifiedIdent .
    TypeArgs = "[" TypeList [","] "]" .
    TypeList = Type { "," Type } .
    TypeLit = ArrayType | StructType | PointerType | FunctionType | InterfaceType | SliceType | MapType | ChannelType .

布尔类型
---------

布尔类型是一个预声明的类型bool，它的值是预声明的常量true和false，它是一个定义的类型
（defined type）。

数值类型
---------

数值类型是预声明的类型，包括uint8 uint16 uint32 uint64 int8 int16 int32 int64
float32 float64 complex64 complex128，另外byte是uint8的别名，rune是int32的别名，
uint和int类型的长度是32或者64位，uintptr类型的长度是平台保存一个指针值所需的长度。

为了避免代码移植问题，所有这些数值类型除了byte和rune都是一个定义的类型，也即是
独立的相互区分的类型。如果不同的数值类型混合使用在一个表达式或者赋值语句中，必须
要使用显式转换。例如int32和int不是相同的类型即使在特定的平台上它们有相同的长度。

字符串类型
-----------

字符串的值是一个字节序列（可以为空），字节的个数表示字符串的长度。字符串是不可修改
的，一旦创建后，不可能改变字符串包含的内容。对应的预声明的类型是string，是一个
定义的类型。字符串s的长度可以使用内置函数len来获取，如果字符串是一个常量字符串，
它的长度是一个编译时常量。字符串中的字节可以通过0到len(s)-1的索引来访问，不能获取
一个字符串中单个字节的地址，例如&s[i]是非法的。

数组类型
---------

数组由同一类型的多个元素组成，这个类型称为数组的成员类型，元素的个数称为数组的大小::

    ArrayType = "[" ArrayLength "]" ElementType .
    ArrayLength = Expression .
    ElementType = Type .

数组长度是数组类型的一部分，长度必须是一个可以用int表示的非负整数常量，数组长度可以
使用内置的len函数获取，数组元素可以通过0到len(a)-1的索引进行访问和修改。数组类型总是
一维的数组，但是形式上可以声明成多维类型。一个数组类型T的元素类型不能是T，也不能是
包含了类型T的类型（如果包含的类型是数组或结构体）::

    [32]byte
    [2*N] struct { x, y int32 }
    [1000]*float64
    [3][5]int
    [2][2][2]float64 // 相当于 [2]([2]([2]float64))
    type ( // 非法的数组类型
        T1 [10]T1
        T2 [10]struct{f T2}
        T3 [10]T4
        T4 struct{f T3}
    )
    type ( // 合法的数组类型
        T5 [10]*T5 // T5包含的是一个指针类型
        T6 [10]func() T6 // T6包含的是一个函数类型
        T7 [10]struct {f []T7} // T7包含的是一个结构体，但结构体包含的是一个切片
    )

切片类型
---------

切片描述的是一个底层数组的片段，这个片段的长度称为切片的长度，未初始化的切片的值
是nil。 ::

    SliceType = "[" "]" ElementType .

切片的长度可以通过内置函数len获取，不同于数组，切片的长度可能在运行时改变，切片的
元素可以通过0到len(s)-1的索引来访问和修改。切片一旦初始化就总是与底层的数组进行
关联，切片与其他使用相同的底层数组的切片一起，共享了这个底层的数组。切片的底层数组
可能超过切片的尾部，切片的容量capacity用来表示切片的长度加上数组超过切片尾部的长度
总和，可以通过内置函数cap来获取容量大小。可以在切片的容量大小之上创建出新的切片，
切片还可以添加元素形成一个新的切片。

一个给定元素类型T的新的初始化的切片可以使用内置函数make来创建，make函数会创建出
切片的底层数组::

    make([]T, length, capacity) // 参数capacity是可选的
    make([]int, 50, 100)
    new([100]int)[0:50] // 这两个表达式是等价的

像数组一样，切片总是一维的，但是在形式上可以声明多维的切片使用。对于数组的数组，
内部的元素数组的长度是一样的。但是切片的数组，或者切片的切片，内部切片的长度可能
是不同的，而且内部切片必须单独进行初始化。

映射类型
---------

一个映射是一种类型元素的无序集合，该类型称为映射的元素类型。集合的元素可以用另一
类型的值来索引，称为键类型。一个未初始化的映射的值是nil。 ::

    MapType = "map" "[" KeyType "]" ElementType .
    KeyType = Type .

键类型必须实现比较操作符==和!=，因此键类型不能是函数、映射、或者切片。如果键类型
是接口类型，那么接口类型的动态类型必须实现了比较操作符，否则会报运行时错误。

映射中元素的个数称为映射的长度，可以用内置函数len来获取，但是映射的长度可以在
运行时改变。映射的元素可能在使用赋值或通过索引表达式获取元素时动态增加，也可以
使用内置函数delete和clear删除元素。

可以用内置函数make创建一个新的值为空的映射，需要传递一个map类型和可选的映射容量
初始值，容量初始值不会与映射的大小绑定，映射会自动增长来容纳它存储的元素。一个
nil映射除了不能添加元素外相当于是一个空的映射::

    map[string]int
    map[*T]struct{x,y float64}
    map[string]interface{}
    make(map[string]int)
    make(map[string]int, 100)

通道类型
---------

通道给并行执行的函数提供了一种沟通的机制，这种机制让函数可以发送和接受特定类型的
值来沟通。未初始化的通道的值是nil。 ::

    ChannelType = ( "chan" | "chan" "<-" | "<-" "chan" ) ElementType .

可选的<-操作符用来表示通道的方向，是发送还是接收。如果指定了一个方向，这个通道是
有方向的通道，否则是双方向的通道。一个通道可以通过赋值或类型转换限制只发送或者只
接收。通道可以使用内置函数make来创建，可以传递一个可选的容量参数，用来设置通道中
的缓存大小。如果通道的容量没有设置或者设置为0，表示这个通道是非缓冲通道，只能在
发送端和接收端都准备好的时候才能成功沟通，在对方准备好之前发送或接收操作会被阻塞。
缓冲通道只要发送端的缓存没有满或者接收端的缓存不为空，就可以无阻塞地进行沟通。
一个值为nil的通道始终是没准备好的，不能进行沟通。 ::

    chan T               // 可以接收和发送T类型数据的通道
    chan<- float64      // 只能发送float64类型数据的通道
    <-chan int          // 只能接收int类型数据的通道
    chan<- chan int   // 操作符<-是首先向左与chan结合，如果左边不能结合则向右与chan结合，这个定义相当于chan<- (chan int)
    chan<- <-chan int // 相当于chan<- (<-chan int)
    <-chan <-chan int // 相当于<-chan (<-chan int)
    chan (<-chan int)
    make(chan int, 100)

通道可以使用内置函数close关闭，如果使用接收操作符的多值赋值形式，会通知通道关闭
前接收的值是否还没有被发送。只有发送端才应该关闭通道，不能是接收端，接收端关闭
通道会导致异常。通道不像文件，你一般不需要去关闭通道，只有在接收端需要被通知没有
更多的值时，接收端可能需要关闭通道，例如接收端需要终止range接收循环。

单个通道可以用在发送语句和接收操作中，可以不加同步让任意数量的goroutine调用cap
和len内置函数。通道是一种先进先出的队列，例如，一个goroutine发送一个值给一个通道，
第二个goroutine会以发送这些值的顺序进行接收。

结构体类型
-----------

结构体类型是一组命名的元素，每个元素有一个类型和一个名称。元素的名称如果是非空，
必须在结构体内是唯一的。没有提供元素名称的元素称为内嵌元素（embedded field），
内嵌元素必须用一个类型名称T指定，或者非接口类型的指针*T指定，而且T自己本身不能是
指针，内嵌元素的名称有对应的类型决定。注意元素的名称必须是唯一的，包括用类型作为
名称的内嵌元素。 ::

    StructType = "struct" "{" { FieldDecl ";" } "}" .
    FieldDecl = (IdentifierList Type | EmbeddedField) [Tag] .
    EmbeddedField = ["*"] TypeName [TypeArgs] .
    Tag = string_lit .

    struct {} // 一个空结构体
    struct {
        x,y int
        u float32
        _ float32 // 空名称元素，用于对齐（padding）
        A *[]int  // 切片指针类型的名称为A的元素
        F func()  // 函数类型的名称为F的元素
    }
    struct {
        T1         // 名称为T1
        *T2        // 名称为T2
        P.T3     // 名称为T3
        *P.T4     // 名称为T4
        x,y int // 名称为x和y
    }
    struct {
        T        // 非法因为名字冲突
        *T        // 非法因为名字冲突
        *P.T    // 非法因为名字冲突
    }

结构体x中内嵌类型的元素或函数f，可以通过x.f形式进行访问。这种形式相当于是结构体x
自己定义的元素一样，除了不能在结构体的复合结构字面量中使用这种名称。对于结构体类型
S和命名类型T，T中的成员函数会以下面的形式包含到结构体S的定义中：

1. 如果S内嵌了类型T，S和 `*S` 的函数集合包含了T的函数集合， `*S` 的函数集合还包含
   了 `*T` 的函数集合；
2. 如果S内嵌了类型 `*T`，S和 `*S` 的函数集合都包含了T和 `*T` 的函数集合；

结构体的每个元素声明可选的可以添加一个字符串标记（Tag），空字符串相当于没有设置标记，标记
可以使用在反射和类型鉴定中，其他情况会忽略标记::

    struct {
        x,y float64 "" // 空字符表示没有标记
        name string "any string is permitted as a tag"
        _ [4]byte "ceci n'est pas un champ de structure"
    }
    struct {
        microsec uint64 `protobuf:"1"`
        serverIP6 uint64 `protobuf:"2"`
    }

一个结构体类型T的元素类型不能是T，也不能是包含了类型T的类型（如果包含的类型是数组或结构体）::

    type ( // 非法的结构体类型
        T1 struct { T1 }
        T2 struct { f [10]T2 }
        T3 struct { T4 }
        T4 struct {f [10]T3 }
    )
    type (
        T5 struct {f *T5} // 合法因为包含的是T5的指针
        T6 struct {f func() T6} // 合法因为包含的是一个函数类型
        T7 struct {f [10][]T7 } // 合法因为包含的是一个切片类型
    )


指针类型
---------

指针指向一个特定类型的变量，这个类型称为指针的基类型（base type），指针如果没有初始化
它的值是nil。 ::

    PointerType = "*" BaseType .
    BaseType = Type .

    *Point  // 一个类型指针
    *[4]int // 一个数组指针


函数类型
---------

函数类型定义了有相同参数类型和返回值类型的一组函数，函数类型的变量如果没有初始化它的
值是nil。 ::

    FunctionType = "func" Signature .
    Signature = Parameters [Result] .
    Result = Parameters | Type .
    Parameters = "(" [ ParameterList [","] ] ")" .
    ParameterList = ParameterDecl { "," ParameterDecl } .
    ParameterDecl = [ IdentifierList ] [ "..." ] Type .

    func() // 如果没有参数和返回值，参数列表和返回值为空
    func(x int) int // 返回值如果没有指定名称并且只有一个，可以省略括号
    func(a,_ int, z float32) bool // 参数的个数是命名参数的个数（包括空名称参数_），和未命名参数的类型个数之和
    func(a,b int, z float32) (bool)
    func(prefix string, values ...int) // 函数的最后一个参数可以边长参数
    func(a,b int, z float64, opt ...interface{})(succss bool) // 命名的返回值必须使用括号
    func(int, int, float64)(float64, *[]int) // 多个返回值必须使用括号
    func(n int) func(p *T) // 函数的参数和返回值可以是一个函数类型

接口类型
---------

接口类型定义了一个类型集合，一个接口类型的变量可以存储这个类型集合中任何类型的值。这种满足
接口类型定义的类型称为该类型实现了这个接口，因而能赋值给该接口变量。未初始化的接口变量的值
是nil。 ::

    InterfaceType = "interface" "{" { InterfaceElem ";" } "}" .
    InterfaceElem = MethodElem | TypeElem .
    MethodElem = MethodName Signature .
    MethodName = identifier .
    TypeElem = TypeTerm { "|" TypeTerm } .
    TypeTerm = Type | UnderlyingType .
    UnderlyingType = "~" Type .

一个接口类型由一个接口元素列表定义，接口元素可以是成员函数元素或者类型元素，类型元素是单个类型，
或者单个底层类型（single underlying type），或多个类型的联合。

基本接口
________

最基本的形式，一个接口只声明了一个成员函数列表（可能为空）。这个接口定义的类型集合即那些实现了
这些成员函数的类型，实现接口的类型必须确切地实现接口声明的成员函数。接口定义的类型集合能够完全
由一个成员函数列表来定义的接口称为基本接口。 ::

    interface { // 一个简单的文件接口
        Read([]byte)(int, error)
        Write([]byte)(int, error)
        Close() error
    }
    interface { // 接口中的函数名称必须是唯一的，并且名称不能为空
        String() string
        String() string // 非法，因为名称不唯一
        _(x int)        // 非法，因为名称不能为空
    }
    type Locker interface {
        Lock()
        Unlock()
    }

    // 一个接口可以被多个类型实现，例如S1和S2两个类型（T表示S1或者S2）都实现了上面的文件接口，
    // 不管S1和S2两个类型还定义哪些其他的成员函数
    func (p T) Read(p []byte)(n int, err error)
    func (p T) Write(p []byte)(n int, err error)
    func (p T) Close() error
    func (p T) Lock() {...}
    func (p T) Unlock() {...}
    
    // 并且一个类型可能同时实现多个接口，例如上面的S1和S2还都实现了Locker接口，另外所有的类型
    // 都实现了下面的空接口（empty interface）。空接口代表了所有非接口类型的集合，为了方便，
    // 预声明的类型any是空接口的一个别名[Go1.18]
    interface {}

内嵌接口
________

接口除了包含成员函数声明，还可以内嵌其他接口类型，比如接口T包含了接口E称作接口E被内嵌到了
接口T中 [Go1.14]_. 接口T定义的类型需要实现T和E两者声明的所有函数。当内嵌接口时，如果声明
的函数名称相同，它们必须拥有相同的函数签名。 ::

    type Reader interface {
        Read(p []byte)(n int, err error)
        Close() error
    }
    type Writer interface {
        Write(p []byte)(n int, err error)
        Close() error
    }
    type ReadWriter interface {
        Reader // 包含Reader声明的所有函数
        Writer // 包含Writer声明的所有函数
    }
    type ReadCloser interface {
        Reader
        Close() // 非法，因为成员函数的签名与Reader接口声明的Close函数冲突
    }


泛型接口
________

更一般的接口包含的元素还可以是一个任意的类型T或者~T表示所有底层类型是类型T的类型集合，
或者类型的联合 ``t1|t2|…|tn`` [Go1.18]_。加上声明的成员函数，接口定义的类型集合如下：

1. 空接口定义了所有非接口类型
2. 非空接口定义的类型是所有接口元素定义的类型的交集
3. 一个成员函数元素定义的类型是那些实现了这个成员函数的非接口类型
4. 一个非接口类型元素T定义的类型是这一单个类型T
5. 类型元素~T定义的类型是所有底层类型是T的类型
6. 类型联合 ``t1|t2|…|tn`` 定义的类型是所有这些类型元素定义的类型的并集

类型元素T可以是任意非接口类型表示限定只有该类型满足该接口，但T还可以是一个接口类型，
相当于内嵌了该接口。

类型元素~T的形式，T不能是接口类型，并且T必须是一个底层类型是自己的类型，比如
type MyInt int，只能书写~int，不能书写~MyInt，因为MyInt的底层类型不是自己而是int。
因此~int代表底层类型是int的所有类型，包括int和MyInt。

类型联合 ``t1|t2|…|tn`` 定义的类型是包含所有这些类型定义的类型的并集，类型联合可以
包含接口类型，但这个接口类型只能是只包含类型元素不包含函数声明的泛型接口或者any。类型
联合中每个非接口类型元素定义的类型不能存在重复，比如~int|MyInt是非法的，因为~int定义
了MyInt。 ::

    interface {
        int // 只有int类型实现了该接口
    }
    interface {
        ~int // 所有底层类型是int的类型都实现了该接口，比如int和MyInt
    }
    interface {
        ~int
        String() string // 除了底层类型是int还需要实现String函数
    }
    interface {
        int
        string // 不可能一个类型即是int也是string，因此这个接口没有定义任何类型
    }
    interface {
        ~[]byte  // 合法，包含字节元素的切片是一个底层类型
        ~MyInt   // 非法，因为MyInt不是一个底层类型
        ~error   // 非法，~T中的T不能是接口类型
    }
    type Float interface {
        ~float32 | ~float64 // 定义了所有的float类型
    }
    interface {
        ~int | MyInt    // 非法，因为定义~int定义了MyInt，与MyInt重复
        float32 | Float // 非法，因为Float已经定义了float32
    }

类型元素T和~T不能是一个类型参数，类型参数是用在泛型类或者泛型函数中的泛型参数。
类型联合 ``t1|t2|…|tn`` 不能包含预声明的标识符comparable或者有函数的接口，也
不能包含内嵌了comparable的接口或者内嵌了有函数接口的接口。

泛型接口（非基本接口）只能用作类型约束，或者在用作类型约束的其他接口中作为一个
类型元素使用，不能作为值或变量类型、元素或成员类型、非接口类型等其他用途。

接口类型T不能内嵌接口T，也不能内嵌一个间接或直接内嵌了T的接口::

    interface {
        P           // 假如P是一个类型参数，非法
        int | ~P    // 假如P是一个类型参数，非法
    }
    var x Float     // 非法，非基本接口只能用于类型约束，不能用来声明接口类型的变量
    var x interface{} = Float(nil) // 非法，非基本接口只能用于类型约束
    type Floatish struct {
        f Float     // 非法，非基本接口只能用于类型约束，不能用来声明接口类型的变量
    }
    type Bad interface {
        Bad // 非法，接口内嵌不能出现嵌套循环
    }
    type Bad1 interface {
        Bad2
    }
    type Bad2 interface {
        Bad1 // 非法，接口内嵌不能出现嵌套循环
    }
    type Bad3 interface {
        ~int | ~string | Bad3 // 非法，接口内嵌不能出现嵌套循环
    }
    type Bad4 interface {
        [10]Bad4 // 非法，接口内嵌不能出现嵌套循环
    }

一个非接口类型T实现了接口I，相当于T是接口I定义的类型集合中的一个类型。
一个接口类型T实现了接口I，相当于T定义的类型集合是I定义的类型集合的一个
子集，I可以包含函数声明只要T也包含了这些函数声明，也即如果实现了接口T
也一定实现了接口I。

一个类型T的值实现了接口I，相当于T实现了接口I。

类型属性
=========

* `底层类型`_
* `核心类型`_
* `类型区分`_
* `可赋值性`_
* `常量可表示性`_
* `成员函数集合`_

底层类型
---------

每一个类型T都有一个底层类型（underlying type）：

1. 如果T是预声明的布尔类型、数值类型、字符串类型、或者一个类型字面量，那么T的
   底层类型就是它自己
2. 否则T的底层类型是声明T的语句中关联的原类型对应的底层类型
3. 如果T是一个类型参数，它的底层类型是对应类型约束的底层类型（一个接口类型）

例如::

    type (
        A1 = string
        A2 = A1 // sting、A1、A2的底层类型都是string
    )
    type (
        B1 string
        B2 B1    // string B1 B2的底层类型是string
        B3 []B1
        B4 B3   // []B1 B3 B4的底层类型是[]B1
    )
    func f[P any](x P) // P的底层类型是interface{}

核心类型
--------

每个非接口类型T都有一个核心类型（core type），其核心类型就是它的底层类型。
接口类型T如果满足以下条件之一也有自己的核心类型：

1. 有单个类型U，是T定义的所有类型的底层类型，此时T的核心类型是U，或者
2. T定义的类型只包含通道类型chan E，那么T的核心类型是chan E，或者
3. T定义的类型只包含通道类型chan<- E和chan E，那么T的核心类型是chan<- E，或者
4. T定义的类型只包含通道类型<-chan E和chan E，那么T的核心类型是<-chan E

除此之外的其他接口类型都没有核心类型。

根据定义，核心类型一定不是一个除预声明类型之外的定义类型（defined type），
类型参数，或接口类型。 ::

    type Celsius float32
    type Kelvin float32    // 核心类型是float32
    interface{ int }    // 核心类型是int
    interface{ Celsius|Kelvin } // 核心类型是float32
    interface{ ~chan int }      // 核心类型是chan int
    interface{ ~chan int|~chan<- int} // 核心类型是chan<- int
    interface{ ~[]*data; String() string } // 核心类型是[]*data
    
    // 以下接口没有核心类型
    interface {}
    interface { Celsius|float64 }
    interface { chan int | chan<- string }
    interface { <-chan int | chan<- int }

另外，一些操作包括切片表达式、append、copy操作对核心类型的限制有放松，可以接受
切片和字符串。在这种放松的限制下，如果T定义的所有类型的底层类型都是[]byte或者
string，那么称T的核心类型是bytestring。注意bytestring不是一个真正的类型，它仅仅
用于描述对字节序列的操作而存在，这种字节序列可以是字节切片或者字符串。 ::

    interface{ []byte | string }     // bytestring
    interface{ ~[]byte | myString }  // bytestring

类型区分
---------

两个类型要么相同（identical）要么不同（different）。一个命名类型（named type）
总是与任何其他类型不同（除非是命名类型的别名）。非命名类型要相同，它们的底层类型
字面结构以及元素或成员的类型必须相同：

1. 两个数组类型是相同的，如果它们有相同的元素类型和相同的长度；
2. 两个切片类型是相同的，如果它们有相同的元素类型；
3. 两个结构体类型是相同的，如果它们有相同的元素顺序，并且每个元素都有对应相同的类型、
   名字、标记，定义在不同包中的非导出成员永远是不同的；
4. 两个指针类型是相同的，如果是指向相同的类型；
5. 两个函数类型是相同的，如果有相同的参数个数和返回值个数，并且对应的参数和返回值类型相同；
6. 两个接口类型是相同的，如果它们定义的类型集合相同；
7. 两个映射类型是相同的，如果它们有相同的键和元素类型；
8. 两个通道类型是相同的，如果它们有相同的元素类型和相同的方向；
9. 两个实例化类型是相同的，如果对应的类型相同，并且类型实参都相同；

例如::

    type (
        A0 = []string
        A1 = A0                               // A0 A1 []string是相同的
        A2 = struct{a,b int}                  // A2 struct{a,b int}是相同的
        A3 = int                              // A3 int是相同的
        A4 = func(A3, float64) *A0
        A5 = func(x int, _ float64) *[]string // A4 A5 func(x int, float64) *[]string是相同的
        B0 A0
        B1 []string                           // B0 B1不相同是因为它们是用类型定义创建的新类型
        B2 struct{a,b int}
        B3 struct{a,c int}
        B4 func(int, float64) *B0             // func(int, float64) *B0 与 func(int, float64) *[]string不同是因为B0是新类型
        B5 func(x int, y float64) *A1
        C0 = B0                               // B0 C0是相同的
        D0[P1, P2 any] struct{x P1; y P2}     // P1 P2不同是因为两个命名类型
        E0 = D0[int, string]                  // E0 D0[int, string]是相同的
    )

D0[int, string]与struct{x int; y string}不同是因为前者是实例化类型，后者是类型字面量，
但两者是可以相互赋值的。

可赋值性
---------

如果满足下面的任意一个条件，类型V的值x可以赋给类型T的变量：

1. 类型V和类型T是相同的
2. 类型V和类型T有相同的底层类型，并且不是类型参数，并且V和T至少一个不是命名类型（named type）
3. 类型V和类型T都是通道有相同的元素类型，V是双向通道，并且V和T至少一个不是命名类型（named type）
4. T是一个接口类型，并且不是类型参数，类型V实现了接口T
5. x是预定义标识符nil，并且T是一个指针、函数、切片、映射、通道、接口类型，但不是一个类型参数
6. x是一个无类型的可以用类型T的值表示的常量

如果类型V或者类型T是类型参数，x可以赋给类型T的变量，如果满足下面的一个条件：

1. x是预定义标识符nil，T是类型参数，x可以赋值给T定义的类型集合中的所有类型的变量
2. V不是一个命名类型（named type），T是类型参数，x可以赋值给T定义的类型集合中的所有类型的变量
3. V是一个类型参数，T不是一个命名类型（named type），V定义的每个类型的值可以赋值给类型T的变量

常量可表示性
------------

满足下面任意一个条件，常量x就可以用类型T的值表示，其中T不是一个类型参数：

1. x是类型T定义的值中的一个
2. T是浮点类型，x可以近似到T类型的精度而不上溢，浮点近似使用IEEE 754的round-to-even规则除了
   IEEE的负零被转成无符号零。注意不能用常量表示一个IEEE负零、NaN、无穷。
3. T是一个复数，并且real(x)和imag(x)都可以用T的内部类型（float32或者float64）的值表示

如果T是一个类型参数，需要x都可以用T定义的每个类型的值表示::

    x                       T            x可以用类型T的值表示的原因
    'a'                     byte         97在一个字节范围内
    97                      rune         97在int32范围内
    "foo"                   string       "foo"是一个字符串值
    1024                    int16        1024在int16范围内
    42.0                    byte         42在一个字节范围内
    1e10                    uint64       10000000000在uint64范围内
    2.718281828459045       float32      可以近似到2.7182817用float32表示
    -1e-1000                float64      可以近似到IEEE -0.0被转换成0.0来表示
    0i                      int          0是一个整数
    (42 + 0i)               float32      42.0是一个float32类型的值
    
    x                       T            x不能用类型T的值表示的原因
    0                       bool         0是整数不是一个布尔值
    'a'                     string       'a'是一个字符不是一个字符串
    1024                    byte         1024超出了字节的范围
    -1                      uint16       -1是一个负数不能用无符号值表示
    1.1                     int          1.1是一个浮点数不能用整数表示
    42i                     float32      42i是一个复数不能用浮点数表示
    1e10000                 float64      1e1000会上溢到IEEE正无穷，超出了float64的范围

成员函数集合
------------

一个类型的成员函数集合，确定了可以用这个类型调用的成员函数。每个类型都拥有一个成员函数
集合（可能为空）：

1. 一个定义类型（defined type）T的成员函数集合，是所有使用T作为参数的成员函数
2. 一个定义类型（defined type）T的指针的成员函数集合，是所有使用T或者*T作为参数的成员
   函数，T不能是一个指针或者接口
3. 一个接口定义的成员函数集合，是接口定义的每个类型定义的成员函数集合的交集

成员函数集合中，每个函数名称都必须是非空的唯一的。

代码块
=======

一个代码块是包含在大括号内的零条或多条语句，代码块可以嵌套，代码块会影响作用域。 ::

    Block = "{" StatementList "}" .
    StatementList = { Statement ";" } .

除了显式的代码块，还有以下隐式代码块：

1. 全局代码块（universe block）包含了所有的Go程序源代码
2. 每个包是一个包代码块，包含了该包中所有的Go源代码
3. 每个文件是一个文件代码块，包含了文件中的所有Go源代码
4. 每个if for switch语句都定义了一个自己的代码块
5. 每个switch select语句中的每一条款都定义了自己的代码块

声明和作用域
=============

* `标签`_
* `标识符`_
* `常量声明和iota`_
* `类型声明`_
* `类型参数声明`_
* `变量声明`_
* `函数声明`_
* `成员函数声明`_

每个声明都绑定到一个非空标识符来表示一个常量、类型、类型参数、变量、函数、标签、包。程序中的每个
标识符都必须先声明。相同代码块中的标识符不能声明两次，一个标识符不能同时声明在一个文件代码块和包
代码块中。

空标识符（即一个下划线字符 `_`）可以像其他标识符一样声明，但这种声明是没有真正绑定的因而是一个未声明
的标识符。在包代码块中，标识符init仅能用在init函数的声明中，也想空标识符一样不引入任何绑定。 ::

    Declaration = ConstDecl | TypeDecl | VarDecl .
    TopLevelDecl = Declaration | FunctionDecl | MethodDecl .

声明的标识符的作用域，是标识符代表的特定常量、类型、变量、函数、标签、包的源代码范围。Go使用代码块
来表达词法作用域：

1. 预声明的标识符的作用域是全局代码块（universe block）；
2. 声明在顶层（任何函数之外）的代表常量、类型、变量、函数（非成员函数）的标识符的作用域是包代码块；
3. 导入的包的包名作用域是包含这个导入声明的文件代码块；
4. 函数参数或返回值标识符的作用域是该函数或成员函数的函数体；
5. 函数的类型参数标识符的作用域是函数名称之后以及整个函数体；
6. 类型的类型参数标识符的作用域是类型名称之后以及整个类型体（TypeSpec）；
7. 函数内声明的常量或变量标识符，它的作用域是声明（ConstSpec VarSpec ShortVarSpec）结束之后到代码块结束；
8. 函数内声明的类型标识符，它的作用域是标识符声明（TypeSpec）之后到代码块结束；

在一个代码块中声明的标识符可以在内嵌的代码块中重新声明，内嵌代码块中使用的标识符，代表的是那个由
内嵌代码块声明的那一个。

文件所属哪个包的包说明不是声明，包名字不出现在任何作用域中，它的目的是用来分辨所属同一个包的文件，
以及提供默认包名给导入声明使用。

标签
------

标签（label）使用标签语句声明的，可以用在break continue goto语句中。定义一个未使用的标签是非法的。
与其他标识符不同，标签是没有代码块作用域的，不会与其他不是标签的标识符冲突。标签的作用域是它声明
的函数的整个函数体，剔除该函数包含的任何嵌套函数的函数体。

标识符
-------

空标识符（blank identifier）用一个下划线字符表示，替代正常的标识符用来匿名占位，
它在声明（如结构体中用来占位对齐，函数声明忽略一个类型参数，常量声明忽略一个值）、
赋值语句（忽略多返回值中的值）中有特殊含义。

下面的标识符是在全局代码块（universe block）[Go1.18]_ [Go1.21]_ 中隐式预声明的::

    类型 any bool byte comparable
         complex64 complex128 error float32 float64
         int int8 int16 int32 int64 rune string
         uint uint8 uint16 uint32 uint64 uintptr
    常量 true false iota
    零值 nil
    函数 append cap clear close complex copy delete
         imag len make max min new panic print println
         real recover

可以导出一个标识符来允许另一包中的代码访问，满足下面两个条件的标识符是导出的：

1. 标识符的第一个字母是Unicode编码区Lu定义的大写字母，并且
2. 这个标识符定义在包代码块或者是一个成员变量名或者成员函数名

其他的标识符都是没有被导出的。

标识符的唯一性表示它在一个标识符集合中是唯一的与集合中其他标识符都不同。名字
不同的标识是不同的，未导出的标识符在别的包中总是与该包中其他标识是不同的，因
为该包不能导入使用这个名字。因此包中两个同名标识符是相同的，与其他包中导出的
同名标识符也是相同的，这样会出现这个标识符名字在这个包中不唯一。

常量声明和iota
--------------

常量声明将一个或多个常量标识符绑定到一个常量表达式表示的常量值上。赋值左边的
标识符个数必须与右边的常量表达式值的个数相同。 ::

    ConstDecl = "const" (ConstSpec | "(" { ConstSpec ";" } ")" ) .
    ConstSpec = IdentifierList [[Type] "=" ExpressionList ] .
    IdentifierList = identifier { "," identifier } .
    ExpressionList = Expression { "," Expression } .

如果指定了类型，声明语句中的常量的类型就被指定了，并且右边常量表达式
的值必须可以赋值给这个类型，指定的类型不能是一个类型参数。如果没有指定
类型，那么常量的类型自动与右边的表达式的类型关联，如果右边常量表达式值
的类型是一个无类型（untyped）的常量那么声明的常量也是无类型的。例如表
达式是一个浮点字面量，那么常量标识符也表示一个浮点常量，即使这个浮点常量
的小数部分为零。 ::

    const Pi float64 = 3.14159265358979323846
    const zero = 0.0 // 无类型（untyped）的浮点常量
    const (
        size int64 = 1024
        eof = -1 // 无类型（untyped）的整型常量
    )
    const a, b, c = 3, 4, "foo" // 无类型的整型和字符串常量
    const u, v, float32 = 0, 3

在括号表示的常量声明列表中，表达式列表可以从除第一个常量标识符外的其他
标识符开始忽略书写表达式，后面省略的表达式列表是前面最后一个未省略的表
达式列表的重复。配合预定义标识符iota一起使用，可以简化常量的声明。iota
是一个常量产生器，表示连续的无类型的整型常量。 ::

    const (
        bit0, mask0 = 1 << iota, 1 << iota - 1 // 后面都重复这个表达式列表
        bit1, mask1 // iota是1，相当于 (1 << 1), (1 << 1 - 1)
        _, _
        bit3, mask3 // iota是3，相当于 (1 << 3), (1 << 3 - 1)
    )

每当const关键字出现时，iota被重置成0，即常量声明列表中第一个常量标识符（或列表）
对应的iota是0，后面每个常量标识符（或列表）对应的iota的值依次加一。 ::

    const (
        Sunday = iota // 0
        Monday        // 1
        Tuesday
        Wednesday
        Thursday
        Friday
        Partyday
        numberOfDays // 这个常量没有被导出
    )
    
    const (
        c0 = iota // 0
        c1 = iota // 1
        c2 = iota // 2
    )
    
    const (
        a = 1 << iota // (1 << 0) 1
        b = 1 << iota // (1 << 1) 2
        c = 3
        d = 1 << iota // (1 << 3) 8
    )
    
    const (
        u         = iota * 42 // 无类型的整型常量0
        v float64 = iota * 42 // float64类型的常量42.0
        w         = iota * 42 // 无类型的整型常量84
    )
    
    const x = iota // 0
    const y = iota // 0

    type ByteSize float64
    const (
        _           = iota // 通过空标识符忽略第一个值
        KB ByteSize = 1 << (10 * iota) // 1 << 10 即 0b10000000000 1024
        MB                             // 1 << 20 即 1024*1024
        GB
        TB
        PB
        EB
        ZB
        YB
    )

类型声明
---------

类型声明将一个标识符绑定到一个类型，即将这个类型以这个标识符命名。类型声明有
两种形式，别名声明和类型定义。 ::

    TypeDecl = "type" ( TypeSpec | "(" { TypeSpec ";" } ")" ) .
    TypeSpec = AliasDecl | TypeDef .
    AliasDecl = identifier "=" Type .
    TypeDef = identifier [ TypeParameters ] Type .
    TypeParameters = "[" TypeParamList [ "," ] "]" .
    TypeParamList = TypeParamDecl { "," TypeParamDecl } .
    TypeParamDecl = IdentifierList TypeConstraint .
    TypeConstraint = TypeElem .

别名声明
________

别名声明没有定义新的类型，在标识符的作用域内它是对应类型的一个别名::

    type (
        nodeList = []*Node // nodeList和[]*Node是完全相同的类型
        Polar = polar      // Polar和polar是完全相同的类型
    )

类型定义
_________

类型定义创建了一个新的类型，完全与其他类型不同，包括绑定到这个标识符的那个
原类型。但创建出的类型与原类型有相同的底层类型（underlying type）。这个新
创建的类型称为定义的类型（defined type）。 ::

    type (
        Point struct{x,y float64} // Point和struct{x,y float64}是两个不同的类型
        polar Point               // polar和Point是两个不同的类型
    )
    type TreeNode struct {
        left,right *TreeNode
        value any
    }
    type Block interface {
        BlockSize() int
        Encrypt(src,dst []byte)
        Decrypt(src,dst []byte)
    }

一个定义的类型可以创建自己的成员函数，但是它不会获得原类型的成员函数，
但是接口声明的函数以及组合类型对元素的操作被原样保留到新类型上::

    type Mutex struct { ... }
    func (m *Mutex) Lock()
    func (m *Mutex) Unlock()
    
    type NewMutex Mutex // NewMutex与Mutex有相同的成员变量，但它的成员函数集合为空
    type PtrMutex *Mutex // PtrMutex是一个指针，它的底层类型*Mutex的成员函数集合保持不变，但PtrMutex没有成员函数
    type PrintableMutex struct { // PrintableMutex与结构体有相同的成员变量，*PrintableMutex包含了Mutex的成员函数集合
        Mutex
    }
    
    type MyBlock Block // Block是一个接口，MyBlock与Block有相同的函数集合

类型定义可以用来定义一个不同的布尔、数值、或字符串类型::

    type TimeZone int
    const (
        EST TimeZone = -(5 + iota)
        CST
        MST
        PST
    )
    func (tz TimeZone) String() string {
        return fmt.Sprintf("GMT%+dh", tz)
    }

如果类型定义中指定了类型参数，那这个类型标识符代表的就不是一个具体的类型，称为泛型
类型或者模板类型。泛型类只有实例化成具体的类型才能正常使用。每个类型参数都有一个
类型约束来限定这个类型参数表示的是哪些类型的集合，只有满足类型约束的类型实参才能传
给类型参数。在类型声明中，原类型不能是一个类型参数。泛型类也可以定义成员函数，成员
函数定义必须指定相同的类型参数。 ::

    type List[T any] struct { // 定义了一个泛型类List，该泛型类有一个类型参数T
            next *List[T]     // 该类型参数的类型约束是any，即可以是任何非接口类型
            value T
    }
    type T[P any] P // 非法，P是一个类型参数，定义了一个泛型类T，有一个可以是任何类型的类型参数P，但P是参数类型不能作为原类型
    func f[T any] {
        type L T    // 非法，T是一个类型参数，定义了一个类型L，但T是类型参数不能作为原类型
    }
    func (l *List[T]) Len() int { … }

函数可以指定类型参数变成泛型函数，成员函数的接收参数也可以指定类型参数::

    func min[T ~int|~float64](x, y T) T {
        if x < y {
            return x
        }
        return y
    }
    func (l *List[T]) Len() int { … } // 成员函数接收参数的类型其实是泛型类的实例化形式
    func (p Pair[A, B]) Swap() Pair[B, A] { … }
    func (p Pair[First, _]) First() First { … }

类型参数声明
------------

泛型函数和泛型类都有一个类型参数列表，类型参数列表跟函数参数列表很像，除了函数参数是一个具体的类型，
而类型参数表示的是一个类型集合，另外类型参数列表不能为空，类型参数列表通过方括号定义 [Go1.18]_。

类型参数列表中的类型参数名称除了空标识符外，其他名字都必须具有唯一性，每个类型参数标识符都是一个新的
不同的命名类型（named type）。就像每个函数参数都对应有一个参数类型一样，每个类型参数也都对应有一个
类型约束（type constraint），类型约束限定了类型参数表示的类型范围。引入类型参数增加了解析的模糊性，
当使用类型约束C声明单个类型参数P时， ``type T[P *C]`` 是一个合法的表达式，但这容易被误解为一个数组
类型的类型定义，例如 ``type T [5]int``。为了解决这种模糊性，需要将类型约束放到interface{}中或者在
尾部加一个逗号。另外，泛型类T的类型参数的类型约束，不能直接或间接的引用泛型类T自己。 ::

    [P any]                             // 类型参数P可以是任何类型
    [S initerface{ ~[]byte|string }]    // S只能是底层类型为字节切片的类型或者字符串类型
    [S ~[]E, E any]                     // S只能是底层类型为E类型的切片的类型，类型参数E可以是任何类型
    [P Constraint[int]]                 // P只能是泛型类Constraint用int实例化的类型
    [_ any]
    
    type T[P *C] …                 // P是C类型指针，容易跟声明数组类型混淆
    type T[P (C)] …                // P只能是类型C，容易跟声明数组类型混淆
    type T[P *C|Q] …               // P只能是C类型指针或者类型Q，容易跟声明数组类型混淆
    type T[P interface{*C}] …      // P只能是类型C的指针，使用interface{}避免混淆
    type T[P *C,] …                // P只能是类型C的指针，尾部加逗号避免混淆

    type T1[P T1[P]] …                  // 非法，类型约束引用了泛型类T1自己
    type T2[P interface{T2[int]}] …     // 非法，类型约束引用了泛型类T2自己
    type T3[P interface{m(T3[int])}] …  // 非法，类型约束引用了泛型类T3自己
    type T4[P T5[P]] …                  // 非法，类型约束间接引用了T4自己
    type T5[P T4[P]] …                  // 非法，类型约束间接引用了T5自己
    type T6[P int]struct{f *T6[P]} // 合法，T6不是在类型参数列表中，并且*T6[P]表示的是泛型类实例化类型T6[int]的指针

类型约束是一个接口，类型约束定义了对应类型参数允许的类型实参集合，并控制类型参数对应值支持
的操作。类型参数列表中的类型约束如果只包含单个类型元素或类型联合，在不引起歧义的情况下可以
省略interface{}。预声明的接口类型comparable表示一个严格可比较（strictly comparable）的
非接口类型的类型集合 [Go1.18]_。尽管不是类型参数的接口是可比较的，但不是严格可比较的，因此
它们没有实现comparable，但是它们满足（satisfy）comparable。comparable接口以及直接或间接
内嵌了comparable接口的接口只能用作类型约束，不能作为值或变量的类型、元素或成员的类型、以及
其他用途。 ::

    [T []P]                     // 相当于 [T interface{[]P}]
    [T ~int]                    // 相当于 [T interface{~int}]
    [T int|string]              // 相当于 [T interface{int|string}]
    type Constraint ~int        // 非法，没有interface{}的单个类型元素或类型联合只能出现在类型参数列表中
    int                         // 整型实现了comparable，整型是严格可比较的
    []byte                      // 没有实现comparable，切片是不可比较的
    interface{}                 // 没有实现comparable
    interface{~int|~string}     // 该接口只能用作类型约束，实现了comparable，整型和字符串是可比较的
    interface{comparable}       // 该接口只能用作类型约束，实现了comparable，comparable实现了自己
    interface{~int|~[]byte}     // 该接口只能用作类型约束，切片是不可比较的
    interface{~struct{any}}     // 该接口只能用作类型约束，any不是严格可比较的

一个类型T满足（satisfy）一个类型约束C，T必须是C定义的类型集合中的一个类型，也即类型T
实现（implement）了C。但一个列外是一个comparable类型（不一定严格可比较）可以满足一个
严格可比较的类型约束。具体的，一个类型T满足（satisfy）一个约束C，如果：

1. T实现（implement）了C，或者
2. C可以写成形式interface{comparable; E}，其中E是一个基本接口并且T实现了E并且可比较

因为这个满足约束的例外，比较类型参数类型的操作数可能引发运行时异常（尽管类型参数总是
严格可比较的，因为comparable自己是严格可比的）。 ::

    类型实参          类型约束                    是否满足
    int              interface{~int}             满足，int实现了接口
    string           comparable                  满足，字符串是严格可比较的，实现了接口
    []byte           comparable                  不满足，切片不可比较
    any              interface{comparable; int}  不满足，any没有实现interface{int}
    any              comparable                  满足，any是空接口interface{}可比较
    struct {f any}   comparable                  满足，struct {f any} 成员变量是一个空接口类型的变量，可比较
    any              interface{comparable; m() } 不满足，没有实现基本接口 interface{ m() }
    interface{ m() } interface{comparable; m() } 满足，interface{m()}可比较，并且实现了基本接口 interface{m()}

变量声明
---------

变量声明创建一个或多个变量，绑定到对应的标识符，并且给定每个变量的类型和初始值::

    VarDecl = "var" (VarSpec | "(" {VarSpec ";"} ")" ) .
    VarSpec = IdentifierList ( Type [ "=" ExpressionList ] | "=" ExpressionList ) .
    ShortVarDecl = IdentifierList ":=" ExpressionList

如果给定了表达式列表，它使用赋值语句规则对变量进行初始化，否则变量的值被初始化为
零值（zero value）。如果给定了类型，每个变量的类型被设定为对应的类型，否则变量的
类型根据赋值中的初始值的类型决定。如果这个初始值是一个无类型（untyped）的常量，
它会隐式的转换成它的默认类型（default type），如果是一个无类型的布尔值，它首
先转换成布尔类型。预声明的值nil不能用来初始化一个没有显式指定类型的变量。

编译器可以将一个在函数内部定义的但没有使用的变量声明当成一个错误。 ::

    var i int
    var U,V,W float64
    var k = 0
    var x, y float32 = -1, -2
    var (
        i int
        u,v,s = 2.0, 3.0, "bar"
    )
    var re, im = complexSqrt(-1)
    var _, found = entries[name]
    var d = math.Sin(0.5) // d类型是float64
    var i = 42            // i类型是int
    var t, ok = x.(T)     // t类型是T，ok类型是bool
    var n = nil           // 非法，类型没有显式指定

声明变量还可以使用短变量声明形式，它是对常规变量声明的简写，但是省去
了var和显式类型指定。与常规变量声明不同，短声明形式中的变量可以被重
声明，只要这个变量在相同的代码块中已经声明或者声明在函数参数列表中，
并且有着相同的类型。这种形式相当于在符号操作:=左边使用原来声明的那个
变量，对它进行赋值，而不是重新定义这个变量。注意操作符:=左边的变量
至少有一个非空的变量是要新定义的，另外这些非空的变量的名字必须唯一。
短声明形式仅能出现在函数内部，或在一些情况如if for switch语句中也
能作为声明局部临时变量使用。 ::

    i, j := 0, 10
    f := func() int { return 7 }
    ch := make(chan int)
    r,w,_ := os.Pipe()
    _,y,_ := coord(p)
    field1, offset := nextField(str, 0)
    field2, offset := nextField(str, offset)
    x,y,x := 1, 2, 3 // 非法，左边变量名不唯一

函数声明
---------

函数声明将一个标识符绑定到一个函数类型::

    FunctionDecl = "func" FunctionName [ TypeParameters ] Signature [ FunctionBody ] .
    FunctionName = identifier
    FunctionBody = Block .

如果函数签名声明了返回值参数，函数体中的语句列表必须以终止语句（例如return）结束。
如果函数指定了类型参数，这个函数变成一个泛型函数或者模板函数。一个泛型函数必须在实例
化之后才能被调用或者当作一个值使用。没有类型参数的函数声明可以没有函数体，这种形式
的声明提供一种外部实现函数的签名，例如这个函数是汇编语言实现的。 ::

    func IndexRune(e string, r rune) int {
        for i, c := range s {
            if c == r {
                return i
            }
        }
        // 非法：没有return语句
    }
    
    func min[T ~int|~float64](x,y T) T {
        if x < y {
            return x
        }
        return y
    }

    func flushICache(begin,end uintptr) // 外部实现

成员函数声明
-------------

成员函数是一个拥有接收参数（receiver）的函数，成员函数声明将一个标识符
绑定到成员函数，并且将函数关联到接收参数指定的基类型上。成员函数也称为
方法（method）。 ::

    MethodDecl = "func" Receiver MethodName Signature [ FunctionBody ] .
    Receiver = Parameters .

接收参数是一个函数名前面的一个额外的参数，这个参数必须是非变长的单个参数。
它的类型必须是一个定义的类型（defined type）T或者T的指针，或者T是一个泛型
类型必须声明为泛型类实例化的形式T[P1,P2,...]。T称为是函数接收的基类型，
基类型不能是一个指针或者接口而且必须是同一个包内定义的类型。成员函数绑定
到了基类型上，表示这个函数只相对这个基类型才是可见的，只有基类型才能调用
这个函数。

一个非空的接收参数名称必须在成员函数签名中是唯一的，如果接收参数的值没有这
函数体中使用，接收参数名称可以在声明是省略。函数的参数以及成员函数的参数也
适用这个规则。

对于绑定到基类型的所有非空名称的成员函数，它们的名称必须是唯一的。如果基类
型是一个结构体类型，非空的成员函数名和成员变量名都必须是唯一的。

如果基类型是一个泛型类，必须提供该泛型类的类型参数列表，这样使这些类型
参数可以在成员函数中使用。所有非空的类型参数名、接收参数名、函数参数名都必
须是唯一的。接收参数的参数列表的类型约束是隐式的由泛型类定义决定的。 ::

    func (p *Point) Length() float64 {
        return math.Sqrt(p.x * p.x + p.y * p.y)
    }
    func (p *Point) Scale(factor float64 {
        p.x *= factor
        p.y *= factor
    }
    type Pair[A,B any] struct {
        a A
        b B
    }
    func (p Pair[A,B]) Swap() Pair[B,A] { … } // 类型参数A B，接收参数p，以及函数参数都是函数的局部变量，这些名称必须唯一
    func (p Pair[First, _]) First() First { … } // 两个类型参数，其中一个是未被使用的

表达式
=======

* `操作数`_
* `复合字面量`_
* `函数字面量`_
* `包限定标识符`_
* `成员选择`_
* `类型的成员函数`_
* `成员函数值`_
* `下标表达式`_
* `切片表达式`_
* `类型断言`_
* `函数调用`_
* `泛型实例化`_
* `类型推导`_
* `类型等价`_
* `操作符和优先级`_
* `算术操作`_
* `比较操作`_
* `逻辑操作`_
* `地址操作`_
* `通道接收操作`_
* `类型转换`_
* `常量表达式`_
* `求值顺序`_

一个表达式表示一个值的计算，计算由作用在操作数上的操作符或函数表达。

操作数
-------

操作数表示一个表达式中的元素值。一个操作数可以是一个字面量，一个非空标识符名称
代表的常量、变量、函数，一个用括号引起的表达式。操作数标识符名称如果表示的是
一个泛型函数，可能跟随一个类型参数列表，最后的操作数表示的是一个实例化的函数。

空标识符仅能在赋值语句中的左边才可能作为一个操作数出现。如果操作数的类型是一个
表示空类型集合的类型参数，也即操作数的类型是一个没有定义任何类型的接口，编译器
不应该报错。但泛型函数不能使用这种空集合类型参数进行实例化，任何这种尝试都将导致
实例化失败。 ::

    Operand = Literal | OperandName [ TypeArgs ] | "(" Expression ")" .
    Literal = BasicLit | CompositeLit | FunctionLit .
    BasicLit = int_lit | float_lit | imaginary_lit | rune_lit | string_lit .
    OperandName = identifier | QualifiedIdent .
    QualifiedIdent = PackageName "." identifier .
    Expression = UnaryExpr | Expression binary_op Expression .
    UnaryExpr = PrimaryExpr | unary_op UnaryExpr .

    PrimaryExpr = Operand | Conversion | MethodExpr |
                PrimaryExpr Selector |
                PrimaryExpr Index |
                PrimaryExpr Slice |
                PrimaryExpr TypeAssertion |
                PrimaryExpr Arguments .
    Selector = "." identifier .
    Index = "[" Expression [ "," ] "]" .
    Slice = "[" [Expression] ":" [Expression] "]" | "[" [ Expression ] ":" Expression ":" Expression "]" .
    TypeAssertion = "." "(" Type ")" .
    Arguments = "(" [ ( ExpressionList | Type ["," ExpressionList ] ) ["..."] [","] ] ")" .
    ExpressionList = Expression { "," Expression } .
    Conversion = Type "(" Expression [ "," ] ")" .
    MethodExpr = ReceiverType "." MethodName .
    ReceiverType = Type .

    CompositeLit  = LiteralType LiteralValue .
    LiteralType   = StructType | ArrayType | "[" "..." "]" ElementType | SliceType | MapType | TypeName [ TypeArgs ] .
    LiteralValue  = "{" [ ElementList [ "," ] ] "}" .
    ElementList   = KeyedElement { "," KeyedElement } .
    KeyedElement  = [ Key ":" ] Element .
    Key           = FieldName | Expression | LiteralValue .
    FieldName     = identifier .
    Element       = Expression | LiteralValue .

复合字面量
-----------

复合字面量相当于通过大括号提供的初始值产生了一个新的复合类型值，
元素的初始化值可以通过可选的键值方式提供。对应复合类型的核心类型
（core type）T必须是一个结构体、数组、切片、或者映射类型。其中元素
的类型以及键的类型必须可以直接赋值的，中间不会有额外的类型转换。其中
的键值被解析成结构体的成员变量，数组和切片的下标，映射类型的键。对应
映射类型，所有的元素都必须有键值。不能用同一个成员变量或键值指定多个
元素。

对于结构体字面量要符合下面的规则：

- 键值必须是结构体类型声明的成员变量名
- 不包含键值的元素列表必须按结构体中成员变量的声明顺序
- 如果任一元素指定了键值，所有元素都必须指定键值
- 使用键值的方式，不需要给每个成员指定值，未指定的元素会被初始化为零值
- 不能给另一个包中的结构体类型中未导出的成员赋初始值

对于数组和切片类型字面量：

- 每个元素都有一个整型索引值表示该元素在数组中的位置
- 带键值的元素使用这个整型索引作为键值，必须是int类型的非负值
- 如果一个元素没带键值，它的键值是前一个有键值的键值加一，如果是第一个元素它的键值是0

结构体字面量和数组字面量示例::

    type Point3D struct {x,y,z float64}
    type Line struct {p,q Point3D}
    origin := Point3D{} // zero value
    line := Line{origin, Point3D{y:-4, z:12.3}} // q.x zero value
    var pointer *Point3D = &Point3D{y: 1000} // 可以获取复合结构字面量的地址

    p1 := &[]int{} // p1指向的是一个初始化的空切片，其长度为0
    p2 := new([]int) // p2指向一个未初始化的nil切片，其长度为0
    buffer := [10]string{} // 数组长度为10，每个元素的值为空字符串
    intset := [6]int{1, 2, 3, 5} // 后两个元素的值为0
    days := [...]string{"Sat", "Sun"} // 数组长度为2
    []T{x1, x2, … xn} // 相当于 tmp := [n]T{x1, x2, … xn} tmp[0:n]

数组、切片、映射类型T的初始化列表，因为元素和键值的类型是确定的，因此可以忽略字面量的
类型。类似的，如果元素和键值的类型是复合类型的指针，可以忽略取地址符&::

    [...]Point{{1.5, -3.5}, {0, 0}} // 相当于 [...]Point{Point{1.5, -3.5}, Point{0, 0}}
    [][]int{{1,2,3}, {4,5}} // 相当于 [][]int{[]int{1,2,3}, []int{4,5}}
    [][]Point{{{0,1}, {1,2}}} // 相当于 [][]Point{[]Point{Point{0,1}, Point{1,2}}}
    map[string]Point{"origin":{0,0}} // 相当于map[string]Point{"origin": Point{0,0}}
    map[Point]string{{0,0}:"origin"} // 相当于map[Point]string{Point{0,0}: "orig"}
    type PPoint *Point
    [2]*Point{{1.5, -3.5}, {}} // 相当于 [2]*Point{&Point{1.5, -3.5}, &Point{}}
    [2]PPoint{{1.5, -3.5}, {}} // 相当于 [2]PPoint{PPoint(&Point{1.5, -3.5}), PPoint(&Point{})}

如果复合字面量使用类型名字进行初始化，并且使用在if for switch语句中，初始化的大括号很容易误解析成
语句块的大括号，为了避免这种解析错误，需要将字面量包含在括号内::

    if x == T{a,b,c}[i] { … } // 会解析错误，T{}会被解析成if语句的代码块
    if x == (T{a,b,c}[i]) { … } // 需要将复合字面量包含在括号中
    if (x == T{a,b,c}[i]) { … } // 或者整个if语句的条件包含在括号中

数组、切片、映射字面量的示例::

    primes := []int{2,3,5,7,9,2147483647}
    vowels := [128]bool{'a':true, 'e':true, 'i':true, 'o':true, 'u':true, 'y':true}
    filter := [10]float32{-1, 4:-0.1, -0.1, 9:-1} // {-1, 0, 0, 0, -0.1, -0.1, 0, 0, 0, -1}
    noteFrequency := map[string]float32{
        "C0": 16.35, "D0": 18.35, "E0": 20.60, "F0": 21.83,
        "G0": 24.50, "A0": 27.50, "B0": 30.87,
    }

函数字面量
-----------

函数字面量是一个匿名函数，函数字面量不能声明类型参数。 ::

    FunctionLit = "func" Signature FunctionBody .

函数字面量可以赋值给一个变量，或者直接调用。函数字面量是一个闭包（closure），
它可以使用当前函数声明的变量，这些变量共享在当前函数和函数字面量中。闭包定义时
引用的外部变量，可能在闭包执行时（不管是在协程中执行还是保存以后执行）已经超出
生存期了，这时Go运行时会自动延长对应变量的生命期，例如将变量从栈中移到堆中以保
证变量在闭包执行过程中总是有效的。 ::

    func(a,b int, z float64) bool { return a*b < int(z) }
    f := func(x,y int) int { return x + y }
    func(ch chan int) { ch <- ACK } (replyChan)

包限定标识符
------------

限定标识符是一个用包名前缀限定的标识符，包名和标识符名称必须都不是空标识符::

    QualifiedIdent = PackageName "." identifier .

限定标识符用来访问另一个包中导出的定义在包代码块中的标识符，并且这个包已经
导入::

    math.Sin // 标识math包中的Sin函数

成员选择
---------

其中x时一个基本表达式，并且不是一个包名，成员选择表达式表示f是值x的一个成员
变量或者成员函数。f不能是一个空标识符，最后表达式的类型是f的类型或者f返回值
的类型。 ::

    x.f

成员选择需要遵循下面的规则:

1. 对于类型T或*T的变量x（T不能是指针或接口），x.f表示类型T第低层次的成员变量
   或成员函数，如果T没有这样的成员，表达式非法;
2. 对于接口类型I的变量x，x.f表示存储在x中的动态值实际实现的成员函数，如果接口
   没有这个成员函数，表达式非法；
3. 如果x是一个定义的指针类型，并且 ``(*x).f`` 合法选择了成员变量f（不能是函数），
   那么可以使用省略的形式 ``x.f`` 来代替；
4. 如果x是一个值为nil的指针，并且x.f是一个结构体成员，对x.f的赋值或者求值都会
   导致运行时异常；
5. 如果x是一个值为nil的接口，调用或者求值成员函数x.f都会导致运行时异常；

例如::

    type T0 struct {
        x int
    }

    func (*T0) M0()

    type T1 struct {
        y int
    }

    func (T1) M1()

    type T2 struct {
        z int
        T1
        *T0
    }

    func (*T2) M2()

    type Q *T2
    var t T2
    var p *T2
    var q Q = p
    t.z
    t.y     // t.T1.y
    t.x     // (*t.T0).x
    p.z     // (*p).z
    p.y     // (*p).T1.y
    p.x     // (*(*p).T0).x
    q.x     // (*(*q).T0).x
    p.M0()  // (*p).T0.M0()
    p.M1()  // (*p).T1.M1()
    p.M2()  // p.M2()
    t.M2()  // (&t).M2()
    q.M0()  // 非法，规则第3条，(*q).T0.M0() 有对应的成员但不是成员变量

类型的成员函数
--------------

如果M是T类型的成员函数，T.M是一个像正常函数一样可调用的函数，不同的是它有
一个额外的接收参数。 ::

    MethodExpr    = ReceiverType "." MethodName .
    ReceiverType  = Type .

例如::

    type T struct {
        a int
    }
    func (t T) Mv(a int) int { return 0 } // 接收参数传值
    func (t *T) Mp(f float32) float32 { return 1 } // 接收参数传指针
    var t T
    T.Mv // 表示传值的成员函数，函数签名为 func(t T, a int) int
    T.Mp // 类型T并没有定义Mp
    (*T).Mp // 类型*T定义了Mp
    (*T).Mv // 类型*T还定义了Mv
    t.Mp(0) // 但是变量可以自由访问，相当于调用了(*T)定义的函数 (*T).Mp(&t, 0)，只要可以获取t的地址
    t.Mv(7) // 下面的调用方式都是等价的
    T.Mv(t, 7)
    (T).Mv(t, 7)
    f1 := T.Mv; f1(t, 7)
    f2 := (T).Mv; f2(t, 7)

    (*T).Mp // 表示传指针的成员函数，其签名为 func(t *T, f float32) float32
    (*T).Mv // 由于指针可以调用接收参数传值的成员函数，这个表达式也是合法的，其签名为 func(t *T, a int) int
    T.Mp // 非法，因为值类型不能调用传指针的成员函数

``func (t *T, a int) int`` 并没有创建新的函数，它只是间接的调用了底层的 func (t T, a int) int，
将指针指向的值传给底层函数。从接口类型中获取成员函数也是合法的，这样获取的函数它的第一个参数接口类型。

成员函数值
-----------

如果表达式 x 对应的类型是静态类型T（T可以是接口类型），并且M是类型T的成员函数，
x.M称为成员函数值，它是一个可调用的函数，在成员函数值求值时，表达式x的值先被求值
并保存，后续调用时保存的值会被当作接收参数传入。 ::

    type S struct { *T }
    type T int
    func (t T) M(a int) { print(t) }
    t := new(T)   // t 的类型是 *T
    s := S{T: t}
    f := t.M // 接收参数的值*t被求值并保存在变量f中
    g := s.M // 接收参数的值*(s.T)先计算并保存在变量g中
    *t = 42  // 这里并不影响保存在f和g中的值
    f(1) // 相当于f保存了一个T类型的值并且与函数M进行了关联

    type T struct {
        a int
    }
    func (t T) Mv(a int) int { return 0 } // 接收参数传值
    func (t *T) Mp(f float32) float32 { return 1 } // 接收参数传指针
    var t T
    var pt *T
    func makeT() T
    t.Mv // 相当于产了一个签名为 func(int) int 的函数
    t.Mv(7) // 下面两种调用方式是等价的
    f := t.Mv; f(7)
    pt.Mp // 相当于产生了一个签名为 func(float32) float32的函数
    pt.Mv // 等价于 (*pt).Mv
    t.Mp // 等价于 (&t).Mp，但是t必须是可以寻址的
    f := pt.Mp; f(7) // 相当于 pt.Mp(7)，等价于 (*T).Mp(pt, 7)
    f := pt.Mv; f(7) // 相当于 pt.Mv(7)，等价于 (*pt).Mv(7)，等价于 T.Mv(pt, 7)
    f := t.Mp; f(7) // 相当于 t.Mp(7)，相当于 (&t).Mp(7)，相当于 (*T).Mp(&t, 7)
    f := makeT().Mp // 非法，makeT()的结果不能获取地址
    var i interface { M(int) } = myVal // 类型可以是接口
    f := i.M; f(7) // 相当于 i.M(7)

下标表达式
----------

下标表达式表示数组、数组指针、切片、字符串、映射中的一个元素，
其中x称为索引或者键值。 ::

    a[x]

如果a不是一个映射类型或者类型参数:

1. 索引x必须是一个无类型的常量，或者核心类型（core type）必须式一个整型
2. 常量索引必须是一个非负值，能用int类型表示
3. 无类型的常量索引，会给定类型为int
4. 索引必须在0到len(a)-1的范围内，否则会越界

如果a是数组:

1. 常量索引必须在范围内
2. 如果x的值超出范围，会出现运行时异常
3. 表达式a[x]的类型是元素类型

如果a是数组指针:

1. 等价于 `(*a)[x]`

如果a是切片:

1. 如果x的值超出范围，会出现运行时异常
2. 表达式a[x]的类型是元素类型

如果a是字符串:

1. 如果字符串是常量，常量索引必须在范围内
2. 如果x的值超出范围，会出现运行时异常
3. a[x]是一个非常量字节值，其类型是byte
4. a[x]不能被赋值

如果a是一个映射:

1. x的类型必须可以赋值给映射的键类型
2. 如果映射包含键x，那么a[x]是对应键的元素值，其类型是元素类型
3. 如果映射是nil或者不包含这个键，那么a[x]表示元素类型的零值

如果a是一个类型参数P:

1. a[x] 必须对所有P定义的类型都是合法的
2. P定义的类型的元素类型都必须是相同的，这种情况下string的元素类型是byte
3. P定义的类型包含一个映射类型，其他类型也必须是映射类型，并且所有的键类型必须相同
4. 如果a[x]是P实例化后的数组、切片、字符串、映射元素，其类型就是这个元素的类型
5. 如果P定义的类型包含字符串类型，a[x]可能不能被赋值

其他形式的a[i]都是非法的。

映射索引可以产生一个额外的无类型的bool值，表示键x是否在映射中
存在::

    v, ok = a[x]
    v, ok := a[x]
    var v, ok = a[x]

对一个nil隐射的元素进行赋值会导致运行时错误。

切片表达式
----------

切片表达式中a的核心类型（core type）必须是字符串、数组、数组指针、切片、或者是bytestring。
low和high表示a中元素范围，从0开始其长度等于hight-low。 ::

    a[low : high]

例如::

    a := [5]int{1,2,3,4,5}
    s := a[1:4] // 创建了一个类型为[]int，长度为3，容量为4的切片s
    s[0] == 2
    s[1] == 3
    s[2] == 4
    a[2:] // 相当于 a[2:len(a)]
    a[:3] // 相当于 a[0:3]
    a[:]  // 相当于 a[0:len(a)]
    pa[low:hight] // 相当于(*pa)[low:hight]

对于数组和字符串，索引必须 ``0<=low<=high<=len(a)``，否则越界。对于切片，high
的最大值是cap(a)而不是len(a)。常量索引必须是非负的int值，对于数组和字符串常量，
常量索引必须在范围内，如果两个索引都是常量必须low<=high。如果索引在运行时越界，
会导致运行时异常。

如果a是字符串或者切片，切片表达式的结果是一个对应类型的非常量值。如果a是一个数组，
那么a必须是可以取地址的。如果切片表达式是一个合法的表示nil的切片，那么其结果是
nil，否则结果切片于原操作数一起共享底层的数组::

    var a [10]int
    s1 := a[3:7]  // s1的底层数组是a，&s1[2] == &a[5]
    s2 := s1[1:4] // s2和s1的底层数组都是a，&s2[1] == &a[5]
    s2[1] = 42 // s2[1] == s1[2] == a[5] == 42，它们都引用相同的底层数组元素
    var s []int
    s3 := s[:0] // s3 == nil

切片表达式还可以指定切片的容量 a[low : high : max]，对应切片的容量为max - low，
这种形式下，只有low可以省略。索引的范围必须满足0<=low<=high<=max<=cap(a)，其中
a的核心类型必须是数组、数组指针、或者切片，不能是字符串::

    a := [5]int{1,2,3,4,5}
    t := a[1:3:5] // 创建了一个类型为[]int长度为2，容量为4的切片
    t[0] == 2
    t[1] == 3
    t[2] // 非法，因为切片的长度为2，索引越界
    pa[low:high:max] // 相当于 (*pa)[low:high:max]

类型断言
--------

x是一个接口类型（不能是类型参数）的变量，类型断言表示x不是nil并且保存的是类型T的值。
如果T不是一个接口类型，表示x的动态类型是T，此时T必须实现了接口，否则断言表达式是
非法的。如果T是一个接口类型，表示x的动态类型实现了接口T。 ::

    x.(T)

如果类型断言成立，那么表达式的值是x，表达式的类型是T。如果断言失败，会产生一个
运行时错误。因此尽管x的动态类型只能在运行时知道，但是表达式x.(T)的类型在正确的程序
中是确定的类型T。

类型断言可以使用的特别的赋值语句中，它返回一个额外的bool类型的值，表示断言成功还是
失败。如果断言失败表达式的值是T的零值，并且bool值为false，这种情况下不会产生运行
时异常::

    var x interface{} = 7      // x的动态类型是int，值为7
    i := x.(int)               // i的类型是int，值为7
    type I interface { m() }
    func f(y I) {
        s := y.(string)        // 非法，字符串没有实现I接口
        r := y.(io.Reader)     // io.Reader是接口，动态类型y必须同时实现接口I和接口io.Reader
        …
    }
    v,ok = x.(T) // v的动态类型是T，值是x或者T的零值，ok是true或者false
    v,ok := x.(T)
    var v,ok = x.(T)
    var v,ok interface{} = x.(T)

函数调用
--------

表达式中f的核心类型函数类型F，参数必须是单值表达式并且可以赋值给F的参数类型，
参数会在函数调用前求值，表达式的值是函数的返回值，表达式的类型是函数的返回值
类型。成员函数的调用是类似的，除了成员函数多了一个接收参数。如果f是一个泛型
函数，在使用前或者用作函数值前必须先实例化。 ::

    f(a1, a2, … an)

函数调用时，函数值和参数值先被求值，然后将值传递给函数并执行。当函数返回时，
会将返回值返回个调用者。如果调用了一个nil的函数值，会导致运行值错误。

特殊的情况，如果函数或成员函数g的返回值个数以及类型与函数或成员函数f的参数
个数和类型时匹配的，可以直接将g的结果当作f的参数进行调用。g必须返回至少一个
参数，并且对f的调用不能除了调用g外的其他参数。如果f时一个变长参数函数，会将
匹配正常参数剩下的g的返回值传递给变长参数。

成员函数的调用x.m()只有当x的类型关联的成员函数集合包含有x并且传的实参可以
赋值给成员函数的参数。如果x是可以取地址的，并且&x对应的类型关联的成员函数
包含m，那么x.m()相当于(&x).m()。

不存在相同的成员函数类型，也不存在成员函数字面量。

函数调用示例::

    math.Atan2(x,y)
    var p Point
    var pt *Point
    p.Scale(3.5)
    pt.Scale(3.5)
    func Split(s string, pos int) (string, string) {
        return s[0:pos], s[pos:]
    }
    func Join(s,t string) string {
        return s + t
    }
    if Join(Split(value, len(value)/2)) != value { // 返回值个数和类型于参数个数和类型匹配
        log.Panic("test fails")
    }

如果f是变长参数类型，那么变长参数相当于是类型T的一个切片类型[]T，如果调用f是没有传递
参数给变长参数，对应的值为nil。否则，对应数量的类型T的值用来生成一个底层数组并绑定到
切片。如果传递一个可以赋值给切片类型[]T的值，并且使用...操作符，那么这个值会直接传递
给切片，不会去创建新的切片。 ::

    func Greeting(prefix string, who ...string)
    Greeting("nobody") // 切片who的值是nil
    Greeting("hello:", "Joe", "Anna", "Eileen") // 切片who绑定到了一个包含三个元素的底层数组
    s := []string{"James", "Jamine"}
    Greeting("goodbye:", s...) // 切片直接传给了who，不会创建新的切片


泛型实例化
----------

泛型函数或类型通过使用具体的类型实参，替换类型形参来实现实列化 [Go1.18]_。
实例化一个类型得到一个新的非泛型的命名类型（named type），实例化一个函数
得到一个非泛型函数。 ::

    T[type arguments]

实例化过程分为两步：第一步进行类型参数替换；第二步检查类型约束是否满足条件，
如果不满足实例化会失败::

    类型参数列表         类型实参       是否满足参数约束
    [P any]             int           int满足any
    [S ~[]E, E any]     []int, int    []int满足~[]int，int满足any
    [P io.Writer]       string        非法，string不满足io.Writer
    [P comparable]      any           any满足comparable，但是没有实现comparable接口

对于泛型类，所有的类型实参都必须提供。对于泛型函数，类型实参需要显式提供，
或者部分的或全部通过函数参数的类型自动进行推导，这种情况可以省略一部分或全部
类型实参，如果部分忽略只能忽略参数的后边部分至少第一个类型实参必须提供::

    func sum[T ~int|~float64|~string](x... T) T { … }
    x := sum // 非法，x的类型不明
    intSum := sum[int] // intSum的类型是 func （x... int) int
    a := intSum(2, 3) // 推导T的类型是int
    b := sum[float64](2.0, 3)
    c := sum(b, -1) // 推导T的类型是float64
    type sumFunc func(x... string) string
    var f sumFunc = sum // 推导T的类型是string，相当于sum[string]
    f = sum // 推导T的类型是string，相当于sum[string]
    func apply[S ~[]E, E any](s S, f func(E) E) S { … }
    f0 := apply[] // 非法，类型实参不能为空
    f1 := apply[[]int] // 第二个类型参数E自动推导
    f2 := apply[[]string, string] // 两个类型参数都显式提供
    var bytes []byte
    r := apply(bytes, func(byte) byte { … }) // 两个类型参数都通过函数参数类型自动推导

类型推导
---------

使用泛型函数时，如果对应的类型实参可以通过函数调用的上下文自动推导，泛型函数的部分或
全部类型实参都可以不提供。如果一个没有提供的类型实参可以自动推导出来，并且可以成功给
被推导的类型参数进行实例化，那么类型推导是成功的。否则类型推导失败，程序不合法。

类型推导利用类型之间的关系来进行推导，例如一个函数实参必须能赋值给对应的函数参数，这
就建立了实参类型与函数参数类型之间的关系。如果这两个类型至少有一个是类型参数，类型推导
可以利用满足赋值关系的类型实参来推导类型参数。同样的，类型推导还利用了一个事实，即一个
类型实参必须满足对应类型参数的约束。

每一对这样的来自于一个或多个泛型函数的类型匹配，形成了包含一个或多个类型参数的类型等式。
推导未提供的类型实参即意味着求解类型等式方程组中对应的那些类型参数。例如::

    func dedup[S ~[]E, E comparable](S) S { … } // dedup移除切片中任何重复的元素
    type Slice []int
    var s Slice
    s = dedup(s) // 等价于s = dedup[Slice, int](s)

为了程序的正确性，切片类型Slice的变量s必须能够赋值给函数参数的类型S。为简单起见，类型
推导忽略赋值的方向性，这样的一个Slice与S两个类型之间的关系可以用一个对称的类型等式表示，
Slice ≡A S（或者 S ≡A Slice），其中≡A中的A表示两个类型必须满足可赋值性规则的类型等价
（详情见下面类型等价部分）。类似的，类型参数S必须满足它的约束~[]E，这个满足约束的关系
可以用类型等式 S ≡C ~[]E 来表示。其中 X ≡C Y 表示X满足约束Y。这样就得到一组两个类型等
式的方程组::

    Slice ≡A S      (1)
    S     ≡C ~[]E   (2)

然后就可以来求解S和E这两个类型参数，从方程式（1）编译器可以推导S的类型实参式为Slice。同样
的，因为Slice的底层类型为[]int，类型[]int必须满足对应的约束[]E，因此编译器可以推导，E的
类型必须是int。因此从这两个类型等式中可以推导出，S的类型是Slice，E的类型是int ::

    S ➞ Slice
    E ➞ int

给定一组类型等式，需要求解的类型参数是那些在函数中需要实例化的并且没有提供类型实参的类型
参数。这些类型参数被称为绑定的类型参数（bound type parameters）。例如，上面dedup函数中，
类型参数S和E都是绑定的类型参数，需要通过类型推导来解决。一个泛型函数调用中一个实参本身也
可能是一个泛型函数，那么这个内部泛型函数的类型参数也是绑定的类型参数的一部分。一个函数的
实参类型可能包含另一个泛型函数的类型参数（比如泛型函数内部的一个函数调用），这些类型参数
可能出现在类型等式中，但它们在当前上下文中不是绑定的。一组类型等式总是为了用来求解绑定的
类型参数，即使方程组中出现了其他类型参数，只有绑定的类型参数才会被求解。

类型推导支持泛型函数的调用以及将泛型函数赋值给显式函数类型的变量两种情况的类型推导。这包括
将泛型函数作为另一个函数（可以也是泛型函数）的实参，或者将泛型函数作为返回值返回。类型推导
利用每个这样的场景特有的一组类型等式，来实现对类型参数的求解。对应的类型等式说明如下（这里
为了清晰性省略了类型参数列表）：

1. 对于函数调用 f(a0, a1, …)，其中f或者实参ai是一个泛型函数：
   每一对类型(ai, pi)将产生一个类型等式 typeof(pi) ≡A typeof(ai)，其中ai不是无类型的常量；
   如果ai是一个无类型的常量cj，并且 typeof(pi) 是一个绑定的类型参数Pk，类型对(cj, Pk)会被
   有别于类型等式单独收集起来；
2. 对于赋值 v = f，其中f是泛型函数，v是函数类型（非泛型）的变量，将产生 typeof(v) ≡A typeof(f)
3. 对于返回语句 return …, f, … 其中f是一个泛型函数，对应的返回值r是一个函数类型（非泛型）的变量，
   那么将产生 typeof(r) ≡A typeof(f)；

额外地，每个类型参数Pk与对应的类型约束Ck，都产生一个类型等式 Pk ≡C Ck。

相比于从无类型的常量中获取类型信息，类型推导给予从有类型操作数中获取类型信息更高的
优先权。因此，推导的流程分为两步：

1. 使用类型等价来求解类型等式中的绑定类型参数，如果等价失败，类型推导失败；
2. 对每个绑定的类型参数Pk，如果没有对应的类型实参被推导出来，将利用与Pk相关的一个或多个
   类型对(cj, Pk)来求解，首先对于所有的cj确认一个常量类别，Pk的类型实参是这个常量类别
   的默认类型。如果常量类型冲突不能确定出一个常量类别，类型推导失败；

如果经过这两步，还存在未找到的类型实参，那么类型推导失败。如果推导成功，类型推导会为每个
绑定的类型参数确定对应的类型实参，即每个类型参数Pk对对应一个实参类型是Ak，Pk ➞ Ak。

类型实参Ak可能是一个复合类型，它的元素类型可能包含一个或多个其他绑定类型参数。在类型推导
不断简化处理过程中，每个类型实参中的绑定类型参数会被它们各自的类型实参替换，直到每个类型
实参中不再包含任何绑定的类型参数。

如果在类型实参中存在绑定的类型参数的循环引用，即某个类型参数间接或直接引用了自身，这将导致
推导无法进行简化，导致类型推导失败。否则，推导是成功的。

类型等价
---------

类型推导是利用类型等价来求解类型等式的。类型等价递归的比较一个等式中的两个类型，其中一个
或两个类型都可能是或者包含绑定的类型参数，为这些类型参数寻找合适的类型实参让等式两边的类型
匹配（根据上下文关系，让两个类型一致或者赋值兼容）。为达到这个目的，类型推导维护一个绑定
的类型参数到推导出的类型实参的映射，这个映射会在类型等价过程中被查阅并更新。

初始情况下，绑定的类型参数是已知的，但是映射为空。在类型等价过程中，如果一个新的类型实参A
被推导出来，对应的类型参数到类型实参的映射 P ➞ A 会添加到映射中。反过来，当比较类型时，
一个已知的类型实参（映射中存在的一个类型实参）会替换到对应类型参数的位置。随着类型推导的
进行，映射会被不断添加元素，直到所有的类型等式都被考虑到，或者直到类型等价失败。如果没有
任何一个类型等价的步骤是失败的，并且映射中对每个类型参数都有一个元素与之对应，那么类型推导
成功。

例如，对给定的包含绑定类型参数P的类型等式::

    [10]struct{ elem P, list []P } ≡A [10]struct{ elem string; list []string }

类型推导从一个空的映射开始，类型等价首先比较等式两边两个类型的顶层结构。上面示例两个都是
相同长度的数组，它们是等价的如果它们的元素类型等价。每个元素都是结构体类型，它们是等价的
如果有相同数量的成员并且名称相同并且成员类型等价。P的实参是未知的（因为不存在对应的映射），
因此将P与string等价添加到映射中 P ➞ string。等价成员list的类型，需要类型[]P和[]string
等价，因为P的类型实参已经是已知的（已存在对应的映射元素），将类型参数P替换成实参string，
替换后的成员类型string与string相同，因此等价的步骤成功完成。最后类型推导成功，因为只有一
个类型等式，并且等价步骤都是成功的，映射元素都完全对应。

类型等价根据两个类型必须是相同的（identical）、赋值兼容的、还是结构相等的，使用精确等价
（exact）和宽松等价（loose）两种匹配模式。具体的类型等价规则详情见附录部分。

对于类型赋值等式 X ≡A Y，其中X和Y是函数参数赋值（包括返回参数）涉及的两个类型，两个类型的
顶层类型结构可以是宽松等价的，但是元素类型必须精确等价。对于类型约束等式 P ≡C C，其中P是
类型参数C是对应约束，它们的等价规则如下：

1. 如果C有一个核心类型（core type），P有一个已知的类型实参A，C的核心类型必须与类型A是宽松
   等价的；如果P没有已知的类型实参并且C只包含一个类型元素T并且T不是用~表示的底层类型，那么
   将P等价到T的映射关系 P ➞ T 添加到映射中；
2. 如果C没有核心类型并且P有一个已知的类型实参A，A必须有C定义的所有成员函数，并且对应的函数
   类型是精确等价的；

当求解类型约束等式时，求解一个等式可能推导出额外的类型实参，根据这些类型实参又可能反过来解决
出其他等式。只要有新的类型实参被推导出来，类型推导会一直重复类型等价的流程。

操作符和优先级
--------------

操作符结合操作数形成表达式。 ::

    Expression = UnaryExpr | Expression binary_op Expression .
    UnaryExpr = PrimaryExpr | unary_op UnaryExpr .
    binary_op  = "||" | "&&" | rel_op | add_op | mul_op .
    rel_op     = "==" | "!=" | "<" | "<=" | ">" | ">=" .
    add_op     = "+" | "-" | "|" | "^" .
    mul_op     = "*" | "/" | "%" | "<<" | ">>" | "&" | "&^" .
    unary_op   = "+" | "-" | "!" | "^" | "*" | "&" | "<-" .

比较操作符在后面介绍，对于其他的二元操作符，操作数的类型必须是一致的，
除非操作涉及左移或右移或无类型的常量。两个操作数都是常量的情况在后面
的常量表达式部分介绍。

除了左移右移操作，如果一个操作数是一个无类型的常量另一个操作数不是，
那么常量会隐式的转换成另一个操作数的类型。左移右移操作的右操作数必须是
一个整型 [Go1.13]_，或者是一个无类型的常量其值可以用uint表示。如果
左移右移操作的左操作数是一个无类型的常量而右操作数不是，这个常量会被
隐式的转换成当这个常量替换这个位移表达式后应该的类型。 ::

    var a [1024]byte
    var s uint = 33        // 假设int是64位整数
    var i = 1<<s           // 假设var i = 1，因此1的类型是int
    var j int32 = 1<<s     // i的类型是int32，j == 0
    var k = uint64(1<<s)   // 1的类型是uint64，k == 1<<33
    var m int = 1.0<<s     // 1.0的类型是int，m == 1<<33
    var n = 1.0<<s == j    // 假设 1.0 == j，因此1.0的类型是int32，n == true
    var o = 1<<s == 2<<s   // 1和2的类型都是int，o == false
    var p = 1<<s == 1<<33  // 1的类型是int，p == true
    var u = 1.0<<s         // 非法，1.0是浮点不能移位
    var u1 = 1.0<<s != 0   // 非法，1.0是浮点类型float64
    var u2 = 1<<s != 1.0   // 非法，1是浮点类型float64
    var v1 float32 = 1<<s  // 非法，1是浮点类型float32
    var v2 = string(1<<s)  // 非法，1是字符串类型
    var w int64 = 1.0<<33  // 1.0<<33是一个常量表达式，w == 1<<33
    var x = a[1.0<<s]      // 报异常，1.0类型是int，但是1<<33超出了数组的范围
    var b = make([]byte, 1.0<<s) // 1.0的类型是int，len(b) == 1<<33
    var mm int = 1.0<<s    // 假设int是32位整数，mm == 0
    var oo = 1<<s == 2<<s  // 1和2是int类型，但是溢出了都为0，oo == true
    var pp = 1<<s == 1<<33 // 非法，1<<33不能用int型表示
    var xx = a[1.0<<s]     // 1.0类型是int，1.0<<s值是0，xx == a[0]
    var bb = make([]byte, 1.0<<s) // 1.0类型是int，len(bb) == 0

一元操作符有最高的优先级，因为++和--是语句不是表达式因此不在操作符的范围内。
因此语句 `*p++` 相当于 `(*p)++`。

    unary_op = "+" | "-" | "!" | "^" | "*" | "&" | "<-" .

二元操作符有5个优先等级，最高的是乘法系列操作，然后是加法系列操作，然后是
比较操作，然后是逻辑与，最后是逻辑或::

    优先级   操作符
    5         * / % << >> & &^
    4         + - | ^
    3         == != < <= > >=
    2         &&
    1         ||

相同优先等级的二元操作符，按照从左至右的顺序进行结合::

    x / y * z       // 相当于 (x / y) * z
    +x              // x
    42 + a - b      // (42 + a) - b
    23 + 3 * x[i]   // 23 + (3 * x[i])
    x <= f()        // x << f()
    ^a >> b         // (^a) >> b
    f() || g()      // f() || g()
    x == y+1 && <-chanInt > 0 // (x == (y+1)) && ((<-chanInt>) > 0)

算术操作
--------

算术操作运用在数值数据之上并产生一个同类型的结果值。四个标准的数术操作符
（+ - * /）用于整型、浮点、复数类型上，操作+也可以用在字符串上。位操作
只能用在整型值上::

    +                           整型、浮点、复数、字符串
    -                           整型、浮点、复数
    *                           整型、浮点、复数
    /                           整型、浮点、复数
    %                           整型
    &  AND                      整型（与）
    |  OR                       整型（或）
    ^  XOR                      整型（异或）
    &^ bit clear (NAND)         整型（与非）
    <<                          整型 integer << interger 右操作数>=0
    >>                          整型 integer >> interger 右操作数>=0

如果操作数的类型是一个类型参数，操作符必须可以应用到类型参数定义的所有类型上。
操作数代表的是类型参数实例化后具体的类型，会根据这个具体类型的精度进行求值::

    func dotProduct[F ~float32|~float64](v1,v2 []F) F {
        var s F
        for i,x := range v1 {
            y := v2[i]
            s += x * y // 会根据F是float32还是float64，按照不同的精度求值
        }
        return s
    }

整数操作
________

对于两个整数x和y，整数的除法q=x/y与取余r=x%y的关系是::

     x = q * y + r 并且 |r| < |y|
     x   y   x/y   x%y   // x/y的结果会向0进行截断（truncated division）
     5   3    1     2
    -5   3   -1    -2
     5  -3   -1     2
    -5  -3    1    -2

一个列外是，如果x是对应整型的最小负数，由于二进制补码整数上溢q=x/-1的值是x，余数是0::

                           x, q=x/-1
    int8                   -128  // -128/-1==128但是最大正数是127溢出了表示的数还是-128
    int16                -32768
    int32           -2147483648
    int64  -9223372036854775808

如果除数是一个常量，它不能是零，如果运行时除数是零会报运行时异常。如果除数不是零并且是
2的幂的常量，那么除法可以用右移操作代替，余数可以用位与操作代替::

     x      x/4      x%4      x>>2                                      x&3
     11      2        3     0b1011>>2==0b10==2                          0b11==3
    -11     -2       -3     0b11110101>>2==0b111101==0b10000011==-3     0b01==1

位移操作的右操作数必须大于等于0，如果右操作数在运行时位负数会倒是运行时异常。如果左操作数
是有符号整数那么位移操作实现的是算术位移，如果左操作数是无符号数那么位移操作实现的是逻辑
位移。

整数的一元操作符有正号负号和取反（+ - ^），取反在C语言中的操作符是~，但是这个操作符在Go
语言中用在了类型约束中::

    +x  // 相当于 0+x
    -x  // 相当于 0-x
    ^x  // 相当于 m^x 其中m对于无符号整型x所有位都是1（例如0xFF），对于有符号整型x其值为-1（其实也是0xFF）

对于无符号整数，操作符+ - * <<计算结果会模上2^n，其中n表示无符号数的位宽。
因此无符号整数的计算会截取掉高位溢出的部分。对于有符号整数，操作符+ - * / <<
的计算可能会上溢出，但是不会产生运行时异常，其结果是有符号数的表示形式对应的
值。编译器不能依赖上溢不会发生来优化代码，例如不能假定x<x+1总会成立。

浮点操作
_________

对于浮点和复数，+x与x相同，-x是x的负值。浮点或复数除以0的结果并没有在IEEE-754
标准中定义，是否抛出运行时异常由特定实现决定。

特定实现可以合并多个浮点操作到单个融合操作（single fused operation），可能
跨越代码语句，其结果可能与单独执行并近似每个操作的值不一样。显式的浮点类型转换
会将值近似到目标类型的精度，可以用来阻止融合操作。例如一些平台提供了FMA（fused 
multiply and add）指令用来直接计算x*y+z的结果，避免x*y中间计算过程中的精度
损失。下面这些例子展示Go的实现如果使用了这个指令的情况::

    r = x*y + z // 以下这些计算都可以使用FMA计算r
    r = z; r += x*y
    t = x*y; r = t + z
    *p = x*6; r = *p + z
    r = x*6 + float64(z) // 这些计算不能使用FMA，因为有显式的精度转换
    r = float64(x*6) + z
    r = z; r += float64(x*y)
    t = float64(x*6); r = t + z

字符串链接
__________

字符串可以使用+或者+=操作符进行字符串连接，字符串连接会创建一个新的字符串::

    s := "hi" + string(c)
    s += " and good bye"

比较操作
--------

比较操作比较两个操作数，产生一个无类型的布尔值。任何比较操作中，第一个
操作数必须可以赋值给第二个操作数的类型，反之也一样。等于操作符== !=用于两个
可比较类型（comparable types）的操作数上，大小比较操作符< <= > >=用于两个
有序类型（ordered types）的操作数上。 ::

    == eq != ne < lt <= le > gt >= ge

比较操作的定义如下:

1. 布尔类型是可比较的
2. 整数是可比较的和有大小的
3. 浮点是可比较的和有大小的，IEEE-754标准定义了两个浮点如何比较
4. 复数是可以比较的，两个复数是相等的如果real(u)==real(v)并且imag(u)==imag(v)
5. 字符串是可比较的和有大小的，字符串按字节序列进行比较
6. 指针是可比较的，两个指针式相等如果都不是nil并且指向同一个变量，指向两个不同的零大小的变量可能也可能不相等
7. 通道是可比较的，两个通道值相等如果它们调用相同的make创建，或者都是nil
8. 非类型参数的接口是可比较的，只要它们的动态类型一样并且动态值相等，或者都是nil
9. 非接口类型X的值x与接口T的值t是可比较的，只要类型X是可比较的并且X实现了T，
   它等价于比较t的动态类型是否与X类型相同，并且t的动态值是否与x相等；
10. 结构体是可比较的只要它所有的成员是可比较的，两个结构体是相等的只要它们
    对应的非空成员值是相等的，成员按照代码定义顺序进行比较，只要出现不相等的
    成员就会停下；
11. 数组是可比较的只要数组的元素可比较，元素按照索引顺序依次进行比较；
12. 类型参数是可比较的只要它们是严格可比较

比较的结果是无类型的布尔值::

    const c = 3 < 4 // c是一个无类型的布尔常量
    type MyBool bool
    var x,y int
    var ( // x==y是无类型的，但当赋值给变量时，如果变量没有指定类型，会自动隐式转换成默认类型
        b3 = x == y // b3的类型是bool
        b4 bool = x == y // b4的类型是bool
        b5 MyBool = x == y // b5的类型是MyBool
    )

比较两个接口类型的值，如果它们有相同的动态类型但是这个动态类型不可比较，会导致
运行时异常。这也包括数组中的接口类型值以及结构体内的接口类型值。

切片、映射、和函数类型是不可比较的。然而，作为特殊情况，一个切片、映射、函数的
值都可以与nil进行比较。另外，指针、通道、接口值都允许与nil进行比较。

一个类型是严格可比较的，如果它是可比较的，并且不是一个接口类型也不包含接口类型，
具体的:

1. 布尔、数值、字符串、指针、通道类型是严格可比较的
2. 结构体类型是严格可比较的，如果它们的成员都是严格可比较的
3. 数组类型是严格可比较的，如果它的元素类型是严格可比较的
4. 类型参数是严格可比较的，如果它定义的所有类型都是严格可比较的

逻辑操作
--------

逻辑操作（&&、||、!）用于布尔值并产生相同类型的结果，逻辑操作首先会对第一个
操作数进行求值，然后按需要对第二个操作数进行求值::

    &&  AND     p && q 相当于 如果p则q否则false
    ||  OR      p || q 相当于 如果p则true否则q
    !   NOT     !p     相当于 如果p则false否则true

地址操作
--------

对于类型T的操作数x，地址操作&x产生一个类型T的指针类型*T。操作数必须是
可取地址的，即是一个变量、指针间接表示的变量、切片元素索引、可取地址结
构体的成员，可取地址数组的元素索引，x还可以是一个复合字面量。如果x的
求值会导致运行时异常，那么求值&x也会。

对于指针类型 `*T` 的值x，指针解引用或指针间接表示的变量（pointer 
indirection） `*x` 表示x指向的那个类型T的变量。如果x的值是nil会导致
运行时错误。

地址操作示例::

    &x // 变量取地址
    pa := &a // 变量取地址
    &a[f(2)] // 数组元素取地址
    &Point{2, 3} // 复合字面量求地址
    *p // 表示p指向的变量
    *pf(x)
    var x *int = nil
    *x // 导致运行时异常
    &*x // 导致运行时异常

通道接收操作
------------

对于核心类型（core type）是通道的操作数ch，接收操作 ``<-ch`` 的值是从通道ch
接收的值。通道的方向必须允许接收，接收操作的结果类型是通道的元素类型。接收操作
表达式会阻塞直到这个值可用，从nil通道进行接收会永远阻塞，对关掉的通道进行接收
操作总会立即处理，其结果是通道元素类型的零值。

接收操作可以用在特殊的赋值或初始化语句中，会额外产生一个无类型的布尔结果
表示通道通信是否成功，如果返回false表示生成了通道类型的零值因为通道关闭了或
者为空，如果为空表示成功接收到了一个值。 ::

    v1 := <-ch
    v2 = <-ch
    f(<-ch)
    <-strobe // 一直等待直到时钟脉冲到来并将接收到的值丢弃
    x, ok = <-ch
    x, ok := <-ch
    var x, ok = <-ch
    var x, ok T = <-ch

类型转换
--------

转换表达式的类型到指定的类型，转换显式的进行，也可能是隐式的在表达式上下文中发生。
显式的转换是 ``T(x)`` 的形式将表达式x的类型转换成类型T::

    Conversion = Type "(" Expression [ "," ] ")" .

如果类型以操作符*或者<-开始，或者类型是以关键字func开始并且没有返回值列表，必须
使用括号避免混淆::

    *Point(p)           // 相当于 *(Point(p))
    (*Point)(p)         // p被转换成*Point
    <-chan int(c)       // 相当于 <-(chan int(c))
    (<-chan int)(c)     // c被转换成(<- chan int)
    func()(x)           // 函数签名 func() x
    (func())(x)         // x转换成func()
    (func() int)(x)     // x转换成func() int
    func() int(x)       // x转换成func() int，不混淆

一个常量x可以转换成类型T，如果x可以用类型T的值表示。特殊的例子，一个整型
常量和非常量x都可以显式的转换成为一个字符串类型，这个整型相当于是一个字符
的值。将一个常量转换成一个不是类型参数的类型将产生一个有类型的常量，将
常量转换成类型参数将产生一个实例化的类型实参类型的非常量值。 ::

    uint(iota)                  // 转换成uint
    float32(2.718281828)        // 转换成float32
    complex128(1)               // 转换成complex128类型的1.0+0.0i
    float32(0.49999999)         // 转换成float32类型的0.5
    float64(-1e-1000)           // 转换成float64类型的0.0，
    string('x')                 // 转换成string类型的"x"
    string(0x266c)              // 转换成string类型的"♬"
    myString("foo" + "bar")     // 转换成myString类型的"foobar"
    string([]byte{'a'})         // 转换成string类型的"a"
    (*int)(nil)                 // 转换成int型指针
    int(1.2)                    // 非法，浮点1.2不能用整数表示
    f := 3.14; a := int(f)      // 虽然常量转换是非法的，但是变量的转换是合法的
    string(65.0)                // 非法，浮点不是一个整数
    func f[P ~float32|~float64]() {
        P(1.1) // P(1.1) 转换成非常量值，其类型是float32或者float64
    }

非常量值x可以转换成类型T，如果满足下面任意一个条件:

1. x可以赋值给类型T
2. 忽略结构体的标记（struct tag），x的类型和T都不是类型参数，且有相同的底层类型
3. 忽略结构体的标记，x的类型和T都是指针类型并且不是命名类型（named types），
   并且指针的基类型不是类型参数且有相同的底层类型
4. x的类型和T都是整数或者浮点类型，即非常量值的整数和浮点可以相互强制转换
5. x的类型和T都是复数类型
6. x是一个整型或者字节类型的或rune类型的切片，而T是一个字符串类型
7. x是字符串类型，而T是一个字节类型的或rune类型的切片
8. x是一个切片，T是一个数组 [Go1.20]_ 或者数组的指针 [Go1.17]_，并且切片和数组有相同的元素类型

另外如果x的类型V或者T是类型参数，x也可以转换成T只要下面其中一个条件满足：

1. V和T都是类型参数，V中的每个类型的值都可以被转换成T中每个类型
2. 只有V是类型参数，V中的每个类型的值都可以被转换成类型T
3. 只有T是类型参数，x可以被转换成T定义的每个类型

转换操作中，结构体的标记（struct tag）在比较结构体类型是否相等是被忽略::

    type Person struct {
        Name string
        Address *struct {
            Street string
            City string
        }
    }

    var data *struct {
        Name string `json:"name"`
        Address *struct {
            Street string `json:"street"`
            City string `json:"city"`
        } `json:"address"`
    }

    var person = *(Person)(data) // 忽略标记，底层类型是相等的

特别的规则应用在非常量值的数值类型和字符串类型的转换上，这些转换改变了x的
表示形式引起了运行时的开销。所有其他转换都只改变了类型而没有修改x的表示
形式。没有语言上的机制对指针和整型值进行转换，但是unsafe包在限制的场景
下实现了这种功能。

数值类型间的转换
________________

对于数值类型非常量值的转换，遵循以下规则：

1. 整数类型之间进行转换时，如果值是有符号的，符号位会被隐式的扩展到无限精度，
   然后被截取适合结果类型的大小。整型之间的转换总是产生一个合法的值，不会提示
   有溢出。例如v:=uint16(0x10F0); uint32(int8(v)) == 0xFFFFFFF0；
2. 如果将一个浮点值转换成整数，小数部分会被丢弃（向0截断）
3. 当将整型或浮点转换成浮点类型时，或者复数转换成另一个复数类型时，结果值会近似
   到目标类型的精度。例如float32类型变量x的一个值（这个值可以是一个表达式）可能
   表示一个比IEEE-754 32位精度更高的值，但是float32(x)会将x的值近似到32位
   精度。类型的，x+0.1可能使用了高于32位的精度，但是float32(x+0.1)则一定是
   32位精度。

所有涉及非常量浮点或复数的类型转换，如果值不能用对应的目标类型表示，类型转换是
成功的，但转换后的值由实现决定。

字符串的转换
____________

可以将字节类型切片、rune类型的切片转换成字符串，反之也可以。另外由于历史原因，
一个整数值可以转换成字符串类型，这个整数值表示一个Unicode编码点然后转换成一个
UTF-8字符，超出Unicode合法范围的值会被转换成 ``\uFFFD``。注意这种转换可能最终
会从语言中移除，go vet工具将一些整数到字符串的转换视为潜在的错误，应该使用库
中的utf8.AppendRune或者utf8.EncodeRune函数。 ::

    string([]byte{'h', 'e', 'l', 'l', '\xc3', '\xb8'})   // "hellø"
    string([]byte{})                                     // ""
    string([]byte(nil))                                  // ""
    type bytes []byte
    string(bytes{'h', 'e', 'l', 'l', '\xc3', '\xb8'})    // "hellø"
    type myByte byte
    string([]myByte{'w', 'o', 'r', 'l', 'd', '!'})       // "world!"
    myString([]myByte{'\xf0', '\x9f', '\x8c', '\x8d'})   // "🌍"
    string([]rune{0x767d, 0x9d6c, 0x7fd4})   // "\u767d\u9d6c\u7fd4" == "白鵬翔"
    string([]rune{})                         // ""
    string([]rune(nil))                      // ""
    type runes []rune
    string(runes{0x767d, 0x9d6c, 0x7fd4})    // "\u767d\u9d6c\u7fd4" == "白鵬翔"
    type myRune rune
    string([]myRune{0x266b, 0x266c})         // "\u266b\u266c" == "♫♬"
    myString([]myRune{0x1f30e})              // "\U0001f30e" == "🌎"

    []byte("hellø")             // []byte{'h', 'e', 'l', 'l', '\xc3', '\xb8'}
    []byte("")                  // []byte{}
    bytes("hellø")              // []byte{'h', 'e', 'l', 'l', '\xc3', '\xb8'}
    []myByte("world!")          // []myByte{'w', 'o', 'r', 'l', 'd', '!'}
    []myByte(myString("🌏"))    // []myByte{'\xf0', '\x9f', '\x8c', '\x8f'}
    []rune(myString("白鵬翔"))   // []rune{0x767d, 0x9d6c, 0x7fd4}
    []rune("")                  // []rune{}
    runes("白鵬翔")              // []rune{0x767d, 0x9d6c, 0x7fd4}
    []myRune("♫♬")              // []myRune{0x266b, 0x266c}
    []myRune(myString("🌐"))    // []myRune{0x1f310}

    string('a')          // "a"
    string(65)           // "A"
    string('\xf8')       // "\u00f8" == "ø" == "\xc3\xb8"
    string(-1)           // "\ufffd" == "\xef\xbf\xbd"
    type myString string
    myString('\u65e5')   // "\u65e5" == "日" == "\xe6\x97\xa5"

数组的转换
___________

将一个切片转换成数组，产生一个数组包含切片底层数组中的元素。将切片转换成数组指针，
会产生一个指针指向切片的底层数组。如果切片的长度小于数组的长度，会导致运行时错误。
注意转换成数组会创建一个新的数组，而转换成数组指针该指针和切片共享底层数组。 ::

    s := make([]byte, 2, 4)
    a0 := [0]byte(s)
    a1 := [1]byte(s[1:])     // a1[0] == s[1]
    a2 := [2]byte(s)         // a2[0] == s[0]
    a4 := [4]byte(s)         // panics: len([4]byte) > len(s)

    s0 := (*[0]byte)(s)      // s0 != nil
    s1 := (*[1]byte)(s[1:])  // &s1[0] == &s[1]
    s2 := (*[2]byte)(s)      // &s2[0] == &s[0]
    s4 := (*[4]byte)(s)      // panics: len([4]byte) > len(s)

    var t []string
    t0 := [0]string(t)       // ok for nil slice t
    t1 := (*[0]string)(t)    // t1 == nil
    t2 := (*[1]string)(t)    // panics: len([1]string) > len(t)

    u := make([]byte, 0)
    u0 := (*[0]byte)(u)      // u0 != nil

常量表达式
-----------

常量表达式只能包含常量操作数并且可以在编译时求值，无类型的布尔、数值、字符串常
量分别可以合法的作为需要布尔、数值、字符串类型的操作数使用。

一个常量比较操作总是产生一个无类型的布尔常量。如果常量位移的左操作数是一个无类型
的常量，其结果是一个整型常量，否则其结果的类型是左操作数的类型且必须是一个整型。

无类型常量的其他操作的结果都是一个相同类型的无类型常量，即一个布尔、整型、浮点、
复数、或字符串常量。如果除位移操作之外的二元操作的两个操作数是两个不同的无类型，
那么表达式的结果类型按照谁表达能力大就是哪个类型，也即整型、字符类型、浮点、复
数的顺序。例如，一个无类型的整型除以一个无类型的复数的结果是一个无类型的复数常
量。可以调用内置函数complex将无类型的整型、字符类型、浮点常量转换成无类型的复
数常量。 ::

    const a = 2 + 3.0          // a == 5.0   (untyped floating-point constant)
    const b = 15 / 4           // b == 3     (untyped integer constant)
    const c = 15 / 4.0         // c == 3.75  (untyped floating-point constant)
    const Θ float64 = 3/2      // Θ == 1.0   (type float64, 3/2 is integer division)
    const Π float64 = 3/2.     // Π == 1.5   (type float64, 3/2. is float division)
    const d = 1 << 3.0         // d == 8     (untyped integer constant)
    const e = 1.0 << 3         // e == 8     (untyped integer constant)
    const f = int32(1) << 33   // illegal    (constant 8589934592 overflows int32)
    const g = float64(2) >> 1  // illegal    (float64(2) is a typed floating-point constant)
    const h = "foo" > "bar"    // h == true  (untyped boolean constant)
    const j = true             // j == true  (untyped boolean constant)
    const k = 'w' + 1          // k == 'x'   (untyped rune constant)
    const l = "hi"             // l == "hi"  (untyped string constant)
    const m = string(k)        // m == "x"   (type string)
    const Σ = 1 - 0.707i       //            (untyped complex constant)
    const Δ = Σ + 2.0e-4       //            (untyped complex constant)
    const Φ = iota*1i - 1/1i   //            (untyped complex constant)
    const ic = complex(0, c)   // ic == 3.75i  (untyped complex constant)
    const iΘ = complex(0, Θ)   // iΘ == 1i     (type complex128)

当指定类型时，如果常量不能用目标类型表示会报错，也即指定类型的常量必须能用目标类型精确表示。
常量表达式总是精确的求值，其中间结果和常量自身可能需要用比语言预定义类型大得多的精度表示。
另外除法和取余的第二个操作数不能为零。 ::

    const Huge = 1 << 100  // Huge的值为1267650600228229401496703205376 无类型整数常量
    const Four int8 = Huge >> 98  // Four一个类型为int8的值为4的常量
    uint(-1)     // 非法，-1不能用uint表示
    int(3.14)    // 非法，3.14不能用int表示
    int64(Huge)  // 非法，1267650600228229401496703205376不能用int64表示
    Four * 300   // 非法，300不能用int8表示
    Four * 100   // 非法，400不能用int8表示
    3.14 / 0.0   // 非法，不能除零

对于取反操作，要注意有符号值的取反，取反操作符^在C语言中对应的操作符是~。 ::

    ^1           // 1是无类型的有符号整数，^1是值为-2的无类型的有符号整数
                 // [000]...01 -> [111]...10 -> [111]...01 -> [100]...10 -> -2
    uint8(^1)    // 非法，-2不能用uint8表示
    ^uint8(1)    // 类型为uint8的常量，值为0xFE，00000001->11111110->0xFE
    int8(^1)     // 类型为int8的常量，值为-2
    ^int8(1)     // 类型为int8的常量，值为-2

对于无类型的浮点和复数常量表达式，编译器可能进行精度近似，见前面章节常量部分的
实现限制。这种精度近似可能导致一个浮点常量表达式在需要整型的上下文中是非法的，尽管
如果按照无限精度求值它就是一个整数。

求值顺序
---------

在包的级别，包的初始化依赖决定单个变量声明中初始化表达式的求值顺序。否则按通用求值规则
求值，即当求值一个表达式的操作数、赋值或返回语句时，所有函数调用、成员函数调用、通道接收
操作、二元逻辑运算都按照词法从左到右的顺序求值。

例如，下面第一个函数内部的赋值语句，函数调用和通道接收的求值顺序是：f()，如果z是false
求值h()，i()，j()，<-c，g()，k()。但是这些操作与x和y的求值的顺序相比，并没有具体指定，
例如y可能在i()之前求值也可能在之后求值。除非逻辑上限制某个操作必须先求值，比如在g求值
前它的所有参数都必须先求值。 ::

    y[f()], ok = g(z || h(), i()+x[j()], <-c), k()
    a := 1
    f := func() int { a++; return a }
    x := []int{a, f()}            // x可能是[1,2]或[2,2]，因为求值a和f()的顺序不定
    m := map[int]int{a: 1, a: 2}  // m可能是{2:1}或{2:2}，因为映射两个元素的赋值的求值顺序不定
    n := map[int]int{a: f()}      // n可能是{2:3}或{3:3}，因为键值和元素值的求值顺序不定

在包的级别，初始化依赖对每个初始化表达式的求值与从左至右的求值规则不同，但是对单个表达式内部的
操作数来说求值还是按照从左至右的规则。 ::

    var a, b, c = f() + v(), g(), sqr(u()) + v() // 函数u和v是独立的，与这里声明的变量和函数无关联
    func f() int { return c } // 函数f依赖变量c，必须先求值c再求值f()，但是f()的值需要赋给a，因此c必须先于a
    func g() int { return a } // 函数g依赖变量a，必须先求值a再求值g()，但是g()的值需要赋给b，因此a必须限于b
    func sqr(x int) int { return x*x } // 函数sqr不依赖其他变量，依据上面的依赖，必须先求职c然后是a和b
    // 因此上面函数的求值顺序是：u() sqr() v() f() v() g()

单个表达式内的浮点操作按照操作符的结合律进行求值。显式的括号会影响默认的结合律求值顺序，例如
表达式x+(y+z)中y+z会先求值。

语句
=====

* `终止语句`_
* `空语句`_
* `标签语句`_
* `表达式语句`_
* `通道发送语句`_
* `自加自减语句`_
* `赋值语句`_
* `If语句`_
* `Switch语句`_
* `For语句`_
* `Go语句`_
* `Select语句`_
* `Return语句`_
* `Break语句`_
* `Continue语句`_
* `Goto语句`_
* `Fallthrough语句`_
* `Defer语句`_

语句控制着程序的执行::

    Statement = Declaration | LabeledStmt | SimpleStmt |
        GoStmt | ReturnStmt | BreakStmt | ContinueStmt | GotoStmt |
        FallthroughStmt | Block | IfStmt | SwitchStmt | SelectStmt | ForStmt |
        DeferStmt .
    SimpleStmt = EmptyStmt | ExpressionStmt | SendStmt | IncDecStmt | Assignment | ShortVarDecl .

终止语句
--------

终止语句用来终止代码块的常规控制流程，下面这些语句是终止语句：

1. return或goto语句
2. 调用内置函数panic
3. 代码块的最后一条非空语句是终止语句
4. if语句中如果else存在，两个分支都是终止语句
5. for语句是终止语句如果没有break没有指定循环条件也没有使用range
6. switch如果没有break，每个case和default都是终止语句，或者由fallthrough语句终止
7. select如果没有break，每个case和default都是终止语句
8. 标签语句可用来作为终止语句的标签

所有其他语句都不是终止的。一个语句列表以一条终止语句终止，如果这个列表不为空
并且最后一条非空语句是终止语句。

空语句
-------

没有任何内容的是空语句::

    EmptyStmt = .

标签语句
--------

一个标签语句用来作为goto、break、continue语句的跳转目标::

    LabeledStmt = Label ":" Statment .
    Lable = identifier .

例如::

    Error: log.Panic("error encountered")

表达式语句
----------

除一些特别的内置函数外，函数调用、成员函数调用、通道接收操作这些表达式可以作为
语句出现，这些语句可能使用括号括起。 ::

    ExpressionStmt = Expression .

例如::

    h(x+y)
    f.Close()
    <-ch
    (<-ch)
    len("foo") // 非法，如果len是内置函数

下面这些内置函数不允许用作语句::

    append cap complex imag len make new real
    unsafe.Add unsafe.Alignof unsafe.Offsetof unsafe.Sizeof
    unsafe.Slice unsafe.SliceData unsafe.String unsafe.StringData

通道发送语句
------------

通道发送操作发送一个值到一个通道，通道表达式的核心类型（core type）必须
是一个通道，并且通道的方向必须允许发送，发送的数据必须可以赋值给通道的
元素类型。 ::

    SendStmt = Channel "<-" Expression .
    Channel = Expression .

通道表达式和值表达式都会在通道通信前进行求值。通信会一直阻塞直到可以执行
发送操作。在非缓冲通道上发送需要等接收端准备好，在缓冲通道上发送需要等有
空余的缓存存在，在关闭的通道上发送将导致一个运行时异常，在nil通道上发送
会导致永远阻塞。 ::

    ch <- 3 // 发送数据3到通道ch

自加自减语句
------------

自增自减语句++、--相当于一个+=、-=赋值语句，只是它的右操作数是一个无类型的常量1。
如赋值语句一样，操作数必须是可取地址的或者是一个映射的索引表达式。 ::

    IncDecStmt = Expression ( "++" | "--" ) .

下面的赋值语句在语义上是等价的::

    x++   x+=1
    x--   x-=1

赋值语句
---------

赋值语句将变量存储的值替换成一个新的用表达式表示的值，一个赋值语句可以给一个或多个
变量赋值::

    Assignment = ExpressionList assign_op ExpressionList .
    assign_op = [ add_op | mul_op ] "=" .

左操作数必须是可以取地址的，或者是映射的索引表达式，或者是空标识符（仅=赋值）。
操作数可能包含在括号内。算术赋值操作 x op= y 相当于 x = x op y但是只会对x求值
一次，算术赋值操作只能是单个的左操作数和右操作数，并且左操作数不能是空标识符。

元组赋值操作可以将多个值赋值给对应的变量列表中的变量，有两种形式。第一是右操作数是
单个多返回值的函数调用、或者通道或映射操作、或者类型断言，左操作数的个数必须与右边
值的个数相等。第二种形式，左操作数的个数必须与右边表达式的个数相等，右边每个表达式
必须只会产生单个值，并且第n个表达式可以给左边的第n个变量对应的类型赋值。空标识符可以
用来忽略右边的值。 ::

    x = 1
    *p = f()
    a[i] = 23
    (k) = <-ch  // same as: k = <-ch
    a[i] <<= 2
    i &^= 1<<n
    x, y = f()
    one, two, three = '一', '二', '三'
    _ = x       // 求x的值当值忽略这个值
    x, _ = f()  // 求f()的值，但是忽略第二个返回值

赋值语句的处理分为两个阶段。首先，左边的索引表达式、指针解引用（包括成员选择时的隐式
指针解引用）的运算对象，以及右边的表达式，都按照通用求值规则先求值。第二步，赋值操作
会按从左至右的顺序执行。 ::

    a, b = b, a  // 交换a和b的值
    x := []int{1, 2, 3}
    i := 0
    i, x[i] = 1, 2  // 设置i为1，x[0]为2
    i = 0
    x[i], i = 2, 1  // 设置x[0]为2，i为1
    x[0], x[0] = 1, 2  // 设置x[0]为1然后再设为2
    x[1], x[3] = 4, 5  // 设置x[1]为4，然后设置x[3]因为越界导致运行时异常
    type Point struct { x, y int }
    var p *Point
    x[2], p.x = 6, 7  // 设置x[2]为6，然后设置p.x因为p是nil而导致异常
    i = 2
    x = []int{3, 5, 7}
    for i, x[i] = range x {  // 首先计算x[i]，因此i,x[2] =0,3
        break // 因此退出后，i为0，x为{3,5,3}
    }

赋值语句中，右边的每个值都必须可以赋值给左边的类型，但是有下面几种特殊情况：

1. 任何有类型的值可以赋值给空标识符
2. 如果无类型的常量赋值给接口变量或空标识符，常量首先会隐式的转换为它的默认类型
3. 如果无类型的布尔值赋值给接口变量或空标识符，该值首先会隐式的转换为bool类型

If语句
-------

If语句根据一个布尔表达式，来条件执行两个分支，如果表达式为true执行if分支否则else分支。
表达式之前可以出现一个简单语句，这条语句在表达式执行之前求值。 ::

    IfStmt = "if" [ SimpleStmt ";" ] Expression Block [ "else" ( IfStmt | Block ) ] .

例如::

    if x > max {
        x = max
    }
    if x := f(); x < y {
        return x
    } else if x > z {
        return z
    } else {
        return y
    }

Switch语句
-----------

Switch语句有两种，表达式switch和类型switch::

    SwitchStmt = ExprSwitchStmt | TypeSwitchStmt .
    ExprSwitchStmt = "switch" [ SimpleStmt ";" ] [ Expression ] "{" { ExprCaseClause } "}" .
    ExprCaseClause = ExprSwitchCase ":" StatementList .
    ExprSwitchCase = "case" ExpressionList | "default" .
    TypeSwitchStmt = "switch" [ SimpleStmt ";" ] TypeSwitchGuard "{" { TypeCaseClause } "}" .
    TypeSwitchGuard = [ identifier ":=" ] PrimaryExpr "." "(" "type" ")" .
    TypeCaseClause = TypeSwitchCase ":" StatementList .
    TypeSwitchCase = "case" TypeList | "default" .

在表达式switch中，switch表达式和case表达式的按照从左至右从上到下的顺序求值。
如果没有switch表达式，等价于一个布尔类型的值true。如果switch表达式是一个无
类型的常量，它首先转换成它的默认类型。预声明的无类型值nil不能作为switch
表达式使用。switch表达式的类型必须是可比较的。

如果case表达式是无类型的，它首先会隐式的转换成switch表达式的类型。对每个case
表达式x和switch表达式t，x==t必须是合法的比较操作。

在case或default语句块内，最后一条非空语句可以是一个（可能打了标签的）fallthrough
语句，表示代码流程从后一个语句块的第一条语句继续执行，否则switch代码流程在当前语句
块结束。

switch表达式前面可以有一个简单语句，这个语句在switch表达式之前执行。如果这个简单语句
是短变量声明，这个声明的变量的作用域只限于switch语句内部。 ::

    switch tag {
    default: s3()
    case 0, 1, 2, 3: s1()
    case 4, 5, 6, 7: s2()
    }
    switch x := f(); {  // 没有switch表达式相当于true
    case x < 0: return -x
    default: return x
    }
    switch {
    case x < y: f1()
    case x < z: f2()
    case x == 4: f3()
    }

编译可能不允许多个case表达式的值是同一个常量，例如当前的编译器不允许
重复的整型、浮点、字符串常量的case表达式出现。

类型Switch
___________

类型switch比较类型而不是值，它使用类型断言语法形式作为switch表达式，但是
是使用type关键字而不是一个实际的类型::

    switch x.(type) {
    // cases
    }

像类型断言一样，x必须是一个接口类型，而不能是类型参数，而且case中的每个非接口
类型T都必须实现了接口。类型switch将表达式x的动态类型与case中的实际类型T进行
匹配，case中列出的类型必须都是不同的。

类型switch语句可以包含一个短变量声明，这个变量的作用域在case或default语句块内。
如果case包含唯一一个类型，那么该变量的类型就是这个类型，否则为x的类型。case除了
类型，也可以使用nil，表示接口变量x是否为nil。下面是x类型为interface{}的类型
switch的一个例子::

    switch i := x.(type) {
    case nil:
        printString("x is nil")                // i的类型是x的类型interface{}
    case int:
        printInt(i)                            // i的类型是int
    case float64:
        printFloat64(i)                        // i的类型是float64
    case func(int) float64:
        printFunction(i)                       // i的类型是函数func(int) float64
    case bool, string:
        printString("type is bool or string")  // i的类型是x的类型interface{}
    default:
        printString("don't know the type")     // i的类型是x的类型interface{}
    }
    // 上面的代码等价于下面的代码
    v := x  // x is evaluated exactly once
    if v == nil {
        i := v                                 // i的类型是x的类型interface{}
        printString("x is nil")
    } else if i, isInt := v.(int); isInt {
        printInt(i)                            // i的类型是int
    } else if i, isFloat64 := v.(float64); isFloat64 {
        printFloat64(i)                        // i的类型是float64
    } else if i, isFunc := v.(func(int) float64); isFunc {
        printFunction(i)                       // i的类型是函数func(int) float64
    } else {
        _, isBool := v.(bool)
        _, isString := v.(string)
        if isBool || isString {
            i := v                         // i的类型是x的类型interface{}
            printString("type is bool or string")
        } else {
            i := v                         // i的类型是x的类型interface{}
            printString("don't know the type")
        }
    }

类型参数或者泛型类可以用在case表达式中，如果实例化后出现两个相同的case，
会选择第一个出现的case ::

    func f[P any](x any) int {
        switch x.(type) {
        case P:
            return 0
        case string:
            return 1
        case []P:
            return 2
        case []byte:
            return 3
        default:
            return 4
        }
    }
    var v1 = f[string]("foo")   // 匹配case P，v1值为0
    var v2 = f[byte]([]byte{})  // 匹配case []P，v2值为2

类型switch的switch表达式之前可以有一个简单语句，这条语句在switch表达式
之前求值。fallthrough语句不允许出现在类型switch中。

For语句
--------

For语句根据迭代方式有三种形式，第一种形式迭代由单个条件控制，第二种形式
迭代由for子语句（ForClause）控制，第三种形式通过range控制::

    ForStmt = "for" [ Condition | ForClause | RangeClause ] Block .
    Condition = Expression .
    ForClause = [ InitStmt ] ";" [ Condition ] ";" [ PostStmt ] .
    InitStmt = SimpleStmt .
    PostStmt = SimpleStmt .
    RangeClause = [ ExpressionList "=" | IdentifierList ":=" ] "range" Expression .

最简单的形式，For语句的代码块一直循环执行只要单个表达式表示的布尔条件为true，如果没有
指定条件，等价于条件恒为true。 ::

    for a < b {
        a *= 2
    }

For子语句中的三个元素都可以省略，如果连条件都省了，相当于条件为true。For语句的
每一次迭代都有自己隔离的变量声明（或多个变量声明）[Go1.22]_。用在第一个迭代中
的变量有init语句声明，后续每个迭代中的变量会隐式的在post语句执行之前声明，其值
初始化为前一次迭代之后这个变量的值。 ::

    for i := 0; i < 10; i++ {
        f(i)
    }
    for cond { // 相当于  for ; cond ; { S() }
        S()
    }
    for { // 相当于 for true { S() }
        S()
    }
    var prints []func()
    for i := 0; i < 5; i++ { // 每次执行i的值为0，2，4
        prints = append(prints, func() { println(i) })
        i++
    }
    for _, p := range prints { // 打印1 3 5，但是[Go1.22]之前打印6 6 6
        p()
    }

Range循环
__________

Range循环可以遍历数组、切片、字符串、映射中的元素、通道中接收的值、0到上界
的整型范围 [Go1.22]_。每个遍历的元素会被赋值给对应的变量，赋值右边的表达式
称为range表达式，它的核心类型必须是数组、数组指针、切片、字符串、映射、允许
接收操作的通道、或者整型。赋值左边的变量必须是可赋值的或者映射索引表达式，如
果range表达式的类型是通道或整型，变量只允许有一个，否则可以有两个，如果最后
一个变量是空变量，则相当于没有这个变量。

Range表达式x在循环开始之前求值一次，除非只存在一个变量并且len(x)的值是常量，
这时range表达式不会求值。赋值左边的函数调用会在每个迭代都执行一次。每次迭代，
会产生如下类型的值::

    a [n]E 或者*[n]E 或者[]E      int类型的索引值i，E类型的元素值a[i]
    s string                     int类型的索引值i，第i个字符类型的字符值
    m map[K]V                    K类型的键值k，V类型的元素值m[k]
    c chan E 或者<-chan E        E类型的元素值e
    n int                        int类型的元素值i

1. 对于数组、数组指针、切片值a，索引值从0开始到len(a)-1，如果只有一个
   变量只返回索引值不访问数组或切片的元素。如果切片值为nil，那么迭代
   次数为0；
2. 对于字符串，会迭代字符串从字节0处开始的Unicode字符，如果成功索引值
   返回下一个UTF-8字符的第一个字节的索引，元素值返回当前UTF-8字符对应
   的Unicode编码点。如果遍历的过程中遇到非法的UTF-8序列，元素值会返回
   0xFFFD，Unicode替代字符，而下一次迭代的索引值只会增加一个字节；
3. 映射元素的迭代顺序是不确定的，也不保证相同的映射下一次迭代会按相同的
   顺序。如果一个映射值在没有访问之前被移除，这个元素不会被迭代，如果
   一个新的元素添加到了映射中，这个元素可能会也可能不会被迭代。如果映射
   的值为nil，迭代次数为0；
4. 对于通道，会一直迭代接收通道中的值直到通道被关闭。如果通道的值为nil，
   迭代次数为0；
5. 对于整数值n，迭代的值为0到n-1，如果n<=0不会执行任何迭代；

如果range表达式是一个（可能无类型的）整型表达式n，n必须可以赋值给
左边的变量对于的类型，如果没有变量，n必须可以赋值给int类型。 ::

    var testdata *struct {
        a *[7]int
    }
    for i, _ := range testdata.a { // i的值从0到6，testdata.a不会被求值
        f(i)
    }
    var a [10]string
    for i, s := range a { // i的类型是int，s的类型是string值为a[i]
        g(i, s)
    }
    var key string
    var val interface{}  // 映射元素可以赋值给val
    m := map[string]int{"mon":0, "tue":1, "wed":2, "thu":3, "fri":4, "sat":5, "sun":6}
    for key, val = range m {
        h(key, val)
    } // 退出循环后，key的值为映射最后移除迭代的键值，val为最后一次迭代的元素值map[key]
    var ch chan Work = producer()
    for w := range ch {
        doWork(w)
    }
    for range ch {} // 清空通道接收的值
    for i := range 10 { // i从0到9，类型是int（无类型常量10的默认类型）
        f(i)
    }
    var u uint8
    for u = range 256 { // 非法，256不能赋值给uint8
    }

Go语句
-------

Go语句在独立控制的并行线程中启动一个函数调用，这种独立并行线程位于相同的
地址空间内，被称为协程goroutine。 ::

    GoStmt = "go" Expression .

表达式必须是一个函数或成员函数的调用，并且不能包含在括号内。内置函数的调用仅
限于表达式语句。函数值和函数参数按通常的方式求值，但是不同于普通的调用，
程序的执行不会等待函数执行完毕。函数会在一个新的goroutine中开始执行，当
函数执行完毕对应的goroutine也会终止，如果函数有返回值，返回值会被丢弃。 ::

    go Server()
    go func(ch chan<- bool) {
        for {
            sleep(10)
            ch <- true
        }
    } (c)

Select语句
-----------

Select语句用于选择执行哪个发送或接收操作，它跟switch语句类似单是它的case涉及
的都是通信操作。 ::

    SelectStmt = "select" "{" { CommClause } "}" .
    CommClause = CommCase ":" StatementList .
    CommCase = "case" ( SendStmt | RecvStmt ) "default" .
    RecvStmt = [ ExpressionList '=' ] IdentifierList ":=" ] RecvExpr .
    RecvExpr = Expression .

Select语句的执行分为几个步骤：

1. 对于语句中的所有case，接收操作的通道以及发送操作中的通道和右表达式会按代码
   顺序求值一次，但这一步中接收语句赋值操作符左边的表达式还不会被求值；
2. 如果有一个或多个case中的通信现在就可以不阻塞的执行，会使用一个统一的随机选
   择算法选择一个执行，否则如果有default选择default，如果没有select语句会一
   直阻塞直到至少有一个通信可以执行；
3. 除非是default，否则执行对应case的通信操作；
4. 如果选择的case是一个接收操作，赋值操作符左边的表达式会求值，然后被赋值为接收
   到的值；
5. 然后执行被选中case的语句列表；

因为nil通道上的通信永远都不会执行，因此只有nil通道并且没有default的select
语句会被永久阻塞。 ::

    var a []int
    var c, c1, c2, c3, c4 chan int
    var i1, i2 int
    select {
    case i1 = <-c1:
        printf("receive ", i1, " from c1\n")
    case c2 <- i2:
        printf("sent ", i2, " to c2\n")
    case i3,ok := (<-c3):
        if ok {
            print("received ", i3, " from c3\n")
        } else {
            print("c3 is closed\n")
        }
    case a[f()] = <=c4:
        // 相当于 case t := <-c4: a[f()] = t
    default:
        print("no communication\n")
    }
    for { // 发送随机的位序列给通道c
        select {
        case c<-0:
        case c<-1:
        }
    }
    select {} // 会永远阻塞

Return语句
-----------

函数F的return语句终止该函数的执行，并且可能提供一个或多个返回值。函数F
中任何被deferred的函数都会在F返回给调用者之前执行，这个返回之前指的是
函数的所有操作都做完了，包括return语句中对返回值的赋值都做完了。 ::

    ReturnStmt = "return" [ ExpressionList ] .

对于没有返回值的函数，return语句不能指定任何结果值。 ::

    func noResult() {
        return
    }

有三种方式在带返回值的函数中返回结果：

1. 每个返回值可以显式的在return语句中指定，每个表达式必须只产生单个值并且可以
   赋值给对应的返回值类型::

    func simpleF() int {
        return 2
    }
    func complexF1() (re float64, im float64) {
        return -7.0, -4.0
    }

2. return语句中的表达式列表可以是一个单独的有单个或多个返回值的函数调用，这个
   函数的返回值相当于作为当前函数的返回值返回::

    func complexF2() (re float64, im float64) {
        return complexF1()
    }

3. 如果返回值参数指定了名字，可以直接在函数中给这些返回值赋值，相当于是普通的
   函数局部变量一样，此时return语句不需要指定表达式列表::

    func complexF3() (re float64, im float64) {
        re = 7.0
        im = 4.0
        return
    }
    func (devnull) Write(p []byte) (n int, _ error) {
        n = len(p)
        return
    }

不管返回值是怎样声明的，它们在进入函数之前会被初始化为对应类型的零值。编译器可
能不允许return语句为空表达式列表，如果有比如常量、类型或变量的名字在return语句
所在的所用域内与返回值参数同名::

    func f(n int) (res int, err error) {
        if _, err := f(n-1); err != nil {
            return  // 非法，必须指定返回值列表，err被if内的变量覆盖了
        }
        return
    }

Break语句
----------

Break语句终止最内层for、switch、select语句的执行流程，并退出
到包围这些语句的最近的外层语句::

    BreakStmt = "break" [ Label ] .

如果指定了标签，那么break语句将终止与该标签相关联的最近一层的for、
switch、select语句，相当于标签指定了break要退出到哪个结构之外。
标签必须在同一个函数内，关联的结构必须包含了break语句。 ::

    OuterLoop:
    for i = 0; i < n; i++ {
        for j = 0; j < m; j++ {
            switch a[i][j] {
            case nil:
                state = Error
                break OuterLoop
            case item:
                state = Found
                break OuterLoop
            }
        }
    }

Continue语句
-------------

Continue语句终止最内层for语句的当前迭代，即跳过当前迭代后面的语句继续
下一次迭代。 ::

    ContinueStmt = "continue" [ Label ] .

如果指定了标签，这个标签关联的必须是一个for循环，相当于标签明确指定了
continue语句继续执行的是哪个嵌套的for循环::

    RowLoop:
    for y, row := range rows {
        for x, data := range row {
            if data == endOfRow {
                continue RowLoop
            }
            row[x] = data + bias(x, y)
        }
    }

Goto语句
----------

Goto语句将执行权跳转到标签对应的语句上，标签必须在同一个函数内。 ::

    GotoStmt = "goto" Label .

执行goto语句不能造成原本没有在作用域中的变量，在跳转后出现在作用域中，
也即跳转不能跨越变量声明，例如::

        goto L // 非法，因为跨越了变量v的声明
        v := 3
    L:

在一个代码块之外的goto语句，不能跳转到这个代码块内部::

    if n%2 == 1 {
        goto L1 // 非法，跨越了代码块，因为L1在for语句块内部而goto没有
    }
    for n > 0 {
        f()
        n--
    L1:
        f()
        n--
    }

Fallthrough语句
----------------

Fallthrough语句用在switch语句中，将程序控制权转移到下一个case子句的第一条
语句上，fallthrough只能在switch的case子句中使用，并且必须式该子句中的最后
一条非空语句。Fallthrough语句不能用在类型switch语句中，因为类型switch的设计
不支持这种控制流方式。 ::

    FallthroughStmt = "fallthrough" .

Defer语句
----------

Defer将一个函数调用的执行延迟到函数即将返回，可能是函数执行了return语句，
或者到了函数体的结束，或者对应的goroutine遇到了异常。 ::

    DeferStmt = "defer" Expression .

Defer后面的表达式必须是一个函数或成员函数调用，不能用括号括起。对内置函数的
调用是有限制的，一些内置函数不能为表达式语句使用。

每次defer语句执行时，函数值以及函数参数会被正常求值并且被保存，但是实际
的函数不会调用。这些函数会在当前函数返回时，以defer声明的相反顺序进行调
用。如果当前函数显式的使用return语句返回，defer函数的调用是所有返回参数
都赋值完毕并且在返回给调用者之前完成。如果一个defer函数的值求值后为nil，
执行异常会在函数真正执行时发生，而不是defer语句声明时发生。如果defer函数
有任何返回值，这些返回者都会被忽略。 ::

    lock(l)
    defer unlock(l)  // unlock的调用在当前函数返回时进行
    for i := 0; i <= 3; i++ {
        defer fmt.Print(i) // 在函数返回时打印3 2 1 0
    }
    func f() (result int) {
        defer func() {    // 函数f返回首先将result设为6
            result *= 7   // 然后defer函数执行将result编程6*7=42
        }()               // 最后函数f返回给调用者，返回值是42
        return 6
    }

内置函数
=========

* `append copy`_
* `clear`_
* `close`_
* `complex real imag`_
* `delete`_
* `len cap`_
* `make`_
* `new`_
* `min max`_
* `print println`_
* `panic recover`_

内置函数是预声明的，它们像普通函数一样调用，但是有一些内置函数可以接受一个
类型而不是一个表达式作为它的第一个参数。内置函数没有标准的Go类型，只能用在
函数调用表达式中，不能当前做一个函数值使用。

append copy
------------

内置函数append和copy辅助切片的操作，两个函数的执行结果都不依赖于参数引用的
内存是否有重叠。 ::

    append(s S, x ...E) S // 切片S的核心类型是[]E
    copy(dst, src []T) int
    copy(dst []byte, src string) int

变长参数函数append将零个或多个元素值x添加到切片s中，并且返回一个新的同样类型
的切片。切片s的核心类型必须是[]E，元素值x被传递到变长参数...E中。特殊的情况，
如果s的核心类型是[]byte，可以将核心类型为bytestring的参数加上...传递给...E，
相当于将字节切片或者字符串中的字节添加到目标切片中。如果目标切片的容量不能容纳
append添加的元素，append会分配一个新的足够大的底层数组，否则会重用当前的底层
数组。 ::

    s0 := []int{0, 0}
    s1 := append(s0, 2)                // 添加单个元素，s1是[]int{0, 0, 2}
    s2 := append(s1, 3, 5, 7)          // 添加多个元素，s2是[]int{0, 0, 2, 3, 5, 7}
    s3 := append(s2, s0...)            // 添加一个切片，s3是[]int{0, 0, 2, 3, 5, 7, 0, 0}
    s4 := append(s3[3:6], s3[2:]...)   // 切片重叠，s4是[]int{3, 5, 7, 2, 3, 5, 7, 0, 0}
    var t []interface{}
    t = append(t, 42, 3.1415, "foo")   // t是[]interface{}{42, 3.1415, "foo"}
    var b []byte
    b = append(b, "bar"...)            // 添加一个字符串，b是[]byte{'b', 'a', 'r' }

函数copy将一个切片中的所有元素拷贝到目标切片中，并且返回拷贝的元素个数。两个
切片的核心类型必须是一致的。拷贝的元素个数是len(src)和len(dst)两个值的最小值。
特殊的情况，如果目标切片的核心类型是[]byte，参数src可以接收bytestring类型的
参数，相当于拷贝字节切片或者字符串中的字节到目标切片中。 ::

    var a = [...]int{0, 1, 2, 3, 4, 5, 6, 7}
    var s = make([]int, 6)
    var b = make([]byte, 5)
    n1 := copy(s, a[0:])            // 返回值n1为6，s是[]int{0, 1, 2, 3, 4, 5}
    n2 := copy(s, s[2:])            // 返回值n2为4，s是[]int{2, 3, 4, 5, 4, 5}
    n3 := copy(b, "Hello, World!")  // 返回值n3为5，b是[]byte("Hello")

clear
------

内置函数clear接受一个映射、切片、或者类型参数为参数，将其中的元素都删除或者
都清零 [Go1.21]_。如果参数是一个类型参数，它定义的类型必须是映射或切片类型，
clear根据类型参数实例化后的具体类型进行操作。如果映射或切片的值是nil，clear
相当于是一个空操作（no-op）。 ::

    clear(m) // 如果m的类型是map[K]T，将删除所有的元素，最后len(m)是0
    clear(s) // 如果s的类型是[]T，将所有元素都清零
    clear(t) // 如果t的类型是一个类型参数，其约束的类型必须是映射或者切片

close
------

内置函数close的参数ch如果是一个核心类型为通道的参数，它将通道标记为不会继续
发送任何值到通道。如果通道是一个只能接收的通道，调用close是一个错误。对一个
关闭的通道进行数据发送或者重复关闭，会导致运行时异常。关闭一个值为nil的通道
也会导致运行时异常。

通道调用close后，并且所有发送的值都被接收之后，接收操作会返回一个通道元素类型
的零值。如果是多值接收操作，会返回接收到的值和一个表示通道是否关闭的布尔值。

complex real imag
------------------

内置函数complex接收浮点实数和虚数生成一个复数，而内置函数real和imag返回一个
复数的实数和虚数部分。 ::

    complex(realPart,imagPart floatT) complexT
    real(complexT) floatT
    imag(complexT) floatT

对于complex，两个参数的类型必须是相同的浮点类型。如果一个参数是一个无类型
的常量，它会隐式地转换成另一个参数的类型。如果两个参数都是无类型的常量，它们
的类型不能是复数常量或者虚数部分必须是零，函数的结果是一个无类型的复数常量。

对于real和imag，如果参数是一个无类型的常量，它必须是一个数值常量，此时函数
返回的是一个无类型的浮点常量。

如果这些函数的参数都是常量，返回值也是常量。这些函数的参数不能是类型参数。 ::

    var a = complex(2, -2)             // 变量a类型是无类型复数常量的默认类型complex128
    const b = complex(1.0, -1.4)       // 无类型复数常量1-1.4i
    x := float32(math.Cos(math.Pi/2))  // x的类型是float32
    var c64 = complex(5, -x)           // c64的类型是complex64，常量5会隐式转换成为x的类型float32
    var s int = complex(1, 0)          // 无类型复数常量1+0i可以转换成int型整数，因为虚数为零且实数没有小数部分
    _ = complex(1, 2<<s)               // 非法，2会被隐式的转换称为浮点类型，浮点类型不能进行位移操作
    var rl = real(c64)                 // rl的类型是float32
    var im = imag(a)                   // im的类型是float64
    const c = imag(b)                  // c是无类型浮点常量-1.4
    _ = imag(3 << s)                   // 非法，3会被隐式的转换称为浮点类型，浮点类型不能进行位移操作

delete
-------

内置函数delete删除映射m中对应键值k的元素，k值必须赋值给映射的键值类型::

    delete(m, k) // 将元素m[k]从映射m中删除

如果m的类型是一个类型参数，该类型参数定义的类型必须都是映射，并且这些映射必须都有
相同的键值类型。如果映射m的值为nil或者元素m[k]不存在，delete相当于是一个空操作。

len cap
--------

内置函数len和cap返回对应类型的长度和容量，返回值的类型是int。 ::

    len(s) 如果s是字符串类型，返回字符串的字节长度
           如果s是数组或数组指针（[n]T *[n]T），返回数组长度n
           如果s是切片[]T，返回切片长度即类型T元素的个数
           如果s是映射map[K]T，返回映射的长度即元素的个数
           如果s是通道chan T，返回通道缓存中的元素个数
           s的类型还可以是类型参数，类型参数约束的类型必须是上面这些类型
    cap(s) 如果s是数组或数组指针（[n]T *[n]T），返回数组长度n
           如果s是切片[]T，返回切片的容量
           如果s是通道chan T，返回通道的缓存容量
           s的类型还可以是类型参数，类型参数约束的类型必须是上面这些类型

切片的容量是该切片对应的底层数组所分配的空间能保存的元素个数。任何情况长度和容量
满足以下关系::

    0 <= len(s) <= cap(s)

值为nil的切片、映射、通道的长度都为0，值为nil的切片、通道的容量都为0。常量字符串
的长度len(s)是一个常量。如果s的类型是数组或者数组指针，并且表达式s不包含通道接收
操作以及非常量的函数调用，那么表达式len(s)和cap(s)也是一个常量，并且s不会求值。
对于其他情况，len(s)和cap(s)都不是常量，并且s都会求值。 ::

    const (
        c1 = imag(2i)                    // imag(2i)是常量
        c2 = len([10]float64{2})         // [10]float64{2}是常量
        c3 = len([10]float64{c1})        // [10]float64{c1}是常量
        c4 = len([10]float64{imag(2i)})  // imag(2i)是常量
        c5 = len([10]float64{imag(z)})   // 非法，imag(z)是一个非常量函数调用，结果不是常量
    )
    var z complex128

make
-----

内置函数make创建切片、映射、通道类型T的一个新值，第一个参数是类型T，后续参数
可选地传递类型T特定的参数。T的核心类型必须是一个切片、映射、或者通道类型，函数
的返回值类型是T（不是*T），内存会初始化为对应类型的零值。 ::

    调用             核心类型    结果
    make(T, n)       切片       类型T的切片，长度和容量都为n
    make(T, n, m)    切片       类型T的切片，长度n，容量m
    make(T)          映射       类型T的映射
    make(T, n)       映射       类型T的映射，并分配大概n个元素的初始空间
    make(T)          通道       类型T的非缓存通道
    make(T, n)       通道       类型T的缓存通道，缓存个数为n

参数n和m的类型必须是整型，或者是无类型的常量。如果是常量，必须不是一个负数并且
可以表示成int类型，如果是无类型常量会被转换成int类型。如果n和m都是常量，m必须
大于等于n。对于切片和通道，如果在运行时n是负数或者大于m，会导致运行时错误。对于映射，
参数n表示创建一个初始空间可以容纳n个元素的映射，但具体值是多少由具体实现决定。 ::

    s := make([]int, 10, 100)       // int切片，长度10，容量100
    s := make([]int, 1e3)           // int切片，长度和容量都是1000
    s := make([]int, 1<<63)         // 非法，1<<63不能用int表示
    s := make([]int, 10, 0)         // 非法，容量小于长度
    c := make(chan int, 10)         // 缓存个数为10的通道
    m := make(map[string]int, 100)  // 映射初始空间大概容量100个元素

new
----

内置函数new接受一个类型T为参数，在运行时创建该类型变量所需要的内存空间，并返回类型
T的指针*T指向这个变量，变量的值被初始化为该类型的零值。 ::

    new(T)

比如下面的例子，分配类型S的内存空间，并初始化为零值（a为0，b为0.0），然后返回指针::

    type S struct { a int; b float64 }
    new(S) // 返回值的类型是*S

min max
--------

内置函数min和max用于计算一个组数的最小最大值，必须指定提供一个参数 [Go1.21]_。
这些值的类型必须是有序类型（ordered types）。对于有序类型参数值x和y，如果x+y
是合法的那么min(x,y)也是合法的，并且min(x,y)的类型是表达式x+y的类型，max类似。
如果所有参数都是常量，结果也是常量。 ::

    var x, y int
    m := min(x)                 // m是x
    m := min(x, y)              // m是x和y中的最小值
    m := max(x, y, 10)          // m是x和y中的最大值，但至少为10
    c := max(1, 2.0, 10)        // c为浮点数10.0
    f := max(0, float32(x))     // f的类型是float32
    var s []string
    _ = min(s...)               // 非法，参数不能传递切片
    t := max("", "foo", "bar")  // t是字符串"foo"，字符串的大小根据字母顺序进行比较

对于数值参数，如果所有的NaN都相等，那么min和max满足交换律和结合率::

    min(x, y)    == min(y, x)
    min(x, y, z) == min(min(x, y), z) == min(x, min(y, z))

对于浮点类型参数负零、NaN、无穷大，它们遵循以下规则::

    x       y     min(x, y)    max(x, y)
    -0.0    0.0   -0.0         0.0     // 负零小于正零
    -Inf    y     -Inf         y       // 负无穷小于任何其他数
    +Inf    y     y            +Inf    // 正无穷大于任何其他数
    NaN     y     NaN          NaN     // 如果任何一个参数是NaN，其结果都是NaN

print println
---------------

当前实现中提供了print和println两个内置函数，可以用于语言启动和初始化阶段。
这些函数仅为了完整性而记录在当前的实现中，后面的实现可能移除这些函数。这些
函数没有返回值。实际开发中，通常使用Go标准库fmt包中提供的函数。而在引导阶段
或者某些特殊的低级编程场景中，内置的print和println函数可能会被用到。 ::

    print    打印所有的参数，参数的格式由实现决定
    println  打印所有的参数，并在参数之间打印空格，在末尾打印换行

这两个函数的具体实现可以不需要接受任何的参数类型，但是布尔、数值、和字符串
类型必须支持。

panic recover
--------------

内置函数panic和recover，用于辅助运行时异常和程序定义错误的产生和处理::

    func panic(interface{})
    func recover() interface{}

当执行一个函数F时，一个运行时异常或者显式调用panic会终止F的执行，函数F中
所有defer的函数会在终止前执行，然后调用F的函数的defer函数也会被执行，直到
当前goroutine环境中的顶层函数为止。这时，程序会被终止并报告对应错误条件，
包含传递给panic的参数值。这个终止流程被称为抛出异常（panicking）。 ::

    panic(42)
    panic("unreachable")
    panic(Error("cannot parse"))

内置函数recover可以让程序管理一个goroutine抛出的异常。假设一个函数G声明了
一个defer函数D，D调用了recover函数，并且在函数G的执行过程中产生了一个异常。
异常抛出后，当执行到函数D时，recover函数的返回值为对应异常的panic参数。如果
D正常返回，没有抛出新的异常，那么这个异常抛出的流程就会被终止，相当于这个异常
被recover捕获处理掉了。然后函数D之前defer的函数会被执行，然后G的执行结束返回
到它的被调函数中。

如果goroutine没有发生异常或者recover函数不是被defer函数直接调用，那么recover
返回的值为nil。相反的，如果发生了异常并且recover直接在defer函数中该调用，那么
recover的返回一定不是nil。为了保证这点，给panic传递nil值会导致一个异常。

下面的protect函数调用函数g，并且保护上层调用者免受函数g产生的异常的影响::

    func protect(g func()) {
        defer func() {
            log.Println("done")  // 即使发生了异常，Println也会正常执行
            if x := recover(); x != nil {
                log.Printf("run time panic: %v", x)
            }
        }()
        log.Println("start")
        g()
    }

错误和异常
===========

预声明的类型error定义如下::

    type error interface {
        Error() string
    }

该接口类型表示一个错误条件，如果值为nil表示没有错误。例如一个从文件读取数据的函数
可能被定义成::

    func Read(f *File, b []byte) (n int, err error)

执行过程中的错误，例如数组的索引值超出数组的范围，会导致抛出一个运行时异常。运行时
异常相当于调用了一个panic函数，传递的参数是runtime.Error接口类型值，该接口满足
预声明类型error。Error具体实现包括具体错误的error值都由编译器具体实现决定::

    package runtime
    type Error interface {
        error
        // 或许还包含其他成员函数声明
    }

包
===

* `源文件组织形式`_
* `Package语句`_
* `Import声明`_

源文件组织形式
--------------

Go程序通过链接多个包形成。一个包由一个或多个源文件组成，属于包的元素包括常量、
类型、变量、函数可以被同一个包中的所有文件访问，这些元素可以导出被其他包使用。

每一个源文件都包含一个packege语句定义这个文件属于哪个包，然后可能包含一系列
import声明表示当前文件想使用的包，然后可能包含一些列函数、类型、变量、和常量
的声明。 ::

    SourceFile = PackageClause ";" { ImportDecl ";" } { TopLevelDecl ";" } .

Package语句
------------

每个源文件以package语句开头，定义该源文件属于哪个包::

    PackageClause = "package" PackageName .
    PackageName = identifier .

包的名称不能是空标识符。共享同一个包名的所有文件组成了一个包的实现。编译器可能限制一个
包中的所有文件必须在同一个目录下。

Import声明
-----------

Import声明引入当前源文件需要访问的其他包，让该文件可以访问其他包中导出的标识符。
其中PackageName是对应包在当前文件中的名称，ImportPath表示要导入的包::

    ImportDecl = "import" ( ImportSpec | "(" { ImportSpec ";" } ")" ) .
    ImportSpec = [ "." | PackageName ] ImportPath .
    ImportPath = string_lit .

如果没有提供PackageName，默认是导入包的包名。如果提供的是一个点号 ``.``，相当于
导入包中的所有导出标识符都声明到了当前的文件作用域中。

ImportPath的解析依赖于具体实现，但通常是编译好的包的全路径文件名的子串，或者是相对于
安装好的包目录的相对路径。编译器可能限制ImportPath不能包含空字符，并且只能使用Unicode
分区L M N P S中的字符，并且不能包含 ``!"#$%&'()*,:;<=>?[\]^`{|}`` 以及Unicode替换
字符U+FFFD。

比如一个编译好的包的包名是math，它导出了符号Sin，并且包对应的安装文件是"lib/math"，
下面的示例说明了import声明的用法::

    import "lib/math"     // Sin的局部名称为math.Sin
    import m "lib/math"   // m.Sin
    import . "lib/math"   // Sin

一个import声明定义了一个当前包与导入包之间的依赖关系。在一个包中直接或间接导入自己
是非法的，直接导入一个包但没有使用其中的内容也是非法的。如果仅仅是为了导入一个包来
执行包的初始化而不使用它，可以使用空标识符作为显式的包名::

    import _ "lib/math"

下面是一个完整的Go语句包，实现了一个并行执行的素数筛选器（prime sieve）::

    package main
    import "fmt"
    func generate(ch chan<- int) {
        for i := 2; ; i++ {
            ch <- i // 这是一个无限循环，不断地发送整数到通道ch
        }
    }
    func filter(src <-chan int, dst chan<- int, prime int) {
        for i := range src {  // 从src通道中接收到每个值i
            if i%prime != 0 { // 去除所有的非质数，不能被prime整除的才是质数
                dst <- i // 如果是质数，将质数发送到dst通道
            }
        }
    }
    func sieve() {
        ch := make(chan int)
        go generate(ch)
        for {
            prime := <-ch
            fmt.Print(prime, "\n")
            ch1 := make(chan int) // 这里为每个filter协程都分配一个通道，避免协程之间相互干扰
            go filter(ch, ch1, prime) // 这里每次都创建一个goroutine是因为每次传入的prime值是最新的
            ch = ch1
        }
    }
    func main() {
        sieve()
    }

程序初始化和执行
=================

* `类型零值`_
* `包初始化`_
* `程序初始化`_
* `程序执行`_

类型零值
--------

当变量的内存分配之后，不管是通过变量声明还是调用内置函数new，或者一个新值被创建之后，
不管是通过复合字面量还是调用内置函数make，只要没有提供显式的初始值，那么这些变量和值
将被赋值为默认值。即这些变量和值的每个元素都会被设置为对应类型的零值：布尔类型false，
数值类型0，字符串类型""，指针、函数、接口、切片、通道、映射类型nil。这一过程会被递归
的执行，例如一个结构体数组中每个结构体中的每个成员都会设成对应的零值。 ::

    var i int // 等价于下面这条语句
    var i int = 0
    type T struct { i int; f float64; next *T }
    t := new(T) // t.i为0，t.f为0.0，t.next为nil
    var t T // t的值与上面一样

包初始化
---------

如果包作用域中的变量，没有初始化也没有提供初始化表达式，或者它的初始化表达式不依赖
任何其他没有初始化的变量，那么称这个变量已经准备好进行初始化。包初始化过程就是不断
的对包作用域中下一个声明最早的并且已准备好初始化的变量进行初始化，直到不再有已准备
好初始化的变量。

这一流程结束后如果还存在未初始化的变量，这些变量一定属于一个或多个初始化循环依赖的
一部分，这时程序是不合法的。

如果变量声明左边有多个变量，右边只有一个表达式（会产生多个值），那么这些变量会一起
进行初始化。包初始化过程中，空标识符变量会被当作其他普通变量一样对待。

包中多个文件中的变量声明的顺序，取决于哪个文件先被编译器解析。为了重现包初始化行为，
构建系统应该保证同一个包中的所有文件总是以相同的顺序（推荐文件名称排序）交给编译器。

初始化依赖性的分析，不依赖于变量的实际值，仅仅基于源代码中变量的词法引用，这种分析
是递归的，它会跟踪所有的间接引用。例如，如果一个变量x的初始化表达式引用了一个函数，
该函数使用了变量y，那么变量x依赖变量y。具体地：

1. 对一个变量或函数的引用，即使用这个变量或函数的标识符名称
2. 对一个成员函数m的引用，可以是成员函数值或成员函数表达式t.m，其中t的静态类型不是接口类型，
   并且m是t的成员函数集合中的一员，只要使用了表达式t.m不管是否调用引用已经发生；
3. 一个变量、函数、或者成员函数x依赖变量y，如果x的初始化表达式或者函数体包含了对y的引用
   或者包含了一个依赖变量y的函数或成员函数；

例如::

    var x = a
    var a, b = f() // a和b会一起初始化，会先于x因为x依赖a
    var ( // 下面变量的初始化顺序是：d b c a
        a = c + b  // a最后初始化，值为9，初始化表达式中变量顺序无影响，不管c+b还是b+c最终初始化顺序一样
        b = f()    // b的值为4，它比c先初始化只因为它出现在c的前面
        c = f()    // c的值为5
        d = 3      // 最初d的值是3，初始化b是变成4，初始化c时变成5
    )
    func f() int {
        d++
        return d
    }

初始化依赖性分析基于包，只有对声明在当前包中的变量、函数、非接口成员函数的引用才是
依赖。如果变量之间存在其他隐藏依赖，这两个变量的初始化顺序不确定。例如以下代码，
变量a在变量b之前初始化，但是变量x的初始化是不确定的，sideEffect()什么时候调用
也是不确定的，因为看不出依赖关系随时初始化都可以::

    var x = I(T{}).ab()   // x的初始化顺序不确定，因为引用了接口成员函数
    var _ = sideEffect()  // 无明确的依赖关系，什么时候初始化都可以
    var a = b // a依赖b，a会先初始化
    var b = 42
    type I interface      { ab() []int }
    type T struct{}
    func (T) ab() []int   { return []int{a, b} }

变量还可以在init函数中进行初始化，init声明在包作用域中，没有参数也没有返回值。
一个包中可以有多个init函数，甚至一个文件中也可以有多个。在包作用域中init标识符
仅能用来声明init函数，并且这个标识符自己没有被声明，因此init函数不能被任何其他
程序引用。 ::

    func init() { … }

整个包完整的进行了初始化，只要包作用域中所有变量都根据依赖关系进行了初始化，并且
所有init函数按照它们在源代码中出现的顺序都被调用了。

程序初始化
----------

一个完整的程序，其中的包是逐步初始化的，每次初始化一个包。如果包使用import导入了
其他包，导入的包会在当前包初始化之前进行初始化。如果多个包导入了同一个包，这一个
包只会初始化一次。导入包的初始化会保证不会产生循环初始化依赖，精确的，导入包列表
按导入路径进行排序，每一步都在列表中查找第一个未初始化的包进行初始化，如果一个包
没有导入包或者所有导入的包都初始化了则记为这个包已经初始化了，否则未初始化，不断
重复这个步骤直到所有包初始化完毕。

包初始化（变量初始化以及init函数的调用）是顺序的在单个goroutine中执行的，每次
初始化一个包。init函数可能启动其他goroutine，这些goroutine会并行的与初始化代码
一起执行。但是初始化流程总是线性的，下一个init函数的执行必须等上一个init函数执行
完成。

程序执行
---------

一个完整的程序，是通过链接单个不能被导入的main包，以及递归地链接main导入的其他包
构成的。main包必须将包名定义为main，并且必须声明一个没有参数没有返回值的main函数::

    func main() { … }

程序的执行以程序初始化开始，然后调用main包中的main函数，当这个函数返回时程序就会
结束，不会等待其他（non-main）goroutine完成。

系统底层考量
============

* `unsafe包`_
* `类型长度和对齐`_

unsafe包
---------

内置包unsafe，可以通过包路径"unsafe"导入使用，提供了低层次的程序功能，包括那些违反
当前类型系统规则的操作。使用unsafe包的代码必须手动检查类型安全，并且代码可能是不可
移植的。unsafe包提供了以下接口::

    package unsafe
    type ArbitraryType int  // 表示任意的Go类型，它不是一个实际的类型
    type Pointer *ArbitraryType
    func Alignof(variable ArbitraryType) uintptr
    func Offsetof(selector ArbitraryType) uintptr
    func Sizeof(variable ArbitraryType) uintptr
    type IntegerType int  // 表示整数类型，它不是一个实际的类型
    func Add(ptr Pointer, len IntegerType) Pointer
    func Slice(ptr *ArbitraryType, len IntegerType) []ArbitraryType
    func SliceData(slice []ArbitraryType) *ArbitraryType
    func String(ptr *byte, len IntegerType) string
    func StringData(str string) *byte

Pointer是一个指针类型，但是该指针值不能解引用。任何类型的指针或者uintptr类型的值
都可以转换成核心类型是Pointer的类型，反之也一样。具体的Pointer与uintptr类型之间
的转换由实现决定。 ::

    var f float64
    bits = *(*uint64)(unsafe.Pointer(&f)) // float64类型指针转换成Pointer转换成uint64类型指针然后取值
    type ptr unsafe.Pointer
    bits = *(*uint64)(ptr(&f)) // 与上面一样，ptr的核心类型是Pointer
    func f[P ~*B, B any](p P) uintptr {
        return uintptr(unsafe.Pointer(p)) // 将指针p转换成Pointer再转换成uintptr
    }
    var p ptr = nil // Pointer类型可以赋值为nil

函数Alignof和Sizeof可以传入任意类型的表达式x，返回该类型的对齐字节数或者类型的长度。
函数Offsetof接受一个变量成员选择表达式s.f，其中s是一个结构体变量或者指针，f是结构体
中的一个成员，该函数返回该结构体成员相对结构体开始地址的偏移字节数。如果f是一个内嵌
成员，它必须不需要通过指针解引用而直接访问到。 ::

    uintptr(unsafe.Pointer(&s)) + unsafe.Offsetof(s.f) == uintptr(unsafe.Pointer(&s.f))

计算机架构可能需要内存地址对齐，即一个变量的地址必须位于该变量对应类型的对齐字节数上，
也即变量的地址必须是对应类型对齐字节数的整数倍::

    uintptr(unsafe.Pointer(&x)) % unsafe.Alignof(x) == 0

一个类型T的变量的长度是不固定的，如果T是类型参数，或者是数组或结构体类型，但是这些
数组和结构体包含了不固定长度的元素或成员。否则变量的长度是固定的常量。如果参数对应
类型是固定长度的，函数Alignof、Offsetof、Sizeof的调用结果也是一个常量，该常量的
类型为uintptr。

函数Add对一个指针指向的地址添加一个偏移，返回更新后的指针值 [Go1.17]_::

    unsafe.Pointer(uintptr(ptr) + uintptr(len))

函数Slice返回一个由ptr和len表示的底层数组的切片，len必须是非负数。在运行
时如果len是负数，或者ptr是nil而len不是零会导致运行时异常 [Go1.17]_。如果
ptr是nil那么Slice会返回nil [Go1.17]_，否则Slice(ptr, len)等价于::

    (*[len]ArbitraryType)(unsafe.Pointer(ptr))[:]

函数SliceData返回指向对应切片的底层数组的指针，如果切片的容量不是零，那么
返回的指针值是&slice[:1][0]，如果切片的值是nil返回的指针值也是nil，否则
指针值是一个指向未知内存的非nil值 [Go1.20]_。

函数String返回由ptr和len表示的底层字节数组对应的字符串，ptr和len参数与
Slice函数的参数有相同的限制。如果len是零，返回的结果是空字符串。因为Go
语言的字符串是不可修改的，这个底层字节数组在传入String之后，要保证不会
被修改。[Go1.20]_

函数StringData返回对应字符串的底层字节数组的指针。对于一个空字符串，返回
的指针是不确定的，可能是nil。由于Go语言的字符串是不可修改的，要保证获得
指针后，不会修改指向的内容 [Go1.20]_。

类型长度和对齐
--------------

数值类型的长度大小保证::

    byte uint8 int8                 长度为1个字节
    uint16 int16                    长度为2个字节
    uint32 int32                    长度为4个字节
    uint64 int64 float64 complex64  长度为8个字节
    complex128                      长度为16个字节

最小的对齐属性保证：

1. 对于任意类型的变量x，unsafe.Alignof(x)至少是1
2. 对于结构体类型变量x，unsafe.Alignof(x)是所有成员对齐字节数的最大值，但至少是1
3. 对于数组类型的变量x，unsafe.Alignof(x)与元素的对齐字节数一样

一个结构体或数组类型，如果它们没有包含元素或成员，或者包含的元素或成员的大小都是
零，那么这个结构体或数组的大小也是零。两个不同的大小为零的变量，可能在内存中有相同
的地址。

附录
=====

* `语言版本`_
* `类型等价规则`_

语言版本
--------

Go语言版本1的兼容性保证，所有用Go 1规范写的程序在对应规范的生命期内，都可以
无修改的正确的编译执行。更一般的，Go语言在一个特定版本上对语言的修订和功能的
添加，会保证在后续的子版本中继续有效。

例如，Go 1.13添加了用0b前缀表示二进制整数字面量的功能，那么这个功能在后续的
子版本Go 1.xx中是有效的，其中xx大于等于13。但是如果编译器使用一个比Go 1.13
老的语言版本，使用0b前缀表示二进制整数字面量会报错。

以下列出了自Go 1规范之后，每个语言特性需要的最小语言版本支持：

.. [Go1.9]

* 类型别名

.. [Go1.13]

* 0b 0B 0o 0O前缀整数字面量
* 0x 0X前缀浮点字面量
* 虚数后缀i可以使用在任意整数或浮点数字面量之后，不仅仅是十进制字面量
* 任意数值字母量的两个数字之间可以用下划线字符 `_` 分隔
* 整数位移操作的有操作符可以是一个有符号整型

.. [Go1.14]

* 允许接口中内嵌接口导致一个相同的成员函数出现多次

.. [Go1.17]

* 允许切片转换成数组指针，只要元素类型匹配并且数组的长度不大于切片的长度
* unsafe包添加Add和Slice两个新函数

.. [Go1.18]

该版本添加了泛型，具体的：

* 添加与泛型相关的操作符 `~`
* 函数和类型可以声明类型参数
* 接口可以内嵌任何类型，不仅仅是接口，还包含类型联合以及~T等类型元素
* 添加了泛型相关的预声明类型any和comparable

.. [Go1.20]

* 允许切片转换成数组，只要元素类型匹配并且数组长度不大于切片的长度
* unsafe包添加SliceData、String、StringData三个新函数
* 即使类型实参不是严格可比较的，可比类型（比如普通接口）仍可以满足
  comparable约束，comparable表示的是严格可比较类型的集合，这相当
  于放松了comparable约束的限制，对泛型参数传参更具灵活性

.. [Go1.21]

* 添加预声明函数min、max、和clear
* 类型推导利用接口成员函数的类型自动推导类型参数的实际类型，即编译器可以
  根据接口成员函数的签名来自动推导类型，当泛型函数赋值给变量或者作为参数
  传递给其他函数时，也会对函数的类型参数进行推导

.. [Go1.22]

* For语句中，每个迭代都有自己版本的迭代变量，而不是共享相同的迭代变量
* 使用range的for语句时，可以根据一个整数进行迭代

类型等价规则
------------

类型等价规则用来描述两个类型是否以及是怎样等价的。类型等价用来求解类型等式，它
比较类型等式中的两个类型，为对应的类型参数匹配类型实参。规则的细节对Go语言实现
很重要，它影响到错误消息的具体内容（比如编译器是否要报一个类型推导错误还是其他
错误），并且可能影响在一些不常见代码情况下类型推导为什么会失败的原因的解释。尽
管如此，在编写Go语言代码时可以忽略这些规则，因为类型推导被设计为在绝大多数情况
下都可以“按预期工作”，并且等价规则都相应的做了精调。

类型等价有两种匹配模式，精确匹配（exact）和宽松匹配（loose）。在类型等价检查
一个组合类型时，会递归的检查这个复合类型的元素或成员，元素等价的匹配模式会与顶
层的匹配模式保持一致，除了可赋值性（assign ability，≡A）类型等价的情况例外。
这种情况下，顶层的匹配模式是宽松的，但是元素等价的匹配模式仍需要精确匹配，这反
映的事实是可赋值的两个类型不需要完全相同。

两个类型都不是绑定的类型参数，如果满足下面任意一个条件，它们是精确等价的：

1. 两个类型完全相同（identical）
2. 两个类型有相同的结构，并且它们的元素精确等价
3. 一个类型的核心类型，根据可赋值性（≡A）等价规则这个核心类型与另一个类型等价

两个类型都是绑定的类型参数，如果满足下面一个条件，它们在给定的匹配模式下是等价的：

1. 两个类型参数完全相同（identical）
2. 只有一个类型参数有一个已知的类型实参，将两个类型参数合并后都表示同一个类型实参
3. 两个类型参数都没有已知的类型实参，一个类型参数推导出的实参能同时应用到另一个类型参数上
4. 两个类型参数都有一个已知的类型实参，并且两个类型实参在给定的匹配模式下是等价的

两个类型P和T只有P是绑定的类型参数，如果满足下面一个条件，它们在给定的匹配模式下等价：

1. P没有已知的类型实参，T可以作为P推导出的类型实参
2. P有一个已知的类型实参A，类型A和T在给定匹配模式下是等价的并且满足以下条件之一：

  - 类型A和T都是接口类型，如果A和T都是定义的类型（defined type）它们必须完全
    相同，如果两个类型都不是定义的类型，它们必须有相同数量的成员函数（函数的等价
    已经在前置条件类型A和T在给定匹配模式下等价中满足）
  - 类型A和T都不是接口类型，如果T是定义的类型，T可以代替A作为P推导出的类型实参

两个类型都不是绑定的类型参数，如果满足下面一个条件，它们是宽松等价的：

1. 两个类型精确等价
2. 一个类型是定义的类型（defined type），一个类型是类型字面量，但都不是接口类型，
   它们的底层类型在元素等价的匹配模式下是等价的
3. 两个类型都是接口类型（但不是类型参数），有相同的类型元素，两个类型要么都有要么
   都没有内嵌comparable接口，两个类型声明的成员函数是精确等价的，一个接口的成员
   函数集合是另一个接口成员函数集合的子集
4. 一个类型是接口类型（但不是类型参数），两个类型声明的成员函数在元素等价的匹配
   模式下是等价的，并且接口的成员函数集合是另一个类型的成员函数集合的子集
5. 两个类型有相同的结构，并且它们的元素类型在元素等价的匹配模式下等价
