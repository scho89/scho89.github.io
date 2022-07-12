---
title:  "Apps for Microsoft 365 update script"
classes: wide
categories:
  - ko-kr 
  - Microsoft365
tags:
  - PowerShell

header:
  teaser: /assets/images/posts/2022-07-10-Offfice365-Update-script-drop-down-list.png

---

## 작성 배경
함께 일하던 [동료][pepuri]의 아이디어로 만들게 된 스크립트.    
업무 특성 상 Office 클라이언트의 업데이트 채널이나 빌드를 변경할 일이 잦았는데, 이를 좀 더 편하게 하고자 만들었었다.  
무엇이든지 지나고 나서 보면, 이전 작업에 부족한게 많이 보이기 마련이지요.  
민망함을 무릅쓰고 이야기를 적어봅니다. 스크립트를 제작했을 당시 버전의 제 머리(?)를 이용합니다.
  
임직원의 PC를 관리할 수 있는 인프라가 갖춰진 경우, Office 클라이언트의 버전을 관리할 수 있는 방법은 [다양합니다][docs_change_channels].  
하지만, 그런 인프라를 갖추지 못한 환경에서는 특정 버전에서 발생하는 문제가 생긴 경우 이를 대처하는게 여간 까다로운 일이 아닙니다.  
그런 환경에서 이를 통해 좀 더 편하게 문제에 대처할 수 있었으면 하는 마음을 담기도 했었습니다.  
  
시간이 되면, 처음 만들 당시 보다는 좀 더 업그레이드된 버전의 머리(?)를 이용해 정리할 예정입니다.

## 수동으로 업데이트 채널, 빌드 변경
Office 클라이언트의 채널, 빌드를 원하는 값으로 지정하려면 아래의 과정을 진행해야 합니다.  

### Office 클라이언트 업데이트 채널 변경
아래 둘 중 하나의 방법을 이용할 수 있습니다.  
  - OfficeC2RClient.exe를 이용 [채널명][docs_channel_name]
    ```powershell
    "C:\Program Files\Common Files\Microsoft Shared\ClickToRun\OfficeC2RClient.exe" /changesetting Channel=ChannelName
    ```
  - 레지스트리 수정
    - 다음 키를 원하는 채널에 맞게 변경해야합니다.  
      `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Office\ClickToRun\Configuration\CDNBaseUrl`
      ![registry]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-07-10-Offfice365-Update-script-registry.png)

      |채널 명|이전|주소|
      |---|---|---|
      |Beta Channel|Insider Fast|http://officecdn.microsoft.com/pr/5440fd1f-7ecb-4221-8110-145efaa6372f|
      |Current Channel (Preview)|Insider Slow|http://officecdn.microsoft.com/pr/64256afe-f5d9-4f86-8936-8840a6a4f5be|
      |Currnet Channel|Monthly Channel|http://officecdn.microsoft.com/pr/492350f6-3a01-4f97-b9c0-c7c6ddf67d60|
      |Montly Enterprise Channel||http://officecdn.microsoft.com/pr/55336b82-a18d-4dd6-b5f6-9e5095c314a6|
      |Semi-Annual Enterprise Channel (Preview)|Semi-Annual Channel (Targeted)|http://officecdn.microsoft.com/pr/b8f9b850-328d-4355-9145-c59439a0c4cf|
      |Semi-Annual Enterprise Channel|Semi-Annual Channel|http://officecdn.microsoft.com/pr/7ffbc6bf-bc32-4f92-8982-f9dd17fd3114|
      |DevMain Channel (Dogfood)||http://officecdn.microsoft.com/pr/ea4a4090-de26-49d7-93c1-91bff9e53fc3|

    - 아래 키 삭제
      `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Office\ClickToRun\Updates\UpdateToVersion`

### 특정 빌드로 업데이트 (다운그레이드) 진행
각 채널에 맞는 [빌드 번호][docs_update_history]를 참고하여 업데이트 (다운그레이드)를 진행합니다.  
빌드 번호가 맞지 않으면 `OfficeC2RClient.exe`를 실행해도 아무 반응이 없습니다.  

```powershell
"C:\Program Files\Common Files\Microsoft Shared\ClickToRun\OfficeC2RClient.exe" /update user updatetoversion=16.0.0000.0000
```

