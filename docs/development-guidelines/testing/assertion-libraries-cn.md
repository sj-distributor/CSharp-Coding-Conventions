
# 断言库
- - -

### **要** 断言扩展方法的使用:
大多数测试框架都有自己的内置断言类。NUnit的Assert断言库通过静态方法调用：

```csharp
Assert.AreEqual(expected, actual);
```

Assert断言库在NUnit首次发布时是合适的，但后来的C#版本给了我们一个更好用的断言库,名为:Shouldly :

```csharp
actual.ShouldBe(expected);
```

Shouldly断言库写起来和读起来都更自然，我们可以用特定于应用程序的速记来增强内置断言，只需用“ShouldXyz”风格的方法命名来定义我们自己的断言库。

### **要** 与测试框架的内置Assert断言库相比，确实更喜欢Shouldly断言库

在.NET断言库中, [Shouldly](https://github.com/shouldly/shouldly) 有一个简单的API，被积极维护，并且适用于.NETFramework和.NETCore。
