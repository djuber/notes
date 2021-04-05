# JS Testing

Just grabbing some miscellaneous facts about the js tests.



tag: :js or type: :js in system and request tests make capybara use chrome headless, and wait for js.

{% code title="https://github.com/forem/forem/blob/master/spec/support/initializers/capybara.rb\#L29-L44" %}
```ruby


# adapted from <https://medium.com/doctolib-engineering/hunting-flaky-tests-2-waiting-for-ajax-bd76d79d9ee9>
def wait_for_javascript
  max_time = Capybara::Helpers.monotonic_time + Capybara.default_max_wait_time
  finished = false

  while Capybara::Helpers.monotonic_time < max_time
    finished = page.evaluate_script("typeof initializeBaseApp") != "undefined"

    break if finished

    sleep 0.1
  end

  raise "wait_for_javascript timeout" unless finished
end

```
{% endcode %}

interestingly, `initializeBaseApp` is the function that's awaited \(or waited to be defined\) - typeof is function - basically the file that's loaded defining this function \(base.js.erb\) _also_ immediately calls the function - so the existence of the definition is evidence of its execution.



initializeBaseApp calls `initializePage` \(or passes that to `InstantClick`, which I assume calls it on ready state?\)

```javascript

// FUNCTIONAL CODE FOR PAGE

  function initializeBaseApp() {
    InstantClick.on('change', function() {
      initializePage();
    });
    InstantClick.init();
  }
```



