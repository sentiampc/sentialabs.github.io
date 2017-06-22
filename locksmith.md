---
layout: project
title: Locksmith
section: Projects
navigation_weight: 2
github:
  - sentialabs/locksmith
  - sentialabs/locksmith-cli
chrome_web_store: idahiicmmneinnceklagffdlmgdmdnhc
---

**Locksmith allows you to easily switch between accounts in the AWS portal!**

Are you managing many different Amazon Web Services accounts? Locksmith is a
Chrome extension that allows you to easily switch between accounts! It provides
a dropdown menu from which you can select one of your accounts, and it will open
a new Chrome window with a session to the web console of that account.

The extension uses IAMs Cross-Account Roles, therefore you only need credentials
for one AWS account. This also means that you only need a single MFA (virtual)
token, no matter how many accounts you manage!

![](/assets/locksmith/bookmarks.png)


**Locksmith CLI allows you to easily assume roles in your terminal!**

Locksmith CLI is a command-line version of Locksmith that allows you to easily
obtain credentials for your accounts! It provides selection menu from
which you can select one of your accounts, it will assume the IAM role and
export the temporary credentials as environment variables in a new shell.

![](/assets/locksmith/locksmith-cli.png)
