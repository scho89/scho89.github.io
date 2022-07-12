---
title:  Defender for Cloud Apps file API "Insufficient role based permissions" issue
classes: wide
categories:
  - ko-kr 
  - Microsoft365
tags:
  - Defender for Cloud Apps

header:
  teaser: /assets/images/posts/2022-07-10-Offfice365-Update-script-drop-down-list.png

---

## 서론
Microsoft Defender for Cloud Apps alert 및 세부내용 분석을 위한 스크립트를 작성하면서 특정 조건에서 [file API][docs_file_api]가 작동하지 않는것을 확인했습니다.  
이곳 저곳 돌아다니며 찾아봤지만 관련 정보를 찾을 수 없었습니다. 혹시 다른이들에게 도움이 될까 하여 글로 남겨둡니다.  

## 증상
[application context][docs_application_context]를 통해 인증한 뒤 file API를 호출하면 권한 오류가 발생합니다.  
증상은 list file, fetch file 모두 발생합니다.  

![symptom]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-07-12-Defender-for-Cloud-Apps-API-file-permission-error-symptom.png)

```
Invoke-RestMethod : {"detail": "Insufficient role based permissions", "correlation_id": "d0a34b91-baa1-4c3d-809d-db8ca7c10460"}
위치 줄:1 문자:8
+ $res = Invoke-RestMethod -Uri "https://kor2.us3.portal.cloudappsecuri ...
+        ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : InvalidOperation: (System.Net.HttpWebRequest:HttpWebRequest) [Invoke-RestMethod], WebException
    + FullyQualifiedErrorId : WebCmdletWebResponseException,Microsoft.PowerShell.Commands.InvokeRestMethodCommand
```

## 해결 방법
Microsoft를 통해 확인 시 [application context][docs_application_context] 통해서 인증하는 경우 file API를 사용할 수 없습니다. (2022/7/7 기준)  


File API를 이용하기 위해서는 MDCA 포털에서 [레거시 방식][docs_legacy_method]의 API 키를 생성해서 사용해야합니다. (보안 확장 -> API 키)  
(문서의 예제에서는 이 방법으로 설명하고 있습니다.)
  
문서의 그림이 오래되어 하나 첨부합니다.  
![securityextension]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-07-12-Defender-for-Cloud-Apps-API-file-permission-error-securityextension.png)  
  
  
![apiToken]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-07-12-Defender-for-Cloud-Apps-API-file-permission-error-apiToken.png)  
  
  
[API token][docs_api_token]을 사용하는 경우, Microsoft에서 권장하는 방식인 [application context][docs_application_context]과는 좀 다른 형태로 요청 헤더를 만들어야합니다.  

  
```powershell
$apiKey = "your api key"
$authToken = @{"Authorization" = "Token "+ $apikey}
$res = Invoke-RestMethod -Uri "https://kor2.us3.portal.cloudappsecurity.com/api/v1/files/" -Headers $authToken 
```

API 토큰을 이용하는 경우 아래와 같이 정상작동합니다.    

![apiToken]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-07-12-Defender-for-Cloud-Apps-API-file-permission-error-legacy.png) 



## 증상 재현 방법
[application context][docs_application_context]를 이용해 file API를 호출하는 샘플 코드는 다음과 같습니다.

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
  

이떄 API 호출에 필요한 권한(investigation.read)는 어플리케이션에 할당되어 있습니다. 다른 모든 권한을 할당하더라도 증상은 해결되지 않습니다.    

![apipermission]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-07-12-Defender-for-Cloud-Apps-API-file-permission-error-apipermission.png)

## 기타 의견
Activity API에서 제공하는 pagination을 alert이나 file에서는 제공하지 않는 것 같습니다.  
`hasNext`가 `true`이지만 `isScan: true`를 body에 넣어 POST를 보내도 다음 페이지의 쿼리 주소는 제공되지 않습니다. Alert이나 file문서에 따로 언급이 없는 것을 보니, Activity 에서만 기능을 제공하는것 같습니다.  

[Activities requset body parameters][docs_activities]

[docs_file_api]: https://docs.microsoft.com/en-us/defender-cloud-apps/api-files
[docs_application_context]: https://docs.microsoft.com/en-us/defender-cloud-apps/api-authentication-application
[docs_legacy_method]: https://docs.microsoft.com/en-us/defender-cloud-apps/api-tokens-legacy
[docs_api_token]: https://docs.microsoft.com/en-us/defender-cloud-apps/api-introduction#api-tokens
[docs_activities]: https://docs.microsoft.com/en-us/defender-cloud-apps/api-activities-investigate-script#request-body-parameters