---
description: Getting unit tests to run in buildkite
---

# Buildkite configuration

I spent some time \(too long, frankly\) in a feature branch shoring up a docker-composed based unit test setup. I had to disable selenium/system tests temporarily \(using `--tag=~js` to filter\) since that part hasn't been setup yet. I am in a place where the docker build runs in 12-13 minutes for me locally, and I'm not seeing any flakiness \(I do get a few failures when I enable test order randomization\).

One of the next steps is to actually trigger a build in buildkite. Molly had been exploring this about 7 or 8 months back \(buildkite says last build was 31 weeks ago\) - so I assume whatever I have here is dated and no maintenance has happened in the meantime. Buildkite's steps use a yaml format \(there's already a .buildkite/ directory in the repo housing the pipeline to build the containers\) - we probably _also_ want to trigger the build for unit tests so I assume that means a second pipeline.yml file.

Environment variables belong to the pipeline \("Forem CI"\) outside of source control - those might need some help.

The branch has a few changes - one is a new compose file tailored down to running tests \(it doesn't launch a sidekiq container, for example\) that I used during testing, plus some unit test refactorings that will be needed to make the tests reliable in the container environment. 

I expect to push the needed changes back into main \(the only "change" left in the feature branch that differs from the main branch on upstream should be the test-compose.yml file, a convenience script to run rspec using it, and the buildkite pipeline for unit tests, which will be the last PR\).

