---
title: 《Refactoring》读书笔记（四）重新组织数据
categories:
  - 读书笔记
date: 2018-12-10 13:37:18
tags: Books
---

[TOC]

# 第八章 重新组织数据

## 8.1 Self Encapsulate Field（自封装字段）

### 概要

你直接访问一个字段，但与字段之间的耦合关系逐渐变得笨拙。

**为这个字段建立取值/设值函数，并且只以这些函数来访问字段。**

（文中提到的两种观点，有点类似于我们 iOS 开发中的 “self.xxx 与 _xxx”）

## 8.2 Replace Data Value with Object（以对象取代数据值）

### 概要

你有一个数据项，需要与其他数据和行为一起使用才有意义。

**将数据项变成对象。**

## 8.3 Change Value to Reference（将值对象改为引用对象）

### 概要

你从一个类衍生出许多彼此相等的实例，希望将它们替换为同一个对象。

**将这个值对象变成引用对象。**

## 8.4 Change Reference to Value（将引用对象改为值对象）

### 概要

你有一个引用对象，很小且不可变，而且不易管理。

**将它变成一个值对象。**

## 8.5 Replace Array with Object（以对象取代数组）

### 概要

你有一个数组，其中的元素各自代表不同的东西。

**以对象替换数组。对于数组中的每个元素，以一个字段来表示。**

## 8.6 Duplicate Observed Data（复制“被监视数据”）

### 概要

你有一些领域数据置身于GUI控件中，而领域函数需要访问这些数据。

**将该数据复制到一个领域对象中。建立一个Observer模式，用以同步领域对象和GUI对象内的重复数据。**

## 8.7 Change Unidirectional Association to Bidirectional（将单向关联改为双向关联）

### 概要

两个类都需要使用对方特性，但其间只有一条单向连接。

**添加一个反向指针，并使修改函数能够同时更新两条连接。**

## 8.8 Change Bidirectional Association to Unidirectional（将双向关联改为单向关联）

### 概要

两个类之间有双向关联，但其中一个类如今不再需要另一个类的特性。

**去除不必要的关联。**

## 8.9 Replace Magic Number with Symbolic Constant（以字面常量取代魔法数）

### 概要

你有一个字面数值，常有特别含义。

**创造一个常量，根据其意义为它命名，并将上述的字面数值替换为这个常量。**

## 8.10 Encapsulate Field（封装字段）

### 概要

你的类中存在一个public字段。

**将它声明为private，并提供相应的访问函数。**

## 8.11 Encapsulate Collection（封装集合）

### 概要

有个函数返回一个集合。

**让这个函数返回该集合的一个只读副本，并在这个类中提供添加/移除集合元素的函数。**

## 8.12 Replace Record with Data Class（以数据类取代记录）

### 概要

你需要面对传统编程环境中的记录结构。

**为该记录创建一个“哑”数据对象。**

## 8.13 Replace Type Code with Class（以类取代类型码）

### 概要

类之中有一个数值类型码，但它并不影响类的行为。

**以一个新的类替换该数值类型码。**

## 8.14 Replace Type Code with Subclasses（以子类取代类型码）

### 概要

你有一个不可变的类型码，它会影响类的行为。

**以子类取代这个类型码。**

## 8.15 Replace Type Code with State/Strategy（以 state/strategy 取代类型码）

### 概要

你有一个类型码，它会影响类的行为，但你无法通过继承手法消除它。

**以状态对象取代类型码。**

### 动机

本项重构和 Replace Type Code with Subclasses（223）很相似，但如果“类型码的值在对象生命周期中发生变化”或“其它原因使得宿主类不能被继承”，你也可以使用本重构。

State模式和Strategy模式非常相似，重构过程是相同的，选择哪个模式并非问题关键所在。如果你打算在完成本项重构后再以 Replace Conditional with Polymorphism（255）简化一个算法，那么选择 Strategy 模式比较合适；如果你打算搬移与状态相关的数据，而且你把新建对象视为一种变迁状态，就应该选择使用 State 模式。

## 8.16 Replace Subclass with Fields（以字段取代子类）

### 概要

你的各个子类的唯一差别只在“返回常量数据”的函数身上。

**修改这些函数，使他们返回超类中的某个（新增）字段，然后销毁子类。**

