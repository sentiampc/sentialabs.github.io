---
layout: post
title: Managing EC2 SSH users with IAM
banner: /assets/posts/2018-04-26-iam-managed-ssh-users/iamec2.png
author: lvandonkersgoed
---

Servers in the cloud are supposed to be immutable and stateless. But what if the users allowed on a system are dynamic? You need a management system for your users!

At Sentia we've learned to treat our server instances as [cattle](http://cloudscaling.com/blog/cloud-computing/the-history-of-pets-vs-cattle/). They should be replaceable at any time, which is achieved through statelessness, and every server in an autoscaling group should be the same, which is achieved by using immutable sources (like an AMI) for every instance.

![Cattle](../../../assets/posts/2018-04-26-iam-managed-ssh-users/cattle.jpg)


Managing users the old way
--------------------------
Most environments we build and maintain are based on Linux. In general, these systems have a limited amount of users; because of security and operability reasons, we do not often allow our customers to login to instances.

In the few cases where the customer needs and is allowed SSH access to his systems, user management is easy. In our AMI creation process, we define the username(s) the customer can use, add a user in our automated AMI creation procedure and include their public SSH key in the `authorized_keys` file.

The AMI is launched (as cattle), and the user has access to them. In case we need to cycle the keys - about once a year - we build a new AMI and update the autoscaling group. Easy as pie.

Dynamic users
-------------
Recently we've worked on a big data project for a financial institution. Without going into details, the situation required data scientists to test and modify scripts on an instance in an AWS environment. These scripts would execute large processes on an EMR cluster. Customer SSH access into this instance was a realistic and appropriate requirement.

Security guidelines dictate that multiple users should have their own user accounts. This allows you to evict users without affecting others, track user activity and assign users specific folders, disks or databases. 

With that, the requirements for the solution were determined. We were to create a system that:
- Would allow us to create and remove users on the fly, on multiple instances simultaneously (think cattle)
- Would allow us to do this without terminating or rebooting the instance (boot time of the instance is about 30 minutes)
- Allowed the instance to remain stateless (so the user management system had to be placed outside of the instance)

IAM to the rescue
-----------------
We try to use AWS services as much as possible. Choosing services over servers significantly improves maintenance and operation; using services moves the responsibility for security and patching to AWS, services integrate nicely with each other, and services are generally a lot harder to break than something you made yourself.

So when we're thinking "user management", our next thought is "IAM". And the cool thing is: IAM supports uploading SSH keys! This allows us to create a solution like this:

![Process](../../../assets/posts/2018-04-26-iam-managed-ssh-users/process.png)

Managing groups
---------------
It's highly likely that you do not want all your IAM users to have SSH access to your machines. That's why we're putting the SSH-allowed users in a IAM Group called `DataScientists`.

![Group](../../../assets/posts/2018-04-26-iam-managed-ssh-users/group.png)

The magic
---------
The real magic happens on the instances. Through an EC2 IAM role they are allowed to get the `DataScientists` group, and for all the users they are allowed to list and get the public SSH keys:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "iam:GetGroup"
            ],
            "Resource": [
                "arn:aws:iam::111136746641:group/DataScientists"
            ],
            "Effect": "Allow"
        },
        {
            "Action": [
                "iam:ListSSHPublicKeys",
                "iam:GetSSHPublicKey"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}
```

Through a cronjob, a script using these permissions is run every 5 minutes:
```bash
#!/bin/bash
echo "====== Starting IAM SSH Import ======"

LINUX_USERGROUP="datascientists"
IAM_GROUP="DataScientists"

USERS=`aws iam get-group --group-name $IAM_GROUP | jq -r '.Users[] .UserName'`

for user in $USERS; do
  SSH_IDS=`aws iam list-ssh-public-keys --user $user | jq -r '.SSHPublicKeys[] .SSHPublicKeyId'`
  id $user >/dev/null 2>&1
  user_exists_code=$?
  if [ $user_exists_code -eq 0 ]; then
    echo "User $user exists on the system"
  else
    echo "Creating user $user"
    useradd $user -m -g$LINUX_USERGROUP
    mkdir -p /home/$user/.ssh/
    chown $user:$LINUX_USERGROUP /home/$user/.ssh/
    chmod 700 /home/$user/.ssh/
    touch /home/$user/.ssh/authorized_keys
    chown $user:$LINUX_USERGROUP /home/$user/.ssh/authorized_keys
    chmod 600 /home/$user/.ssh/authorized_keys
  fi

  if [ -n "$SSH_IDS" ]; then
    echo "Inserting SSH public keys for $user"
    echo "" > /home/$user/.ssh/authorized_keys
    for ssh_id in $SSH_IDS; do
      KEY=`aws iam get-ssh-public-key --user-name $user --ssh-public-key-id $ssh_id --encoding SSH | jq -r '.SSHPublicKey .SSHPublicKeyBody'`
      echo $KEY >> /home/$user/.ssh/authorized_keys
    done
  else
    echo "Removing all public keys for user $user"
    echo "" > /home/$user/.ssh/authorized_keys
  fi
done

USERS_ON_SYSTEM=`/usr/sbin/lid -gn $LINUX_USERGROUP | awk '{gsub(/^ +| +$/,"")}1'`

echo "== Purging users on system =="

for user_on_system in $USERS_ON_SYSTEM; do
  FOUND=0
  for user in $USERS; do
    if [ "$user" = "$user_on_system" ]; then
      FOUND=1
    fi
  done
  if [ $FOUND -eq 1 ]; then
    echo "Retaining user $user_on_system"
  else
    echo "Deleting user $user_on_system"
    userdel -r $user_on_system
  fi
done

echo "====== Finished IAM SSH Import ======"
```

Going through this script from top to bottom:
1. Get all the users in the IAM Group `DataScientists`
1. For each of these, check if they exist on the linux system.
1. If a user does not exist: 
    1. Create the user with a home directory and the `datascientists` linux user group
    1. Create the `.ssh` folder in their home directory
    1. Create the `authorized_keys` file in the `.ssh` folder
1. Get all the public SSH key files for the user from IAM
1. If there are more than zero ssh keys for the user:
    1. Remove all current SSH keys from `authorized_keys`
    1. Add each known SSH key
1. If there are zero SSH keys:
    1. Remove all current SSH keys from `authorized_keys`
1. Check which users on the system are in the `datascientists` user group
1. Loop over these users:
    1. If a user exists in IAM, do nothing
    1. If a user does not exist in IAM, remove the user and its home directory

Bonus points (logging)
----------------------
Log the output of the IAM sync command into syslog with a tag:
`*/5 * * * * root /scripts/import_iam.sh | logger -t iam-import`.

Then use a custom `rsyslog` configuration (like `/etc/rsyslog.d/10-iam.conf`) to log this to a separate file:
```
:syslogtag, isequal, "iam-import:" /var/log/iam-import.log
& stop
```

And finally, make sure that log file is forwarded to CloudWatch. Don't forget to implement log rotation!

Bonus points (expiry)
---------------------
SSH keys don't expire: you can upload them to IAM and use them forever. If you would like to have a forced key cycling system in place, use CloudWatch Events to trigger a Lambda Function. Have the function check on the key age and revoke it if it's older then a certain threshold. Also trigger SES to warn the user (or a security officer) to warn them. Within a few minutes (depending on your cron's execution rate on the EC2 instances), the user won't be able to login with the key anymore, but all the user's data will be retained on the instances.


