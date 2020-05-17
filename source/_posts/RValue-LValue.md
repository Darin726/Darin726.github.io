---
title: c++ 右值引用
date: 2020-05-17 01:21:18
categories:
- 基础知识
tags:
- c++
---

## 左值 & 右值
左值和右值，其实这个概念很一直都在，简单回顾一下.
```c++
int lValue = 10;
```
在上面的代码中 `lValue` 是左值, `10` 是右值; 简单来区分就是 `=` 左边的是左值, 右边的是右值. 

右值是不能再次被赋值的, 譬如 `10 = 1` 是一个非法的操作. 一个左值也可以转变成右值; 譬如
```c++
int lValue = 10;
int lValue2 = lValue;
```
`lValue2` 是左值, `lValue` 变成右值, 此时操作 `lValue2` 不会对影响 `lValue` 的值. 因为 `lValue2` 是 `lValue` 的一个**拷贝**;
简单的 int 拷贝消耗的时间和空间是比较小的, 但对于大型的类消耗就大了, 譬如假设有很长一段字符串.

```c++
std::string lString = "Hello Darin";
std::string lString2 = lString;
lString2[2] = 'L';
std::cout << "lString = " << lString
          << " & "
          << "lString2 = " << lString2 << std::endl;
```
`lString2 = lString` 执行之后, 相当于将 `lString` 内存的内容拷贝到 `lString2`, 我们将 `lString2` 的内容修改也不会影响 `lString` 的内容.


