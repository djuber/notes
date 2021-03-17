# User Moderation is really just scoring

I started looking at the disabled SinkArticles behavior - and then eventually came to the realization that what's desired it "rescore articles, having applied or removed user level feedback". 



Digging around it looks like we have a "one at a time" article score update worker that does only three things:

* fetch article by provided id
* call update\_score on this article, which changes the db copy
* perfom inline/synchronous ES updates \(rather than enqueing an indexing worker to do this\)

The sink article job was crafted initially to act in bulk \(using update\_all and fitting the score into the articles\) which had a few problems:

* duplication of scoring knowledge into the sink articles worker
* performance was gained at the cost of correctness - the method didn't account for individual scores and caused the feed issue that caused it to be disabled. 
* didn't have any impact on elasticsearch, and didn't update the spaminess rating \(these may be newer than the implementation of the sink worker, which ties into a discussion about the dangers of duplicating knowlege\).

I learned a few things, it looks like we also had the update\_score call using a service class to calculate spaminess and hotness \(for ranked display? still not sure how this wires into behavior\) but passing the stale version of the object to it \(so we weren't factoring in the most recent items\). I have a draft PR \(not merged just yet\) at [https://github.com/forem/forem/pull/13009](https://github.com/forem/forem/pull/13009) waiting on me to write test cases for it.



[https://github.com/forem/forem/pull/13012](https://github.com/forem/forem/pull/13012) addresses the missing ES data \(by calling the existing Article score worker instead of calling Article\#update\_score directly\). 

I guess the key takeway here is that the mechanism by which we moderate users is to apply user level reactions \(like an article level reaction, but all of the users articles factor this in for scoring\) followed by an article rescore for each article. This was unclear when I started on the process \(looking at what appeared to be a one-off database update using a formula to find the right levels\) but is probably the concept I want to internalize.

One shaggy end I'd like to shave some day is right now the Articles::ScoreCalcWorker seems to be the one place that knows about how to perform the sequence of scorings \(update the db and sync to ES\) but there's not an easily testable _function_ to return the new scores, calculation and persistence are mingled in the update\_score method, and scoring the articles seems like business logic that could be pulled out of the article model class into a service, like Articles::Ranking, or at least a function that takes an article or an article and its reactions, and returns the tuple of values from the scoring plus the black box that we'll be persisting. 

