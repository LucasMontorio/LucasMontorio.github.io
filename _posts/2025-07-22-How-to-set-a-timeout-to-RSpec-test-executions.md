## Introduction / Motivation

On one of the Rails application on which I am working, a Rails monolith, I recently stumbled upon a quite disturbing CI issue: in some of our RSpec runs, it happens that specs **timeout in an unpredictable way after having reached the CI environment’s (in our case, CircleCI) built-in “no-output timeout”**.

> *I won’t go in depth into the nature of these tests, but let’s say they are simply browser-centric Capybara tests that sometimes get stuck waiting for a condition that can never be satisfied.*

Reasons that could lead to such hangs are multiple: deadlocks, connection pools exhausted, infinite loops…

When these cases happen, the test that hangs terminates with a quite useless output  and causes your CI run to wait for up to 10 minutes just to tell you that your test has failed. This can cause important delays and generate extra costs (CI runs being usually billed by the minute).

```jsx
Error: Task timed out after 10.00 minutes
```

This made me realise that, some times, it could be good to be able to make a test fail after a certain amount of time, be it to prevent the CI env. from spending too much time (and money) on it, or simply to monitor the efficiency of a test suite.

**To my surprise, I didn’t find any simple, RSpec built-in way to do this.**

Hence, I started developing my own.

*This article will go into the details of the strategy adopted for this solution, and the different phases through which I went before reaching a satisfactory implementation.*

## First iteration / Thread-joining

In order to achieve this, my first intention was to create a mechanism that could detect that a test has been running for too long, then terminate it with a helpful error message that would appear in the CI’s output.

One way of doing this would be to instantiate a thread responsible of running the test, and monitor its execution from the main thread. When reaching a timeout, we could then kill the running thread and proceed with the rest of the suite.

Ruby’s `Thread#join` method allows us to make the main thread wait until the thread referenced by the method completes its execution. It also accepts a timeout parameter, which is particularly relevant to our case. When a timeout is provided, the method will wait only up to that many seconds for the thread to complete:

*Example:*

```ruby
# Create a new thread that takes 5 seconds
t = Thread.new do
  sleep 5
  puts "Thread completed"
end

# Wait for at most 2 seconds for the thread to complete
if t.join(2)
  puts "Thread completed within timeout"
else
  puts "Thread did not complete within timeout"
  # The thread is still running in the background
end

puts "Main thread continues regardless"

# OUTPUT:
Thread did not complete within timeout
Main thread continues regardless
=> nil
```

*In this example, `t.join(2)` will return `nil` because the thread doesn't complete within 2 seconds. If the thread had completed before the timeout, it would have returned the thread object itself, which evaluates to `true` in a boolean context.*

Now that I could identify a thread running for too long, I simply had to add a wrapper (hook) around RSpec examples to plug this behaviour on every running spec, and cleanly terminate the timed-out ones.

This lead to the **first, basic implementation of the gem**:

```ruby
RSpec.configure do |config|
  config.around(:each) do |example|
    time_limit_seconds = example.metadata[:time_limit_seconds]

    next example.run unless example.metadata[:time_limit_seconds]

    begin
      thread = Thread.new do
        example.run
      end

      unless thread.join(time_limit_seconds)
        thread.kill
        raise RspecTimeGuard::TimeLimitExceededError,
              "[RspecTimeGuard] Example exceeded timeout of #{time_limit_seconds} seconds"
      end
    end
  end
end
```

*At this stage, a timed-out test generates the following output:*
<details>

