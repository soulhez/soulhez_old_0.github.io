---
layout: post
title:  "深入理解闭包"
date:   2017-10-08 19:04:00 +0800
categories: C# 
tags: C# lua closures 
---

## 起源
最近在敲c#的时候，在循环中加入了闭包，运行后的结果和预期相差很远。  

Code 1:
```csharp
List<Action> actions = new List<Action>();
List<int> oList = new List<int>(){0, 1, 2, 3, 4};

var variable = oList.GetEnumerator();
while(variable.MoveNext()){
	actions.Add(()=> Console.WriteLine(variable.Current));
}
```

```
Output:
0
0
0
0
0
```

预想的结果应该是输出0~4, 然而事实并非如此。看起来像是输出了`oList`的第一个数。但是将0改成9再跑一遍，结果并没有发生变化。
事情并没有这么简单！！！

##  解决方案 
幸运的是搜索`loop closure`很快就在StackOverflow上找到了类似的问题，找到了解决方案。

Code 2:
```csharp
List<Action> actions = new List<Action>();
List<int> oList = new List<int>(){0, 1, 2, 3, 4};

var variable = oList.GetEnumerator();
while(variable.MoveNext()){
	var copy = variable.Current;
	actions.Add(()=> Console.WriteLine(copy));
}
```

```
Output:
0
1
2
3
4
```

初步得到的结果是只要在循环中将要传入的变量的值拷贝一份然后再传入到闭包中。
到此为止，如果对闭包的原理很清楚的话就能马上理解这是为什么。然而我并不是很理解闭包=。=
所以这里我们需要透过现象看本质。

##  什么是闭包
闭包早在高级语言开始发展的年代就产生了。闭包（Closure）是词法闭包（Lexical Closure）的简称。对闭包的具体定义有很多种说法。我认为比较贴切的是这种定义。
> 闭包是由函数和与其相关的引用环境组合而成的实体。闭包在运行时可以有多个实例，不同的引用环境和相同的函数组合可以产生不同的实例。

我们可以用lua代码来快速的理解闭包的定义。

Code 3:
```lua
function ShowMeClosure()
	local foo = 1
	local closure = function ()
		foo = foo * 2
		return foo
	end
	print("foo:", closure())
end

ShowMeClosure()
```

```
Output:
foo:2
```
正常情况下函数`ShowMeClosure`中的临时变量`foo`的生命周期在函数执行完后就结束了。
但是在匿名函数中使用了`foo变量`，因此在匿名函数中可以继续使用`foo变量`。
此时变量foo和匿名函数就形成了一个闭包。在闭包环境中可以继续使用`foo变量`。

理解了闭包的定义后，让我们来拜读[闭包之美(The Beauty of Closures)](http://csharpindepth.com/Articles/Chapter5/Closures.aspx)。
文章中很好的讲解了C#闭包，而且分别从C# 1，C# 2，C# 3不同版本的C#实现来解释。从C# 3最简化的实现，到最原始的C# 1实现。
而我们现在看到的最简化的实现，其实本质就是c#1的实现，只不过我们的`编译器`帮助了我们，让我们不必去做繁琐的工作。

## 加深理解 
现在我们再回头来看起初的问题。 由于c#中的闭包使用的是`变量本身`，并非变量的值，所以当我们上面的Code 1:在循环结束后所使用的`variable变量`已经是一个空的迭代器了。
所以我们输出时直接调用的`variable`，那么他的值是0，一个不存在与oList的值。
我们的目的就是要输出正确的值，所以只要在循环中得到每次循环时的正确变量就能达到效果，于是一个`copy`就解决了问题。

到此对闭包应该算是有所领悟了。当然理解闭包的原理时还有一个很好的办法，也是理解一切语法的万能办法，就是查看`IL`。

以上就是我的理解，欢迎大家讨论和指正我的错误。