![e66d093753756658168ed247200ac46b.png](http://images.qianlicao.cn/hexo/images/RValue/8c8b4ec2e7244fd09093564411ea617d.png)


## 左值引用 & 右值引用

继续上面的例子, 我们申明了一个新的 `std::string &lString3 = lString;`

```c++
std::string &lString3 = lString;
lString3[2] = 'L';
std::cout << "lString = " << lString
          << " & "
          << "lString3 = " << lString3 << std::endl;
```
`lString3` 其实就像是 `lString` 的别名, 对 `lString3` 内容的任何操作都会影响到 `lString`上, 譬如上面的例子, lString 也会变成 "He**L**lo Darin" ， 这就是**左值引用**。 

```c++
int &getValue_l_ref() {
  static int a = 10;
  return a;
}
getValue_l_ref() = 20;
```

例如上面的左值引用, `getValue_l_ref()` 也是一个左值, 如果不加 `&` 号, 会有什么样的效果?

**右值引用** 是在 C++11 引入的新特性,  是为了解决以下问题

1. 实现 `move` 语义
2. 完美转发

**右值引用** 是什么, 例如有一个 **class MyString**, `MyString&` 是左值引用, 那么为了区分 `MyString&&` 就是一个右值引用, 

```c++
void func(MyString& str); // 左值引用重载
void func(MyString&& str); // 右值应用重载

MyString str;
MyString createStr();

func(str); // 入参是左值, 所以调用上述左值引用
func(createStr()); // 参数是右值, 调用上述右值引用
```

## 构造函数 & 拷贝构造函数

为了能够更容易理解后面的内容, 先熟悉一下 c++ 的构造函数和拷贝构造函数.

我们定义一个 `MyString` 类, 后面我们会持续使用到它.

```c++
class MyString {
public:
    MyString() {
        std::cout << "default_constructor is called" << std::endl;
        buffer_ = nullptr;
        size_ = 0;
    };

    MyString(const char *str) {
        std::cout << "char_constructor is called" << std::endl;
        size_ = strlen(str);
        buffer_ = new char[size_ + 1];
        memcpy(buffer_, str, size_);
        buffer_[size_] = 0;
    }
public:
    char *buffer_;
    int size_;
};
```

接着, 我们来使用它

```c++
MyString temp("Hello Darin");
MyString str = temp;
std::cout << temp.buffer_ << std::endl;
std::cout << str.buffer_ << std::endl;
```

OK 你会发现可以正常运行. 但上面的代码是不完整的，`buffer_` 没有释放会内存泄漏的，若我们完善 `MyString` 加上析构函数

```c++
~MyString() {
  size_ = 0;
  delete[] buffer_;
  std::cout << "destructor is called" << std::endl;
}
```

此时运行就会出问题，问题如下。

```shell
(18057,0x10ed435c0) malloc: *** error for object 0x7fc509c029b0: pointer being freed was not allocated
(18057,0x10ed435c0) malloc: *** set a breakpoint in malloc_error_break to debug
```

原因是因为, 默认的拷贝构造函数, 是将 `buffer_` 指针的地址拷贝过去

如图 `debug` 所示, `str` 和 `temp` 内 `buffer_` 地址一模一样
![be6a213db3b28e08fe878f34e50612ce.png](http://images.qianlicao.cn/hexo/images/RValue/a3eab4dbad544100817935291f147b9b.png)


当 `temp` 释放` buffer_` 指向的内容时, `str` 内部的 `buffer_` 指向的内容也就释放了, 再次释放数组就会出现上述错误. 

![752f3bae8e0948625d608b057d78ee86.png](http://images.qianlicao.cn/hexo/images/RValue/9429bc800a474ff3abaf8059852ea8c0.png)

这明显不是我们期望的, 所以这里就需要自行实现一个深度拷贝构造函数. 

```c++
MyString(MyString &lValueString) {
  std::cout << "l_constructor is called" << std::endl;
  size_ = lValueString.size_;
  buffer_ = new char[size_ + 1];
  for (int i = 0; i < size_; i++) {
    buffer_[i] = lValueString.buffer_[i];
  }//memcpy(buffer_, lValueString.buffer_, size_);
  buffer_[size_] = 0;
}
```

实现完之后会发现, 就不会出现上述问题，达到了我们的期望，我们在调用一个函数时, 传入的参数会发生一次拷贝. `void func(MyString copy)`

```c++
MyString temp("Hello Darin");
func(temp);
```

```shell
char_constructor is called
l_constructor is called
```

但如果将函数改为 `void func(MyString& copy)` l_constructor 就不会被调用, 完美的减少了一次拷贝的操作. 

值得一提的是左值引用一般要和 const 进行搭配, 否则在 func 里我们可以操作原值.


![fd3f302ef5d9afdc90aae86850b66666.png](http://images.qianlicao.cn/hexo/images/RValue/a36f1bdab2754cfd9f4302d7baf8260a.png)

继续说构造函数, 在上述场景下, 拷贝构造函数还是会进行一次深度拷贝, 如果不需要深度拷贝该怎么办呢, 这个时候右值引用就可以出场了. 

```c++
MyString(MyString &&rValueString) {
  std::cout << "r_constructor is called" << std::endl;
  size_ = rValueString.size_;
  buffer_ = rValueString.buffer_;
  rValueString.buffer_ = nullptr;
  rValueString.size_ = 0;
}
```

我们定义一个右值引用的构造函数, 当入参是右值时, 入参左边两个 **"&"**, 直接将指针地址赋值过去, 能够大大减少拷贝的操作. 

在例如, 我们重写 MyString 的 equal 操作

```c++
MyString operator =(const MyString &copy) {
  std::cout << "copy equal is called" << std::endl;
  size_ = copy.size_;
  buffer_ = new char[size_ + 1];
  for (int i = 0; i < size_; i++) {
    buffer_[i] = copy.buffer_[i];
  }//memcpy(buffer_, lValueString.buffer_, size_);
  buffer_[size_] = 0;
  return *this;
}

MyString operator =(MyString &&copy) {
  std::cout << "right copy equal is called" << std::endl;
  size_ = copy.size_;
  buffer_ = copy.buffer_;
  copy.buffer_ = nullptr;
  copy.size_ = 0;
  return *this;
}
```

```c++
template<class T>
void swapLValue(T &a, T &b) {
    T tmp(a);
    a = b;
    b = tmp;
}

template<class T>
void swapRValue(T &a, T &b) {
    T tmp(std::move(a));
    a = std::move(b);
    b = std::move(tmp);
}
```

如上述代码所示, `swapRValue` 的效率明显大于 `swapLValue` 的效率, 只需要交换地址即可，不需要做深度拷贝。

似乎解释了上面右值引用的第一个用发，实现 move 转意.

**在 c++ 11, `std::move` 可以将一个 `LValue` 转换成一个 `RValue`, 要达到很高的转化效率, 前提是 class T 实现了右值的 `operator =` 和 右值拷贝构造函数, 否则拷贝性能还是和原来一样.** 

**总结: 右值引用能够避免深度拷贝, 而 std::move 能够将一个左值引用转换成右值引用**

## 右值引用是右值吗?

```c++
MyString(MyString &&rValueString) {
  MyString x = rValueString; // 这里调用的是什么类型的构造函数
  std::cout << "r_constructor is called" << std::endl;
  size_ = rValueString.size_;
  buffer_ = rValueString.buffer_;
  rValueString.buffer_ = nullptr;
  rValueString.size_ = 0;
}
```

例如上述代码, `MyString x = rValueString;` 实际调用的是 lValue 的构造函数, 所以 `rValueString` 是一个 lValue, 左值.

先写到这里, 未完待续.