```bash
Randomized with seed 29957
#<Thread:0x000000015bf7de68 ... exception (report_on_exception is true):
...gems/rspec-core-3.13.3/lib/rspec/core/example.rb:521:in `ensure in run_after_example': undefined method `teardown_mocks_for_rspec' for nil (NoMethodError)

        @example_group_instance.teardown_mocks_for_rspec
                               ^^^^^^^^^^^^^^^^^^^^^^^^^
	from .../rspec-core-3.13.3/lib/rspec/core/example.rb:521:in `run_after_example'
from ...gems/rspec-core-3.13.3/lib/rspec/core/example.rb:283:in `ensure in block in run'
	from .../rspec-core-3.13.3/lib/rspec/core/example.rb:283:in `block in run'
from .../rspec-core-3.13.3/lib/rspec/core/example.rb:511:in `block in with_around_and_singleton_context_hooks'
from ...gems/rspec-core-3.13.3/lib/rspec/core/example.rb:468:in `block in with_around_example_hooks'
from .../rspec-core-3.13.3/lib/rspec/core/hooks.rb:486:in `block in run'
	from .../rspec-core-3.13.3/lib/rspec/core/hooks.rb:626:in `block in run_around_example_hooks_for'
from ...gems/rspec-core-3.13.3/lib/rspec/core/example.rb:352:in `call'
	from .../rspec-rails-8.0.0/lib/rspec/rails/adapters.rb:75:in `block (2 levels) in <module:MinitestLifecycleAdapter>'
from .../rspec-core-3.13.3/lib/rspec/core/example.rb:457:in `instance_exec'
from ...gems/rspec-core-3.13.3/lib/rspec/core/example.rb:457:in `instance_exec'
from .../rspec-core-3.13.3/lib/rspec/core/hooks.rb:390:in `execute_with'
	from .../rspec-core-3.13.3/lib/rspec/core/hooks.rb:628:in `block (2 levels) in run_around_example_hooks_for'
from ...gems/rspec-core-3.13.3/lib/rspec/core/example.rb:352:in `call'
	from .../webmock-3.25.1/lib/webmock/rspec.rb:39:in `block (2 levels) in <main>'
from .../rspec-core-3.13.3/lib/rspec/core/example.rb:457:in `instance_exec'
from ...gems/rspec-core-3.13.3/lib/rspec/core/example.rb:457:in `instance_exec'
from .../rspec-core-3.13.3/lib/rspec/core/hooks.rb:390:in `execute_with'
	from .../rspec-core-3.13.3/lib/rspec/core/hooks.rb:628:in `block (2 levels) in run_around_example_hooks_for'
from ...gems/rspec-core-3.13.3/lib/rspec/core/example.rb:352:in `call'
	from ... (3 levels) in setup'
.../rspec-core-3.13.3/lib/rspec/core/example.rb:463:in `hooks': undefined method `hooks' for class NilClass (NoMethodError)

        example_group_instance.singleton_class.hooks
                                              ^^^^^^
	from .../rspec-core-3.13.3/lib/rspec/core/example.rb:518:in `run_after_example'
	from .../rspec-core-3.13.3/lib/rspec/core/example.rb:283:in `ensure in block in run'
from ...gems/rspec-core-3.13.3/lib/rspec/core/example.rb:283:in `block in run'
	from .../rspec-core-3.13.3/lib/rspec/core/example.rb:511:in `block in with_around_and_singleton_context_hooks'
from .../rspec-core-3.13.3/lib/rspec/core/example.rb:468:in `block in with_around_example_hooks'
from ...gems/rspec-core-3.13.3/lib/rspec/core/hooks.rb:486:in `block in run'
from .../rspec-core-3.13.3/lib/rspec/core/hooks.rb:626:in `block in run_around_example_hooks_for'
	from .../rspec-core-3.13.3/lib/rspec/core/example.rb:352:in `call'
from ...gems/rspec-rails-8.0.0/lib/rspec/rails/adapters.rb:75:in `block (2 levels) in <module:MinitestLifecycleAdapter>'
	from .../rspec-core-3.13.3/lib/rspec/core/example.rb:457:in `instance_exec'
from .../rspec-core-3.13.3/lib/rspec/core/example.rb:457:in `instance_exec'
from ...gems/rspec-core-3.13.3/lib/rspec/core/hooks.rb:390:in `execute_with'
from .../rspec-core-3.13.3/lib/rspec/core/hooks.rb:628:in `block (2 levels) in run_around_example_hooks_for'
	from .../rspec-core-3.13.3/lib/rspec/core/example.rb:352:in `call'
from ...gems/webmock-3.25.1/lib/webmock/rspec.rb:39:in `block (2 levels) in <main>'
	from .../rspec-core-3.13.3/lib/rspec/core/example.rb:457:in `instance_exec'
