---
title:  Defender for Cloud Apps file API "Insufficient role based permissions" issue
classes: wide
categories:
  - en-us 
  - Microsoft365
tags:
  - Defender for Cloud Apps

header:
  teaser: /assets/images/posts/2022-07-12-Defender-for-Cloud-Apps-API-file-permission-error-symptom.png

---

## Introduction
Composing a script for analyzing alert in Microsoft Defender for Cloud Apps, I found that [file API][docs_file_api] is not working correctly in some cases.  
I coundn't find any information about this symptom on internet. So I decided to write down on it.  


## Symptom
An error occur calling file API in [application context][docs_application_context] authentication.  
It's reproducible with all file APIs such as list file, fetch file.

![symptom]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-07-12-Defender-for-Cloud-Apps-API-file-permission-error-symptom.png)

```
Invoke-RestMethod : {"detail": "Insufficient role based permissions", "correlation_id": "d0a34b91-baa1-4c3d-809d-db8ca7c10460"}
At line:1 char:8
+ $res = Invoke-RestMethod -Uri "https://kor2.us3.portal.cloudappsecuri ...
+        ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : InvalidOperation: (System.Net.HttpWebRequest:HttpWebRequest) [Invoke-RestMethod], WebException
    + FullyQualifiedErrorId : WebCmdletWebResponseException,Microsoft.PowerShell.Commands.InvokeRestMethodCommand
```

## Solution
Calling file APIs is not available in [application context][docs_application_context]. It's confirmed by Microsoft technical support. (As of 7th July, 2022)  


To call file APIs, use API key created using [legacy method][docs_legacy_method]. It's available on security extension page in Microsoft Defender for Cloud Apps portal.  
  
I attached some screenshots, Docs' are too old images.  

![securityextension]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-07-12-Defender-for-Cloud-Apps-API-file-permission-error-securityextension.png)  
  
  
![apiToken]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-07-12-Defender-for-Cloud-Apps-API-file-permission-error-apiToken.png)  
  
  
A request headr is a little bit different from the one with [application context][docs_application_context] when using [API token][docs_api_token].  
  
```powershell
$apiKey = "your api key"
$authToken = @{"Authorization" = "Token "+ $apikey}
$res = Invoke-RestMethod -Uri "https://kor2.us3.portal.cloudappsecurity.com/api/v1/files/" -Headers $authToken 
```

It's working correctly when using API token.  

![apiToken]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-07-12-Defender-for-Cloud-Apps-API-file-permission-error-legacy.png) 



## How to reproduce the symptom
Here is example code to call file API using [application context][docs_application_context]  

```powershell
# Basic information
$ClientId = "72898937-bfcf-412e-a554-a4e438d6095c"
$ClientSecret = '*'
$TenantId = '023ef17e-4a76-4230-a8f2-e0b9e89627bf'
$resourceAppIdUri = '05a65629-4c1b-48c1-a78b-804c4abdd4af'
$oAuthUri = "https://login.microsoftonline.com/$TenantId/oauth2/token"

$authBody = [Ordered] @{
    resource = "$resourceAppIdUri"
    client_id = "$ClientId"
    client_secret = "$ClientSecret"
    grant_type = 'client_credentials'
}

$authResponse = Invoke-RestMethod -Method Post -Uri $oAuthUri -Body $authBody -ErrorAction Stop
$token = $authResponse.access_token
$authToken = @{'Authorization'='Bearer '+$token}

$res = Invoke-RestMethod -Uri "https://kor2.us3.portal.cloudappsecurity.com/api/v1/files/" -Headers $authToken 
```
  

At this point, all permissions needed to call APIs were assigned correctly. Even though all the other permissions are selected, it's not working correctly as well.  

![apipermission]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-07-12-Defender-for-Cloud-Apps-API-file-permission-error-apipermission.png)

## Some other story...
Pagination function, which is available on activity APIs, it seems not implemented.  
Invoke POST request with `isScan: true` is not working and URL is for next page not returned.
Even if `hasNext` has `true` value, there is no way to get next pages.  

I guess it's only available with activity APIs, it's not mentioned on Docs for alert or file api.  
[Activities requset body parameters][docs_activities]

[docs_file_api]: https://docs.microsoft.com/en-us/defender-cloud-apps/api-files
[docs_application_context]: https://docs.microsoft.com/en-us/defender-cloud-apps/api-authentication-application
[docs_legacy_method]: https://docs.microsoft.com/en-us/defender-cloud-apps/api-tokens-legacy
[docs_api_token]: https://docs.microsoft.com/en-us/defender-cloud-apps/api-introduction#api-tokens
[docs_activities]: https://docs.microsoft.com/en-us/defender-cloud-apps/api-activities-investigate-script#request-body-parameters