# 2.5. C++ 类包装

CGO是C语言和Go语言之间的桥梁，原则上无法直接支持C++的类。CGO不支持C++语法的根本原因是C++至今为止还没有一个二进制接口规范(ABI)。一个C++类的构造函数在编译为目标文件时如何生成链接符号名称到方法在不太平台甚至是C++到不同版本之间都是不一样的。但是C++的最大优势是兼容C语言，我们可以通过增加一组C语言函数接口作为C++类和CGO之间的桥梁，这样就可以间接地实现C++和Go之间的互联。当然，因为CGO只支持C语言中值类型的数据类型，我们是无法直接使用C++的引用参数等特性的。

## C++ 类到 Go 语言对象

要实现C++类到Go语言对象的包装需要经过以下几个步骤：首先是用纯C函数接口包装该C++类；其次是通过CGO将纯C函数接口映射到Go函数；最后是做一个Go包装对象，将C++类到方法用Go对象的方法实现。

### 准备一个 C++ 类

为了演示简单，我们基于`std::string`做一个最简单的缓存对象MyBuffer。除了构造函数和析构函数之外，只有两个成员函数分别是返回底层的数据指针和缓存的大小。因为是二进制缓存，我们可以在里面中放置任意数据。

```c++
// my_buffer.h
#include <string>

struct MyBuffer {
	std::string* s_;

	MyBuffer(int size) {
		this->s_ = new std::string(size, char('\0'));
	}
	~MyBuffer() {
		delete this->s_;
	}

	int Size() const {
		return this->s_->size();
	}
	char* Data() {
		return (char*)this->s_->data();
	}
};
```

我们在构造函数中指定缓存的大小并分配空间，在使用完之后通过哦析构函数释放内部分配到内存空间。下面是简单的使用方式：

```c++
int main() {
	auto pBuf = new MyBuffer(1024);

	auto data = pBuf->Data();
	auto size = pBuf->Size();

	delete pBuf;
}
```

为了方便向C语言接口过度，我们故意没有定义C++的拷贝构造函数。我们必须以new和delete来分配和释放缓存对象，而不能以值风格的方式来使用。

### 用纯C函数接口封装 C++ 类

如果要将上面的C++类用C语言函数接口封装，我们可以从使用方式入手。我们可以将new和delete映射为C语言函数，将对象的方法也映射为C语言函数。

在C语言中我们期望MyBuffer类可以这样使用：

```c
int main() {
	MyBuffer* pBuf = NewMyBuffer(1024);

	char* data = MyBuffer_Data(pBuf);
	auto size = MyBuffer_Size(pBuf);

	DeleteMyBuffer(pBuf);
}
```

先从C语言接口用户的角度思考需要什么样的接口，然后创建 `my_buffer_capi.h` 头文件接口规范：

```c++
// my_buffer_capi.h
typedef struct MyBuffer_T MyBuffer_T;

MyBuffer_T* NewMyBuffer(int size);
void DeleteMyBuffer(MyBuffer_T* p);

char* MyBuffer_Data(MyBuffer_T* p);
int MyBuffer_Size(MyBuffer_T* p);
```

然后就可以基于C++的MyBuffer类定义这些C语言包装函数。我们创建对应的`my_buffer_capi.cc`文件如下：

```c++
// my_buffer_capi.cc

#include "./my_buffer.h"

extern "C" {
	#include "./my_buffer_capi.h"
}

struct MyBuffer_T: MyBuffer {
	MyBuffer_T(int size): MyBuffer(size) {}
	~MyBuffer_T() {}
};

MyBuffer_T* NewMyBuffer(int size) {
	auto p = new MyBuffer_T(size);
	return p;
}
void DeleteMyBuffer(MyBuffer_T* p) {
	delete p;
}

char* MyBuffer_Data(MyBuffer_T* p) {
	return p->Data();
}
int MyBuffer_Size(MyBuffer_T* p) {
	return p->Size();
}
```

因为头文件`my_buffer_capi.h`是用于CGO，必须是采用C语言规范的名字修饰规则。在C++源源文件包含时需要用`extern "C"`语句说明。另外MyBuffer_T的实现只是从MyBuffer继承的类，这样可以简化包装代码的实现。同时，和CGO通信时必须通过`MyBuffer_T`指针，我们无法将具体的实现暴漏给CGO，因为实现中包含了C++特有的语法，CGO无法识别C++特性。

将C++类包装为纯C接口之后，下一步的工作就是将C函数转为Go函数。

### 将纯C接口函数转为Go函数

