
# 测试覆盖范围
我们期望为项目生成的测试套件能够充分解决系统功能的所有核心领域。此外，每个测试夹具都应该包含一组测试，这些测试可以充分测试被测试代码的特性。我们没有针对特定的代码覆盖率，因为这很容易成为一个虚荣指标，或者被包含琐碎断言的低值测试所玩弄。
每个项目都是不同的，并且可能有不同的测试需求（例如，一些项目需要UI测试或与特定外部服务的集成测试）。在任何情况下，以下是应始终覆盖的核心领域：
1. TOC
   {: toc}

### **要** 测试系统的所有功能
测试覆盖率很重要。我们希望确保测试我们编写的代码的所有区域。一个很好的经验法则是，对于被测试系统公开的每个公共方法，至少需要一个测试，乘以该方法的关键逻辑输入状态的数量。
在这里使用一些判断。如果您的方法需要大量的输入，则不需要为每个可能的输入编写单独的测试用例。然而，您应该能够将输入分解为能够产生有意义覆盖的分组。覆盖率是关于置信度的：您确信这组示例有意义地演示了代码的行为吗？e some judgement here.  If your method takes a huge range of inputs, you don’t need to write a separate test case for every possible input.  You should, however, be able to collapse your inputs down to groupings that produce meaningful coverage. Coverage is about confidence levels: are you confident that the set of examples meaningfully demonstrates how the code behaves?

### **不要** 追逐代码覆盖率指标
当您拥有代码覆盖率工具时，编写或更改测试仅仅是为了改进报告的覆盖率度量是很诱人的。如果测试碰巧触及了更多的代码行，而没有执行有意义的场景，也没有做出有意义的断言，那么“游戏”这些度量就太容易了。当度量驱动测试时，测试往往会受到影响，同时给团队带来错误的信心。

### **避免** 将代码覆盖率工具作为质量的主要衡量标准
我们的默认建议是代码覆盖率工具可以提供错误的安全感，因为它提供了一个具体的度量前端和中心。
我们发现，对所有功能分支使用彻底的[code reviews](/developmentguidelines/pull-requests/)可以实现相同的最终目标，即确保有质量测试覆盖率，并且与静态分析指标相比，人类可以更好地评估覆盖率。
正因为如此，我们发现使用它们会造成更多的伤害，特别是如果整个开发团队没有将意图内化。

代码覆盖率指标可以作为第二道防线。我们推荐JetBrains的[dotCover](https://www.jetbrains.com/dotcover/). 
评估替代测试覆盖率工具时，确保它支持.NET Core和[Azure DevOps](https://azure.microsoft.com/) 等系统-因为一些测试覆盖工具尚未针对最新进展进行维护。

### **要** 用 Queries / Commands 来测试 读取和写入数据

[Query and Command handlers](https://github.com/jbogard/MediatR/wiki#basics)
构成被测系统的绝大多数。通常，这些测试断言处理程序执行的读取和写入。
有关有效测试这些处理程序的更多信息，请参阅 [Integration Testing Patterns](/development-guidelines/testing/integration-test-patterns.html#integration-test-patterns) for more on effectively testing these handlers.

### **要** 测试 核心基础 和 helper类 以及 扩展方法
用作实现业务逻辑的构建块的“Helper”方法通常需要直接测试，这样您就有信心在系统的其余部分中使用它们。

### **要** 基于现实场景编写测试
现实场景让我们对系统在生产中的表现充满信心。它们有助于确保我们的高覆盖率不仅来自于测试系统中的每一条线，还来自于反映生产的方式。

### **要** 确定测试系统的所有预期输入

这包括外部系统状态，如环境变量、文件系统内容和数据库内容。

### **要** 探究边界案例
边界案例是我们可能不希望经常看到的状态，或者代码应该总是表现得不同。这些是错误的常见来源。仔细考虑您知道需要特殊处理的方法的输入，并专门为这些情况编写测试。例如，任何小于或等于1的输入值都将是此代码中的特殊情况，并将保证其自己的测试用例：
```csharp
int Factorial(int n)
{
    if (n < 0)
        throw new InvalidOperationException();

    if (n == 0 || n == 1)
        return 1;

    return n * Factorial(n-1);
}
```

### **要** 验证异常条件下的行为

当输入无效时，确认系统的行为。当代码预期引发异常时，确认系统的行为。

### **要** 验证全部输出和副作用
完全验证类的行为可能很难正确。请记住，在编写测试时，输入并不总是直接提供给被调用的方法——它可能是需要在测试设置中准备的一些外部系统状态（如数据库记录）。



例如，假设有个[`Command`](https://github.com/jbogard/MediatR/wiki#basics) 
它负责从数据库中删除记录。编写一个假设数据库为空的测试非常简单，插入一条测试记录，运行“DELETE”命令，然后断言目标表为空。但是，如果您的“DELETE”命令忽略了包含“WHERE”子句，该怎么办？测试是绿色的，但你有一个大错误。

覆盖率不是关于测试运行时哪些代码行被命中，而是测试是否有意义地演示了特性的行为。