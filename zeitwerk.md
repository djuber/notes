---
description: rails 6 autoloading behavior
---

# Zeitwerk

[https://guides.rubyonrails.org/v6.0.3/autoloading\_and\_reloading\_constants.html](https://guides.rubyonrails.org/v6.0.3/autoloading_and_reloading_constants.html) discusses this

context: Seeing \(suddenly\) errors loading Notifications::WelcomeNotificationWorker when trying to instantiate a new user in tests in docker.

```ruby
[2] pry(#<RSpec::ExampleGroups::User::ProfileImage>)> create(:user)
NameError: uninitialized constant Notifications::WelcomeNotificationWorker
from /opt/apps/forem/app/models/notification.rb:79:in `send_welcome_notification'

```

Before I get much further - my suspicion is that the issue may be tied to two directories with classes in the Notifications module \(app/services/notifications/ and app/workers/notifications/\) and only the first getting searched.

This is only happening for me in docker \(the same behavior does _not_ happen in a local dev env\). 

I can confirm the Notifications module _is_ loading correctly

```ruby
[8] pry(#<RSpec::ExampleGroups::User::ProfileImage>)> show-method Notifications
                                                                               
From: /opt/apps/forem/app/services/notifications.rb:1
Module name: Notifications
Number of lines: 78

module Notifications
  def self.user_data(user)
```

However - I tried the dumbest thing that could possibly work, and copied the worker from workers/notifications/ to services/notifications/ and still could not autoload the class. Additionally, none of the code in either subdirectory belonging to this namespace was loaded:

```text
[7] pry(#<RSpec::ExampleGroups::User::ProfileImage>)> constant = "Notifications::WelcomeNotificationWorker".constantize
NameError: uninitialized constant Notifications::WelcomeNotificationWorker                                             
from /opt/apps/forem/vendor/bundle/ruby/2.7.0/gems/activesupport-6.1.3.1/lib/active_support/inflector/methods.rb:288:in `const_get'

[16] pry(#<RSpec::ExampleGroups::User::ProfileImage>)> Notifications.constants
=> []                                                 
```



