
# Fakes测试


### **要** 在出现困难的有依赖的地方使用Fakes测试

Fakes测试是某些接口的特定于测试的实现。当运行的应用程序使用该接口的真实实现时，测试可以自由地将其替换为自己的伪实现。
这对于练习代码和某些外部系统之间的交互非常有用。
例如，断言您的系统已经发送了一封真实的电子邮件是不切实际的，但使用 `IEmailSender` 接口来表示向某个收件人发送某些有效载荷的请求是合理的。
`IEmailSender`的实际实现相对较小，只执行实际发送电子邮件这一难以测试的行为。
位于测试项目中的“`IEmailSender`”的测试特定实现仅表示发送电子邮件的请求。
它可以保留传递给它的参数并公开它们，以便测试可以断言预期的接收者、正文、附件等实际上已经到达了实现。
对正在运行的系统进行简单的手动测试可以确认真正的`IEmailSender`的实际行为，而自动测试至少可以证明系统与电子邮件发送进行了适当的交互。
对系统的外部依赖项使用测试假，如发送电子邮件、调用外部API以及与系统时钟交互。
[CareerStart](https://bitbucket.org/headspring/headstart-employee-directory/src/master/) (_Internal Repository_)示例应用程序使用一个test fake来测试我们与ASP的交互。
NET自己的登录cookie基础结构。由于测试不是在真正的web服务器中的真正的web应用程序中运行，因此内置的cookie处理方法在测试时将毫无意义。
相反，我们使用`ILoginService`接口将自己与这些内置方法隔离开来。

[ILoginService](https://bitbucket.org/headspring/headstart-employee-directory/src/35734e21140fb7bff6a7db61a0adf8a96a91f39a/src/EmployeeDirectory/Infrastructure/LoginService.cs#lines-16:50)
(_Internal Repository_)的实际实现调用ASP。NET方法：
```csharp
public class LoginService : ILoginService
{
    private const string AuthenticationScheme =
        CookieAuthenticationDefaults.AuthenticationScheme;

    private readonly IHttpContextAccessor _httpContextAccessor;

    public LoginService(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }

    public async Task LogIn(string email)
    {
        var claims = new List<Claim>
        {
            new Claim(ClaimTypes.Name, email)
        };

        var claimsIdentity = new ClaimsIdentity(claims, AuthenticationScheme);
        var principal = new ClaimsPrincipal(claimsIdentity);

        var properties = new AuthenticationProperties
        {
            IsPersistent = true
        };

        await _httpContextAccessor.HttpContext
            .SignInAsync(AuthenticationScheme, principal, properties);
    }

    public async Task LogOut()
    {
        await _httpContextAccessor.HttpContext
            .SignOutAsync(AuthenticationScheme);
    }
}
```

[`LoginService`](https://bitbucket.org/headspring/headstart-employee-directory/src/35734e21140fb7bff6a7db61a0adf8a96a91f39a/src/EmployeeDirectory.Tests/Infrastructure/StubLoginService.cs#lines-6:26) (_Internal Repository_) 伪造测试实现   
模拟登录cookie，只需保存存储在真实cookie中的电子邮件地址，暴露该字符串，以便测试可以断言：

```csharp
public class StubLoginService : ILoginService
{
    public string AuthenticatedEmail { get; private set; }

    public Task LogIn(string email)
    {
        AuthenticatedEmail = email;
        return Task.CompletedTask;
    }

    public Task LogOut()
    {
        AuthenticatedEmail = null;
        return Task.CompletedTask;
    }

    public void Reset()
    {
        AuthenticatedEmail = null;
    }
}
```

### **不要** 使用 Fake/Mock 库

像 [FakeItEasy](https://fakeiteasy.github.io/) 这样的库允许您在不实际提供相关接口的具体测试实现的情况下完成测试伪造。
它们允许您的测试描述您希望这样的测试实现如何运行，并为您创建实现：

```csharp
var lollipop = A.Fake<ICandy>();
var shop = A.Fake<ICandyShop>();

A.CallTo(() => shop.GetTopSellingCandy()).Returns(lollipop);
//...
A.CallTo(() => shop.BuyCandy(lollipop)).MustHaveHappened();
```
对于非常简单的例子来说，这些库似乎可以支持。然而，在实践中，它们都很难正确使用。
“魔力”在很大程度上依赖于C#参数列表的求值顺序以及库中看不见的副作用，因此即使是简单的“安全”重构，比如“引入变量”，也可能成为破坏性的更改。
对这些库的一个常见的批评是，你真正测试的唯一东西是你使用模拟库的能力，而事实上，不管你知道与否，你都在失败。
这些无声无息的测试在通过时给了我们虚假的信心。