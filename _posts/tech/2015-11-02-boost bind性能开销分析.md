---
layout: post
title: boost bind性能开销分析
category: 杂七杂八
tags: boost bind
---

```
#include <boost/bind.hpp>
#include <boost/function.hpp>
#include <boost/scoped_ptr.hpp>
#include <Windows.h>
#include <stdio.h>
#define MAX_LOOP 1000*1000*8

typedef boost::function1<void, int> Funtor;

struct TBase{
	virtual int DoSomething(int i) = 0;
};

struct TChiled : public TBase
{
	int DoSomething(int i){
		Count(i);
		return 0;
	}

	int DoSomethingNotVitrual(int i){
		Count(i);
		return 0;
	}

	int Count(int i){
		i++;
		i *= 2;
		i /= 4;
		return i;
	}
};

void TestMemberFunction(){
	TChiled* pChiled = new TChiled();
	DWORD nBeginTime = GetTickCount();
	for (int i=0; i<MAX_LOOP; i++)
	{
		pChiled->DoSomethingNotVitrual(i);
	}
	
	printf("Member function spend time : %d ms\n", GetTickCount() - nBeginTime);
}

void TestVirtualFunction(){
	TBase* pBase = new TChiled();
	int nBeginTime = GetTickCount();
	for (int i=0; i<MAX_LOOP; i++)
	{
		pBase->DoSomething(i);
	}

	printf("virtual function spend time : %d ms\n", GetTickCount() - nBeginTime);
}

void TestBoostBind(){
	TChiled* pChiled = new TChiled();
	Funtor functor = boost::bind(&TChiled::DoSomethingNotVitrual, pChiled, _1);
	int nBeginTime = GetTickCount();
	for (int i=0; i<MAX_LOOP; i++)
	{
		functor(i);
	}
	
	printf("boost bind spend time : %d ms\n", GetTickCount() - nBeginTime);
}

void TestBoostBindWithFunctor(){
	TChiled* pChiled = new TChiled();
	int nBeginTime = GetTickCount();
	for (int i=0; i<MAX_LOOP; i++)
	{
		Funtor functor = boost::bind(&TChiled::DoSomethingNotVitrual, pChiled, _1);
		functor(i);
	}
	
	printf("boost bind with a new functor spend time : %d ms\n", GetTickCount() - nBeginTime);
}

void TestVirtualFunctionWithNewObject(){
	int nBeginTime = GetTickCount();
	for (int i=0; i<MAX_LOOP; i++)
	{
		TBase* pBase = new TChiled();
		pBase->DoSomething(i);
		delete pBase;
	}
	
	printf("virtual function with new object spend time : %d ms\n", GetTickCount() - nBeginTime);
}

void TestVirtualFunctionWithNewObjectAndScopePtr(){
	int nBeginTime = GetTickCount();
	for (int i=0; i<MAX_LOOP; i++)
	{
		TBase* pBase = new TChiled();
		boost::scoped_ptr<TBase> scoptPtr(pBase);
		scoptPtr->DoSomething(i);
	}
	
	printf("virtual function with new object and scope ptr spend time : %d ms\n", GetTickCount() - nBeginTime);
}

int main()
{
	TestMemberFunction();
	TestVirtualFunction();
	TestBoostBind();
	TestBoostBindWithFunctor();

	printf("\n");
	TestVirtualFunction();
	TestVirtualFunctionWithNewObject();
	TestVirtualFunctionWithNewObjectAndScopePtr();
	return 0;
}
```

release下编译时候选择不同的代码选项得到不同的结果。
Maximize Speed执行结果

```
Member function spend time : 0 ms
virtual function spend time : 32 ms
boost bind spend time : 62 ms
boost bind with a new functor spend time : 156 ms

virtual function spend time : 16 ms
virtual function with new object spend time : 608 ms
virtual function with new object and scope ptr spend time : 624 ms
```

Default 执行结果

```
Member function spend time : 63 ms
virtual function spend time : 78 ms
boost bind spend time : 483 ms
boost bind with a new functor spend time : 1295 ms

virtual function spend time : 78 ms
virtual function with new object spend time : 795 ms
virtual function with new object and scope ptr spend time : 905 ms
```

用的VC6有点老，还有个优化错误在Maximize Speed时候直接调用成员函数花费时间为0。
上述结果可以看出（主要依据Default）：  
1）使用成员函数的的调用时间最短，其次是虚函数的方法，这两者相差很小。  
2）多一层封装到functor中，在调用有一定的开销。  
3）每次都生成一个临时的functor后，开销也有一定增长。每次都分配一个functor的方式优化差异很大，主要是因为在构造functor的过程中构造析构的次数很多（在栈上），如果不优化可能会有将近十次的拷贝构造操作优化后次数会大大减少。  
4）使用scoptr有轻微的开销，可以忽略。  
5）每次在堆上分配对象的开销很大，而且优化空间很小。  

上面的bind参数只是简单的int，在传递一个复杂参数的时候bind的开销会进一步变大。当参数过多时候甚至会分配动态内存，开销明显。

总体上说，使用boost的bind有一定开销，如果不是hot path，那么其实影响不大。但是，在hot path中，尽量少的使用智能指针、new（不仅开销大，还会造成内存碎片）、bind。甚至虚指针都需要少用（虚指针可能伴随着对象new操作），最好直接调用对象的成员方法。
