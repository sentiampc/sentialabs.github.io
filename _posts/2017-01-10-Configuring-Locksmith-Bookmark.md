---
layout: post
title: Add Locksmith Bookmark
banner: /assets/locksmith/bookmarks.png
author: thomas
belongs_to:
  - locksmith.md
---

How to add AWS Account bookmarks to Locksmith.
Locksmith can be used both stand-alone and managed by a service.
Here we show how to use Locksmith as a stand-alone tool.

Instructions for adding an AWS account bookmark to Locksmith are given below.
First, create an IAM Role in the target account.
Second, add a bookmark to Locksmith.

## Create an IAM Role

In the AWS console, [create a new IAM Role](createRole):

* Choose a role name, for example the email address of the user
* Click "Role for Cross-Account Access"
* In "Provide access between AWS accounts you own", click "Select"
* Enter the AccountID of the account in which the IAM User for Locksmith was
  [created](/2017/01/09/Configuring-Locksmith.html)
* Make sure "Require MFA" is **checked**!
* Click "Next Step"
* Select the Policy you wish the user to be able to use
  
  _It is good practice to give the minimum required set of privileges._

  If you _must_ provide almost all privileges, please consider using
  "PowerUserAccess" (allows evertything, except IAM user management) over
  "AdministratorAccess" (allows everything).

* Please make a note of the "Role ARN", this is a string like
  `arn:aws:iam::012345543210:role/foo@bar.baz`
* Click "Create Role", don't forget this step!

## Add a Bookmark

* Click the plus sign in the upper left corner of the Locksmith popup
  
  ![](/assets/posts/2017-01-10-Configuring-Locksmith-Bookmark/add-bookmark.png){: .image-center .image-border }

* Fill following information in the form:
  
  ![](/assets/posts/2017-01-10-Configuring-Locksmith-Bookmark/edit-bookmark.png){: .image-center .image-border }
  
  * **Name:** A name to identify the account
  * **AWS Account Number:** The 12-digit number from the role ARN you made a
    note of, in the example above that is `012345543210`
  * **Role Name:** The part after the last slash of the role ARN you made a note
    of, in the example above that is `foo@bar.baz`
  * The **Avatar URL** is optional:
    * When you leave it empty, it will use the gravatar for the bookmark's
      **Name**
    * When you provide an email address, it will use the gravatar for that
      email address
    * When you provide an URL (a string starting with `http`) it will use
      that URL as avatar
* Click "Save Bookmark"

## Finally

Try to use the new bookmark. Feel free to add as many bookmarks as you like,
there is no limit.
When the amount of AWS accounts you manage is becoming too large for you to
manage manually, implement the Locksmith API, or feel free to give us a
[call](/contact.html)!