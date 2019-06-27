---
layout    : post
title     : "Questions From TGPL: Answers"
date      : 2019-06-27
lastupdate: 2019-06-27
categories: go
---

#### 1. Short variable declarations with `:=`

> TGPL 2.3.1

`:=` is used for declare and initialize local varaibles.

Short varaible declaration takes the form:

```go
var := expression
```

Where the type of `var` is determined by the `expression`.

Keep in mind that `:=` is a **declaration**, whereas `=` is an **assignment**.

One subtle but important point: a short variable declaration does not
necessarily declare all the variables on its left-hand side. If some of them
already declared in the same lexical block , then the short variable declaration
acts like an assignment to those variables.

In the code below, the first statement declares both `in` and `err`. The second
declares `out` but only assigns a value to the existing `err` variable.

```go
a, err := f()
b, err := f()
```

A short variable declaration must declare **at least one new variable**,
however, so this code will not compile:

```go
c, err := f()
c, err := f()
```

To fix it:

```go
c, err := f()
c, err = f()
```

#### 2. Describe the relationship between: Unicode, UTF-8 and `rune`

> TGPL 3.5

Three different things:

* Unicode: character set, contains a complete set of all characters
* `rune`: the character index in a character set, can be represented in a
  4-byte integer
* UTF-8: an encoding/compressing scheme for `rune`


给每个 Unicode 一个索引号，用 int32 表示，这个索引号在 go 里就成为 rune（古日耳
曼字母；神秘的或有魔力的符号；具有神秘意义的诗歌或咒语）。
因此，所有的字符串都其实都可以用 rune 表示，对应的标准是 UTF-32 or UCS-4。但是，
这样的表示每个字符都需要 4 个字节，非常占用存储空间，因为大部分字符只要 1 或 2
字节就能表示了。

有没有更经济的编码方式呢？有：UTF-8，发明者是 Ken Thompson and Rob Pike，没错，
正是 Go 语言的发明者本尊。UTF-8 是对 rune 进行变长编码（UTF-8 is a
variable-length encoding of Unicode code points as bytes. UTF-），最长不超过 4
字节。
注意这里有两层编码：
1. 字符在字符表中的值（例如 0xabcd）
2. 字符组字符表中的索引号（例如 168）
UTF-8 并不是字符在字符表中的值（例如 0xabcd），而是这个字符在字符表中的索引（例
如 168）的编码（压缩）；需要先拿到索引（168），然后去字符表里根据索引拿到字符本
身的值（0xabcd）。

原理：前缀编码，最高的 N bit 表示接下来用几字节编码。

<p align="center"><img src="/assets/img/questions-from-tgpl/2.png" width="60%" height="60%"></p>

因此如果给定是 UTF-8 表示，要转成字符串的话，必须要 UTF-8 解码。
utf8.DecodeRuneInString

UTF-8 的好处：
1. 效率高
1. 与 ASCII 兼容

Unicode 是字符集，UTF-8 是编码。

Unicode literal，以下三者是等价的：'B'  '\u4e16'  '\U00004e16'
Rune literal："\xe4\xb8\x96\xe7\x95\x8c"

#### 3. Convert integer into comma separated format

> TGPL 3.5.4

Iterative:

```go
func comma(s string) string {
	n := len(s)
	if n <= 3 {
		return s
	}
	return comma(s[:n-3]) + "," + s[n-3:]
}
```

Recursive:

```go
func comma(s string) string {
	r := ""
	for n := len(s); n > 3; n -= 3 {
		r = "," + s[n-3:] + r
		s = s[:n-3]
	}
	return s + r
}
```

#### 4. Nil slice & empty slice

> TGPL 4.2

nil slice 的 len 和 cap 都是 0，但反过来不成立，即 len 和 cap 都是 0 的不一定是
nil slice

```go
var s []int    // len(s) == 0, s == nil
s = nil        // len(s) == 0, s == nil
s = []int(nil) // len(s) == 0, s == nil
s = []int{}    // len(s) == 0, s == nil
```

#### 5. Address of a map item

> from chapter 4

```go
_ = &m[k] // compiler error
```

其中一个原因是：map 是会动态扩展的，每次扩展之后内存位置都会变。

