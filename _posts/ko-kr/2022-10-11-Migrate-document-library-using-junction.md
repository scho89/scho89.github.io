---
title:  Junction을 이용한 문서 라이브러리 마이그레이션

categories:
  - ko-kr 
  - Microsoft365
tags:
  - SharePoint

toc_sticky: true

header:
  teaser: /assets/images/posts/2022-10-11-Migrate-cloud-storage-using-junction-powershell1.png


---

# 서론
Microsoft 365 테넌트 간 데이터를 옮겨야하는 일이 종종 발생하곤 합니다.  
이때 Windows의 junction을 이용하여 좀 더 간편하고, 연속성있게 데이터를 옮길 수 방법을 연구해보았습니다.    

본 예시에서는 A 테넌트의 OneDrive 및 팀 사이트의 문서 라이브러리 데이터를 B 테넌트의 OneDrive 또는 팀 사이트로 데이터를 이전하면서, 원본 데이터를 이용할 수 있는 방법을 보여줍니다.  

# 원본 데이터 준비  
여기 현재 사용 중인 OneDrive와 Teams 데이터가 있습니다. 테넌트 이름은 "sanghocho"입니다.
![OneDriveWeb]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-10-11-Migrate-cloud-storage-using-junction-OneDrive1.png)  
  
![TeamsWeb]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-10-11-Migrate-cloud-storage-using-junction-teams1.png)  

원본 데이터를 모두 다운로드하기 위해 OneDrive 클라이언트를 이용해 문서 라이브러리를 모두 동기화 했습니다.  
![synced]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-10-11-Migrate-cloud-storage-using-junction-explorer.png)

그리고, 모든 데이터를 다운로드 받기 위해 요청 기반 파일 다운로드 기능[(On-demand for Windows)][on-demand]을 꺼줬습니다.  
![OnDemandOff]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-10-11-Migrate-cloud-storage-using-junction-OneDrive-DisableOnDemand.png)  

잠시 후 모든 데이터가 다운로드 완료됩니다. 탐색기에서 파일 및 폴더의 [동기화 상태를 확인][sync-status] 할 수 있습니다.  
![synced]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-10-11-Migrate-cloud-storage-using-junction-synced.png)




# 대상 저장소 동기화  
데이터가 이사갈 저장소도 서비스에서 제공하는 동기화 클라이언트를 이용합니다. 이사갈 테넌트는 "scho1"입니다.
![addAccount]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-10-11-Migrate-cloud-storage-using-junction-AddOneDrive.png)  


# Junction 생성  
데이터가 이사갈 저장소 로컬 경로에 [junction][junctions]을 생성합니다.  

각 계정의 동기화 경로는 아래와 같습니다.  
- blog@scho.kr (OneDrive 원본): `C:\Users\ho\OneDrive - Ho`
- blog@scho.kr (팀 사이트 원본): `C:\Users\ho\Ho\scho blog - General`
- a@watc.shop (이동 대상): `C:\Users\ho\OneDrive - Contoso`

PowerShell을 열고 각 경로로 이동 후 junction을 생성합니다. [`New-Item`][New-Item] cmdlet을 사용합니다.  

```powershell
Set-Location "C:\Users\ho\OneDrive - Contoso"
New-Item -Name "Migrated OneDrive data" -Type Junction -Value "C:\Users\ho\OneDrive - Ho" 
New-Item -Name "Migrated Teams data" -Type Junction -Value "C:\Users\ho\Ho\scho blog - General" 

```
  
Juntion을 만들고, 정보를 보면 모드 열에서 link를 의미하는 "l"를 확인할 수 있습니다.  

![create-junction]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-10-11-Migrate-cloud-storage-using-junction-powershell1.png)  

잠시 후 Contoso에 파일이 동기화 되는 것을 확인할 수 있습니다.  
![sycned2]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-10-11-Migrate-cloud-storage-using-junction-synced2.png)  

필요하다면 이를 응용하여 팀 사이트간, 또는 OneDrive에서 팀 사이트로 데이터 이동이 가능합니다.  
데이터 이동 기간 동안 원본에서 수정된 데이터는 자동으로 대상 저장소에도 반영됩니다.  

테스트를 해보니, 간혹 동기화가 바로 안되는 경우가 있었습니다. 이때는 우측 하단 트레이의 OneDrive 아이콘에서 동기화 일시 중지 -> 동기화 재개를 진행하면 됩니다. 또는 컴퓨터를 처음 켤때 OneDrive가 실행되면서 이전 변경 사항을 동기화 합니다.  
![pause]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-10-11-Migrate-cloud-storage-using-junction-pause.png)  

# 데이터 이동 마무리  
데이터를 모두 동기화 한 뒤, 더 이상 옮길 데이터가 없는 경우 <span style="color:red">*대상 테넌트의 동기화를 해제 후*</span> 동기화 경로의 junction을 지워줍니다.  
동기화를 해제하지 않고 junction을 삭제하는 경우, 삭제한 동작도 대상 테넌트의 저장소에 반영됩니다. 그 뒤에 일반적인 경우와 같이 OneDrive (또는 팀 사이트)를 동기화 후 사용합니다.  
동기화를 끊었다가 다시 연결하는 이유는 junction을 삭제하여 원본 데이터와 연결을 끊기 위함입니다.  

![sycned2]({{ site.url }}{{ site.baseurl }}/assets/images/posts/2022-10-11-Migrate-cloud-storage-using-junction-disconnect.png)  



[on-demand]: https://support.microsoft.com/en-us/office/save-disk-space-with-onedrive-files-on-demand-for-windows-0e6860d3-d9f3-4971-b321-7092438fb38e
[sync-status]: https://support.microsoft.com/en-us/office/what-do-the-onedrive-icons-mean-11143026-8000-44f8-aaa9-67c985aa49b3
[Junctions]:https://learn.microsoft.com/en-us/windows/win32/fileio/hard-links-and-junctions
[New-Item]: https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/new-item?view=powershell-7.2