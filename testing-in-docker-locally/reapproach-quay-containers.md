---
description: >-
  Part 2 in an N part series where I try to get our forem:testing containers to
  run the test suite
---

# Reapproach quay containers

Jotting down a few notes before I get started. Talked earlier this week with Joe about how they overcame volume mount and user permissions issues for the forem self-host instances \(Digital ocean droplet running podman running thes quay.io/forem/forem:latest image plus supporting services\).

{% hint style="info" %}
"quay containers" is a horrible name for this - what I mean are the containers I can download now from quay.io/forem/forem that are built and tagged by buildkite both during PR branches and master builds. This is important because these are also the containers we deploy \(in forem cloud, and likely in self-hosting\). S
{% endhint %}

One thing that needed/needs to happen is removing the root user and using the forem user \(sounds like a docker-compose change- the container file already builds a forem user, we weren't using that\). - Snapped this commit together during the discussion [https://github.com/djuber/forem/commit/36dc3c56a1f18351983bd038f28e49c60ff016be](https://github.com/djuber/forem/commit/36dc3c56a1f18351983bd038f28e49c60ff016be)

My last series of tests would copy the tree into the image during build \(so any code changes locally required a rebuild of the image, which wasn't a problem as long as just running the container was the task at hand, but could be a big hassle for live use/reuse\).

My problems with port vs expose are still valid \(I think I want to change to expose basically for everything in testing - unless there's value in seeing port 3000 I shouldn't _need_ to stop the local dev env to interact with docker\). 

How do I isolate the "testing" stage in the container file?

```text
docker build --target="testing" . 
```

I think this might be correct.

Change 1:

make the entrypoint call rspec instead of rails server - I remember this being a weird decision when I was using docker compose to run the tests because of how the output happened

```text

-CMD ["bundle", "exec", "rails", "server", "-b", "0.0.0.0", "-p", "3000"]
+CMD ["bundle", "exec", "rspec"]
 
```

Did some minor surgery on the test compose file and I'm able to get errors running tests now successfully

{% embed url="https://github.com/djuber/forem/tree/djuber/quay-images-for-testing" %}

Commented out the profile image from the users factory and I'm down to a few hundred failures \(out of 6000\) so \(1\) carrierwave is still an impediment, mostly tied to user factory, \(2\) it's the main problem to solve here.

```text
$ docker-compose -f test-compose.yml run rails bundle exec rspec --format=documentation

```

