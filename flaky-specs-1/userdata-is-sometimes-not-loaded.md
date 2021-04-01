# userData is sometimes not loaded

Noticed this after [https://github.com/forem/forem/pull/12862/](https://github.com/forem/forem/pull/12862/files) was merged - and developers started reporting failures that they were sometimes able to replicate in isolation, sometimes not. Julianna was able to work around the failures she was seeing by adding a second page load in [https://github.com/forem/forem/pull/13224](https://github.com/forem/forem/pull/13224)

The rest of this page is describing how we got from failing test to "fixed" test, and what the next steps are - this will probably get distilled into a forem github issue after I get my thoughts together here.



Actual problem is that a non-trusted user \(who should not show the Flag option in another user's profile page\) is in fact showing that the first time they visit the page.

![](../.gitbook/assets/showing-for-non-trusted-user.png)

Internal forem team slack link [https://forem-team.slack.com/archives/C012YSX5ZC1/p1617289014004500](https://forem-team.slack.com/archives/C012YSX5ZC1/p1617289014004500) - but I'll distill what was useful from this discussion here since it's important background.

> Seeing a handful of failures related to `trsuted_user_flags_user_spec.rb` , specifically line 16: `when signed in as a non-trusted user, it does not show the flag button` . When I run the entire spec, the tests pass without an issue, but when I run just the specified test on its own, it fails with the same failures I’m seeing on Travis \(failure in thread\)



