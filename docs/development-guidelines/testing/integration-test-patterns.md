---
layout: default
title: Integration Test Patterns
parent: Testing
grand_parent: Development Guidelines
nav_order: 9
---

# Integration Test Patterns
{: .no_toc}
Although specific technologies vary greatly across projects, a few core principles have driven our most effective integration tests to follow similar patterns.

### Integration Testing Principles
{: .no_toc}

* Exercise the production scenario as much as possible.
* Develop “shorthand” test helper functions so that tests can read like a script of user interactions.

No two apps are the same, so the details will vary, but it helps to focus the discussion on several high-value specifics that we see in many projects. Specific MVC versions, specific IoC tools, specific databases will vary, but when we apply the above principles to these variations, we can still arrive at very similar patterns in our tests.

Consider a typical modern web application: ASP.NET MVC Core with its built-in IoC container, a SQL Server database, and a useful “unit of work” aligned with each web request. Each web request is set up to have a dedicated SQL transaction opened near the start of the request and committed/rolled back at the end, so that most of our code need not be concerned with transactions at all. Aligned with that transaction, we have a scoped IoC container so that each web request has its own dedicated container based on the “global” one initialized during application startup. Controllers use [MediatR](https://github.com/jbogard/MediatR) to `Send` `Command` and `Query` objects to their respective Handlers, where real work is performed. These handlers may be protected by some validation rule library like [FluentValidation](https://fluentvalidation.net/).

With that application in mind, we’ll often have test classes 1-1 with message handler classes. We won’t have automated tests of the controllers: these only really behave under the context of a real web request in a running web server. Thankfully the use of MediatR makes the controllers quite small. We’ll be satisfied with our handler test coverage so long as we do run the application and exercise the controller as part of a typical QA effort.

If our primary integration testing effort is in testing MediatR handlers, then, it may be tempting, then to write a test by explicitly constructing a MediatR handler, passing its dependencies both real and stub to the constructor, explicitly calling the `Handle` method, and asserting on the response and side effects. Doing so would be a mistake, as it violates both of our integration testing principles: it does not mimic production, and it is verbose.

It would be better if a test could simply `Send` the relevant `Command` or `Query` object just like a `Controller` class would, fully exercising the MediatR pipeline, just like in production. It would be better if every such `Send` automatically took part in a unit of work transaction, just like in production. It would be better if every such `Send` also exercised any validation rules before allowing the handler to run, just like in production. It would be better if every such `Send` also executed in the context of a dedicated, scoped IoC container, just like in production. Lastly, it would be better if we could accomplish all of this in an exceptionally small number of characters, so that the test author can simply focus on the scenario at hand rather than any of these cross-cutting concerns.

All of these principles can be seen in the [CareerStart Employee Directory](https://bitbucket.org/headspring/headstart-employee-directory)(_internal repository_) sample application.

Here, we’ll review the infrastructure for the Employee Directory tests, then witness some representative tests, and finally sum up why this approach is so effective in practice. At any point, we can tie back our decision making to the two Integration Testing Principles above.

We’ll achieve shorthand in our tests by defining static class `Testing` which most test files will include with `using static Testing;`. All public methods in this class can therefore be called from any test, as if they were global functions. The static constructor of `Testing` will establish some fundamentals used by the many global helper functions.

Applying these patterns in our integration tests results in tests that are easy to write, exercise as much of the production code as we realistically can, protect themselves from testing unrealistic or impossible scenarios, and remain meaningfully red or green throughout the life of a growing project. Armed with this style, we rarely need to reach for inheritance or test framework-specific oddities like `TestFixtureSetUp`.

1. TOC
{: toc}

### **DO** Initialize integration test runs using the same setup as the application
{: .text-green-100 }
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

The [`static Testing constructor`](https://bitbucket.org/headspring/headstart-employee-directory/src/35734e21140fb7bff6a7db61a0adf8a96a91f39a/src/EmployeeDirectory.Tests/Testing.cs#lines-27:46)(_internal repository_) establishes the test project’s config file in exactly the same way as the running web application.

Then, we set up the root IoC container that will be used across all tests. We don’t do so merely by performing similar IoC setup as the core app, but instead by calling into the same setup code exercised when the web app starts up in production. Yes, we can follow that up with a rare test stub or two registered with the container, overriding the production configuration, but only when the production setup poses too severe of an obstacle such as for interfaces dealing with third party APIs. In this example, we introduce a stub `ILoginService`, allowing tests to simulate the login cookie that would exist at runtime.

### **DO** establish core integration test helpers: Transaction, Query, Validator, and Send
{: .text-green-100 }
Next, we layer in a few core test helper methods.

We define persistence test helper methods. Their details may vary from one project to another as IoC and persistence technologies change, but their purpose is the same from project to project:

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

The `Transaction` helper method allows a test to trivially run any code within the context of a web-request-like unit of work. The supplied action runs within a dedicated transaction, just like production, and a dedicated scoped IoC container, just like production. Built on top of this, we have additional shorthand for common database interactions:

```csharp
var entity = Query<TEntity>(id);

var result = Query(db => …);

var count  = Count<TEntity>();
```

`Transaction` and the database querying shorthand methods should mostly be used during the assertions at the end of your test, when you’re simply trying to verify the new state of the underlying database after exercising the SUT.

The [`Validation` helper](https://bitbucket.org/headspring/headstart-employee-directory/src/35734e21140fb7bff6a7db61a0adf8a96a91f39a/src/EmployeeDirectory.Tests/Testing.cs#lines-176:213)(_internal repository_) 
gives you the validation result for a given `command` or `query`. 
Combined with some custom `ShouldValidate` and `ShouldNotValidate` assertion extensions (see the [Complete Assertions](/development-guidelines/testing/integration-test-patterns.html#consider-complete-assertions) section), 
this makes it easy to write tests of the form, 
“Given a form filled out like so, we expect it to fail validation with these expected error messages.” 
Since production validation rules often need to inject dependencies of their own,
and because they often need to query the database, 
Validation(...) works by establishing the now-familiar database transaction and IoC scope, 
resolving the validator as we would in production, and then exercising it.
**Tests need only say, “This form should validate or not.”**:

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

Our two most powerful helpers, [overloads of `Send(...)`](https://bitbucket.org/headspring/headstart-employee-directory/src/35734e21140fb7bff6a7db61a0adf8a96a91f39a/src/EmployeeDirectory.Tests/Testing.cs#lines-70:120)(_internal repository_), build atop these concepts. They ultimately send a given `command` or `query` through the production MediatR pipeline, in a unit of work transaction, in a scoped IoC container, just like production. Also just like production, validation rules are executed and must pass before we bother to send the message to its handler. This validation check protects us from writing tests that pass while testing _impossible_ scenarios. **Your test will only pass if the scenario in question is one that a user could actually arrive at themselves.**

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

### **AVOID** manual database setup
{: .text-red-300 }
Whenever possible, we must avoid overly-manual setup steps in our code, such as constructing and filling in a few entity classes and asking our ORM to save them all. It is far too easy to set up a scenario that is incomplete, unrealistic, or impossible to arrive at as a real user. These setup steps also tend to age poorly, such that even if they are complete and realistic when first written, they degrade to being incomplete as the surrounding system and data model grows.

Instead, whenever possible, set up the scenario under test by exercising (`Send()`ing) the same `Command` objects that the user would. The scenario mimics production, we further exercise those preliminary command handlers, avoid brittle setup, and generally simplify the test as a script of user actions.

Avoid the tempting use of tools like [AutoFixture](https://github.com/AutoFixture/AutoFixture) to fully populate a given class’s many properties with random values. In simple scenarios it seems like a clear win on brevity, but the unrealistic nature of the property setting quickly degrades as a data model becomes more interesting, such as we see along navigation properties in ORM models. You’d find that you were creating subtly-invalid ORM models, or having to define complex exceptions-to-the-random-generation-rules, and the complexity simply gets away from you. Instead, develop [easily-reasoned-about application-specific shorthand](https://bitbucket.org/headspring/headstart-employee-directory/src/35734e21140fb7bff6a7db61a0adf8a96a91f39a/src/EmployeeDirectory.Tests/Testing.cs#lines-215:250)(_internal repository_) for creating realistic sample instances of types:

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

### **DO** establish a Domain Specific Language for integration tests
{: .text-green-100 }
As you start to use your own Commands for test setup, you’ll find yourself introducing even more shorthand for the most common operations. This builds up a _Domain Specific Language_, the “vocabulary” of your application, further simplifying the writing and reading of each new test.

In the sample Employee Directory application, we found it useful to have [shorthand for the common actions of logging in as various users, registering new employees](https://bitbucket.org/headspring/headstart-employee-directory/src/35734e21140fb7bff6a7db61a0adf8a96a91f39a/src/EmployeeDirectory.Tests/Testing.cs#lines-279:317)(_internal repository_), and setting up roles/permissions. Note how even these work by exercising the same Mediatr handlers that a real user would invoke through the UI, rather than riskily saving poorly-populated entities:

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

### **DO** test against a comprehensive data set
{: .text-green-100 }
Testing with an incomplete data set is the most dangerous pitfall in persistence testing.  It’s very easy to miss code behaviors there are already present but not apparent because we’re starting with a minimally populated database, or perhaps worse, empty database.

* Beware testing Entity Framework using an `InMemoryContext`.  These tests won’t actually generate SQL executed against a real database, and they will suffer from the empty database problem.  See [Limitations of EF In-memory test doubles](https://docs.microsoft.com/en-us/ef/ef6/fundamentals/testing/mocking#limitations-of-ef-in-memory-test-doubles)
* Temporal coupling: Tests that add their own test records and assert around this data can be prone to a situation where you accidentally rely on data added during another test running in same transaction cycle.  If the order in which your tests run suddenly changes, you may see new, confusing test failures. Pay careful attention that your test cases are sufficiently isolated from data added outside of your specific test
* Starting with an empty database on each test is too simple compared to the DB state that your handlers will run under in production, so it’s easy to fool yourself that your test coverage is right (e.g. the [DELETE-without-WHERE-clause scenario](/development-guidelines/testing/coverage.html#do-verify-the-full-range-of-outputs-and-side-effects) already discussed).
  * The implication here: **Use [Respawn](https://github.com/jbogard/Respawn) sparingly!**.
* When possible, write your test to be meaningful and consistent no matter how many records are already in the database. There are good examples in the CareerStart app:

[`ShouldGetAllEmployeesSortedByName()`](https://bitbucket.org/headspring/headstart-employee-directory/src/35734e21140fb7bff6a7db61a0adf8a96a91f39a/src/EmployeeDirectory.Tests/Features/Employee/EmployeeIndexTests.cs#lines-12:89)(_internal repository_)

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

[`ShouldDeleteEmployeeById()`](https://bitbucket.org/headspring/headstart-employee-directory/src/35734e21140fb7bff6a7db61a0adf8a96a91f39a/src/EmployeeDirectory.Tests/Features/Employee/DeleteEmployeeTests.cs#lines-14:35)(_internal repository_)

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

### **DO** use separate Development and Test databases
{: .text-green-100 }
If your build script only sets up one local database, shared by test runs and by the running application itself, you create unnecessary obstacles. Imagine you have used the running application to set up a scenario useful to feature development: merely running the test suite could easily erase or otherwise invalidate that feature development setup effort. Having a single database for development and tests also means that while a long test suite is running, you have to wait for it to finish before you can meaningfully run the application.

When your build script sets up two databases, one dedicated to developing the application, and one dedicated to test runs, you eliminate these obstacles.

### **DO** use meaningful transaction boundaries
{: .text-green-100 }
Transactions are tricky to set up, and they are also tricky to mock in tests.  Under ideal conditions your app would be built around MediatR handlers and the core `Send` test helper method, so you’d already have Transactions handled via the test framework described at the outset of this section. That’s not always possible. Look out for the following problems when you setup any testing around database transactions:

* With ORMs like EF, make sure your test setup is actually persisting data to the database and that your assertions are running in a separate context.  In other words, the data should make a full round trip, not just a trip to the in-memory cache and back.
* With EF, it is easy to experience surprising behavior when entity instances are shared across multiple `DbContext`s. We rarely have multiple `DbContext`s in play in deployable code, but tests often fall into the trap of misusing multiple contexts. See the following [illustrative test](https://bitbucket.org/headspring/headstart/src/1bdb118daf023a9f5c92df2ed28f54c6c48a1b63/src/16.%20ORMs/5.%20LeakyAbstractionTests.cs#lines-73)(_internal repository_) for more.

### **AVOID** using an automated transaction rollback per test
{: .text-red-300 }
In this strategy, a transaction is set up to start at the beginning of each test, and roll back at ehe end of each test. Although that may at first appear to meet the goal of test independence, the approach quickly breaks down. It is not realistic to production scenarios. It also hides debugging information when a test fails, because the data involved in the test failure will be rolled back with the transaction after the test completes.

### **AVOID** using Respawn between each fixture or (even worse!) test
{: .text-red-300 }
While [Respawn](https://github.com/jbogard/respawn) is a great tool for giving you a clean slate between each set of tests, it can lead to slow test runs with even a modest number of tests as it generates extra queries each time the database is respawned.  Per the Respawn documentation

Respawn examines the SQL metadata intelligently to build a deterministic order of tables to delete based on foreign key relationships between tables. It navigates these relationships to build a DELETE script starting with the tables with no relationships and moving inwards until all tables are accounted for.
{: .fs-3 .bg-grey-lt-000 .p-2 .fw-500}

It can be difficult to move **away** from Respawn, since it leads developers to write tests that make lots of assumptions about the database being largely empty at the start of each test case. If you’re going to use it, consider running it only once on test suite startup. Doing so motivates test authors to avoid making assumptions about database state, since other tests in the suite may be saving arbitrary records of their own throughout the run. Running only once on test suite startup also assists the developer during debugging, by omitting data left over from previous test suite runs.

### **CONSIDER** Complete Assertions
{: .text-yellow-300 }
A common complaint about tests comes when they are brittle. A brittle test is one that fails in response to some small change in the SUT. It can be frustrating when you arrive at a failing test only to realize that it is not failing in a _useful_ way but merely failing due to circumstance.

For instance, if a test makes assertions that are needlessly specific, they may begin to fail when the implementation details of the SUT change. The test needed to be updated just to keep up with the design change, even though the overall effect of the feature didn’t really change and the _intended_ claims made by the test didn’t really change. This experience thwarts the two qualities of a good test: it’s not providing me with useful information during such failures, and it’s not giving me the confidence to make further changes as it hurts too much.

**We need to distinguish, though, between this bad kind of brittleness and the kind of brittleness that actually _helps_ us to keep a sound test suite.**

Consider a feature whose test is complete today. It works against today’s schema, filling in some entities appropriately and asserting on the entities’ state after some SUT gets exercised. Although it’s completely testing the feature today, it may become meaningfully incomplete tomorrow after the schema changes. Worse yet, although it’s no longer telling the full story, it may still be passing by unfortunate coincidence! This test was originally brittle in a bad way: a small change to the SUT caused the test to stop providing meaningful information whether passing or failing. The schema change caused the test’s passing status to become a bit of a lie. Should the test be updated? Removed? Augmented with additional tests? **We don’t know, because it is still green**.

We would rather this green brittle test at least be a red brittle test. This is the kind of brittleness we can value: a meaningful change in the SUT, which would have caused the test to be incomplete, causes the test to fail so that we at least know that it deserves attention to remain meaningful.

In a perfect world, we could even make it so that neither kind of brittleness, neither good nor bad brittleness, was even necessary. Some tests can be written so that they are always complete. It’s not always possible, but a goal we can strive for. We do so by using Complete Assertions whenever possible.

Complete Assertions are often tailored to the application in question, but their most common form is to provide an automatic, deep comparison between two complex objects. If (part of) your data model has a natural JSON representation, for instance, you can make [a useful “ShouldMatch” assertion](https://bitbucket.org/headspring/headstart-employee-directory/src/35734e21140fb7bff6a7db61a0adf8a96a91f39a/src/EmployeeDirectory.Tests/Assertions.cs#lines-19:32)(_internal repository_) that takes two complex objects of the same type, turns them each to JSON, and asserts on the resulting strings being equal:

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

Complete Assertions are also valuable when testing validation rules. We define [Complete Validation Assertions ShouldValidate and ShouldNotValidate](https://bitbucket.org/headspring/headstart-employee-directory/src/35734e21140fb7bff6a7db61a0adf8a96a91f39a/src/EmployeeDirectory.Tests/Assertions.cs#lines-34:61)(_internal repository):

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

`ShouldMatch` can provide the useful kind of brittleness. When the intent of your test is to show the full impact of a SUT on your model, you are safer with this course-grained comparison than you would be with many individual property assertions. Without `ShouldMatch`, you have no way of knowing that you need to add a new assertion for a new property. With `ShouldMatch`, you’re alerted the moment that your expected value became incomplete.

The validation helpers similarly protect us from misleadingly-green tests of validation rules. By asserting on the exact and complete set of error messages, we know that the validator is treating the model as valid or invalid as expected, and for the reason we expected. Avoid validation testing that only asserts on the `bool` of whether a model is valid or not; such tests can quickly become misleadingly-green.

Since `ShouldMatch` essentially performs one large string comparison, failure messages of the “Expected (long string), Actual (long string)” can make it hard to quickly diagnose what went wrong. To provide a better developer experience, we can make it so that the test run opens such a failure in the developer’s diff tool of choice. The test fails, I see right away what part of my complete assertion failed, and get right into addressing the change.

`ShouldMatch` as defined in these examples is a good starting point, but this can vary based on project needs. In a project backed by [MongDb](https://www.mongodb.com/), for instance, you’d be better off using the MongoDb C# library’s own notion of BSON string representations, so that the comparisons will be even more complete with respect to BSON types.

When testing that an expected exception is thrown, avoid NUnit-style `[ExpectedException]` attributes, instead using your assertion library’s own ability to assert on exceptions. A complete assertion here may assert on both the expected exception type and the message. It is a judgement call whether doing so would be overly brittle. Simply be wary of whether your avoiding that brittleness allows the test to become incorrectly green. For instance, you may compromise by asserting on a pivotal part of an otherwise brittle exception message.

### **CONSIDER** including entity persistence tests
{: .text-yellow-300 }
When working with an ORM like Entity Framework, there’s a lot of potential for writing entity code that looks right even though it will fail to persist correctly. For instance, typos in a column name or property name may keep the column from persisting and surviving a “round trip” to the database and back.

Following the above guidelines for Mediatr-based integration tests should provide all the coverage you need to prove persistence. If not, then the handler tests themselves are seriously incomplete.

However, you may still consider augmenting those tests with entity persistence tests, whose entire purpose is to ensure that a given populated entity will successfully round-trip to the database and back. The CareerStart sample application does so with a [`ShouldPersist` assertion helper](https://bitbucket.org/headspring/headstart-employee-directory/src/35734e21140fb7bff6a7db61a0adf8a96a91f39a/src/EmployeeDirectory.Tests/Assertions.cs#lines-63:78) (_internal repository_):

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

Entity tests simply construct and populate representative instances and then call `ShouldPersist`. In the sample application, we do so for entities `Employee`, `Role`, `RolePermission`, and `EmployeeRole`:

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

Sample Integration Tests
Putting it all together, here we have [two representative integration tests from the sample application](https://bitbucket.org/headspring/headstart-employee-directory/src/35734e21140fb7bff6a7db61a0adf8a96a91f39a/src/EmployeeDirectory.Tests/Features/Employee/DeleteEmployeeTests.cs#lines-14:44)(_internal repository_):

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
