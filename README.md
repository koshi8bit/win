# win

## Питание
Для выключения при закрытии ноута запустить
```
powercfg.cpl
```
Сбоку выбрать `Действия при закрытии крышки` и там настроить сон при закрытии крышки

## Время
Чтобы дата внизу (в панели задач) отображалась в формате `yyyy-MM-dd`, сделай так:

1. Запусти скрипт PowerShell
   ```PowerShell
   try { Set-ItemProperty -Path 'HKCU:\Control Panel\International' -Name sShortDate -Value 'ddd yyyy-MM-dd' -ErrorAction Stop; Set-ItemProperty -Path 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced' -Name ShowSecondsInSystemClock -Value 1 -Type DWord -ErrorAction Stop; Stop-Process -Name explorer -Force -ErrorAction Stop } catch { Write-Error "Ошибка: $_" }
   ```
   Если всё ок - далее можно не идти по пунктам
   
1. Нажми **Win + R** → введи:
   ```
   control
   ```
1. Перейди в **Часы и регион** 
1. Перейди в **Региональные стандарты**
1. Нажми **Дополнительные параметры**
1. Вкладка **Дата**
1. В поле **Краткая дата** задай:

   ```
   ddd yyyy-MM-dd
   ```
1. Применить → ОК

   После этого дата внизу справа (в трее) будет, например:
   ```
   Ср 2026-03-18
   ```
1. Для вывода секунд нужно запустить в PowerShell с админскими правами эту команду
   ```powershell
   Set-ItemProperty -Path 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced' -Name ShowSecondsInSystemClock -Value 1 -Type DWord
   Stop-Process -Name explorer -Force
   ```

## Смена языка
Для смены языка только по сочетанию клавиш `win`+`space` нужно выполнить команду в powershell
```powershell
$Path='HKCU:\Keyboard Layout\Toggle'; if(-not(Test-Path $Path)){New-Item -Path $Path -Force|Out-Null}; 'Language Hotkey','Hotkey','Layout Hotkey'|ForEach-Object{Set-ItemProperty -Path $Path -Name $_ -Value '3' -Type String}
```
## NTP
Создаём `setup-ntp.bat`
```cmd
@echo off

:: Проверка наличия параметра с IP адресом
if "%1"=="" (
    echo.
    echo ОШИБКА: Не указан IP адрес NTP сервера!
    echo Использование: %~nx0 ^<IP_адрес_сервера^>
    echo Пример: %~nx0 10.104.5.22
    echo.
    pause
    exit /b 1
)

:: Сохраняем IP адрес из первого параметра
set NTP_SERVER=%1

:: Проверка формата IP адреса (простая проверка)
echo %NTP_SERVER% | findstr /r "^[0-9][0-9]*\.[0-9][0-9]*\.[0-9][0-9]*\.[0-9][0-9]*$" >nul
if errorlevel 1 (
    echo.
    echo ОШИБКА: Некорректный формат IP адреса: %NTP_SERVER%
    echo Пожалуйста, укажите IP адрес в формате xxx.xxx.xxx.xxx
    echo.
    pause
    exit /b 1
)

echo Настройка синхронизации времени с сервером: %NTP_SERVER%
echo.

sc config w32time start=auto || goto error
net start w32time
w32tm /config /manualpeerlist:"%NTP_SERVER%" /syncfromflags:manual /update || goto error
net stop w32time || goto error
net start w32time || goto error
w32tm /resync || goto error

echo.
echo Сервер синхронизации времени (NTP) настроен успешно!
echo Используется сервер: %NTP_SERVER%
pause
exit /b 0

:error
echo.
echo ОШИБКА: выполнение команды завершилось неуспешно.
echo Код ошибки: %ERRORLEVEL%
echo Работа программы остановлена.
echo.
pause
exit /b %ERRORLEVEL%
```

```
setup-ntp.bat 10.11.5.41
```