from .../rspec-core-3.13.3/lib/rspec/core/example.rb:457:in `instance_exec'
from ...gems/rspec-core-3.13.3/lib/rspec/core/hooks.rb:390:in `execute_with'
from .../rspec-core-3.13.3/lib/rspec/core/hooks.rb:628:in `block (2 levels) in run_around_example_hooks_for'
	from .../rspec-core-3.13.3/lib/rspec/core/example.rb:352:in `call'
from ... (3 levels) in setup'

RspecTimeGuard::TimeLimitExceededError: [RspecTimeGuard] Example exceeded timeout of 1 seconds

0) Processing::Base#call raises a timeout error
   Failure/Error: raise RspecTimeGuard::TimeLimitExceededError, message

   RspecTimeGuard::TimeLimitExceededError:
   [RspecTimeGuard] Example exceeded timeout of 1 seconds
   # ... (2 levels) in setup'
1 example, 1 failure, 0 passed
Finished in 1.033112 seconds

Randomized with seed 29957

Process finished with exit code 1
```
</details>

### Cleaning the error trace / better error-handling

Despite being able to make a test fail based on its execution time, this would still generate quite a lot of noise, due to our overwriting some of RSpec’s internal logic.

In order to avoid this, there exists an option that prevents the noisy error output by preventing Ruby from automatically reporting exceptions that occur within the thread.


> When using `thread.kill` to terminate the test thread, Ruby was automatically printing all the internal exceptions that happened during the termination process. This created the extremely verbose and confusing stack trace seen in the example, showing internal RSpec errors like "undefined method teardown_mocks_for_rspec' for nil".

By setting `Thread.current.report_on_exception = false` at the beginning of our thread creation, I could tell Ruby not to automatically print these internal exceptions to stderr.

*This setting doesn't suppress the exceptions themselves - they still occur and can be caught and handled - it just prevents their automatically being printed to the console, which was cluttering our output with information that wasn't helpful to users of the gem.*
Beyond avoiding noisy process terminations, I also needed to properly handle exceptions and allow them to re-raise in the main thread to allow RSpec to fail for reasons unrelated to timeouts.

If a test fails for reasons unrelated to timeouts (like a failing assertion), we want that original error to be reported clearly, not obscured by our timeout machinery.
To improve thread termination, we explored using `thread.exit` as a gentler alternative to `thread.kill`. While `thread.kill` forces immediate termination and can leave resources in an inconsistent state, `thread.exit` requests the thread to terminate more cooperatively.

This lead roughly to the following (simplified) implementation:

```ruby
thread = Thread.new do
  Thread.current.report_on_exception = false

  begin
    example.run
  rescue Exception => e
    Thread.current[:exception] = e
  end
end

# Later in the code:
if thread.join(time_limit_seconds)
  raise thread[:exception] if thread[:exception]
else
  RspecTimeGuard.terminate_thread(thread)
  raise RspecTimeGuard::TimeLimitExceededError, "[RspecTimeGuard] Example exceeded timeout of #{time_limit_seconds} seconds"
end

def terminate_thread(thread)
  return unless thread.alive?

  # Attempt to terminate the thread gracefully
  thread.exit

  # Give the thread a moment to exit gracefully and perform cleanup
  sleep 0.1
  # If it's still alive, kill it
  thread.kill if thread.alive?
end
```

*For the same timed-out spec as mentioned above, we now get the following output:*

<details>

```bash
RspecTimeGuard::TimeLimitExceededError: [RspecTimeGuard] Example exceeded timeout of 1 seconds

0) Processing::Base#call raises a timeout error
   Failure/Error: raise RspecTimeGuard::TimeLimitExceededError, message

   RspecTimeGuard::TimeLimitExceededError:
   [RspecTimeGuard] Example exceeded timeout of 1 seconds
   # .../lib/rspec_time_guard.rb:53:in `block (2 levels) in setup'
