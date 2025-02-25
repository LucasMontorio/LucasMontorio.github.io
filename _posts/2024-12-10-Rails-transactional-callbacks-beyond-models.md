In this previous [post](https://lucasmontorio.github.io/2024/11/21/transaction-safety-in-rails-identifying-and-addressing-non-atomic-interactions.html), we discussed non-atomic interactions in transactions and their potential impact on data integrity.
One of the workarounds proposed for the issue described there was to use a **transactional callback.**

Let’s deep-dive into this feature, see how to make maximum profit of it, and how to extend its principle outside of the ActiveRecord with the help of a gem.

&nbsp;

## Legacy / Model Callbacks

Rails callbacks (or, more precisely, `ActiveRecord` callbacks) have been a common and handy practice for triggering logic on a range of events that can occur on a given model.
However, it's important to note that some of them rely on potentially risky patterns that are sometimes overlooked, despite posing risks to data integrity.

According to the official definition,
> *“Callbacks are hooks into the life cycle of an Active Record object that allow you to trigger logic before or after a change in the object state”.*


Here is the list of “classic” supported callbacks available on ActiveRecord:

- `before_validation`
- `after_validation`
- `before_save`
- `before_create`
- `after_create`
- `after_save`

The principle here is simple: when firing a `create` , `save` or a validation event on the `ActiveRecord` model at stake, the event will be preceded or followed by the execution of the block passed to the callback definition.

Hence, their use-cases can be multiple: sending an email after an object is created, formatting a user’s phone number before persisting it, updating relations subsequently to a successful action…


Here is the example we saw in the previous article:

```ruby
class User < ApplicationRecord
  after_save :do_something_asynchronous

  private

  def do_something_asynchronous
    SyncUser.perform_later(user: self) # Some business logic there...
  end
end
```

We observed that doing this represents a non-atomic interaction that, when run inside a transaction (which can implicitly happen as soon as a transaction is opened that includes manipulations on the `user` here), can lead to **race conditions**.


&nbsp;

## Transactional Callbacks

To avoid this type of race condition, we can replace this `after_save` callback with an `after_commit` callback, which will only fire after the transaction is completed.

This is called a **transactional callback**.

The available transactional callbacks to date are:

- `after_commit`
- `after_rollback`

> Transactional callbacks are native to ActiveRecord models and follow a simple principle: instead of relying on a model-related event, they rely on the state of an ActiveRecord transaction, meaning that any block of logic passed to it will be registered and called only when the transaction is closed.

_It is important to note that an `after_commit` statement is not specific to transactions on the current model, but will wait for every level of nested transactions around it to be closed._

Let’s see an example, involving two models and an asynchronous job:


```ruby
class User < ApplicationRecord
  after_commit :do_something_asynchronous

  private

  def do_something_asynchronous
    SyncUser.perform_later(user_id: self.id)
  end
end
```

```ruby
class Post < ApplicationRecord
  after_commit :log_something

  private

  def log_something
    Rails.logger.info("Post with ID #{id} was committed")
  end
end

```

```ruby
class SyncUser < ApplicationJob
  def perform(user_id:)
    user = User.find(user_id)
    Post.first.update(content: 'NEW CONTENT')

    sleep(5)

    # Some business logic on the user and post...

    puts "ASYNC action on user with ID: #{user_id}"
  end
end

```
&nbsp;

When executing the following code in a console...
```ruby
# Rails console
Post.transaction do
  user = User.new(name: 'John DOE')
  user.save

  # We wait for 5 sec to simulate a (very) long transaction
  sleep 5
end

#  TRANSACTION (1.0ms)  BEGIN
#  User Create (1.1ms)  INSERT INTO "users" ("name", "created_at", "updated_at") VALUES ($1, $2, $3) RETURNING "id"  [["name", "John DOE"], ["created_at", "2024-12-05 18:10:51.580199"], ["updated_at", "2024-12-05 18:21:24.721670"]]
#  TRANSACTION (0.9ms)  COMMIT

# Sidekiq server
# Performing SyncUserJob [...] from Sidekiq [...] enqueued at 2024-12-05T18:21:29Z with arguments: {:user_id=>1}
# Post with ID 1 was committed at: 2024-12-05T19:21:30+01:00
# Performed SyncUserJob [...] in 5071.17ms
```

Based on the timestamps of user update, job enqueuing, `Post`'s callback logging, and job ending, we can deduce that:
- The `user` was persisted while the transaction on `Post` was still open (you don't have the timestamp of my tapping ENTER in my console, but believe me, this DB operation was instantly committed)
- The `after_commit` callback on `User` waited 5 seconds before being triggered, resulting in the Sidekiq job being enqueued 5 seconds later than the `user`'s creation date. We can deduce from this part that the transactional callback is not restricted to a DB transaction on the table linked to the model bearing the callback, but waits for any open transaction to be closed.
- The job was quickly performed (~1 second after being enqueued), and the callback on `Post` was triggered right away, as a result of the `Post.first.update(...)` operation it contains
- The job then slept for 5 seconds before ending. At this stage, no ongoing transaction remained open

&nbsp;

> Breaking down this example on a classic ActiveRecord model highlights the benefits of transactional callbacks to ensure data integrity throughout your model’s lifecycle.


I would add that relying on callbacks remains a tricky practice, and shouldn’t be generalised. In many cases, using an interaction pattern or a simple service object to encapsulate DB manipulations and their side-effects might be the most efficient and straightforward move.

**Now, what if we wanted to run a callback on the event of the successful execution of some service object, outside of a specific model?**

&nbsp;

## The after_commit_everywhere gem

Let's illustrate this case with the following example of a service that manipulates `posts` and `users` simultaneously:

```ruby
class BusinessLogicService
  def call(user_id)
    user = User.find(user_id)

    Post.transaction do
      user.posts.update_all(active: false)

      user.update!(active: false)
    end

    puts 'transaction ongoing'
  end
end
```


**What if I want to send an email (or perform any other action) as soon as this transaction is committed?**

> Since we are not in an ActiveRecord model, we can't use the `after_commit` callback mentioned above.

Fortunately, a great gem called `after_commit_everywhere` (GitHub project [here](https://github.com/Envek/after_commit_everywhere)) offers a plug-and-play solution to tackle this case.

The principle is simple: include the `AfterCommitEverywhere` module, and you'll have access to `after_commit`, `after_rollback`, and `before_commit` "callbacks" wherever you need them:

```ruby
include AfterCommitEverywhere

class BusinessLogicService
  def call(user_id)
    user = User.find(user_id)

    Post.transaction do
      user.posts.update_all(active: false)

      user.update!(active: false)

      after_commit do
        puts 'transaction over!'
        # do something
      end

      puts 'transaction ongoing'
    end
  end
end
```

&nbsp;

**Okay, but how does this work under the hood?**

Let's see what a call to `after_commit` does, in the [gem's source code](https://github.com/Envek/after_commit_everywhere/blob/master/lib/after_commit_everywhere.rb#L41)

- It "registers a callback" in the form of a block or proc
    - Checks if a transaction is active
        - Gets the active `ActiveRecord::Base.connection` if any, defines one otherwise
        - Calls [`ActiveRecord::ConnectionAdapters::DatabaseStatements#transaction_open?`](https://apidock.com/rails/v4.2.7/ActiveRecord/ConnectionAdapters/DatabaseStatements/transaction_open%3F) on the connection
    - If a transaction is active
        - If `prepend` option is passed
            - Fetches the array of `records` (models with callbacks defined on them) on the connection's `current_transaction`
        - else (general case)
            - Wraps the callback in an ActiveRecord model-like class called `Wrap`, to be able to register transactional callbacks on it. This is the trick that allows usage of ActiveRecord's native callbacks.
            - Calls [`ActiveRecord::ConnectionAdapters::DatabaseStatements#add_transaction_record`](https://apidock.com/rails/v4.0.2/ActiveRecord/ConnectionAdapters/DatabaseStatements/add_transaction_record) on the connection with the wrapped callback
    - If no transaction is active
        - Yields the callback or raises, depending on config


As you can see, what I like about this gem is that it relies only on ActiveRecord's internals and simply exposes them in a simple, plug-and-play module.

&nbsp;

As a conclusion, transactional callbacks offer a powerful tool for maintaining data integrity and ensuring atomicity in complex DB operations.

With the help of `after_commit_everywhere`, we can take this concept further and use them outside of `ActiveRecord` models, **but**, it's important to remember that over-reliance on callbacks can lead to tightly coupled code and potential performance issues.
