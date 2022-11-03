---
布局: default
标题: C#
nav_order: 4
上级目录: 开发指南
子目录: 无
---

# C#
本指南为开发人员定义了C#开发标准并设定了明确的期望。它还可以作为参考材料，帮助新入行的工程师，使得每个人都可以自给自足。

具有可预测结构并遵循指导原则的代码使开发人员能够适应代码的意图，而不是一种陌生的或分散注意力的约定。公认的约定促进了可读性，并肯定了开发人员在第一次检查时的期望。当我们遵循既定的约定和标准时，代码变得更易于导航和理解。编码标准考虑到将来其他开发人员会阅读和维护代码。

- TOC

## 一致性的重要性

我们的许多项目都不是“greenfield”，成熟的代码库基于或者与现有的客户工程团队甚至其他咨询公司合作。在不考虑对整个代码库的影响的情况下，你可以大胆地使用自己的约定、命名样式等（本文中有很多）。然而，正确的方法是调整你的风格，使之与已经确立的风格相匹配。

我们编写的代码将在未来几年被许多其他人阅读，首先是我们的开发人员和技术负责人。一旦我们为客户编写了软件，我们就将其交付给他们，然后不再自行维护，我们目标是他们或任何其他开发团队都可以轻松地维护它。如果代码库是风格和结构的混合体，读者会发现很难解析，这就很难维护。我们的目标是拥有一个看起来像是由一个开发人员编写的代码库。

### **规定** 默认新项目可以使用本指南

### **规定** 优先使用项目中预先设定的代码样式和风格，即便它们与本指南不同 conform to pre-existing code styles in a project, even when they differ from this guide

### **避免** 在单个代码库中混合多种编码样式和风格

### **规定** 利用工具配置文件在开发人员的环境中自动控制样式和结构
Visual Studio使用 `.editorconfig` 文件以及R#使用 `*.DotSettings` 文件来共享环境配置并存储在源码管理中

Headspring’s 默认ReSharper的配置可以在以下存储库中找到

