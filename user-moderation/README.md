# User Moderation

Notes on the thought process and progress for [https://github.com/forem/forem/issues/12965](https://github.com/forem/forem/issues/12965)&#x20;

Wanted to do the following:

* re-enable user article scoring when a user is flagged
* ensure the sum of all article scores is not factored in when doing this (which was the odd behavior that caused it to be disabled)
* explore enough of the UI in a local dev instance to be comfortable testing this
* explode the UI and documentation in tandem (ideally I should be able to find the documentation about moderation and behavior and determine what was supposed to happen, as well as what should happen).

As always, pull fresh master and check out a branch

```bash
git checkout -b djuber/moderation-articles-sinking
```

Since there were skipped tests, lets start there (it will be nice to have a failing test case so I know when I'm close to having this back working). Basically step one (not actually what I did, but what I _could_ have done) is to reverse the changes made in [https://github.com/forem/forem/commit/0a51540a4cbacb71ed4b20a7afe9105e425cc15f](https://github.com/forem/forem/commit/0a51540a4cbacb71ed4b20a7afe9105e425cc15f#diff-4b0e71e646891f1c8f43bac4678688e291ddf78a5d79acd3b372308f25915738) to the `sink_articles_spec.rb`

While I was there I realized the test didn't say what I thought it should (it checked the aggregate score's sum, not the individual scores of the articles, so I adapted this to check an array of equal values instead of the sum). The change I made in [https://github.com/forem/forem/commit/8281d8ac13af6527fe283360e8874914c0846a67](https://github.com/forem/forem/commit/8281d8ac13af6527fe283360e8874914c0846a67#diff-4b0e71e646891f1c8f43bac4678688e291ddf78a5d79acd3b372308f25915738) doesn't actually test what we need (we should create articles with distinct scores, then assert that the scores each shifted by the correct amounts, rather than setting them all to zero and adding user reactions, which fails to check the case where many articles have non-zero reaction scores.) I experimented a little with that by trying to set the scores to `[0, 10, 20]` and asserting on them, but the `scores.call` check was getting the articles in arbitrary orders. It's totally possible in rspec to test matching an array for containing exactly these elements, but I left it alone for the time being.

I did notice that the catch-all error handling in the worker made iterative development less than straightforward (since a broken worker just rolls back the transaction and logs to stats client, there's no failure the way you would _want_ it, and you only get "did not change" test failures. I eventually just put breakpoints in the worker, it would have made the most sense to put them in the rescue block and inspected the error, something like this

```ruby
module Moderator
  class SinkArticlesWorker
    include Sidekiq::Worker

    sidekiq_options queue: :medium_priority, retry: 10

    def perform(user_id)
      user = User.find_by(id: user_id)
      return unless user

      articles = Article.where(user: user)
      reactions = Reaction.where(reactable: user)
      articles.each do |article|
        article.update(score: article.score + reactions.sum(&:points))
      end
    rescue StandardError => e
      binding.pry  # <--- this is added to catch fall through errors during testin
      ForemStatsClient.count("moderators.sink", 1, tags: ["action:failed", "user_id:#{user.id}"])
      Honeybadger.notify(e)
    end
  end
end
```

Rubocop will reject code checked in with breakpoints so this should be fine (you can't mistakenly put this in a mergeable PR without guardrails from the system catching your mistake).&#x20;

Initially I tried something that looks like the above (for each article, add its current score to the sum of the reactions for the user), this reflected a "partial" understanding of the ways we could enter this code.

I started to think about what this would look like "a user is flagged as inappropriate or abusive, and all of their articles are scored lower" was my first use case, but it was incomplete. I looked around and found we also permit destroying feedback (in fact, when I tried testing this in the UI, I noticed the "destroy" route is just the same "Flag user" flow but the server responds with a json destroy success hash, while in the normal create route the "Flag user" form responds with text that all of the users articles will be marked lower/less visibly.) Once I started to think about needing to handle feedback/moderation removal, as well as creation, the improvements I tried to make here (passing the reaction to the service, letting the service decide whether the feedback was a "user\_vomit?" instead of sprinkling this in the controllers, and letting the reaction's reactable/user point to the articles to sink instead of passing a user and finding all of their user moderation reactions) fell away. There might be a way to do this by passing a user id and score change (+/- N for remove/add vomit reactions) but it didn't seem like a huge gain and recalculating the scores from the ground up seems equivalent.

I reached a point where I realized the things I was doing to the articles (loading all of a users reactions and all of the articles reactions and adding the sum of the scores, then applying it to the article) sure felt like an article concern and not some logic that belonged in the worker, I looked at adding an `update_` `score` __ or `rescore!` method on the article class. Happily, this already existed (the `update_score` method was what I needed) and I ended up with a worker that makes up for in correctness what it lacks in efficiency:

```ruby
module Moderator
  class SinkArticlesWorker
    include Sidekiq::Worker

    sidekiq_options queue: :medium_priority, retry: 10

    def perform(user_id)
      user = User.find_by(id: user_id)
      return unless user

      Article.where(user: user).each(&:update_score)
    rescue StandardError => e
      ForemStatsClient.count("moderators.sink", 1, tags: ["action:failed", "user_id:#{user.id}"])
      Honeybadger.notify(e)
    end
  end
end
```

What would make me happiest is if the update\_score logic were able to push more of this into an action performable on a relation at a time (it looks like that was the thrust during the initial development, to do this in one pass as an update, instead of a loop of updates). Conceivably this could be extracted more, something like this (I never get AR join statements correct on the first pass and this was typed into gitbook instead of a console):

```ruby
Articles.where(user_id: user.id)
        .joins("reactions ON reactions.reactable_id = article.id AND reactions.reaction_type = 'Article'")
        .includes("sum(reactions.points) as reaction_points")
        ...
```

I _think_ you could update the objects in memory then save all at the end, having pulled the sums of the reaction points in a large pass or a hash output from `GROUP BY article.id`

The changeset for the PR is about as small as it could be, like I said there probably should have been articles with non-zero scores in the setup step so we can see that not all articles get the same score when they're finished. [https://github.com/forem/forem/pull/12983/files](https://github.com/forem/forem/pull/12983/files)

One thing I found mildly bothersome when running a single spec locally was a coverage report happening after the specs run, causing a substantial delay from the test suite results showing and the prompt returning. I'm used to tests taking 4-5 seconds to load the database and code, and a few seconds to run during seed/truncate or seed/drop/seed phases, but the extra time spent calculating coverage for a single spec (where it _might_ have value but I was definitely not getting it) was surprising.