map 的遍历顺序是没有规定的，跟实现的 hash 算法相关。

slice 也是会动态扩展的，扩展后内存也会变，但是经测试，slice 是可以取元素地址的，
WHY？另外经测试，slice 扩容后原来的元素地址的确是会变的，所以尽管可以取地址，也
不要这样做，因为它可能很快就是非法地址：


```go
func main() {
	s := []int{1, 2}
	fmt.Printf("%p, %p\n", &s[0], &s[1])
	s = append(s, 3)
	s = append(s, 4)
	fmt.Printf("%p, %p\n", &s[0], &s[1])
}
```

```shell
$ go build
$ ./main
0xc000086010, 0xc000086018
0xc00009c020, 0xc00009c028
```

#### 6. `nil` map operation

```go
var ages map[string]int
fmt.Println(ages == nil)    // "true"
fmt.Println(len(ages) == 0) // "true"
```

对 nil 的 map 进行 get/delete/iterate 都是安全的，但是，对其赋值是不行的：

```go
ages["carol"] = 21 // panic: assignment to entry in nil map
```

#### 7. Decide if two maps are equal

> from chapter 4

注意必须要同时检查是否存在，以及元素是否相等。如果不检查是否存在，下面这个 case
就会被判断为相等：

```go
func equal(x, y map[string]int) bool {
	if len(x) != len(y) {
		return false
	}

	for k, v := range x {
		if vv, ok := y[k]; !ok || vv != v {
			return false
		}
	}
	return true
}
```

#### 8. JSON marshal/unmarshal

> from chapter 4

marshal：v 整顿；配置；汇集。n 陆空军元帅；典礼官；司仪官

Marshal 输入的是 Go struct 实例，返回的是 []byte，去掉了所有空格，只有一行。只有
导出的字段（首字母大写）才会被 marshal。
MarshalIndent 添加了缩进，打印更美观

Unmarshal 将 []byte 转换为 Go struct 实例。

#### 9. `len` and `cap` of a slice

> from chapter 4

slice 内部实现使用的是 go array，即固定长度数组。当 append 数据导致数据长度不够
时，就动态分配一个新数据，将老数据 copy 过去。

len()：slice 当前的元素数量
cap()：在 slice 底层的 array 中，从 slice 的第一个元素（不一定是 array 的第一个
元素）到 array 最后一个元素（不一定是 slice 的最后一个元素）的总容量（包括未使用
部分）

<p align="center"><img src="/assets/img/questions-from-tgpl/4-1-org.png" width="60%" height="60%"></p>

直接取 summer[4] 会导致 panic，因为超出了 summer 的 len；但是，可以通过 slicing
对它扩容：summer[:5]，这时候再取就可以了：

```go

fmt.Println(summer[:20]) // panic: out of range

endlessSummer := summer[:5] // extend a slice (within capacity)
fmt.Println(endlessSummer)
```

注意如果按以上方式扩容，当扩容超过了底层 array 的大小时（例如 endlessSummer :=
summer[:100]），编译不会报错，但运行时会 panic。

slice 指向的是底层的 array，传递一个 slice 作为参数得到的是对底层 array 的一个引
用。

#### 10. Reverse a slice in place

> from chapter 4

```go
func reverse(s []int) {
	for i, j := 0, len(s)-1; i < j; i, j = i+1, j-1 {
		s[i], s[j] = s[j], s[i]
	}
}
```

#### 11. Why can not compare two slices?

> from chapter 4

判断两个简单 slice 是否相同可以用：

但是，要深度判断（deep equivalence）两个 slice 是否相同是有问题的。因为 slice 的元素都是间接的（indirect），一个 slice 可能会包含它自身。虽然这个问题也可以解决，但总体来说，没有简单、高效、显而易见一看就懂得方式。

另外，slice 和 map 是可以被修改的，而一些 golang 特定只会对 map 做浅拷贝（只考 key）。如果 slice 包含了一个元素指向一个 map，那 map 的value 变化时，slice 是感知不到的。
最安全的方式就是不提供深度 equal 函数。唯一的例外：和 nil 比较是安全的。

#### 12. Exponential backoff retry

> chapter 5

