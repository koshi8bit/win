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
