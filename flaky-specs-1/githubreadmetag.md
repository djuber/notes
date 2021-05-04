---
description: dynamic classes and verifying doubles are order dependent
---

# GithubReadmeTag

The `Github::OauthClient` wraps another class \(Octokit\) and uses method\_missing to define api access as needed. Unfortunately, this means instance methods, other than the overridden "inspect", are not defined until they're called - and rspec will fail if this is run out of order.

```text

Randomized with seed 7704
[Zonebie] Setting timezone: ZONEBIE_TZ="Novosibirsk"

GithubTag::GithubReadmeTag
  #id
    renders a repository with a missing README (FAILED - 1)
    renders a repository URL with a trailing slash
    rejects invalid GitHub repository URL
    renders a repository path
    renders a repository URL with a fragment
    rejects GitHub URL without domain
    renders a repository path with a trailing slash
    renders a repository URL
    rejects a non existing GitHub repository URL
    options
      accepts 'no-readme' as an option
      rejects invalid options
    regressions
      parses a repository with invalid img tags

Failures:

  1) GithubTag::GithubReadmeTag#id renders a repository with a missing README
     Failure/Error: allow_any_instance_of(Github::OauthClient).to receive(:readme).and_raise(Github::Errors::NotFound) # rubocop:disable RSpec/AnyInstance                                                   
       Github::OauthClient does not implement #readme
     # ./spec/liquid_tags/github_tag/github_readme_tag_spec.rb:77:in `block (3 levels) in <top (required)>'  
     # ./spec/rails_helper.rb:125:in `block (2 levels) in <top (required)>'

Finished in 2.49 seconds (files took 2.77 seconds to load)
12 examples, 1 failure

Failed examples:

rspec ./spec/liquid_tags/github_tag/github_readme_tag_spec.rb:71 # GithubTag::GithubReadmeTag#id renders a repository with a missing README                              

```

A dirty solution would be to instantiate an oauth client, call readme, accept that it fails, and then execute the test. A "better" solution might be to have the client class list the known instance methods \(we're using maybe 10\) and pre-define them at load time.



[https://github.com/djuber/forem/commit/d414f7e55](https://github.com/djuber/forem/commit/d414f7e55) maybe something like this. Or what Rhymes did the first time [https://github.com/forem/forem/commit/3130261384\#diff-058371fd17700cbee2b084681a6515dc723daf206855881cbf8bb44d9774cab7R21-R27](https://github.com/forem/forem/commit/3130261384#diff-058371fd17700cbee2b084681a6515dc723daf206855881cbf8bb44d9774cab7R21-R27) 