## 클라이언트 업데이트 스크립트
하지만 이런 작업도 한두번이지요. 여러번 작업할때, 또는 장치가 여러대이면 이만저만 번거로운게 아닙니다.  
그래서 아래와 같은 순서로 작동하는 스크립트를 작성합니다.

  1) 각 채널별 `CDNBaseUrl` 확인  
  2) 각 채널에 맞는 빌드 번호를 확인  
  3) 현재 설정된 `CDNBaseUrl`과 원하는 `CDNBaseUrl` 비교하여 값 변경 / `Updatetoversion` 키 삭제  
  4) 업데이트 진행  
  5) 양식을 사용하여 GUI 형태로 구현  
  6) (필요한 경우) 자동 업데이트를 비활성화  

### 1) 각 채널별 CDNBaseUrl 확인
직접... 테스트를 통해 값을 확인하였습니다.  
[DevMain Channel][dogfood]을 제외한 모든 채널은 Docs문서를 참고하여 채널 변경 후 `CDNBaseUrl` 값을 확인할 수 있습니다.

### 2) 각 채널에 맞는 빌드 번호를 확인
Microsoft Docs에서 각 채널별 빌드 번호를 확인할 수 있습니다.
스크립트에서는 Docs 문서의 HTML 본문에서 채널별 빌드를 정규식을 통해 분류했습니다.

문서 본문은 [`Invoke-WebRequest`][Invoke-WebRequest]를 통해 가져올 수 있습니다.

```powershell
$res = Invoke-WebRequest -Uri "https://docs.microsoft.com/en-us/officeupdates/update-history-microsoft365-apps-by-date"
$html = $res.content
```

가져온 HTML 문서에서 태그와 적절한 키워드를 이용하여 각 채널에 맞는 빌드 목록을 추려냅니다.

```powershell
$current = [regex]::matches( $html, '<a href=\"(monthly-channel|current-channel)(.*?)</a>')
```
  
만약 원본 문서의 HTML 구조가 변경된다면, 정규식으로 필요한 데이터를 필터링 할 수 없어 빈 목록이 표시됩니다.


### 3) 현재 설정된 CDNBaseUrl과 원하는 CDNBaseUrl 비교하여 값 변경 / Updatetoversion 키 삭제
레지스트리 키를 확인, 설정 및 삭제하는 cmdlet은 ItemProperty 명사를 이용합니다.
  - [Get-ItemProperty][Get-ItemProperty]
  - [Set-ItemProperty][Get-ItemProperty]
  - [Remove-ItemProperty][Remove-ItemProperty]

레지스트리 값 수정을 위해 [`Start-Process`][Start-Process]를 이용해 필요한 부분만 상승된 권한(`-Verb RunAs`)으로  실행되게 하였습니다.  

```powershell
if((Get-ItemProperty -Path Registry::HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Office\ClickToRun\Configuration).CDNBaseUrl -ne $CDNBaseUrlCurrent)        {
    $ChannelChanged = $true
    Start-Process powershell.exe -Verb runAs{
    Set-ItemProperty -Path Registry::HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Office\ClickToRun\Configuration -Name CDNBaseUrl -Value "http://officecdn.microsoft.com/pr/492350f6-3a01-4f97-b9c0-c7c6ddf67d60"
    Remove-ItemProperty -Path Registry::HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Office\ClickToRun\Updates -Name UpdateToVersion
    }
}
```

### 4) 업데이트 진행
수동으로 진행할 때와 크게 다르지 않습니다. 
DevMain 채널인 경우 정확한 빌드를 확인할 수 없기 때문에 빌드를 입력하지 않습니다. (해당 채널 최신 버전으로 업데이트.) 
```powershell
if($CmbChannel.Text -ne $nameDevMain){
    $build = "16.0."+(($CmbBuild.text -split "Build ")[1] -split "\)")[0]
    & "$env:CommonProgramFiles\microsoft shared\ClickToRun\OfficeC2RClient.exe" /update user updatetoversion=$build}

else{& "$env:CommonProgramFiles\microsoft shared\ClickToRun\OfficeC2RClient.exe" /update user}
```

### 5) 양식을 사용하여 GUI 형태로 구현 
처음에는 단순히 텍스트로 최신 빌드 목록을 표시해주고 번호로 선택하는 형태로 구현했습니다.  
하지만 시간이 지날수록 빌드 목록을 길어지고, 화면에 표시하기 번거로워 채널과 빌드를 드롭 다운 목록 형태로 만들었습니다.  
  ![drop-down-list]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-07-10-Offfice365-Update-script-drop-down-list.png)
  
일부만 보면... 
기본 폼을 만들고, 드롭 다운 리스트(ComboBox), 버튼, 체크박스 등을 추가합니다.  
드롭 다운 리스틀 어떻게 채우는지, 이벤트 처리 (선택된 채널이 변경되는 경우, 그에 따른 빌드 목록 표시)를 어떻게 하는지 찾는데 시간을 좀 소요했던 기억이 납니다.  

