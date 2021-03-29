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
$ docker build --file .buildkite/Dockerfile .

```



