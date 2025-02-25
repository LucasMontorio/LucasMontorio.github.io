*Disclaimer: This article proposes a straightforward approach to request validation in Rails controllers. While it doesn't claim to be a perfect solution for every scenario, it presents a simple pattern that can be applied to many cases and help simplify controller responsibilities.*

## Observation

When dealing with complex Rails controllers, finding an elegant and maintainable way to perform validations on incoming requests can be challenging, be it to simply check the presence of mandatory attributes or to deal with more intricate business-logic checks.

Let’s consider this basic controller as an example:

```ruby
class SimpleController < ApplicationController
  def perform_computation
    service = MyBusinessLogicService(
      param1: params[:param1],
      param2: params[:param2],
      param3: params[:param3]
    ).new
    result = service.perform

    respond_to do |format|
      format.json { render json: result }
      format.html { render text: result.to_s }
    end
  end
end
```

It defines a simple action that calls a business-logic service and returns the result.

Now, suppose we need to perform various validations, of different natures:

- Check that `param1` and `param2` are present
- Check that the current user has the permission to perform this action, depending on its role (RBAC)
- Check that the current user has an attribute `active` set to `true`

Given the nature of the validations at stake here (input-related validation, session-related validation, and business-logic validation), we are likely to come to the conclusion that the controller is not the ideal place to perform them.

The controller's primary role should focus on instantiating/updating/deleting objects, invoking services or libraries containing business logic, and formatting output data before rendering.

_Disclaimer: In real-life cases, parameter validation and authorisation checks are likely to be split and/or factored out of the controllers. I am including them in the same validator here for the sake of the example, by lack of more business logic checks._


## The validator pattern

Instead of cluttering the controller action with validation-related code, let’s define the following ‘validator’ class:

```ruby
class RequestValidator
  class UnauthorizedError < StandardError; end
  class InactiveUserError < StandardError; end

  def initialize(param1:, param2:, param3:, user:)
    @param1 = param1
    @param2 = param2
    @param3 = param3
    @user = user
  end

  def call!
    validate_required_params!
    validate_user_role!
    validate_user_is_active!
  end

  private

	  def validate_required_params!
	    raise ParameterMissing, :param1 if @param1.blank?
	    raise ParameterMissing, :param2 if @param2.blank?
	  end

	  def validate_user_role!
	    raise UnauthorizEderror, 'User not allowed to perform this action' unless RoleChecker(user: user).allowed?
	  end

	  def validate_user_is_active!
	    raise InactiveUserError, "User is not active" unless @user.active?
	  end
end
```

Instead of having a bloated controller, we can now plug validations in a very light fashion:

```ruby
class SimpleController < ApplicationController
  def perform_computation
    validate_computational_request!

    service = MyBusinessLogicService(
      param1: params[:param1],
      param2: params[:param2],
      param3: params[:param3]
    ).new
    result = service.perform

    respond_to do |format|
      format.json { render json: result }
      format.html { render text: result.to_s }
    end

  rescue RequestValidator::UnauthorizedError => e
    render json: { error: e.message }, status: :unauthorized
  rescue RequestValidator::InactiveUserError => e
    # Some custom logic
    # ...
  end

  private

    def validate_computational_request!
      RequestValidator.new(
        param1: params[:param1],
        param2: params[:param2],
        param3: params[:param3],
        user: current_user
      ).call!
    end
end
```

## Principle / Benefits

- **PORO Object**: The validator is implemented as a Plain Old Ruby Object (PORO), avoiding dependencies and complex designs.
- **Single Responsibility**: Each validator object has a single responsibility and a single instance method (`call!`), adhering to the Single Responsibility Principle.
- **Separation of Concerns**: The validation logic is clearly separated from the business logic, improving maintainability and readability.
- **Straightforward Error Handling**:
    - Sequential validations allow each step to raise specific errors with more informative context, aiding in debugging.
    - Namespaced errors enable targeted exception handling, whether in individual actions or parent controllers. This can help logging relevant information prior to rendering custom messages.
- **Modularity and Reusability**: Validator classes can be easily reused across different controllers or actions, promoting code reuse and consistency.
- **Improved Testability**: With validations isolated in separate objects, unit testing becomes more focused and efficient.
- **Reduced Controller Complexity**: By moving validations out of controllers into dedicated objects, we achieve thinner, more manageable controllers.

This approach to request validation using simple PORO objects offers a clean, maintainable way to handle complex validation scenarios in Rails applications, leading to more organised and scalable codebases.
