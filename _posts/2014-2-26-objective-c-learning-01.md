---
layout: post
title: Objective-C学习之property VS instance variable
---

Objective-C提供两种方式来声明一个类（interface）中的成员变量：property以及instance variable。

1. **property**

   如果将成员变量声明成property，编译器会自动生成相应的getter和setter函数，同时还会生成一个对应的instance variable。
   例如，给一个类XYZClass声明一个property member：
   
   	@interface XYZClass : NSObject

   	@property id member;

   	@end

   那么在类的方法中可以通过以下三种方式访问member：

   	// 通过getter方法
   	[self member]
   	// 通过点方法
   	self.member
   	// 通过对应的instance variable
   	_member
   
   类的property对外是公开的，在非本类的代码中可以通过getter以及点方法访问property。
   另外，Objective-C允许为property定义各种属性，不同的属性赋予了property不同的特性，参考http://rypress.com/tutorials/objective-c/properties.html
   
    <!-- more -->

2. **instance variable**

   如果将成员变量声明成instance variable, 那么该成员默认是protected，但是可以通过声明属性来改变其可见性：

   	@interface XYZClass : NSObject｛
   		@public
   		id publicMember;
       
   		@protected
   		id protectedMember;

   		@private
   		id privateMember;
   	｝
   	@end

   在类中通过instance variable的变量名来访问该成员；在非本类的代码中可以通过类似C中的指针运算符->访问public instance variable。

通常来说提倡使用property，它不仅方便而且使代码易于理解，也比较符合封装的思想。
