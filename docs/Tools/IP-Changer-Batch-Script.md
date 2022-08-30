---
title: "Windows IP Changer Batch Script"
author: DDAONG
date: 2022-08-16
category: "Batch"
layout: post
---

# Windows IP Changer Batch Script


윈도우즈용 IP 변경 Batch 스크립트 입니다.  

  > 최초 실행시 인터페이스 잡아주는 과정이 필요합니다. 
> 최초 실행 및 인터페이스명 설정 완료 시  ipch.bat 파일이 존재하는 디렉토리에 ipch_cfg.txt 파일이 자동으로 작성됩니다.
  {: .block-warning}
  
- CUI 메뉴
	- 스크립트 실행 시 CUI 메뉴로 진입합니다.

- 커맨드로 사용 가능
	- Cmd 모드로 사용하려면 아래처럼 CIDR 형식, GW 순서로 작성해주시면 됩니다.  
	- 예시 > ipch.bat 192.168.10.10/24 192.168.10.1  

> IP변경에는 관리자 권한 실행이 필요합니다. 
{: .block-warning}

> 아래 스크립트를 ipch.bat 파일로 저장하여, 바로가기 생성 후 
> 바로가기에서 관리자 권한으로 실행하기 옵션 Enable 해주시면 편리합니다.
{: .block-tip}


