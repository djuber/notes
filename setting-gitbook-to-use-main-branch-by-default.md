# Setting gitbook to use "main" branch by default

Gitbook, by default, when you integrate with github, will push to master. 

Github, by default, will set main as the  default branch \(for cloned repos this is the first checked out branch, and for pull requests this is th default merge target\). 

It makes sense to have these speak the same language. One option is to change the default branch \(in github, under repo settings, branches, chose "Default Branch" using the pencil icon next to main\). The other is to point gitbook at main instead of master.

Under "Integrations" in the left side, setup github. When prompted for branches the default is master only, but we can add "main" on the right option. When you enable this gitbook will sync to github both on the main branch \(where it creates a new empty "variant" with no content and an empty readme\) and master \(not what we wanted, but if you have content this is where it went, we'll fix that in a moment\).



