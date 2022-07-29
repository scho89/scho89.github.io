---
title:  EWS authentication using applicationimpersonation role
classes: wide
categories:
  - en-us 
  - Microsoft365
tags:
  - Exchange Server
  - PowerShell

header:
  teaser: /assets/images/posts/2022-07-26-EWS-impersonated-authentication-ews-download.png


---

# Introduction
There are more options like [Graph API][docs_graph_api] on Exchange Online compared with Exchange Server for bulk tasks for user mailboxes.
It's necessary to get used to EWS for those jobs on Exchange Server.  
Using service account with [Applicationimpersonation][docs_impersonation] role, let's practice authentication process for bulk jobs. 

# Prerequisites
1. Download EWS managed API
2. Assign Applicationimpersonation for to service account  

## Download EWS Managed API
You can get it from following link: 
[NuGet Gallery | Exchange.WebServices.Managed.Api][EWS_download]  

Using package manager or, direct download is available.  
![ews-download]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-07-26-EWS-impersonated-authentication-ews-download.png)

You can find it on folder named 'lib' after extract ZIP file.    
We will use `Microsoft.Exchange.WebServices.dll`  
![ews-download]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-07-26-EWS-impersonated-authentication-ews-download2.png)


## Assign Applicationimpersonation role
We can use ECP or PowerShell to assign role.  

### Using ECP
On ECP, add new role group by using permissions -> Admin role  
For new added group, assign `Applicationimpersonation` role.  
Add your service account as member.  
Applicationimpersonation also used to mailbox migration from Exchange to 3rd party email services.  

![ecp-role]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-07-26-EWS-impersonated-authentication-role.png)

### Using PowerShell
It can be done by one simple line of cmdlet.  

```powershell
New-RoleGroup -Name "Application impersonation" -Roles "Applicationimpersonation" -Members "admin@3namu.shop"
```

# Connection
In this article, we are uing PowerShell.    

Import downloaded EWS managed API library.  
Please modify file path depends on your environment.    

```powershell
Import-Module "C:\Users\ho\Downloads\microsoft.exchange.webservices.2.2.0\lib\40\Microsoft.Exchange.WebServices.dll"
```

Set target mailbox.   


```powershell
$MailboxName ="on@3namu.shop"
```

Provide credential for service account.  

```powershell
$PSCredential = Get-Credential
```

Create EWS service instance, and add credential, service URL.  
Normally, application (script) connecting to their own mailbox, steps to this point are sufficient to access mailbox using EWS.    

```powershell
$Service = [Microsoft.Exchange.WebServices.Data.ExchangeService]::new()
$Service.Credentials = New-Object System.Net.NetworkCredential($PSCredential.UserName.ToString(),$PSCredential.GetNetworkCredential().password.ToString())
$Service.Url = "https://mail.3namu.shop/EWS/Exchange.asmx"
```

But accessing other mailboxes without permission, an error occurs.  

When trying to connect target mailbox,  
```powershell
$folderid= new-object Microsoft.Exchange.WebServices.Data.FolderId([Microsoft.Exchange.WebServices.Data.WellKnownFolderName]::Inbox ,$MailboxName)
$folder = [Microsoft.Exchange.WebServices.Data.Folder]::Bind($service,$folderid)
```

A permission error occurs like below:  

```
Exception calling "Bind" with "2" argument(s): "The specified object was not found in the store."
At line:2 char:1
+ $folder = [Microsoft.Exchange.WebServices.Data.Folder]::Bind($service ...
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : NotSpecified: (:) [], MethodInvocationException
    + FullyQualifiedErrorId : ServiceResponseException
```

We need to fill `ImpersonatedUserId` property to use impersonated permission.

```powershell
$ImpersonatedUserId = New-Object Microsoft.Exchange.WebServices.Data.ImpersonatedUserId([Microsoft.Exchange.WebServices.Data.ConnectingIdType]::SMTPAddress,$MailboxName)
$service.ImpersonatedUserId = $ImpersonatedUserId
```

If trying to connect mailbox again, we can see it work now.  

![mailbox-connected]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-07-26-EWS-impersonated-authentication-connected.png)



# Apply to... 
A good example for using impersonated permission, we can imagine to empty RecipientCache folder for all mailboxes in organization, retrieved by [`Get-Mailbox`][docs_get_mailbox] cmdlet.  


```powershell
$mailboxes = Get-Mailbox -Resultsize Unlimited

$mailboxes | %{
  $MailboxName = $_.WindowsEmailAddress
  $ImpersonatedUserId = New-Object Microsoft.Exchange.WebServices.Data.ImpersonatedUserId([Microsoft.Exchange.WebServices.Data.ConnectingIdType]::SMTPAddress,$MailboxName)
  $service.ImpersonatedUserId = $ImpersonatedUserId
  $folderid= new-object Microsoft.Exchange.WebServices.Data.FolderId([Microsoft.Exchange.WebServices.Data.WellKnownFolderName]::RecipientCache, $MailboxName)
  $folder = [Microsoft.Exchange.WebServices.Data.Folder]::Bind($service, $folderid)
  $folder.Empty([Microsoft.Exchange.WebServices.Data.DeleteMode]::HardDelete, $true)
}
```

[docs_graph_api]: https://docs.microsoft.com/ko-KR/graph/overview
[EWS_download]: https://www.nuget.org/packages/Exchange.WebServices.Managed.Api/api-authentication-application
[docs_get_mailbox]:https://docs.microsoft.com/ko-kr/powershell/module/exchange/get-mailbox?view=exchange-ps
[docs_impersonation]: https://docs.microsoft.com/en-us/exchange/client-developer/exchange-web-services/impersonation-and-ews-in-exchange