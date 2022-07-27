---
title:  Applicationimpersonation 역할을 통한 EWS 인증
classes: wide
categories:
  - ko-kr 
  - Microsoft365
tags:
  - Exchange Server
  - PowerShell

header:
  teaser: /assets/images/posts/2022-07-26-EWS-impersonated-authentication-ews-download.png


---

# 서론
Microsoft 365의 Exchange Online의 경우 EWS뿐 아니라 [Graph API][docs_graph_api]를 제공하기 때문에 사용자 사서함에 일괄 작업을 하기가 Exchange Server와 비교했을 때 더 다양한 선택지가 있습니다.  
Exchange Server의 사서함에 일괄 작업을 하기 위해서는 EWS에 익숙해져야 할 필요가 있습니다. [Applicationimpersonation][docs_impersonation] 역할을 가진 계정으로 일괄 작업을 위한 인증 과정을 정리해봅니다.

# 사전 준비
1. EWS managed API 다운로드
2. Applicationimpersonation 역할 할당

## EWS Managed API 다운로드
아래 링크에서 다운로드 받을 수 있습니다.  
[NuGet Gallery | Exchange.WebServices.Managed.Api][EWS_download]  

패키지 관리자를 이용하거나, 직접 파일을 다운로드 받을 수 있습니다.  
![ews-download]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-07-26-EWS-impersonated-authentication-ews-download.png)

ZIP 파일을 압축해제 후 lib 폴더를 보면 필요한 파일들이 담겨있습니다.  
`Microsoft.Exchange.WebServices.dll`를 사용합니다.  
![ews-download]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-07-26-EWS-impersonated-authentication-ews-download2.png)


## Applicationimpersonation 역할 할당
ECP 또는 PowerShell을 통해 진행합니다.  

### ECP를 이용해 역할 할당
ECP에서 permissions -> Admin roles 메뉴로 이동 후 새로운 역할 그룹을 추가합니다.  
추가한 그룹에는 `Applicationimpersonation` 역할을 넣어줍니다.  
구성원으로는 작업에 사용할 서비스 계정을 넣어줍니다.  
Applicationimpersonation 역할은 Exchange에서 3rd party 메일 서비스로 마이그레이션 작업 시 사용하기도 합니다.  

![ecp-role]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-07-26-EWS-impersonated-authentication-role.png)

### PowerShell을 이용해 역할 할당
간단하게 한줄로 생성과 구성원 추가를 해줄 수 있습니다.  

```powershell
New-RoleGroup -Name "Application impersonation" -Roles "Applicationimpersonation" -Members "admin@3namu.shop"
```

# 연결
PowerShell을 이용합니다.  


다운로드 받은 EWS managed API 라이브러리를 가져옵니다.  
경로는 상황에 맞게 변경해주세요.  

```powershell
Import-Module "C:\Users\ho\Downloads\microsoft.exchange.webservices.2.2.0\lib\40\Microsoft.Exchange.WebServices.dll"
```

연결할 대상 사서함을 지정합니다.  


```powershell
$MailboxName ="on@3namu.shop"
```

자격증명 입력. 서비스 계정의 자격증명을 입력받습니다.  

```powershell
$PSCredential = Get-Credential
```

EWS 서비스 개체를 생성하고 필요한 생성한 자격증명과, URL을 넣어줍니다.  
어플리케이션 (스크립트)이 본인의 사서함에 어떤 작업을 진행하는 경우에는 여기까지만 진행해도 사서함에 접근이 가능합니다.  

```powershell
$Service = [Microsoft.Exchange.WebServices.Data.ExchangeService]::new()
$Service.Credentials = New-Object System.Net.NetworkCredential($PSCredential.UserName.ToString(),$PSCredential.GetNetworkCredential().password.ToString())
$Service.Url = "https://mail.3namu.shop/EWS/Exchange.asmx"
```

다만, 다른 사용자의 사서함에 접근한 경우에는 아래와 같은 오류가 발생합니다.

대상 사서함의 받은 편지함 연결을 시도한 경우,
```powershell
$folderid= new-object Microsoft.Exchange.WebServices.Data.FolderId([Microsoft.Exchange.WebServices.Data.WellKnownFolderName]::Inbox ,$MailboxName)
$folder = [Microsoft.Exchange.WebServices.Data.Folder]::Bind($service,$folderid)
$folder.Empty([Microsoft.Exchange.WebServices.Data.DeleteMode]::HardDelete, $true);
```

아래와 같은 권한 오류가 발생합니다.  

```
"2"개의 인수가 있는 "Bind"을(를) 호출하는 동안 예외가 발생했습니다. "The request failed. 원격 서버에서 (401) 권한이 없음 오류를 반환했습니다."
위치 줄:1 문자:1
+ $folder = [Microsoft.Exchange.WebServices.Data.Folder]::Bind($service ...
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : NotSpecified: (:) [], MethodInvocationException
    + FullyQualifiedErrorId : ServiceRequestException
```

Impersonated 된 권한을 사용하기 위해서는 생성한 서비스 객체에 `ImpersonatedUserId` 값을 채워줘야합니다. 

```powershell
$ImpersonatedUserId = New-Object Microsoft.Exchange.WebServices.Data.ImpersonatedUserId([Microsoft.Exchange.WebServices.Data.ConnectingIdType]::SMTPAddress,$MailboxName)
$service.ImpersonatedUserId = $ImpersonatedUserId
```

다시한번 받은 편지함 연결을 시도해[보면 정상적으로 연결됩니다.
![mailbox-connected]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-07-26-EWS-impersonated-authentication-connected.png)



# 활용
한가지 예시를 들어보자면, [`Get-Mailbox`][docs_get_mailbox]로 가져온 사서함 목록을 이용해 모든 사용자의 RecipientCache 폴더를 비워주는 작업을 생각해 볼 수 있습니다.  

```powershell
$mailboxes = Get-Mailbox -Resultsize Unlimited

$mailboxes | %{
  $MailboxName = $_.WindowsEmailAddress
  $folderid= new-object Microsoft.Exchange.WebServices.Data.FolderId([Microsoft.Exchange.WebServices.Data.WellKnownFolderName]::RecipientCache, $MailboxName)
  $folder = [Microsoft.Exchange.WebServices.Data.Folder]::Bind($service, $folderid)
  $folder.Empty([Microsoft.Exchange.WebServices.Data.DeleteMode]::HardDelete, $true)
}
```

[docs_graph_api]: https://docs.microsoft.com/ko-KR/graph/overview
[EWS_download]: https://www.nuget.org/packages/Exchange.WebServices.Managed.Api/api-authentication-application
[docs_get_mailbox]:https://docs.microsoft.com/ko-kr/powershell/module/exchange/get-mailbox?view=exchange-ps
[docs_impersonation]: https://docs.microsoft.com/en-us/exchange/client-developer/exchange-web-services/impersonation-and-ews-in-exchange