将纯C函数包装为对应的Go函数的过程比较简单。需要注意的是，因为我们的包中包含C++11的语法，因此需要通过`#cgo CXXFLAGS: -std=c++11`打开C++11的选项。

```go
// my_buffer_capi.go

package main

/*
#cgo CXXFLAGS: -std=c++11

#include "my_buffer_capi.h"
*/
import "C"

type cgo_MyBuffer_T C.MyBuffer_T

func cgo_NewMyBuffer(size int) *cgo_MyBuffer_T {
	p := C.NewMyBuffer(C.int(size))
	return (*cgo_MyBuffer_T)(p)
}

func cgo_DeleteMyBuffer(p *cgo_MyBuffer_T) {
	C.DeleteMyBuffer((*C.MyBuffer_T)(p))
}

func cgo_MyBuffer_Data(p *cgo_MyBuffer_T) *C.char {
	return C.MyBuffer_Data((*C.MyBuffer_T)(p))
}

func cgo_MyBuffer_Size(p *cgo_MyBuffer_T) C.int {
	return C.MyBuffer_Size((*C.MyBuffer_T)(p))
}
```

为了区分，我们在Go中的每个类型和函数名称前面增加了`cgo_`前缀，比如cgo_MyBuffer_T是对应C中的MyBuffer_T类型。

为了处理简单，在包装纯C函数到Go函数时，除了cgo_MyBuffer_T类型本书，我们对输入参数和返回值的基础类型依然是用的C语言的类型。

### 包装为Go对象

在将纯C接口包装为Go函数之后，我们就可以基于包装的Go函数很容易地构造出Go对象来。因为cgo_MyBuffer_T是从C语言空间导入的类型，它无法定义自己的方法，因此我们构造了一个新的MyBuffer类型，里面的成员持有cgo_MyBuffer_T指向的C语言缓存对象。

```go
// my_buffer.go

package main

import "unsafe"

type MyBuffer struct {
	cptr *cgo_MyBuffer_T
}

func NewMyBuffer(size int) *MyBuffer {
	return &MyBuffer{
		cptr: cgo_NewMyBuffer(size),
	}
}

func (p *MyBuffer) Delete() {
	cgo_DeleteMyBuffer(p.cptr)
}

func (p *MyBuffer) Data() []byte {
	data := cgo_MyBuffer_Data(p.cptr)
	size := cgo_MyBuffer_Size(p.cptr)
	return ((*[1 << 31]byte)(unsafe.Pointer(data)))[0:int(size):int(size)]
}
```

同时，因为Go语言的切片本身含有长度信息，我们将cgo_MyBuffer_Data和cgo_MyBuffer_Size两个函数合并为`MyBuffer.Data`方法，它返回一个对应底层C语言缓存空间的切片。

现在我们可以很容易在Go语言中使用包装后的缓存对象了（底层是基于C++的`std::string`实现）：

```go
package main

//#include <stdio.h>
import "C"
import "unsafe"

func main() {
	buf := NewMyBuffer(1024)
	defer buf.Delete()

	copy(buf.Data(), []byte("hello\x00"))
	C.puts((*C.char)(unsafe.Pointer(&(buf.Data()[0]))))
}
```

例子中，我们创建了一个1024字节大小的缓存，然后通过copy函数向缓存填充了一个字符串。为了方便C语言字符串函数处理，我们在填充字符串的默认用'\0'表示字符串结束。最后我们直接获取缓存的底层数据指针，用C语言的puts函数打印缓存的内容。

## Go 语言对象到 C++ 类

要实现Go语言对象到C++类的包装需要经过以下几个步骤：首先是将Go对象映射为一个id；然后基于id导出对应的C接口函数；最后是基于C接口函数包装为C++对象。

### 构造一个Go对象

为了便于演示，我们用Go语言构建了一个Person对象，每个Person可以有名字和年龄信息：

```go
package main

type Person struct {
	name string
	age  int
}

func NewPerson(name string, age int) *Person {
	return &Person{
		name: name,
		age:  age,
	}
}

func (p *Person) Set(name string, age int) {
	p.name = name
	p.age = age
}

func (p *Person) Get() (name string, age int) {
	return p.name, p.age
}
```

Person对象如果想要在C/C++中访问，需要通过cgo导出C接口来访问。

### 导出C接口

我们前面仿照C++对象到C接口的过程，也抽象一组C接口描述Person对象。创建一个`person_capi.h`文件，对应C接口规范文件：

```c
// person_capi.h
#include <stdint.h>

typedef uintptr_t person_handle_t;

person_handle_t person_new(char* name, int age);
void person_delete(person_handle_t p);

void person_set(person_handle_t p, char* name, int age);
char* person_get_name(person_handle_t p, char* buf, int size);
int person_get_age(person_handle_t p);
```