[Resharper Configuration](https://bitbucket.org/headspring/resharper-configuration/src/master/)(_internal repository_)

## 通用的C#标准

### **规定** 尽可能的使用var而不是显式类型.
例外：需要显式子类型，例如：

```csharp
ReadOnlyCollection<People> people = GetPeople(); // returns List<People>()
```

### **禁止** 引用成员字段和方法时使用显式this限定符
```csharp
// Bad
public void SampleMethod(string name)
{
    this.Name = name;
    this.AddToList(name);
}

// Good
public void SampleMethod(string name)
{
    Name = name;
    AddToList(name);
}
```

### **规定** 对内置类型使用别名，而不是BCL类名

```csharp
// Bad
String.IsNullOrEmpty(person.Name);

// Good
string.IsNullOrEmpty(person.Name);
```

### **仔细考虑** 使用字符串插值代替string.Format()
{: .text-yellow-300 }
```csharp
// Bad
var message = string.Format("You live in {0}, {1}.", address.City, address.State);

// Good
var message = $"You live in {address.City}, {address.State}.";
```

### **规定** 每个缩进使用四个空格，而不是两个空格或制表符

```csharp
// Bad
public void DoSomething()
{
••Foo();
}

// Good
public void DoSomething()
{
••••Foo();
}
```

### **禁止** 有多个连续的空（空白）行

```csharp
// Bad
public void DoSomething()
{
   Foo();
   Bar();


   Baz();
}

// Good
public void DoSomething()
{
   Foo();
   Bar();

   Baz();
}
```

### **规定** 使用语句（或指令）删除未使用的或冗余的

### **禁止** 将多行语句的大括号放在与自己同一行上

```csharp
// Bad
public void DoSomething() {
   Foo();
}

// Good
public void DoSomething()
{
   Foo();
}
```

### **禁止** 在左大括号或右大括号的后面或前面加一个空行
{: .text-red-300 }
```csharp
// Bad
public void DoSomething()
{

   Foo();
   Bar();

}

// Good
public void DoSomething()
{
   Foo();
   Bar();
}
```

## 类成员的基本布局

代码应该是可读的，这样没有任何项目背景的开发人员无需适应特定的约定，就可以根据经验进行阅读，对类的成员进行排序并提供了一种预期的结构，使开发人员能够快速而彻底地理解新代码。

下面的陈述适用于一个green-field项目——在那里我们可以对我们建立的新类做出决定。通常，我们被要求帮助现有系统。在这些情况下，我们应该遵循客户的既定模式。

### **规定** 按照下面的常规顺序编写类成员，以便更容易找到成员
{: .text-green-100 }
**成员排序**

按以下顺序放置所有类、接口和结构的成员：

Constant Fields
Fields
Constructors
Finalizers (Destructors)
Delegates
Events
Enums
Interfaces (interface implementations)
Properties
Indexers
Methods
Structs
Classes

对于上面的每个成员类型，按访问修饰符排序如下：

public
internal
protected internal
protected
private

在访问修饰符中，将静态元素放在实例（非静态）元素之前

将只读元素放在可变元素之前。

## 命名约定

命名约定增强了代码可读性，并在整个系统中描述领域对象。乍一看，开发人员应该能够确定类型、方法等的用途，而不必在实现中查找上下文去理解。

### **规定** 将PascalCasing（首字母大写）用于类名、方法名、属性和命名空间

### **规定** 将camelCasing（首字母小写）用于方法参数、局部变量和字段名

大小写的例外情况应包括域对象中使用的特殊业务术语。ReSharper可能会提示缩写违反了大小写约定。将缩写添加到ReSharper配置文件中的忽略列表中，以跳过对域缩写的分析。在团队级别共享DotSettings文件。。

### **禁止** 使用匈牙利符号，数据类型缩写作为变量或字段名的前缀

### **规定** 使用大写字母分隔标识符区分名称中的两个不同单词

### **避免** 使用缩写或更短版本的单词-如GetWin 表示 GetWindow

例外：通常用作名称的缩写，如Id、Xml、Ftp、Uri等。

### **规定** 使用PascalCasing表示三个或更多字符的缩写

### **禁止** 使用下划线分隔单词

例外：测试方法名称通常在单词之间用下划线书写。在这个特定的实例中，您可以使用下划线来提高描述性方法名称的可读性，否则很难直观地解析这些名称。

### **考虑** 在共享的ReSharper dotsettings文件中创建排除列表，以保存众所周知的域名缩写

比如:

MASMModule 中 MASM 表示 Medical Administrative Service Module

MROWeb 中 MRO 表示 Medical Review Officer

### **规定** 使用名词或名词短语命名类

### **避免** 使用过于泛型的类型名称

好的名称应该描述类或实例所做的一切。要避免的一些常见后缀是：

Manager
Builder
Writer
Getter
Setter
Provider
Facade

### **规定**  接口以'I'作为前缀

接口名称应该是名词、名词短语或形容词。

## 类型和成员的可见性

支持更多限制而不是更少限制的访问修饰符。对于不能在当前类外部访问的方法使用private，对于表示发布可访问接口且不受限制的方法使用public。限制访问修饰符以防止实现细节泄漏到当前上下文之外。

### **避免** 使用内部或受保护的修饰符

`Internal` 修饰符通常没有用处，除非您有特殊的用例，例如类库开发。“Protected“修饰符可能表明类继承被滥用。

### **规定** 使用显式修饰符，即使它与默认访问级别匹配

示例：默认情况下，字段声明是“private”，但添加显式private修饰符清楚地表明了这一意图。

### **规定** 对只能在类内部访问的类型或成员使用私有修饰符，对所有其他类型或成员都使用公共修饰符。

在考虑如何访问类型及其方法和属性的同时，请考虑该类的可测试性。如果使用私有修饰符对类进行过多限制，则很难或不可能为其所有功能编写测试。

## DRY准则 （Don't Repeat Yourself Principle）
将代码组织成更小的、可维护的和可重用的代码块可以使代码更强大，更不容易出错，因为每个代码块都是可测试的，并按设计执行其功能。在多个位置重复代码更容易出错并浪费时间，因为您必须在多个地方修改实现。

然而，DRY准则也可能被过度使用。过于“枯燥”的代码可能很难阅读，为了理解它的实际功能，需要遵循太多的逻辑调用和重定向。

此外，当我们使用功能文件夹和MediatR编写应用程序时，我们可能会发现自己在几个不同的处理程序中编写相同的查询或更新操作。很容易将重复重构为某种实用程序或服务，但首先需要确定重复是巧合还是故意的结果。例如，命令和查询处理程序都表示特定的业务逻辑。跨多个处理程序的重复代码更可能是巧合而不是有意的，并且不是重构为实用程序方法的好时机。

你可以考虑的一种方法是WET——“write everything twice.”。重复两次代码通常是可以的，因为它可以让我们免于基于直觉反应的过早优化。相反，我们观察当代码重复三次或三次以上时出现的模式。这允许我们检查代码的实际用例，并就如何正确重构做出明智的决定。

补充资料: https://dev.to/wuz/stop-trying-to-be-so-dry-instead-write-everything-twice-wet-5g33

### **考虑** 当代码重复三次或多次时，将代码重构为通用的可重用方法

### **避免** 过度使用DRY原则故意复制代码

代码应该是有目的的，并且在于解决一个非常具体的问题。如果重构看似重复的代码改变了代码的特性或功能，那么就别管它了。

## 使用#Regions
#Regions 是.NETFramework1.x时代（约2002年）的一个IDE功能，在2.0引入分部类之前用于隐藏生成的设计器代码。当IDE隐藏细节时，代码的可读性可能会受到极大的阻碍。如果源代码文件太长，请考虑重构实现，而不是在区域中隐藏代码。

### **禁止** 使用 #regions

## 代码本身就是文档

可以通过仔细的命名和可识别的模式来表达意图。自描述代码减轻了强制或冗长注释的需要。代码注释只能用于记录非自解释的实现选择，如RegEx模式或位掩码等简洁的语言特性。如果你认为你需要写评论，你是在描述“为什么”还是“它是什么”？如果您发现需要编写“它是什么”注释，那么您应该重新考虑代码的命名和结构，以更好地传达意图。

信息来源 :

[Clean Code](https://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship-dp-0132350882/dp/0132350882/ref=mt_paperback?_encoding=UTF8&me=&qid=) (Martin) chapters 2 (Meaningful Names) and 4 (Comments)

Headspring “Mindful Development - Week 4” video (~20:00 - 27:00 segment)(_internal_)

### **规定** 在命名方法、变量等时，使用能记住的命名和可识别的模式来表达意图

例如，参数名称应具有足够的描述性，以便可以使用名称及其类型来确定其在大多数情况下的用途。

### **考虑** 在提交文本中提供与代码相关的任何附加信息，而不是注释块

注释块变得陈旧，而提交消息与更改的历史一起存在。

### **禁止** 将TODO注释推送到主分支或开发分支

`TODO`注释对于开发人员跟踪正在进行的工作可能很有用，但它们不应在feature或bug分支之外持续存在.在靠近`TODO`的地方`throw new NotImplementedException()；`，以确保在合并分支之前完成工作。

对于较长时间的“待办事项”，跟踪JIRA或相关项目管理工具中的未来工作，以便在整个团队中保持可见、计划和跟踪.

### **避免** 使用注释来解释写得不好的代码

如果你发现自己需要写一个注释来解释你所做的事情，这可能是一个迹象，表明可以进行一些重构，以使代码更可读。

### **规定** 当客户端要求或需要文档时，使用注释

例如，第三方工具（如Swagger）可以使用XML参考文档来记录RESTful API。

### **禁止** 注释旧的代码

如果需要恢复或引用更改，源代码管理将提供代码更改的历史记录。

## 童子军规则

随着时间的推移，在任何项目中，代码库的质量都会下降，技术债务也会增加。为了积极应对技术债务的累积，我们会尽最大努力让事情变得比我们发现的更好。例如，如果你正在改变一种方法，花几分钟时间看看是否有什么可以在短时间内改进的地方。变化不必很大，只要更好。如果您继续进行这些微小的更改，那么随着时间的推移，代码的整体质量将显著提高。

有关如何在源代码管理中捕获重构工作的详细信息，请阅读源代码管理指南-自行提交时进行工具更改。

附加来源:

[Clean Code](https://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship-dp-0132350882/dp/0132350882/ref=mt_paperback?_encoding=UTF8&me=&qid=) (Martin) chapters 1 (The Boy Scout Rule)

### **规定** 将您的更改保留在您正在处理的功能或bug的范围内

技术负责人需要能够审查您的更改，重构不相关的类或方法将使确定更改的原因和影响更加困难和耗时。

### **考虑** 重命名变量、参数和方法

例如，变量名命名不明确或不正确，需要调查以确定其用途，例如day vs.birthDay，price vs.priceAfterTax。

### **考虑** 修复您发现的小错误

在您的功能范围内发现的可以快速修复的Bug应该得到修复。但是，如果您发现一个更大的错误，或者不在您的功能范围内的错误，请创建一个新的错误单。

### **考虑** 重构大型类以提高可读性

大型类通常是过度泛化的实用程序类的标志。

### **考虑** 你的更改可能与团队存在合并冲突

重构其他人或者团队正在处理的代码可能会导致繁琐的合并冲突。确保您的更改不会扰乱或延迟总体进度。

### **考虑** 创建“Refactor”或“Cleanup”的tickets，以跟踪说明你大量的清理工作。

这些变更应与团队和项目领导协调，以确保不会造成破坏。

## 组合与继承

继承是一种很容易被滥用的OOP模式。

滥用继承可能导致:

- 两个具体类的紧密耦合
- 基类变得脆弱
- 封装性减弱
- 测试混乱问题
- 额外的维护成本

### **规定** 倾向于组合而非继承

组合使我们的设计具有更高的灵活性，能够在不违反契约的情况下修改未来的行为。

### **规定** 使用接口定义类之间的契约

如果您在应用程序的关键点定义接口，那么您应该仔细考虑它们应该支持的行为，并致力于该行为。

### **禁止** 当子类不合适作为父类的子类型时使用继承

使用大量“virtual”、“override”和“base”关键字的类很难阅读。确定从基类（或基类的基类）中提取或更改行为的类的用途会使跟踪正在执行的实际行为变得困难。子类只能用于添加基类的功能，而不能修改它。

### **禁止** 跨越领域边界继承

如果通用的机械结构是要求跨越领域边界的，使用通用的实用基类代替。

### **考虑** 在应用程序层面建立单一继承的约定

例如，我们经常创建一个基本控制器类，其中包含我们希望所有其他控制器都可以使用的约定，例如用于返回特定格式的JSON响应的帮助方法。

### **避免** 三层以上的继承

超过三个层以上的继承是一个强有力的指标，您应该考虑重构到组合。

可接受的三层以上继承的一个例子是，当你需要在第三方扩展前使用自己的基类实现。

```csharp
public class MyBaseController : ClientLibrary.ClientBaseController
{
   // …
}

public class EmployeeController : MyBaseController
{
   // …
}
```
