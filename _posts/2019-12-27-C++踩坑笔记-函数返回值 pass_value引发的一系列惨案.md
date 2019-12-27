---
layout:     post
title:      函数返回值 pass-value引发的一系列惨案
subtitle:   传参、RAII
date:       2019-12-27
author:     BY ZYZ
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - C++踩坑笔记
    - RAII
    - effective C++
---
一个关于函数返回值是传值还是传引用的坑。

调试代码中返回一个TcpServer对象，由于TcpServer类并**没有禁用拷贝构造函数和赋值运算符**，使得getTcpServer()函数中返回的是一个TcpServer副本。

在使用完这个副本对象的时候，会调用该副本的析构函数。

在析构函数中，会等待StartThread这个线程结束（pthread_join）而非强制退出线程。

StartThread线程标识startThreadHandle也被拷贝至副本里，也就是说，这个副本的析构完成需要等到原件里的StartThread退出。这个StartThread的生命周期与软件的生命周期一样长，所以程序会一直阻塞在副本的析构处。

究其原因，是因为资源管理类TcpServer没有考虑对象的copying行为。当需要析构函数时，也需要考虑拷贝构造函数。一个好的RAII类，在使用析构函数时，如果不希望被拷贝，需要把其拷贝函数和赋值构造函数屏蔽，如果有拷贝行为存在，要么添加引用计数，要么进行深度拷贝。

>effective C++ item 14: 小心资源管理类的copying行为。
 + 复制RAII对象必须一并复制它所管理的资源，所以资源的copying行为决定RAII对象的copy行为
 + 普遍而常见的RAII class copying行为是：抑制copying、实行引用计数。

 后来，把getTcpServer()函数的返回该为返回TcpServer*，即解决该问题。

 >effective C++ item20: Prefer pass-by-reference-to-const to pass-by-value.宁以pass-by-reference-to-const替换pass-by-value

 我们不能保证类库的开发者能够遵守RAII类设计规范，所以改为返回指针。
 
 例如：
 ```
 #include <iostream>

using namespace std;

class A
{
public:
    A(int i):m_i(i)
    {
        cout<<"A constructor"<<endl;
    }

    A(const A&)
    {
        cout<<"A copy constructor"<<endl;
    }

    A operator=(const A& other)
    {
        cout<<"A assign constructor"<<endl;
        this.m_i = other.m_i;
        return *this;
    }
private:
    int m_i;
};

class single
{
public:
    static single* instance()
    {
        if(m_s==nullptr)
            m_s = new single();
        return m_s;
    }

    A getA()
    {
        return m_a;
    }

	void print()
	{
		cout<<"hello"<<endl;
	}

private:
    static single* m_s;
    single():m_a(0){}
    single(const single&)=delete;
    single operator=(const single&)=delete;

    A m_a;
};
single* single::m_s = nullptr;


int main()
{
	single::instance()->print();
    single::instance()->getA();
    return 0;
}
 ```
 这段代码运行时可以`getA()`看到返回的是m_a的副本，调用了拷贝构造函数。

 **最后，在返回引用或者指针时，也需要注意其有效性。**