```powershell
#Form 

Add-Type -AssemblyName System.Windows.Forms

$Form = New-Object system.Windows.Forms.Form
$Form.Text = "Office 365 Update Tool"
$Form.Size = New-Object System.Drawing.Size(370,120)
$form.MaximumSize = New-Object System.Drawing.Size(370,150)
$Form.MinimumSize = New-Object System.Drawing.Size(370,150)
$CenterScreen = [System.Windows.Forms.FormStartPosition]::CenterScreen;
$Form.StartPosition = $CenterScreen
$Form.TopMost = $True

$CmbChannel = New-Object System.Windows.Forms.ComboBox
$CmbChannel.Text = "Select release channel..."
$CmbChannel.Location = New-Object System.Drawing.Point(10,15)
$CmbChannel.Size = New-Object System.Drawing.Size(330,80)
$Form.controls.Add($CmbChannel)

# 중략...

$BtnUpdate = New-Object System.Windows.Forms.Button
$BtnUpdate.Text = "Update"
$BtnUpdate.Location = New-Object System.Drawing.Point(265,75)
$BtnUpdate.Enabled = $false
$Form.Controls.Add($BtnUpdate)

$ChkUpdate = New-Object System.Windows.Forms.Checkbox
$ChkUpdate.Text = "Disable updates"
$ChkUpdate.Location = New-Object System.Drawing.Point(10,75)
$ChkUpdate.Size = New-Object System.Drawing.Size(200,20)
$Form.Controls.Add($ChkUpdate)


#Cmb contents
$CmbChannel.Items.Add($nameCurrent) >> $null
$CmbChannel.Items.Add($nameMonthlyEnt) >> $null

# 중략...

#Event handler

$CmbChannel_SelectedIndexChanged =
{
    $BtnUpdate.Enabled = $false
    if($CmbChannel.Text -eq $nameCurrent){
        $CmbBuild.Items.Clear()
        $CmbBuild.Text = "Select build number..."

        for($i=0;$i -lt $current.count;$i++){
            $date_build = ([regex]::matches($current.value[$i],'Version \d{4} \(Build \d{4,5}\.\d{4,5}\)' )).value
            $CmbBuild.Items.Add($date_build)
        }
    }

# 중략...

    elseif($CmbChannel.Text -eq $nameDevMain){
        $BtnUpdate.Enabled = $true
        $CmbBuild.Items.Clear()
        $CmbBuild.Text = "Update to the latest build."       
    }

}

$CmbBuild_SelectedIndexChanged = { $BtnUpdate.Enabled = $True }

```

### 6) (필요한 경우) 자동 업데이트를 비활성화
만약 특정 빌드에 문제가 있어 이전 버전으로 다운그레이드 하는 경우, 다시 자동으로 업데이트가 된다면 힘들게 되돌린 의미가 없어집니다.

  
필요한 경우 자동 업데이트를 중지합니다.  
  ![drop-down-list]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-07-10-Offfice365-Update-script-checkbox.png)
```powershell
if($ChkUpdate.Checked -eq $true){
    Write-Host "Disable updates......."
    Start-Process powershell.exe -Verb runAs{
    Set-ItemProperty -Path Registry::HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Office\ClickToRun\Configuration -Name UpdatesEnabled -Value "False"
    }
```

## 결과
코드 전문은 아래에서 확인할 수 있습니다.  
[Update-Office365][update-office365]

또는 PowerShell에서 아래의 명령어로 스크립트 설치도 가능합니다.  
```powershell
Install-Script Update-Office365
```

설치된 스크립트는 보통 아래 경로에 저장됩니다.  
`C:\Users\<UserName>\Documents\WindowsPowerShell\Scripts`  

[update-office365]: https://github.com/scho89/Update-Office365
[pepuri]: https://blog.limcm.kr/
[docs_change_channels]: https://docs.microsoft.com/en-us/deployoffice/change-update-channels
[docs_channel_name]: https://docs.microsoft.com/en-us/deployoffice/update-channels-changes#office-deployment-tool
[docs_update_history]: https://docs.microsoft.com/en-us/officeupdates/update-history-microsoft365-apps-by-date
[dogfood]: https://devblogs.microsoft.com/oldnewthing/20110802-00/?p=10003
[Invoke-WebRequest]: https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-webrequest?view=powershell-7.2
[Get-ItemProperty]: https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-itemproperty?view=powershell-7.2
[Set-ItemProperty]: https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.management/set-itemproperty?view=powershell-7.2
[Remove-ItemProperty]: https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.management/remove-itemproperty?view=powershell-7.2
[Start-Process]: https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.management/start-process?view=powershell-7.2