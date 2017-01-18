---
layout: post
title: Introducing Locksmith
banner: /assets/posts/2016-09-23-Introducing-Locksmith/locksmith-screenshot.png
author: thomas
belongs_to: locksmith.md
---

At SENTIA we manage many AWS accounts, and for this we quite often need to
login to the AWS Console of these accounts. We wanted a way to access the AWS
Console that was both secure, and easy to use. 


## Options for AWS Console Authentication

We have looked at a couple of options (see table) for authentication to the AWS
console:

  * **Shared root credentials:**
    Using a system like 1Password or LastPass the root credentials of all AWS accounts could be shared.
    This means that everybody has full privileges within all accounts.
    A person leaving the team requires all credentials to be rotated.
    Requires one MFA token per account.
  * **Personal IAM Users:**
    Each person could have its own IAM user in every account.
    User actions can be audited.
    IAM policies can be used to finetune user privileges.
    A person leaving the team requires all his/hers IAM users to be deleted.
    Requires one MFA token per user per account.
  * **Personal IAM Roles:**
    Each person could have its own IAM role in every account, using a single IAM user to assume this role.
    User actions can be audited.
    IAM policies can be used to finetune user privileges.
    A person leaving the team requires only one IAM user to be deleted.
    Requires a single MFA token per user.

<div markdown="1" class="table-responsive feature-last-column">

|                       | Shared<br/>Root   | IAM<br/>Users | IAM<br/>Roles | IAM Roles<br/>&<br/>Locksmith |
| --------------------: | :----: | :-------: | :-------: | :-------------------: |
| **Security**          |        |           |           |                       |
| No shared credentials | ❌     | ✅        | ✅        | ✅                    |
| No shared MFA         | ❌     | ✅        | ✅        | ✅                    |
| User policies         | ❌     | ✅        | ✅        | ✅                    |
| Auditability          | ❌     | ✅         | ✅        | ✅                    |
| --------------------- | ------ | --------- | --------- | --------------------- |
| **Ease of Use**       |        |           |           |                       |
| Single MFA            | ❌     | ❌        | ✅        | ✅                     |
| Simple Login          | ❌     | ❌        | ❌        | ✅                     |
|=======================|========|===========|==========|========================|
| **Conclusion**       | 🙈     | 😫        | 😉        | 😍💕 🍻🎉                |

</div>

Of these options we found using IAM Roles to be the most secure, but
logging into the AWS Console using IAM Roles is quite a hassle, therefore
Locksmith – a Chrome Extension for AWS Console login using
Cross-Account IAM Roles – was created.


## How does Locksmith look?

![](/assets/posts/2016-09-23-Introducing-Locksmith/locksmith-screenshot.png)

## How does this work under the hood? 

We use a single IAM user per person. This user has a single MFA, and you can
easily remove the IAM user to revoke a person access to all accounts. 

  * [Assume a role][api-sts-assume-role] using the user's personal IAM User credentials
  * [Obtain a a signin token][federation-service] from the AWS Federation service
  * Construct a signin URL
  * Open the signin URL in the webbrowser

## But, wait a minute...
...doesn't the AWS Console support this [already][x-account-console]?

Yes indeed, we developed Locksmith before AWS announced this feature. Even so,
we still might have developed Locksmith since it has the following advantages
over the tool built into the AWS Console:

  * The list of accounts is **not limited** to the last five you accessed
  * Supports **MFA** for assuming the cross-account role
  * Consumes an **API** to present an up to date list of your AWS accounts
  * **Quicker**, you don't have to login to your personal AWS Console first
  * Shows the name of a bookmark (AWS account), not that of the assumed role

[api-sts-assume-role]: http://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html
[federation-service]: http://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_enable-console-custom-url.html
[x-account-console]: https://aws.amazon.com/blogs/aws/new-cross-account-access-in-the-aws-management-console/