1 example, 1 failure, 0 passed
Finished in 1.127036 seconds
```
</details>

### The `RSpec.current_example` issue / better thread handling

Now that I was able to set an example-specific timeout threshold and get a clean error output in case of timeout, the next step was to allow the gem to **set a global value for this threshold**, thus allowing us to run an entire test suite with the same timing expectations.

In the traditional Ruby gem fashion, I implemented a configuration object responsible for storing the timeout threshold value globally, and ran the test suite locally on a production-ready application with >1000 tests:

```bash
time_limit_seconds = example.metadata[:time_limit_seconds] || RspecTimeGuard.configuration.max_execution_time
```

To my surprise, I ran into several issues that led to a significant proportion of my examples ending with the following error when setting the global `max_execution_time` param: tests that had previously worked fine suddenly began failing with puzzling errors indicating that `RSpec.current_example` was nil. This was particularly perplexing since the tests had worked correctly when using per-example time limits through metadata.

```bash
      NoMethodError:
        undefined method `metadata' for nil
```

Further investigation revealed that some of our tests were trying to access `RSpec.current_example.metadata` as part of their business, which caused the error. The intention behind these tests will not be described here, but let’s just say that they needed to access the current example’s metadata to conditionally load certain fixtures. It turned out this getter was actually broken because of my `RspecTimeGuard` implementation. Here’s why:

The root cause lay in how RSpec manages the current example context. **RSpec uses thread-local storage** to track the currently executing example, making it available via `RSpec.current_example`. This works well in standard RSpec usage where tests run in the main thread, but our implementation was breaking this fundamental assumption by **running each test in a separate thread so that the main thread could monitor its execution time:**

```ruby
thread = Thread.new do
  Thread.current.report_on_exception = false
  begin
    example.run
  rescue Exception => e
    Thread.current[:exception] = e
  end
end

unless thread.join(time_limit_seconds)
  # Handle timeout...
end

```

This design created a disconnection: when the test called `RSpec.current_example`, it returned nil because the thread-local reference only existed in RSpec's original thread, not in my custom thread where the test was actually running. I had to update the gem’s design and invert the approach:

Since I had to keep the test running in RSpec’s main thread where all the context was properly set up, I could now use a separate thread solely for timeout monitoring.
This inverted design became the new implementation:

```ruby
RSpec.configure do |config|
  config.around(:each) do |example|
    time_limit_seconds = example.metadata[:time_limit_seconds] || RspecTimeGuard.configuration.global_time_limit_seconds

    next example.run unless time_limit_seconds

    completed = false

    # NOTE: We instantiate a monitoring thread, to allow the example to run in the main RSpec thread.
    # This is required to keep the RSpec context.
    monitor_thread = Thread.new do
      Thread.current.report_on_exception = false

      # NOTE: The following logic:
      #  - Waits for the duration of the time limit
      #  - If the main thread is still running at that stage, raises a TimeLimitExceededError
      sleep time_limit_seconds

      unless completed
        message = "[RspecTimeGuard] Example exceeded timeout of #{time_limit_seconds} seconds"
        if RspecTimeGuard.configuration.continue_on_timeout
          warn "#{message} - Running the example anyway (:continue_on_timeout option set to TRUE)"
          example.run
        else
          Thread.main.raise RspecTimeGuard::TimeLimitExceededError, message
        end
      end
    end

    # NOTE: Main RSpec thread execution
    begin
      example.run
      completed = true
    ensure
      # NOTE: We explicitly clean up the monitoring thread in case the example completes before the time limit.
      monitor_thread.kill if monitor_thread.alive?
    end
  end
end
```

**The monitoring thread simply sleeps for the duration of the timeout** and then checks if the test is still running. If it is, it raises an error in the main thread to terminate the test.

> A key component of this implementation is the `completed` flag that provides communication between the main thread and the monitoring thread. When the test finishes successfully, the main thread sets this flag to true, signaling to the monitoring thread that it shouldn't take any action even if it wakes up after its sleep period.

**This redesign solved the `RSpec.current_example` issue!**

### Second iteration / single-thread monitoring implementation

Now that the the RSpec example context issue was fixed, I was finally able to finalise a first (WIP) version of the gem: add specs, a CI suite, a proper README, … And finally test it on a production app’s CI run.

Although this was a nice milestone, the latest implementation at this staged still was unsatisfying in a way: **Each test was creating its own monitoring thread**, and in test suites with thousands of examples, this could potentially create thousands of threads over the course of execution. This proliferation of threads could impact system resources and overall performance.

To address this issue, my idea was to reimagine the current implementation to use a a **single, persistent monitoring thread** that would track all active tests. Instead of each test creating its own monitor, tests would register themselves with this central monitor when they started and unregister when they completed.

