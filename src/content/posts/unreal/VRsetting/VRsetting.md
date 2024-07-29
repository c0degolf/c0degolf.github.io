---
title: 언리얼 VR 개발환경 구축
published: 2024-07-10
description: 'for META QUEST'
image: './banner.jpg'
tags: [VR, Develop]
category: 'Unreal Engine'
draft: false
---

# 서론
프로젝트 생성할때마다 여기저기 찾아보면서 세팅하기 귀찮아서 정리한다.<br>
필자는 UE5.3.2, QUEST2로 개발한다. 프로그램마다 기본 언어가 필자와 다르게 설정되어 있다면 엉뚱한 걸 설치할 수 있다는 점을 유의해두길(그래봤자 기초 영단어다).

# 설치해야 할 것
별도의 언급이 없는 한, 모든 installer는 default로 설치한다. 자세한 방법은 아래에 설명한다.
1. Unreal Engine 5.3.2 (당연하다)
2. Android Studio Flamingo | 2022.2.1 Patch 2 May 24, 2023 (Installer > Windows (64-bit)) [#](https://developer.android.com/studio/archive)
3. Java SE Development Kit 17.0.10 Windows x64 Compressed Archive [#](https://www.oracle.com/java/technologies/javase/jdk17-archive-downloads.html)
4. Meta Quest (핸드폰 앱)
5. Oculus Desktop App [#](https://www.meta.com/quest/setup/)  
6. Meta Quest Developer Hub for Windows [#](https://developer.oculus.com/downloads/package/oculus-developer-hub-win)  
7. Unreal Engine 5 Integration v65 (Meta XR Plugin) [#](https://developer.oculus.com/downloads/package/unreal-engine-5-integration/)  
8. Unreal Engine 5 Platform v65 (Meta XR Platform) [#](https://developer.oculus.com/downloads/package/unreal-5-platform-sdk-plugin)  
9. Meta XR Simulator v65 [#](https://developer.oculus.com/downloads/package/meta-xr-simulator/)

### 사전 준비
+ [oculus](https://developer.oculus.com/)에서 개발자 계정을 만든다. (혹시 모르니 나이는 그냥 성인으로 하자)

### Android Studio Flamingo
installer 설치 후, 시작 화면 아래에 `More Actions > SDK Manager`메뉴에 들어가 아래 것들을 체크한다.<br>
만일 아래의 버전들이 안보인다면 우측 하단의 옵션을 on/off 해보자
+ SDK Platform: 
	+ `Android API 34` 
	+ `Android 12L (Sv2)`
+ SDK Tools (Name 말고 Version 기준): 
	+ `Android SDK Build-Tools 35-rc2 > 34.0.0, 33.0.1`
	+ `NDK (Side by Side) > 25.1.8937393`
	+ `Android SDK Command-line Tools (latest) > 11.0`
	+ `CMake > 3.10.2`
	+ `Android Emulator`
	+ `Android Emulator hypervisor driver (installer)`,
	+ `Android  SDK Platform-Tools`
  
다 했다면 확실히 체크했는지 검토하고 Apply를 누른다. 끝났다면 윈도우 재시작!

### Java SE Development Kit
`C:\Program Files`에 다운로드 받은 파일을 압축 해제한다. `jdk-17.0.10`폴더가 생긴다.

### Meta Quest Mobile App 
**이 단계가 가장 중요하다 안된다고 넘어가고 나중에 하면 안된다**<br>
Android / IOS 상관없이 스토어에서 `Meta Quest`를 설치한다.<br>
QUEST를 근처에 두고 `메뉴 > 기기 > 새로운 연결`에서 본인 기기를 누른다.<br>
그 후 페어링이 완료되면 QUEST의 PIN 번호를 입력한다.

:::note[PIN 번호는]
+ QUEST 초기 세팅부터 할 경우: 세팅 단계 중 폰에 연결하라고 PIN 번호를 알려준다
+ QUEST 세팅이 완료되어 있는 경우: `설정 > 시스템 > 정보`에서 페어링 코드를 확인 할 수 있다.
:::

나는 이 단계에서 1주일 이상 소요되었다. PIN 번호를 입력해도 이미 연결된 기기라고 뜨면서 연결을 거부했다.<br>
내가 해결한 방법으로 추론하건데(어디까지나 나의 경험이다. 맹신하지 말자), QUEST의 ROOT 사용자 계정으로 앱에 로그인해서 연결해야 하는 것 같다.그리고 한 기기당 앱에서 연결할 수 있는 계정 수는 하나인 것 같다.<br>
그래서 나는 그냥 초기화하고 다시 세팅하면서 앱에 연결했다.

연결되면 QUEST를 킨 상태로 `기기 > (QUEST 모델) > 헤드셋 설정 > 개발자 모드`에 들어가 `개발자 모드`를 켜준다.

추가적으로, 헤드셋 설정에서 QUEST wifi연결 등등 꽤 많은 것을 할 수 있다.

### Oculus Desktop App
Desktop App을 설치한 후, QUEST와 컴퓨터를 USB로 연결한다(같은 wifi라면 무선연결도 된다).<br>
그 후, QUEST를 끼고, `설정 > 시스템 > 개발자`메뉴의 모든 옵션들을 킨다.

이제 잠깐 쉬자! YOUTUBE VR로 우월하게 영상을 감상하자!

### Meta Quest Developer Hub
installer 설치 후, 실행하면 자동으로 ADB를 감지하는데, Meta ADB말고 Android ADB를 사용하자.<br>
만일 자동으로 감지하지 않는다면, `Settings > ADB path`를 수정해준다.
끝났다면 윈도우 재시작!

### 7, 8, 9 plugins
이 친구들은 Zip 파일을 그냥 다운로드만 받아둔다. 나중에 쓰인다.

### _이제 힘든 설치는 다 끝났다 언리얼 설정만 마치면 된다_

# Unreal Engine
### Android Setup
`UE_5.3\Engine\Extras\Android\SetupAndroid.bat`을 실행한다. 보통 `C:\Program Files\Epic Games\`에 있다.<br>
만일 `Unable to locate sdkmanager.bat. Did you run Android Studio and install cmdline-tools after installing?`란 오류가 뜰 경우, `SetupAndroid.bat`의 코드를 약간 수정해야 한다. 

vscode/메모장으로 `SetupAndroid.bat`를 열고 86번 줄을 보면 
```bat
set  SDKMANAGER=%STUDIO_SDK_PATH%\cmdline-tools\latest\bin\sdkmanager.bat
```
여기서 우린 cmdline-tools를 11.0으로 설치했기 때문에 `cmdline-tools\latest`가 아닌 `cmdline-tools\11.0`으로 수정해야 한다.<br>
혹시 모르니 `%LOCALAPPDATA%\Android\Sdk\cmdline-tools\11.0\bin`에 `sdkmanager.bat`이 있는지 확인하자. ~~이외의 오류는 구글링으로~~

성공적으로 설치했다면
```bat
if  EXIST  "%NDKINSTALLPATH%" (
	echo Success!
	setx NDKROOT "%NDKINSTALLPATH%"
	setx NDK_ROOT "%NDKINSTALLPATH%"
)
```
이런 느낌의 문자열이 출력된다.

### Unreal Project
`meta_xr_simulator_v65.zip`을 `C:\Users\이름\Documents\Unreal Projects`에 압축 해제한다.<br>
`.tgz`까지 압축을 풀면 `com.meta.xr.simulator`폴더가 생긴다.<br>
이를 `meta_xr_simulator_v65`라는 이름으로 `Documents\Unreal Projects\`에 둔다. < 나중에 쓰인다

언리얼을 실행시킨 후, `GAMES > Virtual Reality` 프로젝트를 하나 생성한다.<br>
그 후 프로젝트 폴더(보통 `C:\Users\사용자\Documents\Unreal Projects\프로젝트명`)에 `Plugins`폴더를 생성하고, 아까 다운로드한 `UnrealMetaXRPlugin.65.0.zip`, `Unreal5PlatformSDKPlugin.65.0.zip`를 압축 해제한다.<br>
`Plugins`폴더에 `MetaXR`, `MetaXRPlatform`폴더가 있어야 한다. 이제 프로젝트를 언리얼에서 실행시킨다.<br>

`Edit > Plugins`에서 `Meta XR`을 검색해 두 플러그인이 모두 켜져있는지 확인한다.<br>
`Edit > Project Settings > Platforms > Android`에서 `Minimum SDK Version: 29`, `Target SDK Version: 32`로 설정하고, 빨간 상자의 `Configure now`를 눌러 초록색으로 만든다(맨 위에 하나 있고, 스크롤하면 빨간 상자가 더 있으니 모두 눌러주자).<br>
`Platforms > Android SDK`에서 `SDKConfig`를 설정한다.
```
Location of Android SDK: C:/Users/이름/AppData/Local/Android/Sdk  
Location of Android NDK: C:/Users/이름/AppData/Local/Android/Sdk/ndk/25.1.8937393  
Location of Java: C:/Program Files/jdk-17.0.10  
SDK API Level: android-32  
NDK API Level: android-29
```
기본적으로 다 설정되어 있다.

`Project Settings > Plugins > Meta XR`에서 
```
General > 
XR API: Epic Native OpenXR with Oculus vendor extensions
Color Space: P3 (Recommended)
Controller Pose Alignment: Default
```
```
PC > 
Meta XR Simulator JSON file : C:\Users\이름\Documents\Unreal Projects\meta_xr_simulator_v65\package\MetaXRSimulator\meta_openxr_simulator.json
```
로 설정해주고 재시작한다.

***이제 진짜 끝났다!!***<br>
컴퓨터와 QUEST를 연결해 QUEST LINK에 접속한다.<br>
프로젝트 재시작 후, 상단의 플레이 버튼(초록 삼각형) 옆 점 3개 버튼에서 `VR preview`를 선택한 다음 실행한다.<br>
그러면 QUEST에 연동되어 VR 플레이가 가능해진다!!!!!

### BUILD
Epic Games Launcher에서 `Library > 5.3.2 luanch 옆 삼각형 > Options > Target Platforms > Android`를 체크하고 Apply를 눌러 설치한다.

설치되었다면 프로젝트 상단의 `Platforms > Android > Package Project`를 누르고, 프로젝트 폴더에 `Quest`란 폴더를 만들고 선택한다.<br>
빌드(좀 걸린다)가 다 되었다면, `Meta Quest Developer Hub`를 실행한다. QUEST연결 후, `Device Manager > +Add Build`를 누르고, 방금 빌드한 `.apk`파일을 선택한다(`프로젝트 폴더 > Quest > Android_ASTC > *.apk`).<br>
install 후, `QUEST > App Library`모든 앱에서 `Unknown Sources`를 찾으면 당신의 언리얼 프로젝트를 컴퓨터 연결 없이 플레이 할 수 있다.

# 후기
이 글은 2시간만에 썼지만 삽질하면서 VR연결하는데 엄청 걸렸습니다... GG<br>
창업동아리에서 게임 만드느라 산 VR인데 YOUTUBE VR 360 영상이 맛도리여서 애용하고 있습니다.<br>
그냥 VR << 얘 끼고 있기만 해도 재미있음. 단점은 3D멀미 유발, 오래끼면 머리가 무겁다 정도?
### 읽어주셔서 감사합니다!
![Amelia Watson Winking](https://cdn3.emoji.gg/emojis/7050_Amelia_Watson_Winking.gif)