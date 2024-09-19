---
description: Writeup for the 1st flag (Emerge through the breach) in the Thunderdome Cyber Range from PwnedLabs
title: Pwned Labs - Thunderdome Flag 1 of 9
date: 2024-09-19 00:00:00 +0200
categories: [Cloud, Pwned Labs, Hacking]
tags: [cloud, hacking, pwnedlabs, thunderdome]
#show_image_post: true                                   # Change this to true
#image: /assets/img/                # Add infocard image here for post preview image

---

## Target Entry Point: 44.208.228.94

## Enumeration

Browsing to the IP we are presented with a website labeled "Massive Pharma"

![insert](/assets/img/MassivePharma.png)

Looking at the source code we see an interesting comment referring to a Bitbucket repository `mp-website`:

```console
<footer class="u-align-center u-clearfix u-footer u-grey-80" id="sec-1c6e">
    <div class="u-clearfix u-sheet u-sheet-1">
    </div>
</footer>
<!-- http://bitbucket.org/massive-pharma/mp-website -->
<br>
```

Browsing to project on bitbucket we see the website files:

![insert](/assets/img/PharmaBitbucket.png)

Looking at the files and commits we don't find anything of use. Perhaps Massive Pharma has some other projects in the Bitbucket account? 

And we find the `trial-data-management-poc` repository:

![insert](/assets/img/PharmaClinicBucket.png)

Looking through some commits we get some info that will assist us in getting enough info to compromise the AWS account.. We have the user Haru's email `haru@massive-pharma.com` and we have an AWS key `AKIATCKANV3QK3BT3CVG`

```console
From c167543e30628c5a76f79f519a0adb752b238106 Mon Sep 17 00:00:00 2001
From: Haru Sato <haru@massive-pharma.com>
Date: Thu, 25 Jan 2024 22:14:55 +0000
Subject: [PATCH] Bucket name change

---
 tests/uploader/data-uploader.php | 4 ++--
 1 file changed, 2 insertions(+), 2 deletions(-)

diff --git a/tests/uploader/data-uploader.php b/tests/uploader/data-uploader.php
index b2f70e5c..ded53fc2 100644
--- a/tests/uploader/data-uploader.php
+++ b/tests/uploader/data-uploader.php
@@ -8,12 +8,12 @@ $s3 = new S3Client([
     'version' => 'latest',
     'region'  => 'us-east-1',
     'credentials' => [
-        'key'    => '',
+        'key'    => 'AKIATCKANV3QK3BT3CVG',
         'secret' => '',
     ],
 ]);
 
-$bucketName = 'clinical-trial-data';
+$bucketName = 'mp-clinical-trial-data';
 $directory = '../export/';
 
 $files = glob($directory . 'TRIAL*-*.csv');
-- 
2.46.1
```

