# 集成测试模式

尽管具体的技术在不同的项目中差异很大，但一些核心原则促使我们最有效的集成测试遵循类似的模式。

### 集成测试原则

* 尽可能多地锻炼生产场景。
* 开发“shorthand”测试助手函数，以便测试可以像用户交互脚本一样读取。

没有两个应用程序是相同的，因此细节会有所不同，但这有助于将讨论集中在我们在许多项目中看到的几个高价值细节上。特定的MVC版本、特定的IoC工具、特定的数据库会有所不同，但当我们将上述原则应用于这些变化时，我们仍然可以在测试中得出非常相似的模式。

考虑一个典型的现代web应用程序：ASP。NET MVC Core及其内置IoC容器、SQL Server数据库和与每个web请求对齐的有用“工作单元”。
每个web请求都被设置为在请求开始时打开一个专用的SQL事务，并在结束时提交/回滚，这样我们的大部分代码就完全不需要关心事务。与该事务一致，我们有一个作用域的IoC容器，这样每个web请求都有自己的专用容器，基于应用程序启动期间初始化的“全局”容器。
控制器使用[MediatR](https://github.com/jbogard/MediatR) 将`Command` `Command` 和 `Query`
对象发送到它们各自的处理程序，在那里执行实际工作。这些处理程序可能受到某些验证规则库的保护，例如[FluentValidation](https://fluentvalidation.net/).

考虑到这个应用程序，我们通常会有带有消息处理程序类的测试类1-1。我们不会对控制器进行自动化测试：这些控制器只会在运行的web服务器中的真实web请求的上下文中真正运行。谢天谢地，Mediator的使用使控制器非常小。只要我们运行应用程序并将控制器作为典型QA工作的一部分，我们就会对处理程序测试覆盖率感到满意。

如果我们的主要集成测试工作是测试MediaR处理程序，那么，编写一个测试可能很诱人，方法是显式构造一个MediaR处理程序，将其依赖项（real和stub）传递给构造函数，显式调用`Handle`
方法，并断言响应和副作用。这样做是一个错误，因为它违反了我们的两个集成测试原则：它不模拟生产，而且冗长。

如果测试可以像`Controller`类一样简单地 `Send` 相关的`Command`或`Query`对象，就像在生产中一样，完全使用 MediatR管道，那就更好了。 如果每一个这样的`Send`
都像在生产中一样自动参与一个工作单元事务，那就更好了。 如果每一个这样的`Send`在允许处理程序运行之前都执行了任何验证规则，就像在生产中一样，那会更好。 如果每一个这样的`Send`
都像在生产中一样，在专用的、有范围的IoC容器的上下文中执行，那就更好了。 最后，如果我们能够用非常少的字符数来完成所有这一切，那就更好了，这样测试作者就可以简单地关注手头的场景，而不是任何这些横切的问题。

所有这些原则都可以在 [CareerStart Employee Directory](https://bitbucket.org/headspring/headstart-employee-directory) (_internal
repository_) 中看到示例应用程序。

在这里，我们将回顾员工目录测试的基础结构，然后见证一些有代表性的测试，最后总结为什么这种方法在实践中如此有效。在任何时候，我们都可以将我们的决策与上面的两个集成测试原则联系起来。

我们将通过定义静态类`Testing`来实现测试中的速记，大多数测试文件都会包含`using static Testing;`。 因此，该类中的所有公共方法都可以从任何测试中调用，就像它们是全局函数一样。`Testing`
的静态构造函数将建立许多全局帮助函数所使用的一些基础。

在我们的集成测试中应用这些模式会产生易于编写的测试，尽可能多地练习生产代码，保护自己免受不现实或不可能的场景的测试，并在不断增长的项目的整个生命周期中保持有意义的红色或绿色。
有了这种风格，我们很少需要触及继承或测试框架特定的奇怪之处，比如`TestFixtureSetUp`。

### **要** 使用与应用程序相同的设置初始化集成测试运行

```csharp
private static readonly IServiceScopeFactory ScopeFactory;

public static IConfigurationRoot Configuration { get; }

static Testing()
{
    Configuration = new ConfigurationBuilder()
        .SetBasePath(Directory.GetCurrentDirectory())
        .AddJsonFile("appsettings.json", true, true)
        .AddEnvironmentVariables(Program.ApplicationName + ":")
        .Build();

    var startup = new Startup(Configuration);
    var services = new ServiceCollection();
    startup.ConfigureServices(services);
    services.AddSingleton<ILoginService, StubLoginService>();

    var rootContainer = services.BuildServiceProvider();
    ScopeFactory = rootContainer.GetService<IServiceScopeFactory>();
}
```

首先[`static Testing constructor`](https://bitbucket.org/headspring/headstart-employee-directory/src/35734e21140fb7bff6a7db61a0adf8a96a91f39a/src/EmployeeDirectory.Tests/Testing.cs#lines-27:46) (_
internal repository_)
以与运行的web应用程序完全相同的方式建立测试项目的配置文件。 然后，我们设置将在所有测试中使用的根IoC容器。我们不仅仅通过执行与核心应用程序类似的IoC设置来实现，而是通过调用web应用程序在生产中启动时执行的相同设置代码来实现。
是的，我们可以在容器中注册一个或两个罕见的测试存根，覆盖生产配置，但只有当生产设置造成了太严重的障碍时，例如对于处理第三方API的接口。 在这个例子中，我们引入了一个存根`ILoginService`
，允许测试模拟运行时存在的登录cookie。

### **要** 建立核心集成测试helpers: Transaction, Query, Validator, 和 Send

接下来, 我们将几个核心测试helpers方法分层。

我们定义持久性测试助手方法。随着IoC和持久化技术的变化，它们的细节可能会因项目而异，但它们的目的在不同的项目中是相同的：

```csharp
public static void Transaction(Action<DirectoryContext> action)
{
    using (var scope = ScopeFactory.CreateScope())
    {
        var database = scope.ServiceProvider.GetService<DirectoryContext>();

        try
        {
            database.BeginTransaction();
            action(database);
            database.CloseTransaction();
        }
        catch (Exception exception)
        {
            database.CloseTransaction(exception);
            throw;
        }
    }
}

public static TResult Query<TResult>(Func<DirectoryContext, TResult> query)
{
    var result = default(TResult);

    Transaction(database =>
    {
        result = query(database);
    });

    return result;
}

public static TEntity Query<TEntity>(Guid id) where TEntity : Entity
{
    return Query(database => database.Set<TEntity>().Find(id));
}

public static int Count<TEntity>() where TEntity : class
{
    return Query(database => database.Set<TEntity>().Count());
}
```

`Transaction` 助手方法允许测试在类似web请求的工作单元的上下文中轻松运行任何代码。提供的操作在专用事务中运行，就像生产一样，在专用范围内的IoC容器中运行，像生产一样。在此基础上，我们为常见数据库交互提供了额外的简写：

```csharp
var entity = Query<TEntity>(id);

var result = Query(db => …);

var count  = Count<TEntity>();
```

`Transaction` 和数据库查询速记方法通常应在测试结束时的断言期间使用，此时您只需在执行SUT之后尝试验证底层数据库的新状态。

[`Validation` helper](https://bitbucket.org/headspring/headstart-employee-directory/src/35734e21140fb7bff6a7db61a0adf8a96a91f39a/src/EmployeeDirectory.Tests/Testing.cs#lines-176:213)
(_internal repository_)给出给定`command`或`query`的验证结果。 结合一些自定义的`ShouldValidate`和`ShouldNotValidate`
断言扩展（请参阅[Complete Assertions](/development-guidelines/testing/integration test-patterns.html#consure Complete Assertions)
部分）， 这使得编写表单测试变得很容易，“如果表单是这样填写的，我们预计它会因这些预期的错误消息而无法通过验证.“ 由于生产验证规则通常需要注入它们自己的依赖项，并且因为它们经常需要查询数据库，
所以validation（…）的工作方式是建立现在熟悉的数据库事务和IoC范围，像我们在生产中一样解析验证器，然后执行它
**测试只需要说“此表单是否有效。”**：

```csharp
public static ValidationResult Validation<TResult>(IRequest<TResult> message)
{
    using (var scope = ScopeFactory.CreateScope())
    {
        var serviceProvider = scope.ServiceProvider;

        var database = serviceProvider.GetService<DirectoryContext>();

        try
        {
            database.BeginTransaction();
            EmulateUserContextFilter(serviceProvider, database);


            var validator = Validator(serviceProvider, message);

            if (validator == null)
                throw new Exception($"There is no validator for {message.GetType()} messages.");

            var validationResult = validator.Validate(message);

            database.CloseTransaction();

            return validationResult;
        }
        catch (Exception exception)
        {
            database.CloseTransaction(exception);
            throw;
        }
    }
}

private static IValidator Validator<TResult>(IServiceProvider serviceProvider, IRequest<TResult> message)
{
    var validatorType = typeof(IValidator<>).MakeGenericType(message.GetType());
    return serviceProvider.GetService(validatorType) as IValidator;
}
```

我们最强大的两个助手, [overloads of `Send(...)`](https://bitbucket.org/headspring/headstart-employee-directory/src/35734e21140fb7bff6a7db61a0adf8a96a91f39a/src/EmployeeDirectory.Tests/Testing.cs#lines-70:120) (_
internal repository_), 构建在这些概念之上. 它们最终通过生产MediatR管道，在工作单元事务中，在作用域IoC容器中发送给定的`command`或`query`，就像生产一样

同样，与生产一样，验证规则是执行的，必须在我们费心将消息发送给处理程序之前通过。 这个验证检查可以防止我们在测试 _impossible_ 场景时编写通过的测试.
**只有当所讨论的场景是一个用户可以自己实现的场景时，您的测试才会通过.**

```csharp
public static async Task Send(IRequest message)
{
    using (var scope = ScopeFactory.CreateScope())
    {
        var serviceProvider = scope.ServiceProvider;

        var database = serviceProvider.GetService<DirectoryContext>();

        try
        {
            database.BeginTransaction();
            EmulateUserContextFilter(serviceProvider, database);
            Validator(serviceProvider, message)?.Validate(message).ShouldBeSuccessful();
            await serviceProvider.GetService<IMediator>().Send(message);
            database.CloseTransaction();
        }
        catch (Exception exception)
        {
            database.CloseTransaction(exception);
            throw;
        }
    }
}

public static async Task<TResponse> Send<TResponse>(IRequest<TResponse> message)
{
    TResponse response;

    using (var scope = ScopeFactory.CreateScope())
    {
        var serviceProvider = scope.ServiceProvider;

        var database = serviceProvider.GetService<DirectoryContext>();

        try
        {
            database.BeginTransaction();
            EmulateUserContextFilter(serviceProvider, database);
            Validator(serviceProvider, message)?.Validate(message).ShouldBeSuccessful();
            response = await serviceProvider.GetService<IMediator>().Send(message);
            database.CloseTransaction();
        }
        catch (Exception exception)
        {
            database.CloseTransaction(exception);
            throw;
        }
    }

    return response;
}
```

### **避免** 手动数据库设置

只要有可能，我们必须避免在代码中过度手动设置步骤，例如构造和填充一些实体类，并要求ORM将它们全部保存。建立一个不完整、不现实或不可能成为真实用户的场景太容易了。这些设置步骤也容易老化，因此，即使它们在第一次编写时是完整的和真实的，但随着周围系统和数据模型的增长，它们也会退化为不完整的。

相反，只要可能，通过使用 (`Send()`ing) 相同 `Command` 对象来设置测试场景. 用户会这样做。该场景模拟生产，我们进一步练习这些初步的命令处理程序，避免脆弱 并且通常将测试简化为用户操作的脚本。

避免诱人地使用AutoFixture等工具来用随机值完全填充给定类的许多属性。在简单的场景中，这似乎是对简洁性的明显胜利，但随着数据模型变得更加有趣，属性设置的不切实际性质会迅速退化，比如我们在ORM模型中看到的导航属性。你会发现你正在创建一个微妙的无效ORM模型，或者不得不为随机生成规则定义复杂的异常，而复杂的情况就不复存在了。相反，[开发易于推理的特定于应用程序的速记](https://bitbucket.org/headspring/headstart-employee-directory/src/35734e21140fb7bff6a7db61a0adf8a96a91f39a/src/EmployeeDirectory.Tests/Testing.cs#lines-215:250)
，以创建真实的类型示例实例：

```csharp
public static string SampleEmail() => SampleString() + "@example.com";
public static string SamplePassword() => SampleString();
public static string SampleFirstName() => SampleString();
public static string SampleLastName() => SampleString();
public static string SampleTitle() => SampleString();

public static Role SampleRole()
{
    return new Role
    {
        Name = SampleString()
    };
}

public static Employee SampleEmployee()
{
    return new Employee
    {
        Email = SampleEmail(),
        HashedPassword = HashPassword(SamplePassword()),
        FirstName = SampleFirstName(),
        LastName = SampleLastName(),
        Title = SampleTitle(),
        Office = Sample<Office>(),
        PhoneNumber = SamplePhoneNumber()
    };
}

private static string SampleString([CallerMemberName]string caller = null)
    => caller.Replace("Sample", "") + "-" + Guid.NewGuid();

public static TEnum Sample<TEnum>() where TEnum : struct
{
    var values = Enum.GetValues(typeof(TEnum));
    return (TEnum)values.GetValue(Random.Next(values.Length));
}

public static string SamplePhoneNumber()
    => $"({Random.Next(100, 1000)}) {Random.Next(100, 1000)}-{Random.Next(1000, 10000)}";
```

### **要** 为集成测试建立特定于域的语言

当您开始使用自己的命令进行测试设置时，您会发现自己引入了更多 最常见操作的缩写。这将建立一个_Domain Specific Language_，即您的“词汇表” 应用程序，进一步简化了每个新测试的编写和阅读。

在示例[员工目录](https://bitbucket.org/headspring/headstart-employee-directory/src/35734e21140fb7bff6a7db61a0adf8a96a91f39a/src/EmployeeDirectory.Tests/Testing.cs#lines-279:317)
（内部存储库）应用程序中， 我们发现，对于以各种用户身份登录、注册新员工和设置角色/权限等常见操作，使用简写非常有用。 请注意，即使是通过使用真实用户将通过UI调用的相同中介处理程序，而不是冒着保存填充不足的实体的风险，也可以实现这些功能：

```csharp
public static async Task<Employee> Register(Action<RegisterEmployee.Command> customize = null)
{
    var password = SamplePassword();

    var command = new RegisterEmployee.Command
    {
        Email = SampleEmail(),
        Password = password,
        ConfirmPassword = password,
        FirstName = SampleFirstName(),
        LastName = SampleLastName(),
        Title = SampleTitle(),
        Office = Sample<Office>(),
        PhoneNumber = SamplePhoneNumber()
    };

    customize?.Invoke(command);

    var employeeId = (await Send(command)).EmployeeId;

    return Query<Employee>(employeeId);
}

public static async Task<Employee> LogIn()
{
    var email = SampleEmail();
    var password = SamplePassword();

    var employee = await Register(x =>
    {
        x.Email = email;
        x.Password = password;
        x.ConfirmPassword = password;
    });

    await Send(new LogIn.Command { Email = email, Password = password });

    return employee;
}

public static async Task<Role> CreateRole(Action<CreateRole.Command> customize = null)
{
    var command = new CreateRole.Command
    {
        Name = SampleRole().Name
    };

    customize?.Invoke(command);

    var roleId = (await Send(command)).RoleId;

    return Query<Role>(roleId);
}

public static async Task AssignRoles(Employee employee, params Role[] roles)
{
    await Send(new RoleAssignment.Command
    {
        EmployeeId = employee.Id,
        Roles = roles.Select(x => new RoleSelection
        {
            RoleId = x.Id,
            Selected = true
        }).ToArray()
    });
}

public static async Task AssignPermissions(Role role, params Permission[] permissions)
{
    await Send(new PermissionAssignment.Command
    {
        RoleId = role.Id,
        Permissions = permissions
    });
}
```

### **要** 根据综合数据集进行测试

使用不完整数据集进行测试是持久性测试中最危险的陷阱。很容易错过已经存在但不明显的代码行为，因为我们从一个填充最少的数据库开始，或者更糟糕的是，空数据库。

* 小心使用`InMemoryContext`
  测试实体框架。这些测试实际上不会生成针对真实数据库执行的SQL，并且会遇到空数据库问题。参见[Limitations of EF In-memory test doubles](https://docs.microsoft.com/en-us/ef/ef6/fundamentals/testing/mocking#limitations-of-ef-in-memory-test-doubles)
*
时间耦合：添加自己的测试记录并围绕该数据断言的测试可能会出现这样的情况，即您意外地依赖于在同一事务周期中运行的另一个测试期间添加的数据。如果测试运行的顺序突然改变，您可能会看到新的、令人困惑的测试失败。请注意，您的测试用例与添加到特定测试之外的数据充分隔离
* 与处理程序将在生产环境中运行的数据库状态相比，在每个测试中从空数据库开始太简单了，因此很容易欺骗自己测试覆盖率 (例如已经讨论过的
  [DELETE-without-WHERE-clause scenario](/development-guidelines/testing/coverage.html#do-verify-the-full-range-of-outputs-and-side-effects))
  .
    * 这里的含意是: **谨慎使用 [Respawn](https://github.com/jbogard/Respawn) !**.
* 在可能的情况下，无论数据库中已有多少记录，都要编写有意义且一致的测试。CareerStart应用程序中有一些很好的例子：

[`ShouldGetAllEmployeesSortedByName()`](https://bitbucket.org/headspring/headstart-employee-directory/src/35734e21140fb7bff6a7db61a0adf8a96a91f39a/src/EmployeeDirectory.Tests/Features/Employee/EmployeeIndexTests.cs#lines-12:89)
(_internal repository_)

```csharp
public async Task ShouldGetAllEmployeesSortedByName()
{
    var patrickEmail = SampleEmail();
    var patrick = await Register(x =>
    {
        x.Email = patrickEmail;
        x.FirstName = "Patrick";
        x.LastName = "Zed";
        x.Title = "Principal Consultant";
        x.Office = Office.Austin;
        x.PhoneNumber = "555-123-0001";
    });

    var alonsoEmail = SampleEmail();
    var alonso = await Register(x =>
    {
        x.Email = alonsoEmail;
        x.FirstName = "Alonso";
        x.LastName = "Smith";
        x.Title = "Senior Consultant";
        x.Office = Office.Austin;
        x.PhoneNumber = "555-123-0002";
    });

    var sharonEmail = SampleEmail();
    var sharon = await Register(x =>
    {
        x.Email = sharonEmail;
        x.FirstName = "Sharon";
        x.LastName = "Smith";
        x.Title = "Principal Consultant";
        x.Office = Office.Dallas;
        x.PhoneNumber = "555-123-0003";
    });

    var expectedIds = new[] { patrick.Id, alonso.Id, sharon.Id };

    var query = new EmployeeIndex.Query();

    var result = await Send(query);

    result.Length.ShouldEqual(Count<Employee>());

    result
        .Where(x => expectedIds.Contains(x.Id))
        .ShouldMatch(
            new EmployeeIndex.ViewModel
            {
                Id = alonso.Id,
                FirstName = "Alonso",
                LastName = "Smith",
                Title = "Senior Consultant",
                Office = Office.Austin,
                Email = alonsoEmail,
                PhoneNumber = "555-123-0002"
            },
            new EmployeeIndex.ViewModel
            {
                Id = sharon.Id,
                FirstName = "Sharon",
                LastName = "Smith",
                Title = "Principal Consultant",
                Office = Office.Dallas,
                Email = sharonEmail,
                PhoneNumber = "555-123-0003"
            },
            new EmployeeIndex.ViewModel
            {
                Id = patrick.Id,
                FirstName = "Patrick",
                LastName = "Zed",
                Title = "Principal Consultant",
                Office = Office.Austin,
                Email = patrickEmail,
                PhoneNumber = "555-123-0001"
            }
        );
}
```

[`ShouldDeleteEmployeeById()`](https://bitbucket.org/headspring/headstart-employee-directory/src/35734e21140fb7bff6a7db61a0adf8a96a91f39a/src/EmployeeDirectory.Tests/Features/Employee/DeleteEmployeeTests.cs#lines-14:35)
(_internal repository_)

```csharp
public async Task ShouldDeleteEmployeeById()
{
    var employeeToDelete = await Register();
    var employeeToPreserve = await Register();
    await LogIn();

    var countBefore = Count<Employee>();

    await Send(new DeleteEmployee.Command
    {
        Id = employeeToDelete.Id
    });

    var countAfter = Count<Employee>();
    countAfter.ShouldEqual(countBefore - 1);

    var deletedEmployee = Query<Employee>(employeeToDelete.Id);
    deletedEmployee.ShouldBeNull();

    var remainingEmployee = Query<Employee>(employeeToPreserve.Id);
    remainingEmployee.ShouldMatch(employeeToPreserve);
}
```

### **要** 使用单独的开发和测试数据库

如果生成脚本仅设置一个本地数据库，则由测试运行和运行 应用程序本身，会造成不必要的障碍。假设您已经使用正在运行的应用程序设置了一个场景 对特性开发有用：仅仅运行测试套件就可以很容易地擦除或以其他方式使该特性无效
开发设置工作。拥有一个用于开发和测试的数据库也意味着，虽然长测试套件 运行时，您必须等待它完成，然后才能有意义地运行应用程序。

当构建脚本设置两个数据库时，一个专用于开发应用程序，另一个专用用于测试 跑步，你就能消除这些障碍。

### **要** 使用有意义的事务边界

事务很难设置，在测试中也很难模拟。理想状态下 您的应用程序将围绕MediatR处理程序和核心`Send`测试助手方法构建的条件，因此您已经
通过本节开头描述的测试框架处理的事务。这并不总是可能的。看当您围绕数据库事务设置任何测试时，请检查以下问题：

* 使用像EF这样的ORM，确保测试设置实际上是将数据持久化到数据库中，并且断言在单独的上下文中运行。换句话说，数据应该进行完整的往返，而不仅仅是往返于内存缓存。
* 使用像EF这样的ORM， 当实体实例跨多个`DbContext`共享时，很容易体验到令人惊讶的行为。 在可部署代码中，我们很少有多个`DbContext`
  在发挥作用，但测试经常会误用多个contexts。有关详细信息，请参阅以下[illustrative test](https://bitbucket.org/headspring/headstart/src/1bdb118daf023a9f5c92df2ed28f54c6c48a1b63/src/16.%20ORMs/5.%20LeakyAbstractionTests.cs#lines-73) (_
  internal repository_) 说明性测试。

### **避免** 每次测试使用自动事务回滚

在此策略中，事务设置为在每个测试开始时开始，并在 每次测试结束。尽管这一点乍看起来可能符合测试独立性的目标，但这种方法很快就实现了 故障。这对于生产场景来说是不现实的。当测试失败时，它还会隐藏调试信息，因为
测试失败中涉及的数据将在测试完成后随事务回滚。

### **避免** 在每个fixture或test（甚至更糟！） 使用Respawn

虽然 [Respawn](https://github.com/jbogard/respawn)
是一个很好的工具，可以让您在每一组测试之间有一个清晰的记录，但它可能会导致测试运行缓慢，即使是少量的测试，因为每次重新绘制数据库时，它都会生成额外的查询。根据重生文件

Respawn智能地检查SQL元数据，以基于表之间的外键关系构建要删除的表的确定顺序。 它导航这些关系以构建一个DELETE脚本，从没有关系的表开始，向内移动，直到所有表都被考虑在内。

很难将 **away** 从Respawn中移除, 因为它引导开发人员编写测试 假设数据库在每个测试用例开始时基本上是空的。如果你打算使用它，请考虑 在测试套件启动时只运行一次。这样做可以激励测试作者避免对数据库进行假设
因为套件中的其他测试可能会在整个运行过程中保存自己的任意记录。仅运行一次 在测试套件启动时，通过省略先前测试套件遗留的数据，还可以在调试期间帮助开发人员跑。

### **考虑** 完整声明

对测试的一个常见抱怨来自于测试的脆弱性。脆性试验是对SUT中的一些小变化做出反应而失败的试验。当您到达一个失败的测试时，却发现它不是以一种有用的方式失败，而是由于环境而失败，这可能会让您感到沮丧。

例如，如果一个测试做出了不必要的特定断言，那么当SUT的实现细节发生变化时，它们可能会开始失败。测试需要更新，以跟上设计的变化，即使功能的整体效果并没有真正改变，测试的预期声明也没有真正改变。这种经历挫败了一个好的测试的两个品质：在这样的失败中，它不能为我提供有用的信息，也不能让我有信心做出进一步的改变，因为它伤害太大了。

**然而，我们需要区分这种糟糕的脆性和实际上有助于我们保持良好测试套件的脆性。**

考虑一个今天测试完成的功能。它与今天的模式相反，适当地填充一些实体，并在一些SUT被执行之后断言实体的状态。尽管它今天已经完全测试了这个特性，但在模式改变之后，明天它可能会变得有意义地不完整。更糟糕的是，尽管它不再讲述完整的故事，但它仍然可能是不幸的巧合！这个测试本来就很脆弱：对SUT的一个小改动导致测试停止提供有意义的信息，无论是通过还是失败。模式更改导致测试的通过状态有点像谎言。是否应更新测试？远离的？通过附加测试增强？**
我们不知道，因为它还是绿色的**.

我们希望这种绿色脆性测试至少是红色脆性测试。这是一种我们可以重视的脆弱性：SUT中的一个有意义的变化会导致测试不完整，导致测试失败，因此我们至少知道它值得关注，以保持有意义。

在一个完美的世界里，我们甚至可以使它变得既不需要脆性，也不需要好的或坏的脆性。可以编写一些测试，使其始终完整。这并不总是可能的，但这是我们可以争取的目标。我们尽可能使用完全断言来实现这一点。

完整断言通常针对所讨论的应用程序进行定制，但它们最常见的形式是在两个复杂对象之间提供自动、深入的比较。例如，如果您的数据模型（部分）有一个自然的JSON表示，
您可以做出 [a useful “ShouldMatch” assertion](https://bitbucket.org/headspring/headstart-employee-directory/src/35734e21140fb7bff6a7db61a0adf8a96a91f39a/src/EmployeeDirectory.Tests/Assertions.cs#lines-19:32)
(_internal repository_) 它接受两个相同类型的复杂对象，将它们分别转换为JSON，并断言结果字符串相等：

```csharp
public static void ShouldMatch<T>(this IEnumerable<T> actual, params T[] expected)
    => actual.ToArray().ShouldMatch(expected);

public static void ShouldMatch<T>(this T actual, T expected)
{
    //Perform an initial deep copy of the given objects, to avoid
    //surprise members introduced by lazy load proxies.

    actual = DeepCopy(actual);
    expected = DeepCopy(expected);

    if (Json(expected) != Json(actual))
        throw new MatchException(expected, actual);
}
```

完整断言在测试验证规则时也很有价值。我们定义了 [Complete Validation Assertions ShouldValidate and ShouldNotValidate](https://bitbucket.org/headspring/headstart-employee-directory/src/35734e21140fb7bff6a7db61a0adf8a96a91f39a/src/EmployeeDirectory.Tests/Assertions.cs#lines-34:61)
(_internal repository):

```csharp
public static void ShouldValidate<TResult>(this IRequest<TResult> message)
    => Validation(message).ShouldBeSuccessful();

public static void ShouldNotValidate<TResult>(this IRequest<TResult> message, params string[] expectedErrors)
    => Validation(message).ShouldBeFailure(expectedErrors);

public static void ShouldBeSuccessful(this ValidationResult result)
{
    var indentedErrorMessages = result
        .Errors
        .OrderBy(x => x.ErrorMessage)
        .Select(x => "    " + x.ErrorMessage)
        .ToArray();

    var actual = String.Join(NewLine, indentedErrorMessages);

    result.IsValid.ShouldBeTrue($"Expected no validation errors, but found {result.Errors.Count}:{NewLine}{actual}");
}

public static void ShouldBeFailure(this ValidationResult result, params string[] expectedErrors)
{
    result.IsValid.ShouldBeFalse("Expected validation errors, but the message passed validation.");

    result.Errors
        .OrderBy(x => x.ErrorMessage)
        .Select(x => x.ErrorMessage)
        .ShouldMatch(expectedErrors.OrderBy(x => x).ToArray());
}
```

`ShouldMatch`可以提供有用的脆性。 当测试的目的是显示SUT对模型的全部影响时，与许多单独的属性断言相比，这种粗粒度比较更安全。 如果没有`ShouldMatch`，则无法知道需要为新属性添加新断言。
使用`ShouldMatch`，当您的预期值变得不完整时，您会收到警报。

验证助手类似地保护我们免受对验证规则的误导性绿色测试。 通过断言准确和完整的错误消息集， 我们知道验证器正在按预期将模型视为有效或无效， 出于我们所期望的原因。 避免仅根据模型是否有效的`bool`
进行验证测试；这样的测试很快就会变成误导性的绿色。

由于“ShouldMatch”本质上只执行一个大字符串比较，因此“Expected（long string），Actual（long
string）”的失败消息可能会使快速诊断出错误的原因变得困难。为了提供更好的开发人员体验，我们可以让测试运行在开发人员选择的diff工具中打开这样一个失败。测试失败了，我立即看到我的完整断言的哪一部分失败了，并立即着手解决更改。

这些示例中定义的“ShouldMatch”是一个很好的起点，但这可能会根据项目需求而有所不同。在[MongDb](https://www.mongodb.com/) 支持的项目中例如，您最好使用MongoDb
C#库自己的BSON字符串表示概念，这样就可以更完整地比较BSON类型。

在测试是否抛出了预期的异常时，请避免使用NUnit样式的[`ExpectedException`]
属性，而应使用断言库自己对异常进行断言的能力。这里的完整断言可以断言预期的异常类型和消息。这是一个判断，这样做是否过于脆弱。只需小心避免这种脆性是否会使测试变成不正确的绿色。例如，您可以通过断言异常消息的关键部分来进行折衷。

### **考虑** 因素包括实体持久性测试

当使用类似ORM的实体框架时，有很多编写看起来正确的实体代码的潜力，即使它无法正确持久。 例如，列名或属性名中的输入错误可能会阻止该列在往返数据库的“round trip”过程中持续存在。
遵循上述基于中介的集成测试指南，应该可以提供证明持久性所需的所有覆盖范围。如果没有，那么处理程序测试本身就严重不完整。

但是，您仍然可以考虑使用实体持久性测试来增强这些测试，其全部目的是确保给定的填充实体能够成功地往返于数据库。
CareerStart示例应用程序使用 [`ShouldPersist` assertion helper](https://bitbucket.org/headspring/headstart-employee-directory/src/35734e21140fb7bff6a7db61a0adf8a96a91f39a/src/EmployeeDirectory.Tests/Assertions.cs#lines-63:78)
(_internal repository_):

```csharp
public static void ShouldPersist<TEntity>(this TEntity entity) where TEntity : Entity
{
    entity.Id.ShouldEqual(Guid.Empty);

    Transaction(database => database.Set<TEntity>().Add(entity));

    entity.Id.ShouldNotEqual(Guid.Empty);

    Transaction(database =>
    {
        var loaded = database.Set<TEntity>().Find(entity.Id);

        loaded.ShouldMatch(entity);
    });
}
```

实体测试只需构造和填充代表性实例，然后调用`ShouldPersist`。在示例应用程序中， 我们对实体 `Employee`, `Role`, `RolePermission`, and `EmployeeRole`:

```csharp
public class EmployeeTests
{
    public void ShouldPersist()
    {
        var employee = SampleEmployee();

        employee.ShouldPersist();
    }
}

public class RoleTests
{
    public void ShouldPersist()
    {
        var role = SampleRole();

        role.ShouldPersist();
    }
}

public class RolePermissionTests
{
    public void ShouldPersist()
    {
        var rolePermission = new RolePermission
        {
            Role = SampleRole(),
            Permission = Sample<Permission>()
        };

        rolePermission.ShouldPersist();
    }
}

public class EmployeeRoleTests
{
    public void ShouldPersist()
    {
        var employeeRole = new EmployeeRole
        {
            Employee = SampleEmployee(),
            Role = SampleRole()
        };

        employeeRole.ShouldPersist();
    }
}
```

集成测试 示例

总而言之，我们有 [two representative integration tests from the sample application](https://bitbucket.org/headspring/headstart-employee-directory/src/35734e21140fb7bff6a7db61a0adf8a96a91f39a/src/EmployeeDirectory.Tests/Features/Employee/DeleteEmployeeTests.cs#lines-14:44) (_
internal repository_):

```csharp
public class DeleteEmployeeTests
{
    public async Task ShouldDeleteEmployeeById()
    {
        var employeeToDelete = await Register();
        var employeeToPreserve = await Register();
        await LogIn();

        var countBefore = Count<Employee>();

        await Send(new DeleteEmployee.Command
        {
            Id = employeeToDelete.Id
        });

        var countAfter = Count<Employee>();
        countAfter.ShouldEqual(countBefore - 1);

        var deletedEmployee = Query<Employee>(employeeToDelete.Id);
        deletedEmployee.ShouldBeNull();

        var remainingEmployee = Query<Employee>(employeeToPreserve.Id);
        remainingEmployee.ShouldMatch(employeeToPreserve);
    }

    public async Task ShouldNotAllowDeletingSelf()
    {
        var anotherEmployee = await Register();
        var self = await LogIn();

        new DeleteEmployee.Command { Id = anotherEmployee.Id }.ShouldValidate();
        new DeleteEmployee.Command { Id = self.Id }.ShouldNotValidate("Employees cannot delete themselves.");
    }

    ...
}
```