I encapsulated this functionality in a `TimeoutMonitor` class with a clean API:

```ruby
class TimeoutMonitor
  def register_test(example, timeout, thread)
    # Add test to the active test list with its metadata
  end

  def unregister_test(example)
    # Remove test from the active test list
  end
end

```

This approach required careful attention to thread safety, as multiple tests could be registering and unregistering concurrently. I used a mutex to ensure that updates to the active tests list were atomic:

```ruby
def register_test(example, timeout, thread)
  @mutex.synchronize do
    @active_tests[example.object_id] = {
      example: example,
      start_time: Time.now,
      timeout: timeout,
      thread_id: thread.object_id,
      warned: false
    }

    start_monitor_thread if @monitor_thread.nil? || !@monitor_thread.alive?
  end
end

```

The backbone of the monitoring system was now a single thread that periodically checked the list of active tests to see if they have reached their respective time limits. This thread would be initiated in our RSpec `around` block as follows:

```ruby
def start_monitor_thread
  @monitor_thread = Thread.new do
    Thread.current[:name] = 'rspec_time_guard_monitor'

    loop do
      check_for_timeouts
      sleep 0.5 # Check every half second

      # Exit thread if no more tests to monitor
      break if @mutex.synchronize { @active_tests.empty? }
    end
  end
end

```

For each active test, the monitor compared its elapsed running time against its specified timeout limit. If a test had exceeded its limit, the monitor would take appropriate action, either raising an error or outputting a warning depending on configuration:

```ruby
def check_for_timeouts
  now = Time.now
  timed_out_examples = []

  @mutex.synchronize do
    @active_tests.each do |_, info|
      elapsed = now - info[:start_time]
      timed_out_examples << info if elapsed > info[:timeout]
    end
  end

  # Handle timeouts outside the mutex to avoid deadlocks
  timed_out_examples.each do |info|
    # Create error message and handle timeout...
  end
end

```

One challenge I faced was how to **track thread objects across different contexts**. Storing direct references to thread objects could lead to memory leaks, so I instead stored thread IDs and used `ObjectSpace._id2ref` to retrieve the actual thread when needed:

```ruby
thread = begin
  ObjectSpace._id2ref(@active_tests[example.object_id][:thread_id])
rescue
  nil
end

next unless thread&.alive?
```

To improve the user experience when using the `continue_on_timeout` option, I added a `warned` flag to track which tests had already received timeout warnings. This prevented the monitor from outputting repeated warnings for the same test.

Finally, a tiny bit of resource conservation logic. The monitor thread automatically terminates itself when there are no more active tests to monitor, and it only starts when needed:

```ruby
# Exit thread if no more tests to monitor
break if @mutex.synchronize { @active_tests.empty? }
```

This single-threaded monitoring approach **dramatically reduced the number of threads** created during test suite execution, from potentially thousands to just one.

### Notes / final comments


> `RspecTimeGuard` was developed and tested primarily with MRI/CRuby, but we gave careful consideration to compatibility with other Ruby interpreters. The gem's core functionality relies on standard Ruby libraries and APIs that are implemented across different Ruby interpreters, making it broadly compatible in most scenarios.
>
> However, there are some potential compatibility considerations worth noting. The gem's use of `Thread.raise` behavior can vary between interpreters - while it works reliably in MRI/CRuby, JRuby and TruffleRuby may exhibit different timing characteristics and reliability patterns.


Eventually, `RspecTimeGuard` represents a new gem that provides a simple, focused solution to a quite common problem in Ruby test suites - the need to prevent tests from hanging indefinitely and wasting CI resources. Through its evolution from a basic thread-joining approach to a sophisticated single-threaded monitoring system, it taught me a lot of interesting things and made me want to share this project, which is the point of this whole article!

The gem doesn't pretend to be perfect, and we actively welcome challenging comments, issues, and contributions from the community. Real-world usage will undoubtedly reveal edge cases and improvement opportunities that we haven't yet encountered.

In the coming months, `RspecTimeGuard` will undergo more intensive testing across different Ruby applications, environments, and use cases. I plan to write more on the outcome of such usage. In the meantime, I hope you enjoyed reading this as much as I enjoyed writing it!

Cheers,

Lucas
