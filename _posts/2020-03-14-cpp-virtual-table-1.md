---
title: Cpp 面向对象基础 - 多态
tags: cpp
---
<!-- toc -->

- [1编译时多态](#1编译时多态)
  * [1.1函数重载](#11函数重载)
  * [1.2操作符重载](#12操作符重载)
- [2运行时多态](#2运行时多态)
  * [2.1虚函数](#21虚函数)
  * [2.2虚表](#22虚表)

<!-- tocstop -->

# C++多态

![Polymorphism-1.jpg](https://media.geeksforgeeks.org/wp-content/uploads/20190705113259/Polymorphism-1.jpg)

## 1编译时多态
### 1.1函数重载
- 参数个数不同
- 参数类型不同,或参数顺序不同
- **函数名相同,返回值不同不是重载,编译出错:重定义**
```cpp
/// case 1
void func(){}
int func(){}
/* error: ambiguating new declaration of 'int func()' */
/// case 2
void func1(int ,int){}
void func1(int ,int ,int c = 0){}
int main(){
    func1(1,2);
}
/* error: call of overloaded 'func(int, int)' is ambiguous */
```
### 1.2操作符重载
- 重载运算符: +/-/x/÷/>/< 等

## 2运行时多态

### 2.1虚函数
- 只能使用指针,**和引用**

  > 想了想这里为什么只能用指针和引用呢?  
 
	```c++
	class Base{
	public:
	    virtual void  P(){cout<<"Base"<<endl;}
	};
	class Derived: public Base{
	public:
	    void P() {cout << "Derived"<<endl;}
	};
	int main(){
	    auto d = new Derived;
	    ((Base)*d).P(); // object slicing, 子类属性,函数信息丢失
	    ((Base *)d)->P();
	}
	/* 输出: 
	Base
	Derived
	*/
	```
在父类构造函数及析构函数加入输出提示,见证奇迹的时刻,准备!!!!

	```c++
	class Base{
	public:
	    virtual void  P(){cout<<"Base"<<endl;}
	    void dispatch(){
	        P();
	    }
	    Base(){
	        cout<<"Base Constructed!\n";
	    }
	    ~Base(){
	        cout<<"Base Deconstructed!\n";
	    }
	};
	class Derived: public Base{
	public:
	    void P() override {cout << "Derived"<<endl;}
	};
	int main(){
	    auto d = new Derived;
	    ((Base)*d).dispatch();
	    ((Base &)*d).dispatch();
	}
	----------------------------------
	输出:
	----------------------------------
	Base Constructed!
	Base
	Base Deconstructed!
	Derived
	---------------------------------
	BOOOOOMMM.......   (其实就是一个强制类型转换,调用构造函数
	看到在子类对象转化父类时,会进行隐式基类构造函数,并且调用结束后立即就进行了析构(并未
	等到主函数结束才进行),此时子类多态已经丢失,可以通过指针及**引用**进行调用 ^_^
	---------------------------------
	```

- 运行时多态,动态绑定(dynamic binding) 
<br>定义一个基类成员函数`dispatch()`下调用虚函数`P()`,子类重载虚函数`P()`,子类调用`dispatch()`

	```c++
	class Base{
	public:
	    virtual void  P(){cout<<"Base"<<endl;}
	    void dispatch(){
	        P();
	    }
	};
	class Derived: public Base{
	public:
	    void P() override {cout << "Derived"<<endl;}
	};
	int main(){
	    auto d = new Derived;
	    d->dispatch();
	    ((Base)*d).dispatch();
	}
	/* output: 
	Derived
	Base
	*/
	```

### 2.2虚表
- 概念
  1. 虚函数,C++的动态调用,实现运行时多态
  1. 每个包含虚函数的类会生成一个虚函数表(`virtual table`),vtable在编译时生成
  2. 虚函数为类所有,不属于对象.但具有虚函数的类实体会包含一个指针指向此类对应的虚表
  3. 虚表中存储为函数地址,最开始为`top_offset`四字节,`typeinfo`四字节,剩下未函数指针列表,每个占四字节,

	```c++
	// https://www.cnblogs.com/sunbines/p/9121123.html
	class Base{
	public:
	    virtual void  P(){cout<<"Base"<<endl;}
	    void dispatch(){
	        P();
	    }
	};
	class Derived: public Base{
	public:
	    void P() override {cout << "Derived"<<endl;}
	};
	class NonVirtualClass{};
	int main(){
	    Base b;
	    auto d = new Derived;
	    d->dispatch();
	    cout<<sizeof(b)<<"\t" << sizeof(*d)<<endl;
	    void (*f) ();
	    f = (void(*)())*(int*)*(int*)(&b);
	    f();
	    return 0;
	}
	-------------------------------------------
	输出:
	-----------------------------------------
	Derived
	4       4       1
	Base
	------------------------------------------
	```

输出里看到含有虚函数(包括继承)的类内存占用为4个字节(使用MingW32-make编译),未包含虚函数的`NonVirtualClass`类(不包含成员变量)所占空间为1个字节.
点击运行,在`return 0`处加上断电,使用GDB调试

![](http://img.shargeebar.cc/20200312140940.png)
- - - 
![](http://img.shargeebar.cc/20200312140957.png)

在CLion Debug 下看到实例 `b` 和`d`中无法继续展开,既然啥都没有,那为什么还会占用4个字节呢??? ^_^
<br> 在GDB中输入 `p b` 和 `p *d` 打印对象信息,看到其中还有一个隐藏的`_vptr`地址指针,这就是虚表指针.
输入`x/200xb 0x405168`输出200字节内存数据,以`0x405218` 为起始点输出200字节的数据,其中包括`_vptr.Base = 0x405218 <vtable for Base+8>` 所指向的内存数据.

![x/200xb](http://img.shargeebar.cc/20200311231538.png)

|指针地址|指向值|指向地址说明|
|:-|:-|:-:|
|`0x00405218`|`0x00403cb8`|Base::P()|
|`0x00405224`|`0x00403d20`|Derive::P()|

![](http://img.shargeebar.cc/20200311232855.png)

上图中红色部分为内存函数地址,其中`0x405218 <vtable for Base+8>`处为`Base::P()`的函数地址,`0x405224 <vtable for Derived+8>`为`Derived::P()`函数地址.在代码中将`Base::P()` 赋值给`void(*f)()`函数指针,因此调用`f()` 结果中输出`Base`,见上方代码. 输出`f`函数信息对照`info symbol 0x403cb8` ,信息匹配
蓝色部分指向类信息地址
```c++
    f = (void(*)())*(int*)*(int*)(&b);
    //每个包含虚函数的类对象都包含一个虚表指针,但是无法像成员变量直接获取
    //通过取地址获取对象 b 的地址,随后强制转化为 int* 指针,在解引用获取到前四个字节数据 *(int*)(&b)
    //转化为int 的值即为 0x403cb8, 它指向该类的虚表内存地址起始
    //将得到的地址再一次强制转化为 int* ,对其解引用就可以赋值给 void()类型的函数指针 f.   *(int*)*(int*)(&b)
    //因此调用 f() 就输出了 Base
```
![](http://img.shargeebar.cc/20200312094540.png)
- - - -
![](http://img.shargeebar.cc/20200312094707.png)