然后是在Go语言中实现这一组C函数。

需要注意的是，通过CGO导出C函数时，输入参数和返回值类型都不支持const修饰，同时也不支持可变参数的函数类型。同时如内存模式一节所述，我们无法在C/C++中直接长期访问Go内存对象。因此我们前一节所讲述的技术将Go对象映射为一个整数id。

下面是`person_capi.go`文件，对应C接口函数的实现：

```go
// person_capi.go
package main

//#include "./person_capi.h"
import "C"
import "unsafe"

//export person_new
func person_new(name *C.char, age C.int) C.person_handle_t {
	id := NewObjectId(NewPerson(C.GoString(name), int(age)))
	return C.person_handle_t(id)
}

//export person_delete
func person_delete(h C.person_handle_t) {
	ObjectId(h).Free()
}

//export person_set
func person_set(h C.person_handle_t, name *C.char, age C.int) {
	p := ObjectId(h).Get().(*Person)
	p.Set(C.GoString(name), int(age))
}

//export person_get_name
func person_get_name(h C.person_handle_t, buf *C.char, size C.int) *C.char {
	p := ObjectId(h).Get().(*Person)
	name, _ := p.Get()

	n := int(size) - 1
	bufSlice := ((*[1 << 31]byte)(unsafe.Pointer(buf)))[0:n:n]
	n = copy(bufSlice, []byte(name))
	bufSlice[n] = 0

	return buf
}

//export person_get_age
func person_get_age(h C.person_handle_t) C.int {
	p := ObjectId(h).Get().(*Person)
	_, age := p.Get()
	return C.int(age)
}
```

在创建Go对象后，我们通过NewObjectId将Go对应映射为id。然后将id强制转义为person_handle_t类型返回。其它的接口函数则是根据person_handle_t所表示的id，让根据id解析出对应的Go对象。

### 封装C++对象

有了C接口之后封装C++对象就比较简单了。常见的做法是新建一个Person类，里面包含一个person_handle_t类型的成员对应真实的Go对象，然后在Person类的构造函数中通过C接口创建Go对象，在析构函数中通过C接口释放Go对象。下面是采用这种技术的实现：

```c++
extern "C" {
	#include "./person_capi.h"
}

struct Person {
	person_handle_t goobj_;

	Person(const char* name, int age) {
		this->goobj_ = person_new((char*)name, age);
	}
	~Person() {
		person_delete(this->goobj_);
	}

	void Set(char* name, int age) {
		person_set(this->goobj_, name, age);
	}
	char* GetName(char* buf, int size) {
		return person_get_name(this->goobj_ buf, size);
	}
	int GetAge() {
		return person_get_age(this->goobj_);
	}
}
```

包装后我们就可以像普通C++类那样使用了：

```c++
#include "person.h"

#include <stdio.h>

int main() {
	auto p = new Person("gopher", 10);

	char buf[64];
	char* name = p->GetName(buf, sizeof(buf)-1);
	int age = p->GetAge();

	printf("%s, %d years old.\n", name, age);
	delete p;

	return 0;
}
```

### 封装C++对象改进

在前面的封装C++对象的实现中，每次通过new创建一个Person实例需要进行两次内存分配：一次是针对C++版本的Person，再一次是针对Go语言版本的Person。其实C++版本的Person内部只有一个person_handle_t类型的id，用于映射Go对象。我们完全可以将person_handle_t直接当中C++对象来使用。

下面时改进后的包装方式：

```c++
extern "C" {
	#include "./person_capi.h"
}

struct Person {
	static Person* New(const char* name, int age) {
		return (Person*)person_new((char*)name, age);
	}
	void Delete() {
		person_delete(person_handle_t(this));
	}

	void Set(char* name, int age) {
		person_set(person_handle_t(this), name, age);
	}
	char* GetName(char* buf, int size) {
		return person_get_name(person_handle_t(this), buf, size);
	}
	int GetAge() {
		return person_get_age(person_handle_t(this));
	}
};
```

我们在Person类中增加类一个New静态成员函数，用于创建新的Person实例。在New函数中通过调用person_new来创建Person实例，返回的是`person_handle_t`类型的id，我们将其强制转型作为`Person*`类型指针返回。在其它的成员函数中，我们通过将this指针再反向转型为`person_handle_t`类型，谈话通过C接口调用对应的函数。

到此，我们就实现了将Go对象导出为C接口，然后基于C接口再包装为C++对象便于使用。
