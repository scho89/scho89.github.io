---
title:  "Apps for Microsoft 365 update script"
classes: wide
categories: 
  - en-us
  - Microsoft365
tags:
  - PowerShell
hidden: true
---

## Background
It started from idea of [colleague][pepuri].  
  
It was frequent job, changing Office client's update channel and build number. Due to nature of technical supporting things. So, I composed this script to make my work easier and more conveniently.  
    
Sometimes, what we made always seems too ugly, with time flies go on…  
(In other words, maybe it can say that our skills are better now than the time when birth of our works…)  
Even though It’s embarrassment on my work, I write down a story on it.  
  
Someday, if I have a chance, I’ll make some enhancement on logics and entire structure.  

## Changing update channel and builds manually
Following tasks need to be done to change update channel and builds for Office 365 click to run client.  

### Changing update channel for Office client
You can choose one of the below:  
  - Using OfficeC2RClient.exe with [Channel name][docs_channel_name]
    ```powershell
    "C:\Program Files\Common Files\Microsoft Shared\ClickToRun\OfficeC2RClient.exe" /changesetting Channel=ChannelName
    ```
  - Modifing registry key
    - Set following key as proper value, according to channel name.      
      `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Office\ClickToRun\Configuration\CDNBaseUrl`
      ![registry]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-07-10-Offfice365-Update-script-registry.png)

      |Channel name|Previous name|Value|
      |---|---|---|
      |Beta Channel|Insider Fast|http://officecdn.microsoft.com/pr/5440fd1f-7ecb-4221-8110-145efaa6372f|
      |Current Channel (Preview)|Insider Slow|http://officecdn.microsoft.com/pr/64256afe-f5d9-4f86-8936-8840a6a4f5be|
      |Currnet Channel|Monthly Channel|http://officecdn.microsoft.com/pr/492350f6-3a01-4f97-b9c0-c7c6ddf67d60|
      |Montly Enterprise Channel||http://officecdn.microsoft.com/pr/55336b82-a18d-4dd6-b5f6-9e5095c314a6|
      |Semi-Annual Enterprise Channel (Preview)|Semi-Annual Channel (Targeted)|http://officecdn.microsoft.com/pr/b8f9b850-328d-4355-9145-c59439a0c4cf|
      |Semi-Annual Enterprise Channel|Semi-Annual Channel|http://officecdn.microsoft.com/pr/7ffbc6bf-bc32-4f92-8982-f9dd17fd3114|
      |DevMain Channel (Dogfood)||http://officecdn.microsoft.com/pr/ea4a4090-de26-49d7-93c1-91bff9e53fc3|

    - Remove following key
      `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Office\ClickToRun\Updates\UpdateToVersion`

### Update (downgrade) to specific build
Update or downgrade to proper [build numbers][docs_update_history].  
With incorrect build number, there will be no response with executing `OfficeC2RClient.exe`  

```powershell
"C:\Program Files\Common Files\Microsoft Shared\ClickToRun\OfficeC2RClient.exe" /update user updatetoversion=16.0.0000.0000
```

## Office 365 client update script
But it's too annoying job, if it should be done multiple time for lots of devices.  
So, I wrote down a script running following tasks.  

  1) Get `CDNBaseUrl` for each update channels   
  2) Get proper build numbers for each update channels    
  3) Read current `CDNBaseUrl` and compare with ones for channel to change / Remove `Updatetoversion`   
  4) Update client  
  5) Create GUI with forms    
  6) (In case of needed) disable automatic update    

### 1) Get `CDNBaseUrl` for each update channels
I checked it manually... for each update channels.    
Except [DevMain Channel][dogfood], you can get all `CDNBaseUrl` value for each channel by changing it!

### 2) Get proper build numbers for each update channels
You can refer [Microsoft Docs][docs_update_history] for all build numbers for Office click to run client.  
I filter it using regular expression from HTML body.  

You can get HTML body using [`Invoke-WebRequest`][Invoke-WebRequest].

```powershell
$res = Invoke-WebRequest -Uri "https://docs.microsoft.com/en-us/officeupdates/update-history-microsoft365-apps-by-date"
$html = $res.content
```

Filter out build numbers with proper keywords from HTML.

```powershell
$current = [regex]::matches( $html, '<a href=\"(monthly-channel|current-channel)(.*?)</a>')
```

It may displays empty lists, in case of changing on structure of HTML documents, by Docs author.  


### 3) Read current `CDNBaseUrl` and compare with ones for channel to change / Remove `Updatetoversion` 
Use noun as ItemProperty in cmdlets to get, set, remove registry key.  
  - [Get-ItemProperty][Get-ItemProperty]
  - [Set-ItemProperty][Get-ItemProperty]
  - [Remove-ItemProperty][Remove-ItemProperty]

I used `-Verb RunAs` as parameter for the least privilege principle, to modify registry keys.   

```powershell
if((Get-ItemProperty -Path Registry::HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Office\ClickToRun\Configuration).CDNBaseUrl -ne $CDNBaseUrlCurrent)        {
    $ChannelChanged = $true
    Start-Process powershell.exe -Verb runAs{
    Set-ItemProperty -Path Registry::HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Office\ClickToRun\Configuration -Name CDNBaseUrl -Value "http://officecdn.microsoft.com/pr/492350f6-3a01-4f97-b9c0-c7c6ddf67d60"
    Remove-ItemProperty -Path Registry::HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Office\ClickToRun\Updates -Name UpdateToVersion
    }
}
```

### 4) Update client
Same as the one manually.  
For DevMain channel, execute it without the build number, because it's hard to get correct build number.  
```powershell
if($CmbChannel.Text -ne $nameDevMain){
    $build = "16.0."+(($CmbBuild.text -split "Build ")[1] -split "\)")[0]
    & "$env:CommonProgramFiles\microsoft shared\ClickToRun\OfficeC2RClient.exe" /update user updatetoversion=$build}

else{& "$env:CommonProgramFiles\microsoft shared\ClickToRun\OfficeC2RClient.exe" /update user}
```

### 5) Create GUI with forms
It was only in text originally. It just displays latest build number and make you choose it.  
But it was getting harder and harder as more update released.  
So I made a from with drop-down list for builds and channels.  
  ![drop-down-list]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-07-10-Offfice365-Update-script-drop-down-list.png)
  
On part of full script... 
You can see creating basic form and adding drop-down list (ComboBox), checkbox, etc...   
I remember that it took a few hours to fill drop-down list with values and event handler (builds should shows according changes of selected channel)    

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

# ...

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

# ...

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

# ...

    elseif($CmbChannel.Text -eq $nameDevMain){
        $BtnUpdate.Enabled = $true
        $CmbBuild.Items.Clear()
        $CmbBuild.Text = "Update to the latest build."       
    }

}

$CmbBuild_SelectedIndexChanged = { $BtnUpdate.Enabled = $True }

```

### 6) (In case of needed) disable automatic update
What if, we downgraded it because some bugs on specific build, but it update again itself automatically...  what we have done is in vain..  

In that case, we can disable automatic update.    
  ![drop-down-list]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-07-10-Offfice365-Update-script-checkbox.png)
```powershell
if($ChkUpdate.Checked -eq $true){
    Write-Host "Disable updates......."
    Start-Process powershell.exe -Verb runAs{
    Set-ItemProperty -Path Registry::HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Office\ClickToRun\Configuration -Name UpdatesEnabled -Value "False"
    }
```

## Result
Entire code is available here: [Update-Office365][update-office365]  

Or, you may install it from PowerShell.  
```powershell
Install-Script Update-Office365
```

Installed scripts are stored following path by default:    
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