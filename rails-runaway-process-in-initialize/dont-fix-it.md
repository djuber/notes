---
description: reading the bug tracker is a good thing
---

# Don't fix it

Ended up looking for related errors in the bug tracker in ruby  - and found one reported 3 months ago tied to activesupport and a run-away process.



{% embed url="https://bugs.ruby-lang.org/issues/17494" %}

Especially, Jeremy Evans had a suggested patch at [https://bugs.ruby-lang.org/issues/17494\#note-9](https://bugs.ruby-lang.org/issues/17494#note-9) - I applied this and recompiled - and confirmed the issue went away \(complete Gem set from Michael's branch, no run-away process, bin/setup completed without issue\) - as a guard I re-ran against the rbenv installed unpatched ruby 3.0.0 binary and confirmed there still was a problem \(I didn't accidentally fix this somewhere else\) - and I'm parking this effort on "watch for \#17494 to be resolved"