This alone is not enough to compromise the AWS account as we still need the password and AWS Account ID. The Key will enable us to find the account ID with the AWS CLI using `sts:GetAccessKeyInfo` as indicated in the AWS Documentation [here](https://docs.aws.amazon.com/STS/latest/APIReference/API_GetAccessKeyInfo.html)

Before we can start we need to ensure we are authenticated with a valid AWS account by running `aws configure` and entering your AWS Access Key and Secret key. 

Running the aws cli command we get the account id that the access key belongs to.

```console
C:\Users\MichaelCoetzee>aws sts get-access-key-info --access-key-id=AKIATCKANV3QK3BT3CVG
{
    "Account": "211125382880"
}
```

Now that we have the account id all that is needed is the password.. Looking further at Haru's commits we see a commit pushing his local config that has a password for MySQL in the config file `Treatment!`.. 

```console
From 14129237ea34eeefbced772092c9264f60b2cefa Mon Sep 17 00:00:00 2001
From: hsato <hsato@MacBook.local>
Date: Tue, 23 Apr 2024 16:38:20 +0100
Subject: [PATCH] Pushing local changes

---
 .env        | 52 ++++++++++++++++++++++++++++++++++++++++++++++++++++
 version.php |  4 ++--
 2 files changed, 54 insertions(+), 2 deletions(-)
 create mode 100644 .env

diff --git a/.env b/.env
new file mode 100644
index 00000000..8c5557ed
--- /dev/null
+++ b/.env
@@ -0,0 +1,52 @@
+APP_NAME=MPSatRecall
+APP_ENV=local
+APP_KEY=
+APP_DEBUG=true
+APP_URL=http://localhost:8000
+
+LOG_CHANNEL=stack
+LOG_DEPRECATIONS_CHANNEL=null
+LOG_LEVEL=debug
+
+DB_CONNECTION=mysql
+DB_HOST=127.0.0.1
+DB_PORT=3306
+DB_DATABASE=laravel
+DB_USERNAME=root
+DB_PASSWORD=Treatment!
+
+BROADCAST_DRIVER=log
+CACHE_DRIVER=file
+FILESYSTEM_DISK=local
+QUEUE_CONNECTION=sync
+SESSION_DRIVER=file
+SESSION_LIFETIME=120
+
+MEMCACHED_HOST=127.0.0.1
+
+REDIS_HOST=127.0.0.1
+REDIS_PASSWORD=null
+REDIS_PORT=6379
+
+MAIL_MAILER=smtp
+MAIL_HOST=mailhog
+MAIL_PORT=1025
+MAIL_USERNAME=null
+MAIL_PASSWORD=null
+MAIL_ENCRYPTION=null
+MAIL_FROM_ADDRESS="devmailer@massive-pharma.com"
+MAIL_FROM_NAME="${APP_NAME}"
+
+AWS_ACCESS_KEY_ID=
+AWS_SECRET_ACCESS_KEY=
+AWS_DEFAULT_REGION=us-east-1
+AWS_BUCKET=
+AWS_USE_PATH_STYLE_ENDPOINT=false
+
+PUSHER_APP_ID=
+PUSHER_APP_KEY=
+PUSHER_APP_SECRET=
+PUSHER_APP_CLUSTER=mt1
+
+MIX_PUSHER_APP_KEY="${PUSHER_APP_KEY}"
+MIX_PUSHER_APP_CLUSTER="${PUSHER_APP_CLUSTER}"
diff --git a/version.php b/version.php
index e3192029..1717eb48 100755
--- a/version.php
+++ b/version.php
@@ -13,7 +13,7 @@
 // upgrade file is the starting point for the next upgrade.
 $v_major = '5';
 $v_minor = '0';
-$v_patch = '2';
+$v_patch = '3';
 $v_tag   = ''; // minor revision number, should be empty for production releases
 
 // A real patch identifier. This is incremented when we release a patch for a
@@ -25,7 +25,7 @@ $v_realpatch = '1';
 // is a database change in the course of development.  It is used
 // internally to determine when a database upgrade is needed.
 //
-$v_database = 295;
+$v_database = 300;
 
 // Access control version identifier, this is to be incremented whenever there
 // is a access control change in the course of development.  It is used
-- 
2.46.1
```
## Account Compromise

Looking at the poor security practices of the user Haru i am sure he reuses passwords for multiple accounts as well. Lets test this password with all the information we have discovered as below

AccountID: 211125382880
IAM User: haru@massive-pharma.com
Password: Treatment!

And we have successfully Pwned Haru's account!

![insert](/assets/img/HaruAccountCompromise.png)

## Finding the flag

Looking around the environment i managed to eventually find the flag in `AWS Secret Manager` 

![insert](/assets/img/ThunderdomeFlag1.png)

And that concludes the "Emerge through the breach" flag from the Thunderdome Cyber Range hosted by [PwnedLabs](https://pwnedlabs.io)

Big shoutout to Ian and the people at PwnedLabs for pushing incredible Cloud Labs at an affordable price!