# [Go的嵌套机制](http://www.hydrogen18.com/blog/golang-embedding.html)
我最近在做一个Go项目，其间仔细研究了这个语言许多特征的含义，到现在为止，最有趣的一个特征便是[结构体的相互嵌套](http://www.golang-book.com/9)
## Go的类型
Go使用结构体来支持用户定义类型
```Go
type Discount struct {
    percent float32
    startTime uint64
    endTime uint64
}
```
以上定义了一个可实例化的`Discount`类型，而且你还可以为其添加方法：
```Go
func (d Discount) Calculate(originalPrice float32) float32{
    return originalPrice*d.percent
}
```
假设你有一个名为`holidaySale`的`Discount`实例，你能使用`holidaySale.Calculate(49.99)`来调用方法，但方法的接收者并不限定只能是值类型：
```Go
func (d *Discount) IsValid(currentTime uint64) bool{
    return currentTime > d.startTime && currentTime < d.endTime
}
```
不论方法的接收者是值还是指针，他们的调用是一致的，比如`holidaySale.IsValid(100.0)`。当这个方法被调用时，Go语言会自动将值类型转变为一个指针类型。方法接收者采用指针类型的主要原因，是这种情况下方法能修改指针所参引的值，而且这也能避免潜在的拷贝开销。


## 嵌套(Embedding)
类型可以具有零个或者多个匿名域。
```Go
type PremiumDiscount struct{
    Discount //Embedded
    additional float32
}
```
在结构体中包含一个匿名域被称为嵌套。在这个例子中，`Discount`类型被嵌套到`PremiumDiscount`类型中。所有`Discount`的方法都能被`PremiumDiscount`类型直接使用。而且那些同名方法还能被隐藏：
```Go
func (d *PremiumDiscount) Calculate(originalPrice float32) float32 {
    return d.Discount.Calculate(originalPrice) - d.additional
}
```
在这个例子中，`PremiumDiscount`的`Calculate`方法，调用了`Discount`的`Calculate`方法，并将其减去一个额外的数额，因为结构体的匿名域可以使用嵌套结构体的全名（即`d.Discount`）来参引。
也许这看起来非常类似于java中的继承(inheritance)和方法重载(method overriding)，但这只是一种表面现象。
## 经由值的嵌套
使用值来内嵌一个结构体是最直观的方式。这与面向对象语言中的继承机制很类似，但并不完全一样：
```Go
type Parent struct{
    value int64
}

func (i *Parent) Value() int64{
    return i.value
}

type Child struct{
    Parent
    multiplier int64
}

func (i Child) Value() int64{
    return i.value * i.multiplier
}
```
在这个例子中，假设`myParent`是一个`Parent`实例，那么调用`myParent.Value()`将返回其`value`域。类似的，假设`myChild`是一个`Child`实例，那么调用`myChild.Value()`将返回`value`和`multiplier`域的乘积。这与方法重载有本质的不同，因为此时调用`myChild.Parent.Value()`仍然会触发调用`Parent`的`Value()`方法，而非`Child`的。这两种调用(`myChild.Value()`和`myChild.Parent.Value()`)都是在没有用任何[虚表](https://en.wikipedia.org/wiki/Virtual_method_table)机制的情况下，在编译时解析的。
> *译注：在使用虚表机制实现方法重载的情况下，你不可能在子类实例中直接调用到被重载的父类方法，因为它的地址已经在编译时被覆盖掉了。而在go中，你仍然能通过嵌套结构体的全名来访问到父类的同名方法定义*


这并没有什么不妥。考虑下面这个接口例子：
```Go
type Valueable interface {
    Value() int64
}
```
这个接口需要一种类型来实现`Value()`方法。`Parent`类型满足这个接口。因为`Child`类型嵌套了`Parent`类型，因此它也满足这个接口。
> *译注：Go的嵌套机制是继承的一种实现形式，从继承的角度看，新的类型是既有类型的子类，因此**应该**具有既有类型的所有方法，继而也**应该**实现既有类型已经实现的接口，只不过由于Go没有采用虚表，这种继承不是自动的，你必须像上面那样，手动实现子类的所有接口相关方法。*

然而，如果一个`Child`实例被作为`Valueable`使用，想要调用`*Parent`的`Value()`方法并不太容易。当然你可以使用类型申明或者反射来达成这一目的。
> *译注：原文为However, if an instance of Child is used from a context referring to it as a Valueable, there is no easy way to invoke the Value method receiving a &#42;Parent. It is still possible by using type assertions or reflection.不懂如何用type assertion或者reflect来调用Parent的Value方法，如果你知道，欢迎来信告知。*

注意这里有一个重要的差异。如果`myParent`是一个`Parent`实例，我们不能用`myParent`来作为`Valueable`使用，你必须使用`&myParent`，因为在`Parent`的类型定义中，`Value()`接收者是`*Parent`类型，而不是`Parent`。

虽然`myParent`不能满足`Valueable`接口，但`myChild`可以！因为`Child`的`Value()`接收者就是值类型。指针`&myChild`也可以满足接口！
> *译注：当你在函数调用中，将具类传递给其接口类参数，以便基于接口方法调用来实现动态分派时，具类的类型（值或者指针）必须和具类中接口方法的接收者类型保持一致！
> 如果方法定义的接收者是指针类型，你必须在函数调用中传递具类的指针；但如果方法的接收者是值类型，那么你即可传递具类的值也可以传递具类的指针。比如*
> ```Go
> func func1(p Valueable) {
>	fmt.Printf("Valuable context: p.Value() is %d\n", p.Value())
> }
> myParent := Parent{4}     //创建一个值对象
> func1(myParent)   // 编译错误，missing method Value。即Parent类型并没有Value方法，它的指针才有！
>
> myChild := &Child{multiplier: 3}  // myChild := Child{multiplier: 3}也没问题！
> func1(myChild)    // 调用函数时，传递具类的指针和值均合法
> ```
你可能并不希望使用`myChild`来满足接口`Valueable`，因为在调用过程中涉及到一次拷贝，这意味着你在方法中对`myChild`的任何改变都无法被使用该接口的调用者所看到。
> *译注：所以你必须确保方法的接收者类型对象的核心数据能以指针形态被参引，这意味着这个接收者本身就是指针，或者接收者对象中的核心数据是经由指针参引，比如go中的slice类型，其核心数据的内部表达就是一个指向数组的指针。虽然slice本身并非指针类型，但你仍然可以经由slice对象的值拷贝来共享同一个底层数组。*

使用`Child`和`Parent`来满足`Valueable`接口仍然不是传统意义上的继承和方法重载，你也许会争辩，认为这个例子等同于下面的C++代码：
```C++
class Valueable{
    public:
        virtual int Value() = 0;
};
```
接下来你可能会分别定义名为`Parent`和`Child`的两个`class`，但`Parent`需要显式的继承`Valueable`，这在Go中是不需要的。最为显著的不同在于，C++使用`Valueable*`抽象类型不会导致**copy-on-assignment**语义，而这在Go的接口和实现定义中就会发生！

进一步的，在C++中，如果不继承其他抽象类，`Child`类就不能实现这些抽象类的方法，和Go相比，这似乎只是一种语法层面的不同。C++中你必须明确的声明你继承了什么抽象类，而在Go中，一个类的实现要么满足某个接口要么不满足，不同语言的编译器都会对此进行检查，所以Go获得了某种简洁性。
> *译注：在Go中，你不必在结构体类型的定义中显式声明要继承哪些接口类型，编译器会进行这种检测并建立类型与接口之间的实现关系，从语法层面看，这无疑比C++要简单*

而且，你还可能会给一些你无法维护也不能改变的代码定义接口。比如我可能已经写了很多曾实现`Value`方法的不同类型，然后有一天，有人创建了一个`Valueable`接口，这样根本不用有任何代码变动，我先前创建的那些类型自然就满足了这个`Valueable`接口。这种特性使得一个框架可以直接使用另外一个框架的定义，而无需为了粘合两个框架，定义额外的包装类型和一大堆相关包装函数。但在C++中，如果一个框架定义并实现了`Foo::Reader`抽象类，另外一个框架实现了`Bar::Readable`抽象类，这两个类型本来都具有完全相同的`read`函数，但你仍然需要定义一大堆胶水代码，来使这两个库协同工作。

在比较Go接口和C++抽象类中，你最终会发现真正的问题在于处理钻石继承。我们都能排除二义性父类导致的数以百计的编译错误，但所有的努力并没有真正解决你手中的问题。当然java通过禁止多重继承似乎“解决”了这个问题，但这并没有减缓大家给一个单一类型添加16种不同接口的速度！

## 你知道，这并不是你想做的
在Go中通过值来嵌套还有一个根本问题：如果你所嵌套的类型有未导出的域，而你当前又处于另外一个包中，那么你就不能访问这些域。
> *译注：就是说被嵌套的类型中有名称为小写的域，当前类型代码与被嵌套的类型代码不属于同一个包*

你当然能在一个新的类型定义中嵌套包含一个`HTMLElementTree`类型，但如果你不能初始化它，这又有什么用呢？即便因为包含`HTMLElementTree`，新类型会获得一堆`HTMLElementTree`的相关方法，但这些方法因为`HTMLElementTree`域为`nil`而无法调用。
Go中的一个惯例是通过`NewX`方法返回指向一个就绪实例的指针。比如Go的`json`包定义`NewDecoder`来从流中解码JSON，但因为`NewDecoder`返回一个`*Decoder`指针，你也许不得不在初始化一个包含`Decode`的新类型对象时，解引用这个指针，然后创建一份拷贝，但这将导致在堆中创建一个临时`Decode`对象。可实际上并没有什么东西阻止你使用指向既有类型的指针来构造一个创建新类型的`Init`方法，来完成该新类型的所有初始化工作。
> *译注：  `NewDecoder`返回的是一个指针，为此你需要通过&#42;来解引用得到一个`Decode`对象，再将其赋值给新类型实例中的对应域，而这个赋值动作将引发一次拷贝，此时原来通过`NewDecode`在堆中创建的`Decode`对象也再没有用处。也就是说，因为在定义新类型时使用值来嵌套某种既有类型，你不得不在初始化该新类型时承受如下两个动作的代价：*
> - *在堆中创建一个临时对象*
> - *将该临时对象拷贝到另外一个地方*

所有这些都引发了对[Go缺省导出规定](https://golang.org/ref/spec#Exported_identifiers)的强烈争论。[RAII](https://en.wikipedia.org/wiki/Resource_Acquisition_Is_Initialization)的概念听起来不错，但在C++中它最终导致了过高的复杂度。
> *译注：RAII是Resource Acquisition Is Initialization的缩写，大意是说通过类型对象的状态（在go中就是结构体的域）来映射资源，确保资源和对象的生命周期一致：对资源的访问必须发生在对象初始化成功后和对象销毁之前。相关的议题包括资源的封装（在类中定义资源的引用及其状态表示）、异常安全（当异常发生时，保证回收栈中曾分配的资源）、以及局部性（资源的创建、使用和释放必须以一种特定的顺序结构来定义，比如java中的try/catch/finally）。作者的意思是说，基于RAII来实现内存资源的自动化管理可以规避上面的问题，但语言要支持RAII会非常麻烦，而这种复杂度可能会对语言的其他特性产生负面影响*

以上考虑都是假定你希望完全控制被嵌套类型对象实例化过程。我们继续来看下一节。

## 经由指针的嵌套
go中还可以使用指针来实现嵌套：
```Go
type Bitmap struct{
    data [4][4]bool
}

type Renderer struct{
    *Bitmap //Embed by pointer
    on uint8
    off uint8
}

func (r *Renderer)render() {
    for i := range(r.data){
        for j := range(r.data[i]){
            if r.data[i][j] {
                fmt.Printf("%c",r.on)
            } else {
                fmt.Printf("%c",r.off)
            }
        }
        fmt.Printf("\n")
    }
}
```
`Renderer`类型使用指针包含嵌套了`Bitmap`。这样做的第一个好处是，你能使用返回结构体指针的`NewX`来实施初始化。第二个好处是你在使用被嵌套类型对象的功能时，不必知道它是什么时候实例化的。被嵌套的`Bitmap`指针与Go中的其他指针相比并无二致，所以它能被多次赋值，这样你就可以在运行时动态修改它所指向的对象。
> *译注：在经由值嵌套的情况下，每次创建一个新类型对象时，你都需要显式的在堆中创建一个被嵌套对象并初始化它，然后再执行一次拷贝。而在这种经由指针嵌套的情况下，创建新类型对象时，你不必每次都初始化这个指针所指向的嵌套对象：你只需要在运行时初始化一次即可，这样所有新类型的实例即可共享这个被嵌套对象实例。*

在这个例子种，一个单一的`Bitmap`实例作为被嵌套的实例，被多个`Renderer`实例共享：
```Go
var renderA,renderB Renderer   // 构造并初始化新类型变量
renderA.on = 'X'
renderA.off = 'O'
renderB.on = '@'
renderB.off = '.'

var pic Bitmap      // 运行时：在其他地方初始化嵌套的类型对象
pic.data[0][1] = true
pic.data[0][2] = true
pic.data[1][1] = true
pic.data[2][1] = true
pic.data[3][1] = true

renderA.Bitmap = &pic
renderB.Bitmap = &pic

renderA.render()
renderB.render()
```
这里两个不同的`Renderer`对象共享同一个`Bitmap`实例。每个`Renderer`都有其自己的展示字符集，从而允许两种不同的位图打印。下面是输出：
```
OXXO
OXOO
OXOO
OXOO
.@@.
.@..
.@..
.@..
```
这个例子实际上示范了[蝇量模式](http://en.wikipedia.org/wiki/Flyweight_pattern)。尽管本例中节省的内存微不足道，但如果是成千上万个实例来共享单一的底层数据结构，就会显著的降低系统的内存消耗。

## 接口嵌套
Go的用户定义类型并不限定嵌套结构体，我们也能嵌套接口。

这样做的一个效果是，任何包含某个接口的类型将自动满足对应的接口，而你必须实现这些接口方法，在这个过程中，你可以给现存的接口方法表示的行为附加额外动作：
```Go
package main

import "io"
import "os"
import "fmt"

type echoer struct{
    io.Reader //嵌套一个接口
}

func (e * echoer) Read(p []byte) (int, error) {  // 实现这个接口
    amount, err := e.Reader.Read(p)    // 实施原接口的行为
    if err == nil {             // 附加额外的打印动作
        fmt.Printf("Overridden read %d bytes:%s\n",amount,p[:amount])
    }
    return amount,err
}

func readUpToN(r io.Reader, n int) {
    buf := make([]byte,n)
    amount, err := r.Read(buf[:])
    if err == nil {
        fmt.Printf("Read %d bytes:%s\n",amount,buf[:amount])
    }
}

func main(){
    readUpToN(os.Stdin,3)

    var replacement echoer
    replacement.Reader = os.Stdin

    readUpToN(&replacement,3)
}
```
在这个例子种，被嵌套的接口是`io.Reader`。`os.Stdin`对象是实现该接口的标准输入对象。第一次调用`readUpToN`将读取并显示3个字节。第二次调用`readUpToN`使用了内嵌`os.Stdin`的`echoer`实例。输出如下：
```
ericu@luit:~$ echo 'ABCDEF' | ~/go/bin/embedinterface 
Read 3 bytes:ABC
Overridden read 3 bytes:DEF
Read 3 bytes:DEF
```
不能嵌套包含一个指向接口的指针[^1]，像下面这样的做法：
```Go
type echoer struct{
    *io.Reader //Embedding an interface by pointer
}
```
将产生如下编译错误：
```
../go/src/hydrogen18.com/embedinterface/main.go:8: embedded type cannot be a pointer to interface
```

## 结论
Go提供了嵌套机制来代替传统面向对象编程语言的继承。我觉得它在没有增加语言复杂度的情况下提供了恰当的灵活性。可能要到完成整个项目我才能弄清自己对Go的感受，但目前的经历已经使我受益匪浅。


## 附录
译注：本部分内容为补充参考，非原文内容。
### 匿名域的隐式名称
匿名域都具有隐式的名称，即类型名。所以：
1. 结构体中不能包含两个同类型的匿名域
2. 可以借助类型名来参引子域，比如
```Go
type Point struct {
    X, Y int
}
type Circle struct {
    Point
    Radius int
}
type Wheel struct {
    Circle
    Spokes int
}

var w Wheel
w.Circle.Center.X = 8  //等同于w.X = 8
w.Circle.Center.Y = 8  //等同于w.Y = 8
w.Circle.Radius = 5    //等同于w.Radius = 5
w.Spokes = 20
```
如果上面中间类型（Circel和Center）都没有被导出，就不能在包的外部引用他们，比如w.circle.center.X会导致编译错误，但w.X仍然是合法的！

### 指针类型的匿名域
不能在结构体中嵌套匿名域形式的指针，来指向接口或者指针，比如：
```Go
type MyIface interface {
    Abs()
}

type pInt *int

type MyStruct struct {
    *pInt
    *MyIface
}
```
如果编译将会报错：
```
prog.go:11: embedded type cannot be a pointer
prog.go:12: embedded type cannot be a pointer to interface
```
因为**匿名域之所以存在的根本原因，在于它有助于提升域和方法（The whole point of anonymous fields is that methods get promoted.），简化对被嵌套域的子域或方法的访问。**

但接口本身是没有域和方法的（interfaces don't have methods），因此指向接口的指针无助于提升，也就没有意义了。有人可能会争辩，使用&lt;interfaceName&gt;.&lt;func&gt;可以基于动态派发调用具体类型的方法。的确如此，但动态派发发生在runtime，而上面的promote发生在编译时。

### 接口没有方法
接口只是提供了方法的描述，但并没有定义方法本身。
在Golang中，接口的内部表示也是一种结构体
```Go
type iface struct { // 16 bytes
	tab  *itab
	data unsafe.Pointer
}

type itab struct { // 32 bytes
	inter *interfacetype
	_type *_type        // 与inter一起表示具体类型 
	hash  uint32        // 对类型的hash值，当类型转换时用于快速判断转换的是否一致合法
	_     [4]byte
	fun   [1]uintptr    // 原始的指针，指向用于动态派发的虚函数表数组
}
```
一个没有被赋值的接口，其fun字段指向的虚函数组表是空的，只有在运行时才会动态注入。

### 实现接口的类型
实现接口的类型一定要区分是值类型，还是指针类型。比如：
```Go
type Abser interface {
	Abs() float64
}

type Vertex struct {
	X, Y float64
}

func (v *Vertex) Abs() float64 {
	return math.Sqrt(v.X*v.X + v.Y*v.Y)
}

func main() {
	var a Abser
    v := Vertex{3, 4}

    // In the following line, v is a Vertex (not *Vertex)
	// and does NOT implement Abser.
    a = v
    
    fmt.Println(a.Abs())
}
```
在这个例子中，我们从
```Go
func (v *Vertex) Abs() float64 {
	return math.Sqrt(v.X*v.X + v.Y*v.Y)
}
```
才能看出，不是Vertex而是&#42;Vertex实现了接口Abser！Vertex是值类型，而&#42;Vertex是指针类型。在Go中，指针类型本身就是用含有特定类型信息的一个结构体表示，其操作也是类型受限的（比如&#42;Vertex类型的变量v不能执行v++操作），且完全不同于其指向的类型，比如本例中&#42;Vertex完全不同于Vertex类型。但在C或者C++中指针就是一个单纯的地址，因此也支持自增操作。Go中也有类似的对象，那就是uintptr，但其本身实际是一个无符号的值类型，使用上也有非常多的限制，具体可参考[《理解Golang的uintptr》](https://golangbyexample.com/understanding-uintptr-golang/)