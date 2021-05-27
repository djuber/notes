# Sidekiq Memory Use

{% embed url="https://github.com/forem/forem/pull/13861" %}

We have noticed a number of memory issues with the self-hosted forem test instance \(a 4GB digital ocean droplet\). Initially the assumption was the pressure was coming from ElasticSearch \(which is a fairly heavyweight process sitting on the jvm, a cost we couldn't spread to other components since the ruby stack is not using jruby\). ES had been removed but the OOM lockups on the test instance continued.



Joe was able to install datadog monitoring in the pod so we could get a view of memory use \(identify the problem service\) - it turned out to consistently be sidekiq.

I attempted to disable the ActiveRecord cache for sidekiq and we let the process run for a day - the results were disappointing.

![sidekiq process from datadog](.gitbook/assets/screenshot-from-2021-05-27-09-46-15.png)

Strangely - there's little to no correlation between the "steps" up in the graph and running jobs - when I zoom in closer \(1h scale\) the step is actually just a larger hump in what looks like a sawtooth function, with the troughs coming 100MB higher than before after the process.

Another look at sidekiq \(showing restarts both yesterday and today\)

![datadog sidekiq processes over time](.gitbook/assets/screenshot-from-2021-05-27-10-38-19.png)

I dumped the service logs for a relevant timespan - the sidekiq allocations during an extended period make little to no sense \(this is a quiet instance, there's only the scheduled jobs, they all complete in under 2 seconds\).

Possible suspicions:

could fog be allocating and not releasing something when we \(fail\) to create the sitemap \(issue is aws config is required but not present\)

could something with unique job or another sidekiq middleware/plugin unrelated to any job be allocating regularly? 

What's next?



