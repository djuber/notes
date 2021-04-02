# userData is sometimes not loaded

Noticed this after [https://github.com/forem/forem/pull/12862/](https://github.com/forem/forem/pull/12862/files) was merged - and developers started reporting failures that they were sometimes able to replicate in isolation, sometimes not. Julianna was able to work around the failures she was seeing by adding a second page load in [https://github.com/forem/forem/pull/13224](https://github.com/forem/forem/pull/13224)

The rest of this page is describing how we got from failing test to "fixed" test, and what the next steps are - this will probably get distilled into a forem[ github issue](https://github.com/forem/forem/issues/13228) after I get my thoughts together here.

Actual problem is that a non-trusted user \(who should not show the Flag option in another user's profile page\) is in fact showing that the first time they visit the page.

![](../.gitbook/assets/showing-for-non-trusted-user.png)



### How did we get here

Internal forem team slack link [https://forem-team.slack.com/archives/C012YSX5ZC1/p1617289014004500](https://forem-team.slack.com/archives/C012YSX5ZC1/p1617289014004500) - but I'll distill what was useful from this discussion here since it's important background.

> Seeing a handful of failures related to `trsuted_user_flags_user_spec.rb` , specifically line 16: `when signed in as a non-trusted user, it does not show the flag button` . When I run the entire spec, the tests pass without an issue, but when I run just the specified test on its own, it fails with the same failures I’m seeing on Travis \(failure in thread\)

First thing I did was git blame on the file - it was just committed earlier that day \(about 7 hours before the report of the issue\) so that is a relief \(it's much more likely to be a problem with the test when it's a new test failing sometimes\). 

I checked out master and attempted to reproduce, but the spec passed for me. I know Forem's tests run sequentially/deterministically, so I added --random in order to see if sequencing would recreate the issue. A few random sorts \(there were only four tests in the file so if it will fail it will fail quickly\) down got me to a reproduction case:

```bash
bundle exec rspec spec/system/user/trusted_user_flags_user_spec.rb --order=random --seed=9374 --format=documentation
```

Julianna \(initial reporter\) tried adding a sign out to ensure state between runs was not being carried over. That "worked" the first time she tried it and stopped "fixing" this after a second run, but it was a valid attempt to isolate the two tests.

Capybara will snap a screenshot of the failing page when the expect fails, these go in tmp/screenshots/ and have a format that includes the spec's text description. I could see the wrong information \(the Flag item present on the page\) but not any more information than that.

I added the `HEADLESS=false` environment variable \([https://github.com/forem/forem/blob/master/spec/support/initializers/capybara.rb\#L14-L26](https://github.com/forem/forem/blob/master/spec/support/initializers/capybara.rb#L14-L26) checks for either a remote selenium url, or the HEADLESS environment variable, when configuring JS tests with capybara, basically the default is not to show the browser when the tests run, but you can change this when you call the test by passing in the right env var\). 

Unfortunately, this is really fast \(capybara reads the pages much faster than I do, the only delays are with chrome's rendering agent, which is also pretty fast\) - so in order to view the page I put a breakpoint in just before the `expect()` line in the failing spec \(I'm a firm believer in direct observation, which reduces the leverage a little and slows things down so I can explore the problem more\).

```ruby
  context "when signed in as a non-trusted user" do
    it "does not show the flag button" do
      sign_in create(:user)
      
      visit user_profile_path(user.username)
      click_button(id: "user-profile-dropdown")
      
      binding.pry # added in this step
      expect(page).not_to have_link(flag_text)
    end
  end
```

This permitted two things - the page stopped as the test was running leaving it up and inspectable, and the console popped open a debugger in the backend.

Since the code asserted that a "non-trusted" user was active, I looked at the page to get the current username \(top right dropdown\) and saw it was username4. I queried in the debugger what roles the user with that username had - and it came back empty - this is in fact a non-trusted user.

```ruby
User.find_by(username: :username4).roles
=> []
```

We spent some time looking at what controlled the display of the dropdown. There's a piece of code in the profile view that [adds the dropdown ](https://github.com/forem/forem/blob/master/app/views/users/show.html.erb#L35-L50)when a user is logged in \(rather than a crawler, for example\).

```ruby
              <div id="user-profile-dropdownmenu" class="crayons-dropdown top-100 right-0 p-1">
                <% if user_signed_in? %>
                  <a href="javascript:void(0);" id="user-profile-dropdownmenu-block-button" data-profile-user-id="<%= @user.id %>" class="border-none crayons-link crayons-link--block">Block @<%= @user.username %></a>
                  <a
                    href="javascript:void(0);"
                    id="user-profile-dropdownmenu-flag-button"
                    data-profile-user-id="<%= @user.id %>"
                    data-profile-user-name="@<%= @user.username %>"
                    data-is-user-flagged="<%= @is_user_flagged %>"
                    class="border-none crayons-link crayons-link--block">
                    <%= @flag_status ? "Unflag" : "Flag" %> @<%= @user.username %>
                  </a>
                  <%= javascript_packs_with_chunks_tag "profileDropdown", defer: true %>
                <% end %>
                <span class="report-abuse-link-wrapper" data-path="/report-abuse?url=<%= user_url(@user) %>"></span>
              </div>
```

It is useful to note that the user-profile-dropdownmenu-flag-button is included in the html response from user/show for all logged in users \(even non-trusted users\) - this suggests something is happening in the front end to alter the content of the dropdown menu on page load \(and changes the direction of the search\). As an aside, it would be possible, I think, to alter the code here and check the user's trusted role in the view at render time server side - very much like the `<% if user_signed_in? %>`check on the second line above. That could resolve the issue, at the expense of possibly adding an extra query at in the backend.

So what is the front end doing \(or not doing\) here? Also included in Michael's PR \(that added the tests that was failing\) is app/javascript/profileMenu/flagButton.js which handles the logic for display.

Notes on this file \(might be obvious to someone more conversant in javascript than I am\).

* there's a global function userData declared early in a comment
* The code here defines an exported function `initFlag()` \(which I assume is called elsewhere during initialization\) - if this is _not_ being called that could cause problems
* The first thing the function does is `getElementById` to find the `"user-profile-dropdownmenu-flag-button"` from the view, namign this `flagButton` 
* There's a guard in case that didn't exist \(if you can't see the flag, nothing to do, return early\).
* The second step is to load `userData()` into the const `user`
* If that's not present \(no user\) also exit early
* Otherwise, load the dataset from the `flagButton` to find the user id \(of the button, basically who would be flagged\) and whether they've already been flagged. The user name is also loaded to handle adapting the text later after actions \(when we send flag and unflag, what should the text show\).
* Check to  see if this user is trusted \(there are two keys checked, `user.trusted` and `user.admin`, either permits flagging other users\)
* If the user is not trusted, `remove()` the flag button - it should only be visible for trusted users.

A little sleuthing confirmed that initFlag was being called but that user was not present \(`userData()` was null when initFlag was called\). By the time I had the inspector/console open on the page, `userData()` returned a nice hash of the user - with trusted false and admin false and all of the expected conditions in the proper states - such that calling initFlag\(\) at this point would have removed the item from the dropdown - but I think this is only invoked once and apparently it was invoked too early.

![](../.gitbook/assets/should-have-called-remove.png)



