---
layout: post
title: Java finally
category: 技术基础
tags:  Java finally
description: Java finally
---


### finally 


finally 语句块在 try 语句块中的 return 语句之前执行
try 语句块正常结束还是异常结束，finally语句块是保证要执行的。
如果 try 语句块正常结束，那么在 try 语句块中的语句都执行完之后，再执行 finally 语句块。
如果 try中有控制转移语句（return、break、continue），那么会先执行finally，（包括catch在内）


#### 进阶

##### 代码1


	public static void main(String args[]) {

		FinallyTest ft = new FinallyTest();
		System.out.println("test out is "+ft.test1());

	}

	public int test1() {

		int i = 1;
		try {
			return i;
		} finally {
			i++;
		}

	}

代码1的输出结果为1。
按照道理来说，在return i的时候先去执行i++，此时i为2。为什么最后输出为1.

讲test1方法反编译之后得到如下字节码：


	 public static int test1(); 
	  Code: 
	   0: 	 iconst_1 
	   1: 	 istore_0 
	   2: 	 iload_0 
	   3: 	 istore_1 
	   4: 	 iinc 	 0, 1 
	   7: 	 iload_1 
	   8: 	 ireturn 
	   9: 	 astore_2 
	   10: 	 iinc 	 0, 1 
	   13: 	 aload_2 
	   14: 	 athrow 
	  Exception table: 
	   from   to  target type 
	     2     4     9   any 
	     9    10     9   any 
	 }


字节操作码 

1. iconst 把变量放在操作栈上
2. istore 把操作栈上的变量放在本地变量表中
3. iload_ 从本地变量表中加载数据到操作栈中
4. iinc index, const    本地变量表中对应的索引位置加上对应的变量。

运行如下

![](http://7x00ae.com1.z0.glb.clouddn.com/16-8-21/68142853.jpg)

 
finally 语句块（iinc0,1）执行之前，
test1（）方法保存了其返回值（1）到本地表量表中1的位置，
完成这个任务的指令是 istore_1 ；然后执行finally语句块（iinc0,1），
finally 语句块把位于 0 这个位置的本地变量表中的值加 1，变成 2；待 finally 语句块执行完毕之后，把本地表量表中 1 的位置上值恢复到操作数栈（iload_1），
最后执行 ireturn 指令把当前操作数栈中的值（1）返回给其调用者（main）。
这就是为什么清单 6 的执行结果是 1，而不是 2 的原因。 

Java 虚拟机会把 finally 语句块作为 subroutine（名词）直接插入到 try 语句块或者 catch 语句块的控制转移语句之前。但是，还有另外一个不可忽视的因素，那就是在执行 subroutine（也就是 finally 语句块）之前，try 或者 catch 语句块会保留其返回值到本地变量表（Local Variable Table）中。待 subroutine 执行完毕之后，再恢复保留的返回值到操作数栈中，然后通过 return 或者 throw 语句将其返回给该方法的调用者（invoker）。

接下来看代码2，代码3

代码3：


	public static void main(String args[]) {

		FinallyTest ft = new FinallyTest();
		System.out.println("test out is "+ft.test2().getName());

	}
	public User test2() {
		 User u=new User("王");
		
	     try{
	    	 return u;
	     }finally{
	    	 
	    	 u=new User("宋");
	    //	 u.setName("宋");
	    	
	     }

	}


输出结果为 王
如果把 u=new User("宋"); 改成u.setName("宋"); 输出为宋。引文new 是新变量。

代码3


	public static void main(String args[]) {

		FinallyTest ft = new FinallyTest();
		System.out.println("test out is "+ft.test3());

	}
	public String test3() {
		
		String msg="aa";
	     try{
	    	 return msg;
	     }finally{
	    	 
	    	 msg="bb";
	     }

输出结果为aa。

接下来看几个例子
代码3



	 public  int test3() { 
	        int i = 1; 
	        try { 
	                 i = 4; 
	        } finally { 
	                 i++; 
	             return i; 
	        } 
		 } 
	 }

输出结果5


代码4



 	public  int getValue() { 
        int i = 1; 
        try { 
                 i = 4; 
        } finally { 
                 i++; 
        } 
        
        return i; 
	 } 


输出:5


代码5


	
	 public staitc String test5() {  
	 try {  
	    System.out.println("try block");  
	    return test();  
	 } finally {  
	    
	    System.out.println("finally block");  
			 
			 }  
		 } 
		 
	 public static String test() {  

	    System.out.println("return statement");  
	    return "after return";  
		 
		}  
	 }

输出结果为:

 try block 

 return statement
 
 finally block
 
 after return

在这里return test1();这条语句等同于 :

	 String tmp = test1(); 
	 return tmp;


#### 总结

本文主要介绍了finally在一些特殊情况下的使用，finally并不会一定执行，在finally中如果含有控制转移语句的时候需要额外注意执行顺序。

