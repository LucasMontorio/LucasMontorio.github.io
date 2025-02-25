##Context

Let's consider the following code, defining basic classes and one custom error:

```rb
class CustomError < StandardError; end

class Parent
  def raise_custom_error
    raise CustomError
  rescue StandardError
    puts "StandardError rescued"
  end
end

class Child < Parent
  def call
    raise_custom_error
  rescue CustomError
    puts "CustomError rescued"
  end
end
```

Now, let's examine the following method call::
```rb
Child.new.call
```

To summarize, this code does the following:

* Defines a `CustomError` class that inherits from `StandardError`
* Creates a `Child` class with a `#call` method that invokes  `raise_custom_error` from its parent class `Parent`
* Implements exception handling in both `Child#call` and `Parent#raise_custom_error`

Given this setup, we might expect the exception handling mechanism in `Child#call` to be triggered, as it appears to be the most specific. However, the actual output of the call is:

```rb
StandardError rescued
```

Surprisingly, we end up executing the more generic error handling defined in `Parent#raise_custom_error`.

&nbsp;

## How Rescuing Works
When Ruby encounters a `raise` statement, it **looks up the call stack** for matching `rescue` clauses. It checks whether the raised exception is a subclass of any exception specified in the `rescue` clause.

&nbsp;

## Explanation of the behaviour
The output "StandardError rescued" occurs because:

- When `Parent#raise_custom_error` raises `CustomError`, Ruby looks for matching rescue clauses in the method where the exception was raised
- It finds the `rescue StandardError` clause in `Parent#raise_custom_error`
- Since `CustomError` inherits from `StandardError`, this `rescue` clause matches and handles the exception
- By the time execution returns to `Child#call`, the exception has already been handled, so the rescue `CustomError` clause in `Child#call` never gets a chance to execute

&nbsp;

## Conclusion
Error-rescuing blocks cannot be written completely agnostically of the current class hierarchy. The order and specificity of `rescue` clauses matters due to exception inheritance.
When dealing with exceptions raised in parent classes, **the rescue block in the parent class may catch exceptions before they reach the child class**.

To avoid such scenarios, we can apply a principle related to exception hierarchy, which is closely tied to the Liskov Substitution Principle (LSP). The LSP states that subtypes should be substitutable for their base types, meaning that any code using a parent class should be able to work with a child class without knowing the difference.

In the context of exception handling specifically:

* More general exceptions should be caught at lower levels in the call stack or inheritance hierarchy.
* More specific exceptions should be caught closer to where they are raised.
