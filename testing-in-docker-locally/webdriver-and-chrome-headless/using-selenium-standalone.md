---
description: Trying to replicate the code from the spike branch into the current branch
---

# Using Selenium standalone

The first pass \(using one of molly's old branches\) had docker-compose setting up the selenium standalone container and serving on port 4444

This seemed to encounter a few \(small\) hiccups with webmock blocking the DELETE /session call \(which was allowed for vcr, but cleanup maybe happening after the vcr block closes and only webmock is live\). 

My first attempt with the forem quay.io containers was to install chromium directly in the container and attempt to run that \(webdriver gem probably contains all the code I need for this - and this mirrors how we run the tests locally/native/outside container\). I hit snag one when I found the sandboxing was requiring a kernel capacity \(user namespaces\) that appears to be disabled \(not sure if that's a debian issue, a chromium default, or a fedora container on debian host issue - this didn't happen in the spike where I built on a debian/ruby container\).