## Samba

### Настройка

Для разрешения доступа без пароля нужно создать файл с кодировкой 866. Запускать из под админа.

`samba-configure-without-pass.bat`

```bat
@echo off
net session >nul 2>&1
if errorlevel 1 (
    echo Ошибка: запустите скрипт от имени администратора.
    goto end
)

for /f "tokens=2 delims==" %%A in ('wmic useraccount where "LocalAccount=TRUE and SID like '%%-501'" get Name /value 2^>nul ^| find "="') do (
    net user "%%A" /active:yes
)

reg add "HKLM\SYSTEM\CurrentControlSet\Control\Lsa" /v ForceGuest /t REG_DWORD /d 1 /f

netsh advfirewall firewall add rule name="Guest SMB TCP 445" dir=in action=allow protocol=TCP localport=445 profile=private

echo.
echo Настройка гостевого SMB завершена.

:end
echo.
pause
```

### Добавление папки без авторизации

Для добавления папки доступной для общего пользования без пароля нужно создать файл с кодировкой 866. Запускать из под админа.

`samba-add-to-everyone.bat "D:\home\koshi8bit\temp-lan"`

```bat
@echo off

set "FOLDER=%~1"

if not defined FOLDER (
    echo Введите или вставьте путь к папке:
    set /p "FOLDER=> "
)

set "FOLDER=%FOLDER:"=%"

if not defined FOLDER (
    echo Ошибка: путь не указан.
    goto end
)

for %%A in ("%FOLDER%") do set "FOLDER=%%~fA"

if not exist "%FOLDER%" (
    echo Ошибка: папка не существует: "%FOLDER%"
    goto end
)

for %%A in ("%FOLDER%") do set "SHARE_NAME=%%~nxA"

net share "%SHARE_NAME%=%FOLDER%" /GRANT:Все,FULL

if errorlevel 1 (
    echo.
    echo Ошибка создания SMB-ресурса.
) else (
    echo.
    echo SMB-ресурс успешно создан!
    echo \\%COMPUTERNAME%\%SHARE_NAME%
)

:end
echo.
pause
```


## RDP
### Генерация сертификата
PowerShell (admin)
```powershell
sertlm.msc

$cert = New-SelfSignedCertificate `
-DnsName "192.168.1.10" `
-CertStoreLocation "Cert:\LocalMachine\My" `
-KeyUsage DigitalSignature,KeyEncipherment `
-Type SSLServerAuthentication

# Для подключению по имени и IP заменить строчку ниже в команде выше
# -DnsName "ENTER_IP_HERE","ENTER_NAME_HERE"

Set-ItemProperty `
-Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" `
-Name SSLCertificateSHA1Hash `
-Value $cert.Thumbprint

Restart-Service TermService -Force
```

Для экспорта выполить команду и запомнить `Thumbprint`
```powershell
Get-ChildItem Cert:\LocalMachine\My | 
Select Subject, Thumbprint
```

```powershell
Export-Certificate `
-Cert "Cert:\LocalMachine\My\<Thumbprint>" `
-FilePath "C:\rdp.cer"
```

Импорт
PowerShell (admin)
```powershell
Import-Certificate `
-FilePath "C:\rdp.cer" `
-CertStoreLocation "Cert:\LocalMachine\Root"
```

PowerShell (admin)
```powershell
New-SelfSignedCertificate `
-DnsName $env:COMPUTERNAME `
-CertStoreLocation "Cert:\LocalMachine\My"

$cert = Get-ChildItem Cert:\LocalMachine\My | Select-Object -First 1

Set-ItemProperty `
-Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" `
-Name SSLCertificateSHA1Hash `
-Value $cert.Thumbprint

Restart-Service TermService -Force

Export-Certificate `
-Cert "Cert:\LocalMachine\My\$($cert.Thumbprint)" `
-FilePath "C:\rdp.cer"
```

### 
