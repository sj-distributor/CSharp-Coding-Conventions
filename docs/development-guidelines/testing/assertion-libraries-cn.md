
# 断言库
- - -

### **要** 对于断言使用扩展方法
大多数测试框架都有自己的内置断言类。它们通常基于NUnit的模型，其中断言作为静态方法调用工作：

```csharp
Assert.AreEqual(expected, actual);
```

这些在NUnit首次发布时是合适的，但后来的C#版本给了我们一个更有用的选择。Shouldly 允许我们使用扩展方法:

```csharp
actual.ShouldBe(expected);
```

这些扩展方法写起来和读起来都更自然，我们可以用特定于应用程序的速记来增强内置断言，只需用“ShouldXyz”风格的方法命名来定义我们自己的扩展方法。

### **要** 与测试框架的内置静态断言相比，确实更喜欢Shouldly断言库

在.NET断言库中, [Shouldly](https://github.com/shouldly/shouldly) 有一个简单的API，被积极维护，并且适用于.NETFramework和.NETCore。
