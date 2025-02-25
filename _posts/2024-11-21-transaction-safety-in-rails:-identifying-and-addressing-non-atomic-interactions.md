Database transactions are a crucial mechanism for maintaining data integrity in the face of unexpected errors or failures. They ensure that multiple related operations are treated as a single, indivisible unit of work.

While transactions are very common and often implicit in many Rails operations, in some cases mixing them with asynchronous actions might lead to unexpected results.

Let's consider a practical example to illustrate potential issues, with a simple model and an asynchronous job (in our case, a Sidekiq job):

```ruby
class User < ApplicationRecord
  after_save :do_something_asynchronous

  private

  def do_something_asynchronous
    SyncUser.perform_later(user: self)
  end
end
```

```ruby
class SyncUser < ApplicationJob
  def perform(user_id:)
    user = User.find(user_id)

    # Some business logic on the user...

    puts "ASYNC action on user with ID: #{user_id}"
  end
end
```

In normal circumstances, this code works as expected:

```bash
# Rails console
user = User.new(name: 'John DOE')
user.save
# => true

# Sidekiq server
## Performing SyncUserJob [...] from Sidekiq [...] with arguments: {:user_id=>1}
## ASYNC action on user with name: John DOE
## Performed SyncUserJob [...] in 37.25ms

```

However, let’s see what happens if a wrapping transaction prevents my `save` call to be committed immediately to the DB:

```ruby
# Rails console
User.transaction do
  user = User.new(name: 'John DOE')
  user.save

  # We wait for 5 sec to simulate a (very) long transaction
  sleep 5
end

# Sidekiq server
# Performing SyncUserJob [...] from Sidekiq [...] with arguments: {:user_id=>2}
# Discarded SyncUserJob due to a ActiveRecord::RecordNotFound.
# Performed SyncUserJob [...] in 37.25ms

```

We can see here that the job was performed in Sidekiq with an `ActiveRecord::RecordNotFound` error.

This is due to the fact that, at the time of executing the job’s logic (in the Sidekiq server’s runtime), the DB operation hasn’t been committed yet (in the Rails app’s runtime), meaning the object does not exist in the DB.

## Non-atomic interactions in transactions

Non-atomic interactions occur when asynchronous actions are triggered within a transaction that hasn't been committed yet. This situation can lead to race conditions on the executed async job due to the following reasons:

- The transaction can potentially rollback, causing **data inconsistencies**
- Even in successful operations, the transaction might take longer than expected to commit due to additional tasks being executed, leading to unexpected behaviours

In a Rails environment, a common source of **implicit** non-atomic interactions is ActiveRecord's native `after_save` callback, which wraps its content in a transaction and runs side actions regardless of the transaction's outcome.

While identifying them manually can sometimes be cumbersome, fortunately a great tool has been developed for this exact purpose...

## The Isolator gem

[Isolator](https://github.com/palkan/isolator) is a great 'plug-n-play' gem designed to detect non-atomic interactions within database transactions automatically.

It raises an error every time it detects such an interaction, helping you identify and address these issues as early as possible in your development process.

It supports multiple adapters to catch operations in different contexts that are at risk of non-atomic interactions.

## Using Isolator locally

The gem needs little to no configuration to work, depending on your context.

In our case, let’s Install the gem, and add minimal config:

```ruby
# Gemfile

group :development, :test do
  gem 'isolator'
end
```



```ruby
# initializers/isolator.rb

Isolator.configure do |config|
  # Specify a custom logger to log offenses
  config.logger = nil

  # Raise exception on offense
  config.raise_exceptions = true # true in test env

  # Send notifications to uniform_notifier
  config.send_notifications = false

  # Customize backtrace filtering (provide a callable)
  # By default, just takes the top-5 lines
  config.backtrace_filter = ->(backtrace) { backtrace.take(5) }

  # Define a custom ignorer class (must implement .prepare)
  # uses a row number based list from the .isolator_todo.yml file
  config.ignorer = Isolator::Ignorer
end
```

Let’s now re-run the same example:

```ruby
# Rails console
User.transaction do
  user = User.new(name: 'John DOE')
  user.save

  # We wait for 5 sec to simulate a (very) long transaction
  sleep 5
end

# Isolator::BackgroundJobError: You are trying to enqueue background job inside db transaction. In case of transaction failure, this may lead to data inconsistency and unexpected bugs
# Details: SyncUserJob ({:user_id=>2})
```

We now get an `Isolator::BackgroundJobError` error that prevents `SyncUserJob` from being enqueued, hence protecting data consistency.

In this specific case, several simple solutions exist to prevent this behaviour:

- Use a **transactional callback**
    - Transactional callbacks like `after_commit` or `after_rollback` are native Rails tools that work the same as the standard callbacks, except that they don’t execute until after database changes have either been committed or rolled back.
- Avoid the use of callbacks for such sequential actions, and favour an *interaction* (as in the Interaction Pattern) or other kinds of service objects to encapsulate and sequentially run the logic of user creation followed by its job scheduling.

## Conclusion

While the aforementioned example describes a simple case of non-atomic interaction wrapped in a transaction, real-world applications can obviously involve far more complex scenarios, where identifying these issues manually can be very tricky.

Although it is important to note that it won’t fix all your issues, [Isolator](https://github.com/palkan/isolator) is a great tool to help with such concerns. It is recommended for use in both test and development environments, preferably in a ‘whiny’ mode that raises errors when detecting dangerous operations. While it can be plugged into staging environments, extra careful consideration should be given to this approach.
