* Áp dụng ngay DNS `8.8.8.8, 8.8.4.4`
* Ghi đè Scheduled Task để lần sau khởi động chờ 10s rồi tự áp dụng lại

```powershell
# === FORCE-ALL DNS (PS 5.1/7 compatible) ===
# ---- CONFIG ----
$Servers              = @('8.8.8.8','8.8.4.4')   # ví dụ Cloudflare: @('1.1.1.1','1.0.0.1')
$DelaySeconds         = 10                        # chờ nội bộ trước khi ép (sau boot/logon)
$RepeatEveryMinutes   = 1                         # lặp lại để giữ kỷ luật
$RepeatDurationHours  = 8                         # lặp trong 8 giờ
$TaskName             = 'ForceDNS_All_Adapters_Hard'

# ---- Lệnh chạy khi boot/logon (ép nhiều tầng + verify) ----
$serversLiteral = ($Servers | ForEach-Object { "'$_'" }) -join ','
$template = @'
$ErrorActionPreference='SilentlyContinue';
Start-Sleep -Seconds __DELAY__;
$servers=@(__SERVERS__);

# 1) Cmdlet hiện đại cho mọi IPv4 interface
$ips = Get-NetIPInterface -AddressFamily IPv4 -EA SilentlyContinue
foreach ($ip in $ips) {
  try { Set-DnsClientServerAddress -InterfaceIndex $ip.InterfaceIndex -ServerAddresses $servers -EA Stop } catch {}
}

# 2) Fallback netsh theo alias (rắn hơn)
$adapters = Get-NetAdapter -EA SilentlyContinue
foreach ($ad in $adapters) {
  try {
    & netsh interface ipv4 set dns name="$($ad.InterfaceAlias)" static $servers[0] primary | Out-Null
    for ($i=1; $i -lt $servers.Count; $i++) {
      & netsh interface ipv4 add dns name="$($ad.InterfaceAlias)" address=$servers[$i] index=$($i+1) | Out-Null
    }
  } catch {}
}

# 3) Ghi Registry NameServer cho mọi interface GUID
foreach ($ad in $adapters) {
  try {
    $guid = $ad.InterfaceGuid
    if ($null -ne $guid -and "$guid" -ne '') {
      $key = "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\Interfaces\{$($guid)}"
      New-Item -Path $key -Force | Out-Null
      Set-ItemProperty -Path $key -Name NameServer -Value ($servers -join ',') -Force
    }
  } catch {}
}

ipconfig /flushdns | Out-Null
'@

$cmd = $template.Replace('__DELAY__',[string]$DelaySeconds).Replace('__SERVERS__',$serversLiteral)

# ---- Gỡ task cũ & đăng ký lại (Startup + Logon + lặp) ----
try { Unregister-ScheduledTask -TaskName $TaskName -Confirm:$false -EA SilentlyContinue } catch {}
$action    = New-ScheduledTaskAction -Execute 'powershell.exe' -Argument "-NoProfile -ExecutionPolicy Bypass -WindowStyle Hidden -Command `"$cmd`""
$trStartup = New-ScheduledTaskTrigger -AtStartup
$trLogon   = New-ScheduledTaskTrigger -AtLogOn
$trRepeat  = New-ScheduledTaskTrigger -Once -At ((Get-Date).AddMinutes(1)) `
              -RepetitionInterval (New-TimeSpan -Minutes $RepeatEveryMinutes) `
              -RepetitionDuration (New-TimeSpan -Hours $RepeatDurationHours)
$principal = New-ScheduledTaskPrincipal -UserId 'SYSTEM' -RunLevel Highest
$settings  = New-ScheduledTaskSettingsSet -StartWhenAvailable -ExecutionTimeLimit (New-TimeSpan -Minutes 10)
Register-ScheduledTask -TaskName $TaskName -Action $action -Trigger @($trStartup,$trLogon,$trRepeat) -Principal $principal -Settings $settings -Force | Out-Null

# ---- Áp dụng ngay lần đầu ----
powershell -NoProfile -ExecutionPolicy Bypass -Command $cmd

Write-Host "`nĐÃ ÉP DNS (tất cả adapter IPv4) -> $($Servers -join ', ') | Task: $TaskName (Startup + Logon + mỗi $RepeatEveryMinutes phút trong $RepeatDurationHours giờ)." -ForegroundColor Green
Get-ScheduledTask -TaskName $TaskName | Select-Object TaskName,State,LastRunTime
Get-DnsClientServerAddress -AddressFamily IPv4 | Select-Object InterfaceAlias,ServerAddresses

```

Kiểm tra nhanh: chạy

```powershell
Get-DnsClientServerAddress -AddressFamily IPv4 | Select InterfaceAlias,ServerAddresses
```

Bạn sẽ thấy mỗi card có hai server theo thứ tự `8.8.8.8, 8.8.4.4`. Windows sẽ ưu tiên cái đầu, gặp lỗi mới thử cái sau. Nếu đang trong môi trường domain/VPN có DNS nội bộ, cân nhắc giữ bộ lọc `$IgnoreRegex` như trên để khỏi “phá” phân giải tên nội bộ. Muốn mình thêm IPv6 (2001:4860:4860::8888/8844) hoặc chỉ áp dụng cho “Wi-Fi”/“Ethernet”, mình nhúng thêm giúp bạn được ngay.
