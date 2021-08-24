---
description: >-
  Docker / Docker-compose locally has some warts (we should fix) but there's
  gitpod - is it any better?
---

# gitpod development test

{% embed url="https://dev.to/ben/spin-up-a-local-instance-of-dev-in-the-cloud-with-gitpod-it-s-incredibly-simple-pij" %}

{% embed url="https://developers.forem.com/installation/gitpod" %}

Just on a lark I clicked the link in the install docs. I was prompted to sign in - I chose github - and I logged in pretty quickly.

Some notices about exposed ports being private by default \(good, I didn't think we wanted to host world-readable redis!\) and a theia editor being removed popped up - and I'm waiting on a prebuild now \(with an option to skip?\)



![Prebuild in Progress](.gitbook/assets/screenshot-from-2021-08-24-14-38-37.png)

I opened the developer tools console to see what was happening \(the black box is supposed to be a log output\). There were some 404 errors being handled from the headless logs here 

```javascript
const retry = async (reason: string, err?: Error) => {
      console.debug("re-trying headless-logs because: " + reason, err);
      await new Promise((resolve) => {
        setTimeout(resolve, 2000);
      });
      startWatchingLogs().catch(console.error);
    };
```

```text
re-trying headless-logs because: error while listening to stream Error: Headless logs for 8fcda768-55ba-407b-8c42-5d261fa32fd5 not found
```

After 5 minutes I'm clicking "Dont wait for Prebuild" to see what happens next.

I get to "Creating" and pulling container image

![Creating workspace](.gitbook/assets/screenshot-from-2021-08-24-14-45-13.png)

Once it loads it appears you've got vscode in a browser \(think github codespaces, but 2 years ahead of them\) 

![vscode in the browser](.gitbook/assets/screenshot-from-2021-08-24-14-47-59.png)

Not sure why pg\_ctl failed to start \(my assumption is the gitpod code might not be up to date with any other changes - looking into Ben's initial PR to see what he did to wire this up\)

