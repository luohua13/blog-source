---
title: 设计模式学习笔记（一）
date: 2019-10-21 18:00
tags: [design-pattern,c++]
categories: language
---

<!--more-->

## Before you begin

- [图解设计模式](https://design-patterns.readthedocs.io/zh_CN/latest/creational_patterns/creational.html)

- [三种工厂模式的C++实现](https://blog.csdn.net/silangquan/article/details/20492293)

## 简单工厂模式

### 模式定义
简单工厂模式(Simple Factory Pattern)：又称为静态工厂方法(Static Factory Method)模式，它属于类创建型模式。在简单工厂模式中，可以根据参数的不同返回不同类的实例。简单工厂模式专门定义一个类来负责创建其他类的实例，被创建的实例通常都具有共同的父类。

### 实例
```c++
#include <iostream> 
using namespace std; 
class AbstractProduct { 
    public: virtual ~AbstractProduct() {}
    virtual void Operation() = 0; 
}; 

class ProductA : public AbstractProduct { 
    public: void Operation() { cout << "ProductA" << endl; } 
}; 

class ProductB : public AbstractProduct { 
    public: void Operation() { cout << "ProductB" << endl; } 
}; 

class Factory { 
    public: AbstractProduct* createProduct(char product) { 
        AbstractProduct* ap = NULL; 
        switch(product) { 
            case 'A': ap = new ProductA(); break; 
            case 'B': ap = new ProductB(); break; 
        } 
        return ap; 
    } 
}; 

int main() { 
    Factory* f = new Factory(); 
    AbstractProduct* apa = f->createProduct('A'); apa->Operation(); // ProductA AbstractProduct* apb = f->createProduct('B'); apb->Operation(); // ProductB delete apa; 
    delete apb; 
    delete f; 
    return 0; 
}

```

## 工厂模式

### 模式定义
工厂方法模式(Factory Method Pattern)又称为工厂模式，也叫虚拟构造器(Virtual Constructor)模式或者多态工厂(Polymorphic Factory)模式，它属于类创建型模式。在工厂方法模式中，工厂父类负责定义创建产品对象的公共接口，而工厂子类则负责生成具体的产品对象，这样做的目的是将产品类的实例化操作延迟到工厂子类中完成，即通过工厂子类来确定究竟应该实例化哪一个具体产品类。

### 实例
```c++
#include <iostream>
using namespace std;
enum SOAPTYPE {SFJ,XSL,NAS};
 
class soapBase
{
	public:
	virtual ~soapBase(){};
	virtual void show() = 0;
};
 
class SFJSoap:public soapBase
{
	public:
	void show() {cout<<"SFJ Soap!"<<endl;}
};
 
class XSLSoap:public soapBase
{
	public:
	void show() {cout<<"XSL Soap!"<<endl;}
};
 
class NASSoap:public soapBase
{
	public:
	void show() {cout<<"NAS Soap!"<<endl;}
};
 
class FactoryBase
{
	public:
	virtual soapBase * creatSoap() = 0;
};
 
class SFJFactory:public FactoryBase
{
	public:
	soapBase * creatSoap()
	{
		return new SFJSoap();
	}
};
 
class XSLFactory:public FactoryBase
{
	public:
	soapBase * creatSoap()
	{
		return new XSLSoap();
	}
};
 
class NASFactory:public FactoryBase
{
	public:
	soapBase * creatSoap()
	{
		return new NASSoap();
	}
};
 
 
 
int main()
{
	SFJFactory factory1;
	soapBase* pSoap1 = factory1.creatSoap();
	pSoap1->show();
	XSLFactory factory2;
	soapBase* pSoap2 = factory2.creatSoap();
	pSoap2->show();
	NASFactory factory3;
	soapBase* pSoap3 = factory3.creatSoap();
	pSoap3->show();
	delete pSoap1;
	delete pSoap2;
	delete pSoap3;
	return 0;
}

```

## 抽象工厂模式

### 模式定义
抽象工厂模式(Abstract Factory Pattern)：提供一个创建一系列相关或相互依赖对象的接口，而无须指定它们具体的类。抽象工厂模式又称为Kit模式，属于对象创建型模式。
抽象工厂模式与工厂方法模式最大的区别在于，工厂方法模式针对的是一个产品等级结构，而抽象工厂模式则需要面对多个产品等级结构，一个工厂等级结构可以负责多个不同产品等级结构中的产品对象的创建 。当一个工厂等级结构可以创建出分属于不同产品等级结构的一个产品族中的所有对象时，抽象工厂模式比工厂方法模式更为简单、有效率。

```c++
#include <iostream>
using namespace std;
enum SOAPTYPE {SFJ,XSL,NAS};
enum TOOTHTYPE {HR,ZH};
 
class SoapBase
{
	public:
	virtual ~SoapBase(){};
	virtual void show() = 0;
};
 
class SFJSoap:public SoapBase
{
	public:
	void show() {cout<<"SFJ Soap!"<<endl;}
};
 
class NASSoap:public SoapBase
{
	public:
	void show() {cout<<"NAS Soap!"<<endl;}
};
 
class ToothBase
{
	public:
	virtual ~ToothBase(){};
	virtual void show() = 0;
};
 
class HRTooth:public ToothBase
{
	public:
	void show() {cout<<"Hei ren Toothpaste!"<<endl;}
};
 
class ChinaTooth:public ToothBase
{
	public:
	void show() {cout<<"China Toothpaste!"<<endl;}
};
 
class FactoryBase
{
	public:
	virtual SoapBase * creatSoap() = 0;
	virtual ToothBase * creatToothpaste() = 0;
};
 
class FactoryA :public FactoryBase
{
	public:
	SoapBase * creatSoap()
	{
		return new SFJSoap();
	}
	
	ToothBase * creatToothpaste()
	{
		return new HRTooth();
	}
};
 
class FactoryB :public FactoryBase
{
	public:
	SoapBase * creatSoap()
	{
		return new NASSoap();
	}
	
	ToothBase * creatToothpaste()
	{
		return new ChinaTooth();
	}
};
 
 
int main()
{
	FactoryA factory1;
	FactoryB factory2;
	SoapBase *pSoap1 = NULL;
	ToothBase *pToothpaste1 = NULL;
	pSoap1 = factory1.creatSoap();
	pToothpaste1 = factory1.creatToothpaste();
	pSoap1->show();
	pToothpaste1->show();
	
	SoapBase *pSoap2 = NULL;
	ToothBase *pToothpaste2 = NULL;
	pSoap2 = factory2.creatSoap();
	pToothpaste2 = factory2.creatToothpaste();
	pSoap2->show();
	pToothpaste2->show();
	
	delete pSoap1;
	delete pSoap2;
	delete pToothpaste1;
	delete pToothpaste2;
	
	return 0;
}
```
