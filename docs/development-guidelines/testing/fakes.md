---
layout: default
title: Test Fakes
parent: Testing
grand_parent: Development Guidelines
nav_order: 8
---

# Test Fakes
{: .no_toc}

1. TOC
{: toc}

### **DO** Use test fakes around difficult dependencies
{: .text-green-100 }
A test fake is a test-specific implementation of some interface. While the running application uses a real implementation of that interface, the tests are free to replace that with a fake implementation of their own. This is useful for exercising the interactions between your code and some external system. For instance, it is impractical to assert that a real email has been sent by your system, but it is reasonable to use an `IEmailSender` interface to represent the request to email some payload to some recipient. The real implementation of `IEmailSender` is relatively small, performing only the hard-to-test act of actually sending an email. The test specific implementation of `IEmailSender`, located in the test project, merely represents the request to send the email. It can hold onto the arguments passed to it and expose them so that tests can assert that the expected recipient, body, attachments, etc, are in fact arriving at the implementation. A simple manual test of the running system can confirm the actual behavior of the real `IEmailSender`, while the automated tests at least prove that the system interacts with email sending appropriately.

Use test fakes for the system’s external dependencies, like sending email, calling external APIs, and interacting with the system clock.

The [CareerStart](https://bitbucket.org/headspring/headstart-employee-directory/src/master/)(_Internal Repository_) sample application uses a test fake to test our interacting with ASP.NET’s own login cookie infrastructure. Since the tests are not running in a real web application in a real web server, the built-in cookie handling methods would be meaningless at test time. We instead insulate ourselves from those built-in methods with the `ILoginService` interface.

The real implementation of [ILoginService`](https://bitbucket.org/headspring/headstart-employee-directory/src/35734e21140fb7bff6a7db61a0adf8a96a91f39a/src/EmployeeDirectory/Infrastructure/LoginService.cs#lines-16:50) (_Internal Repository_) calls ASP.NET methods:

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

The test fake implementation of [`LoginService`](https://bitbucket.org/headspring/headstart-employee-directory/src/35734e21140fb7bff6a7db61a0adf8a96a91f39a/src/EmployeeDirectory.Tests/Infrastructure/StubLoginService.cs#lines-6:26)(_Internal Repository_)  simulates the login cookie by merely holding the email address that would have been stored in the real cookie, exposing that string so that tests can assert on it:

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

### **DO NOT** use Fake/Mock libraries
{: .text-red-300 }
Libraries like [FakeItEasy](https://fakeiteasy.github.io/) allow you to accomplish test fakes without actually providing a concrete test implementation of the interface in question. They allow your test to describe how you wish such a test implementation would behave, and they create the implementation for you:

```csharp
var lollipop = A.Fake<ICandy>();
var shop = A.Fake<ICandyShop>();

A.CallTo(() => shop.GetTopSellingCandy()).Returns(lollipop);
//...
A.CallTo(() => shop.BuyCandy(lollipop)).MustHaveHappened();
```

For extremely simple examples, these libraries appear to hold up. In practice, though, they have all been notoriously difficult to use correctly. The “magic” relies a great deal on the order of evaluation of C# parameter lists in combination with unseen side effects within the libraries, so that even simple “safe” refactorings like “Introduce Variable” can become breaking changes. A common criticism of these libraries is that the only thing you’re really testing is your ability to use the mocking library, and you are in fact failing at it whether you know it or not. Such silently broken tests give us false confidence when they pass.