缺点：最后一次 sleep 可能会很长，导致总的 sleep 时间接近 timeout 的两倍。

#### 13. Lexical scope

> chapter 5

dir := d 是必须的，原因和 for 的 scope rule 有关。for 创建了一个 lexical block，d 在这个 block 之中，每次循环创建出来的变量都是对同一块内存地址的引用，而不是 d 的值本身）。

此如果不声明一个新变量 dir 来保存每次都 d，最后 rmdirs 里面所有的元素都是相同的，指向 d 的最后一次值所指向的地址。要真正准确地理解，得去看 golang 的实现。

下面一个例子也是一样的（由于变量 i 的作用域）：

在 go 或 defer statement 中经常会遇到这种问题，但本质上这个问题和 go 和 defer 没有关系。

另一个例子，来自 golang wiki：
https://github.com/golang/go/wiki/Range

<p align="center"><img src="/assets/img/questions-from-tgpl/go-wiki-range.png" width="80%" height="80%"></p>

#### 14. Variadic function

在背后，调用方会生成一个 array，将所传的变量拷进去，然后将 array 转换为 slice 传给函数。

#### 15. `defer` inside loop

defer 只有到函数执行结束的时候才会执行，如果在 for 循环中用 defer，就会存在很大
风险，看例子：

这里的 defer 只有等到 for 循环结束之后才会开始执行，也就是说每次循环都会创建一个 fd 并且不会释放，直到函数结束的时候才开始释放。如果 for 循环很大，那有 fd 用尽的风险。
解决方式：将 open 和 defer 放到一个单独函数，保证每个 fd 用完就释放：

#### 16. `defer` execution order after panic

panic 后，defer 以逆序执行，最后 push 的 defer 最先执行。

#### 17. get call stack after panic

#### 18. Simple breadth first crawler

#### 19. Custom type: auto convert value to pointer

假设自定义类型 Point，那方法签名有两种形式，根据传入的（在 golang 里叫 receiver
）是否是指针类型：
func (p Point) f() {}
func (ptr *Point) f() {}

变量的实例也有两种形式：变量或指针：
p := Point{}
ptr := &Point{}

因此有三种调用方式：
1. 实例和 receiver 都不是指针，或者都是指针：调用正常，因为实例和 receiver 一直
1. 实例是指针， receiver 不是指针：调用正常，编译器会将指针转换为实例
1. 实例不是指针， receiver 是指针：编译器会将实例转换为指针，但是，实例必须是可
   寻址的（例如没有赋值给某变量的临时变量就是不可寻址的）

   第三种情况的一个例子：
   Point{1, 2}.ScaleBy(2) // compile error: can't take address of Point literal
   因为这个变量是（临时变量）literal，没有地址。

#### 20. safely copy

如果 type T 的所有方法 receiver 都是 T 类型的（不是 *T 类型），那 copy 这种类型
的实例是安全的；调用 T 的任何方法都会产生一份拷贝。例子：time.Duration（内部是
type Duration int64）。
https://golang.org/pkg/time/#example_Duration

如果 type T 有 *T 类型的签名方法，那 copy 这种类型的实例就是不安全的，可能会破坏
内部结构。举例：拷贝 bytes.Buffer  是不安全的。

由于非指针类型的 receiver 方法在调用时会产生实例的拷贝，因此效率没有指针
receiver 的方法高。

#### 21. Nil Is a Valid Receiver Value

> chapter 6

对某些类型来说，nil 是一个合法的零值，因此它也可以做 receiver。

Values(nil).Get("item"))  是合法的，但是 nil.Get("item")) 是不合法的（编译不能通
过），因为 nil 的类型无法确定。
第三行失败是因为无法向一个 nil map 插入元素，属于运行时错误。


#### 22. Interface as contracts

go 中有两种类型（type）：

1. concrete type：有具体的内部结构和方法
1. abstract type（interface type）：没有（不暴露）内部结构，只有方法

对于 interface type 类型的 value，只能知道它的行为，不知道其内部数据结构（也不用
关心）。例子：标准库中的 fmt：

接口类型只关心行为，使得执行这个接口方法的变量具有可替换性（substitutability），
这是 OOP 的一个特征（hallmark）。
