---
title: c++ 单例模式的实例自动销毁
date: 2015-03-18 15:47:44
categories: 设计模式
tags: [单利模式,c++,自动回收]
---

## 背景
前些日志看到一篇博文，关于C++单例模式下m_pinstance指向空间销毁问题，m_pInstance的手动销毁经常是一个头痛的问题，内存和资源泄露也是屡见不鲜，能否有一个方法，让实例自动释放。网上已经有解决方案(但是具体实现上表述不足，主要体现在自动析构未能正常运行)，那就是定义一个内部垃圾回收类，并且在Singleton中定义一个此类的静态成员。程序结束时，系统会自动析构此静态成员，此时，在此类的析构函数中析构Singleton实例，就可以实现m_pInstance的自动释放。
## 测试代码
```c++
#include <iostream>
using namespace std;

class Singleton
{
public:
	static Singleton *GetInstance()
	{
		if (m_Instance == NULL)
		{
			m_Instance = new Singleton();
			cout<<"get Singleton instance success"<<endl;
		}
		return m_Instance;
	}

private:
	Singleton(){cout<<"Singleton construction"<<endl;}
	static Singleton *m_Instance;

	// This is important
	class GC // 垃圾回收类
	{
	public:
		GC()
		{
			cout<<"GC construction"<<endl;
		}
		~GC()
		{
			cout<<"GC destruction"<<endl;
			// We can destory all the resouce here, eg:db connector, file handle and so on
			if (m_Instance != NULL)
			{
				delete m_Instance;
				m_Instance = NULL;
				cout<<"Singleton destruction"<<endl;
				system("pause");//不暂停程序会自动退出，看不清输出信息
			}
		}
	};
	static GC gc;  //垃圾回收类的静态成员

};

Singleton *Singleton::m_Instance = NULL;
Singleton::GC Singleton::gc; //类的静态成员需要类外部初始化，这一点很重要，否则程序运行连GC的构造都不会进入，何谈自动析构
int main(int argc, char *argv[])
{
	Singleton *singletonObj = Singleton::GetInstance();
	return 0;
}
```
## 运行结果
![这里写图片描述](http://img.blog.csdn.net/20170618224515650)

## 参考文献
[http://blog.csdn.net/hackbuteer1/article/details/7460019](http://blog.csdn.net/hackbuteer1/article/details/7460019)