```bat
@echo off
setlocal enabledelayedexpansion

:: HEAD
title IPCH Script
mode con cols=90 lines=40
color F0

:: HEAD SET IPcmd Vars
set "dir=%~dp0"
set "cidr=%1"
set "gw=%2"

:: -- ENTRY FORKS --

:: ipch_cfg.txt 파일이 있으면 변수 로드
IF exist %dir%ipch_cfg.txt FOR /f "delims== tokens=1,2" %%G in (%dir%ipch_cfg.txt) do set %%G=%%H
:: ipch_cfg.txt 파일이 있어도 Target NIC 변수값이 없으면 Config 실행
IF "!tnic!X"=="X" GOTO _config
:: 파라미터가 있으면 IPcmd 실행
IF NOT "%*"=="" GOTO _IPcmd
:: ipch_cfg.txt 파일이 없으면 config 실행
IF NOT exist %dir%ipch_cfg.txt GOTO _config
GOTO _Menu 

:_Menu
cls
echo =========================================
echo =                                       =
echo =          IPCH script                  =
echo =                                       =
echo =========================================
echo = Current Directory is %dir%
echo = Target NIC is %tnicindex% %tnic%
echo =========================================
echo   1. Ethernet Static IP
echo   2. Ethernet DHCP IP
echo   3. Ethernet Secondary IP 추가
echo   4. Ethernet Secondary IP 삭제
echo   5. 내 IP 확인
echo =========================================
echo   c. IPCH 설정
echo   h. Command Mode 안내
echo   q. Exit
echo =========================================

set /p num=Choose the task : 
if "%num%"=="1" goto _IPstatic
if "%num%"=="2" goto _IPdhcp
if "%num%"=="3" goto _IPsecond
if "%num%"=="4" goto _IPseconddelete
if "%num%"=="5" goto _my_eth
if "%num%"=="q" goto _exit
if "%num%"=="c" goto _config
if "%num%"=="h" goto _help
goto _Menu

:_config
cls
echo ***** Interface List *****
netsh interface ipv4 show interface | findstr /V disconnected
echo **************************
set /p tnicindex=Choose a Target Network Interface: 

FOR /F "usebackq skip=3 tokens=1,5* delims= " %%G in (`netsh interface ipv4 show interface ^| findstr /V disconnected`) do ( 
	IF %%G==%tnicindex% set tnic1=%%H
	IF %%G==%tnicindex% set tnic2=%%I
	)
	IF "%tnic2%x"=="x" set tnic="%tnic1%"
	IF NOT "%tnic2%x"=="x" set tnic="%tnic1% %tnic2%"
	
echo Target NIC is %tnic%

(
    echo tnic=%tnic%
    echo tnicindex=%tnicindex%
    echo dir=%dir%
)>%dir%ipch_cfg.txt
goto _Success_menu


:_IPdhcp
netsh -c int ip set address name=%tnic% source=dhcp
goto _Success_menu

:_my_eth
netsh -c int ip show addresses %tnic%
pause
goto _Menu

:_IPstatic
echo 고정 IP를 설정합니다.
set /p addr=IP를 입력하세요 : 
set /p netmask=Network Mask를 입력하세요 : 
set gw="0.0.0.0"
set /p gw=게이트웨이를 입력하세요[기본값 0.0.0.0] : 
netsh -c int ip set address name=%tnic% source=static addr=%addr% mask=%netmask% gateway=%gw% gwmetric=0
::netsh interface ip set dns "%tnic%" static 8.8.8.8
goto _Success_menu

:_IPsecond
echo Secondary IP를 설정합니다.
set /p addr=IP를 입력하세요 : 
set /p netmask=Network Mask를 입력하세요 : 
set "gw=0.0.0.0"
set /p gw=게이트웨이를 입력하세요[기본값 0.0.0.0] : 

echo %tnic% "%addr%" "%netmask%" "%gw%"
pause
netsh -c interface ip add address name=%tnic% addr="%addr%" mask="%netmask%" gateway="%gw%"
::netsh interface ip set dns "%tnic%" static 8.8.8.8
goto _Success_menu


:_IPseconddelete
echo 설정된 Secondary IP를 삭제합니다.
set /p addr=삭제할 IP를 입력하세요 : 
netsh -c interface ip delete address name=%tnic% addr="%addr%"
goto _Success_menu



:_IPcmd
echo Target NIC is %tnicindex% %tnic%
echo %*
FOR /F "tokens=1,2 delims=/ " %%G in ("%cidr%") do (
    set addr="%%G"
    set cmask=%%H
)
IF "%cmask%"=="8" set mask="255.0.0.0"
IF "%cmask%"=="9" set mask="255.128.0.0"
IF "%cmask%"=="10" set mask="255.192.0.0"
IF "%cmask%"=="11" set mask="255.224.0.0"
IF "%cmask%"=="12" set mask="255.240.0.0"
IF "%cmask%"=="13" set mask="255.248.0.0"
IF "%cmask%"=="14" set mask="255.252.0.0"
IF "%cmask%"=="15" set mask="255.254.0.0"
IF "%cmask%"=="16" set mask="255.255.0.0"
IF "%cmask%"=="17" set mask="255.255.128.0"
IF "%cmask%"=="18" set mask="255.255.192.0"
IF "%cmask%"=="19" set mask="255.255.224.0"
IF "%cmask%"=="20" set mask="255.2550.240.0"
IF "%cmask%"=="21" set mask="255.255.248.0"
IF "%cmask%"=="22" set mask="255.255.252.0"
IF "%cmask%"=="23" set mask="255.255.254.0"
IF "%cmask%"=="24" set mask="255.255.255.0"
IF "%cmask%"=="25" set mask="255.255.255.128"
IF "%cmask%"=="26" set mask="255.255.255.196"
IF "%cmask%"=="27" set mask="255.255.255.224"
IF "%cmask%"=="28" set mask="255.255.255.240"
IF "%cmask%"=="29" set mask="255.255.255.248"
IF "%cmask%"=="30" set mask="255.255.255.252"
IF "%cmask%"=="31" set mask="255.255.255.254"

IF "%gw%x"=="x" set "gw=0.0.0.0"

echo %tnic% %addr% %mask% "%gw%"
pause
netsh -c int ip set address name=%tnic% source=static addr=%addr% mask=%mask% gateway="%gw%"

GOTO _Success_exit


:_help
cls
echo ===============================
echo =                             =
echo =      IPCH script            =
echo =                             =
echo ===============================
echo.
echo  - 메뉴에 들어오지 않고도 커맨드만으로 IP 변경이 가능합니다.
echo  - 아래 형식으로 실행하면 됩니다.
echo.
echo  ^>^> Usage \: echo ipch [IP Address/Netmask (CIDR)]
echo  ^>^> sample \: echo ipch 192.168.0.100/24
echo.
pause
GOTO _Menu

:_exit
echo 종료합니다.
pause
exit

:_Success_exit
cls
echo    The operation completed successfully.
pause > nul
GOTO _exit

:_Success_menu
echo    The operation completed successfully.
pause > nul
goto _Menu
```

