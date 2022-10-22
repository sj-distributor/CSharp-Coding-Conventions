---
layout: default
title: Good vs. Bad Tests
parent: Testing
grand_parent: Development Guidelines
nav_order: 1
---

# Good vs. Bad Tests
{: .no_toc }
What makes a good test good, and a bad test bad?

1. TOC
{: toc}

### **DO** write tests that provide useful information, both when they pass and when they fail
{: .text-green-100 }
When a test passes, that success needs to be telling us something valuable about the system. When a test fails, that failure needs to be telling us something valuable about the flawed system. Test failures should provide the developer with important clues in the test name and exception details. If it is difficult to determine why a test is failing, it’s not yet written well.

### **DO** write tests that give you the confidence to make changes
{: .text-green-100 }
A healthy test suite enables developers to make changes to the system with confidence. The test suite provides coverage so that negative consequences of a change can be detected immediately.

### **DO** write clear, concise, and clean tests
{: .text-green-100 }
We take our test code as seriously as our deployed code, for the same reasons. Tests follow all our normal code practices in terms of quality.

### **DO** make one logical assertion per test
{: .text-green-100 }
There’s a common rule that says each test should have only a single assertion.  We don’t take that literally: each test may have multiple `Assert` statements (or equivalent), but all of these assertions should be testing a single logical result.

For example: if you’re testing a result object that contains multiple properties, you may have an assertion on each property in order to assert the overall result matches the expected logical state.

### **DO** write tests that add value to the system
{: .text-green-100 }
We test methods and classes with critical business logic or core functionality in the system.  Testing these methods adds the most value to the project because bugs in the core of the project are going to be the most serious if not caught before we deliver.

Conversely, we don’t test trivial methods.  Tests on trivial methods simply add clutter, extra test runtime and are of no value to the project.  For instance, we wouldn’t write a test for this method, though its effects may appear in a more useful test of a more elaborate method:

```csharp
string Concat(string a, string b) { return a + b; }
```

### **DO** name tests clearly, as it is the first clue when diagnosing a test failure
{: .text-green-100 }
Tests are named after the thing they are asserting (more details below), so we can see at a glance when they fail what went wrong.

### **DO** write tests that execute consistently
{: .text-green-100 }
A test that passes should pass every time it is executed. A test that is failing should fail, and fail in the same way, every time it is executed.

### **DO** write tests that are independent
{: .text-green-100 }
Tests should be consistent regardless of which tests are being executed in one run, and regardless of the order they run.

### **AVOID** testing too many things at once
{: .text-red-300 }
Testing too many things at once leads to a test method that’s long and hard to read.  It also makes it more challenging to see if we’ve got full test coverage and are properly testing edge cases.

### **AVOID** writing tests that remain Green when the system under test experiences a breaking change
{: .text-red-300 }
If you can change your system under test in a way that should break your tests, and yet they remain green, that’s a sign of a serious problem with your test.  It’s likely such tests were testing trivial things, not making good assertions or not given valid test data to begin with.

### **DO** write tests for both “happy path” and edge cases
{: .text-green-100 }
If you’re only testing the results of your code when everything is running normally, you’re only testing half of your code.  Expect the unexpected - test the behavior of your code when it should fail.

Place the happy path tests before the edge case tests in the file, so that when reading the test class top to bottom, you learn about the feature as you would while reading a requirements document: “Normally it would go like this, but we do cover these related edge cases as well.”

### **AVOID** explicit loops and conditional logic within test methods
{: .text-red-300 }
One of the reasons we write tests is that it can be too easy to write production code with flawed logic. When our tests are just as complex as the system under test, we can easily write tests that are incorrect but still pass, giving us a false sense of security.

Sometimes, loops/conditionals are unavoidable in tests, or provide enough brevity that they become worthwhile, but we should emphasize straightforward code in tests as much as possible.

### **DO NOT** write tests that give you a false sense of security
{: .text-red-300 }
Good tests tell us useful information when they are passing, and they tell us useful information when they fail.

A bad test may be passing while telling us little of value. It may exercise a feature while making no assertions, or it may make irrelevant assertions while missing out on vital assertions. It may fail to truly exercise the intended feature or edge case. Having passing tests that provide little value gives us a false sense of security. We won’t know about the lurking untested reality until we really use the deployed system.

### **DO NOT** write tests to superficially meet an arbitrary code-coverage metric
{: .text-red-300 }
As we’ve discussed previously, we don’t like trivial tests because they just add clutter.  Generally, a code-coverage metric is an arbitrary target and ultimately encourages poor tests.  We care about quality more than volume.

### **DO** write async tests for async systems
{: .text-green-100 }
All .NET test frameworks allow test methods to be declared async. When a test calls into an async system, the test method should naturally be `async` as well. Do not, instead, block on `async` systems with `Task.Wait()`, `Task.Result`, or `Task.GetAwaiter().GetResult()`. Instead, as we do with controller actions in MVC, simply allow the surrounding test framework to handle the returned `Task` appropriately.

### **DO** leverage test framework support for async setup
{: .text-green-100 }
With xUnit and Fixie, test class constructors are the default place to put common setup steps to be performed
before each test case executes. 
However, since most of our integration test setup is going to involve a 
MediatR `Send` or other `async` call, constructor setup is impossible. An `async` call in a
constructor wouldn’t even compile!

With xUnit, you can rely on [IAsyncLifetime](https://stackoverflow.com/a/45906269)

With Fixie, we can create our [own convention](https://github.com/fixie/fixie/wiki#putting-it-all-together) that any test method named `SetUp` should be treated as a setup method rather than be treated as a test.
