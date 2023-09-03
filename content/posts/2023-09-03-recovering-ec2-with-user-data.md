---
title: "Recovering EC2 Instances With User Data"
date: 2023-09-03T14:04:53+10:00
subtitle: ""
tags: [AWS,EC2]
---

So, every now and then when I do something in AWS, I mess it up. One
problem that I give myself every now and then is breaking SSH access to
an EC2 instance. In that case, you need some sort of out-of-band access
to go and fix the SSH configuration.

EC2 instance user data is generally set to be a shell script that
[cloud-init](https://cloudinit.readthedocs.io/en/latest/)
runs once when the instance is created. However,
[this](https://repost.aws/knowledge-center/execute-user-data-ec2)
extremely helpful article describes how to update the user data for
an EC2 instance so that it runs on every boot. This lets you to run an
arbitrary shell script as root on each boot, from which you can generally
debug or diagnose almost anything.

Here's the user data that I use to set a password on the "ubuntu"
account, so that I can log in using the AWS web console. SSH password
authentication is disabled, so it's reasonably safe to use an insecure
password.


```
Content-Type: multipart/mixed; boundary="//"
MIME-Version: 1.0

--//
Content-Type: text/cloud-config; charset="us-ascii"
MIME-Version: 1.0
Content-Transfer-Encoding: 7bit
Content-Disposition: attachment; filename="cloud-config.txt"

#cloud-config
cloud_final_modules:
- [scripts-user, always]

--//
Content-Type: text/x-shellscript; charset="us-ascii"
MIME-Version: 1.0
Content-Transfer-Encoding: 7bit
Content-Disposition: attachment; filename="userdata.txt"

#!/bin/bash
set -x
sudo sed -e '/PasswordAuthentication/d' -e'$ a\PasswordAuthentication no' -i /etc/ssh/sshd_config
sudo systemctl restart sshd
echo ubuntu:PAssword123# | sudo chpasswd
--//--
```
