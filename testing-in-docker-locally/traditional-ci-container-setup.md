# Traditional CI container setup

Molly had tried a few months ago to run ci in buildkite - and reported everything except front end timeouts seemed to be working - there's also a forem ci pipeline in buildkite \(that keeps asking for an irreversible yaml transition\) that's probably tied to these

first-buildkite-pipeline [https://github.com/forem/forem/tree/mstruve/first-buildkite-pipeline](https://github.com/forem/forem/tree/mstruve/first-buildkite-pipeline)

try-containers-again [https://github.com/forem/forem/tree/mstruve/try-containers-again](https://github.com/forem/forem/tree/mstruve/try-containers-again) 

buildkite-ci [https://github.com/forem/forem/tree/mstruve/buildkite-ci](https://github.com/forem/forem/tree/mstruve/buildkite-ci)



Let's get one of these working \(I think I can use a local buildkite agent using the buildkite cli to execute them, otherwise the CI config should have an equivalent docker configuration\).

copy [https://raw.githubusercontent.com/forem/forem/f44bfccb82e33e0266100c21f6de0183442d051e/.buildkite/Dockerfile](https://raw.githubusercontent.com/forem/forem/f44bfccb82e33e0266100c21f6de0183442d051e/.buildkite/Dockerfile) into a local branch and call `Docker build .` to see what happens \(only change I made was to base off of the ruby:2.7.2 build 



```text
djuber@forem:~/src/forem-for-docker$ docker build .buildkite/
Sending build context to Docker daemon   5.12kB
Step 1/33 : FROM quay.io/forem/ruby:2.7.2
... SNIP ...

Step 18/33 : ENV DOCKERIZE_VERSION=v0.6.1
 ---> Using cache
 ---> 55641e4c5e18
Step 19/33 : RUN wget https://github.com/jwilder/dockerize/releases/download/"${DOCKERIZE_VERSION}"/dockerize-linux-amd64-"${DOCKERIZE_VERSION}".tar.gz     && tar -C /usr/local/bin -xzvf dockerize-linux-amd64-"${DOCKERIZE_VERSION}".tar.gz     && rm dockerize-linux-amd64-"${DOCKERIZE_VERSION}".tar.gz
 ---> Using cache
 ---> 1fe7857d3521
Step 20/33 : WORKDIR "${APP_HOME}"
 ---> Using cache
 ---> 659f209f360e
Step 21/33 : COPY ./.ruby-version "${APP_HOME}"
COPY failed: file not found in build context or excluded by .dockerignore: stat .ruby-version: file does not exist
djuber@forem:~/src/forem-for-docker$ ls -l .ruby-version 
-rw-r--r-- 1 djuber djuber 6 Mar 23 11:01 .ruby-version
djuber@forem:~/src/forem-for-docker$ cat .ruby-version 
2.7.2

# I think I needed instead 
$ docker build --file .buildkite/Dockerfile . -t testcontainer
```

That worked \(with the file option building from `.`\) - it looks like the coordination for this container requires postgresql up and running \(which docker-compose would have done for you\).

I'm still getting the carrierwave issue - and still not having this problem locally. Since molly wasn't crying about this 5 months ago, did something change with the carrierwave setup since then?

oh, I guess I had a typo - we're good - we're good! wow! whoa. this was stupid \(and I still don't know what happened to make this work correctly\).



```text
djuber@forem:~/src/forem-for-docker$ docker-compose --file=docker-compose-test.yml run -e RAILS_ENV=test -e DATABASE_URL_TEST="postgresql://forem:forem@db:5432/PracticalDeveloper_test"  rails bundle exec rspec spec/models/user_spec.rb   
WARNING: The KNAPSACK_PRO_TEST_SUITE_TOKEN_RSPEC variable is not set. Defaulting to a blank string.
WARNING: Found orphan containers (forem_bundle) for this project. If you removed or renamed this service in your compose file, you can run this command with the --remove-orphans flag to clean it up.
Starting forem_yarn                  ... done
Starting forem-for-docker_selenium_1 ... done
Starting forem_redis                 ... done
Starting forem_postgresql            ... done
Starting forem_elasticsearch         ... done
2021/03/29 20:44:11 Waiting for: http://selenium:4444
2021/03/29 20:44:11 Waiting for: tcp://db:5432
2021/03/29 20:44:11 Waiting for: http://elasticsearch:9200
2021/03/29 20:44:11 Waiting for: tcp://redis:6379
2021/03/29 20:44:11 Connected to tcp://db:5432
2021/03/29 20:44:11 Connected to tcp://redis:6379
2021/03/29 20:44:11 Received 200 from http://elasticsearch:9200
2021/03/29 20:44:11 Received 200 from http://selenium:4444
Running command:
bundle exec rspec spec/models/user_spec.rb
DEPRECATION WARNING: Devise::Models::Authenticatable::BLACKLIST_FOR_SERIALIZATION is deprecated! Use Devise::Models::Authenticatable::UNSAFE_ATTRIBUTES_FOR_SERIALIZATION instead. (called from const_get at /opt/apps/forem/vendor/cache/devise-0cd72a56f984/lib/devise/models.rb:90)
[Zonebie] Setting timezone: ZONEBIE_TZ="Chihuahua"
..........................................................................................
..........................................................................................
...................................

Finished in 34.94 seconds (files took 5.73 seconds to load)
217 examples, 0 failures

2021/03/29 20:44:52 Command finished successfully.
```

[https://github.com/forem/forem/commit/2dd87a52b1235bcdea78caefd6a680fc637c59e7](https://github.com/forem/forem/commit/2dd87a52b1235bcdea78caefd6a680fc637c59e7) running all of spec/ shows a _few_ failures - these might be systems tests - will let it finish before I decide what to do.



[https://gist.github.com/djuber/607f96332f7dc3ff7de8cf05301a1708](https://gist.github.com/djuber/607f96332f7dc3ff7de8cf05301a1708) okay - summary of issues

* can't find chrome \(all system/front-end tests require this for selenium\)
* seems like redis url is localhost:6379 and should not be \(a few tests use rpush or put feature flags in redis\) - might be a missing RPUSH\_URL env var that has been set to redis://redis:6379 in a subsequent commit
* admin should show last commit "some date" \(might be tied to the entrypoint.sh not finding git?\) - this looks like a one-off build process or execution order issue, rather than a problem with the container orchestration.

Next step was "add chrome to the build" since that seems like the biggest improvement if it works correctly. 

