* Áp dụng ngay DNS `8.8.8.8, 8.8.4.4`
* Ghi đè Scheduled Task để lần sau khởi động chờ 10s rồi tự áp dụng lại

```powershell
# === ONE-SHOT (Win PowerShell 5.1 compatible): ép DNS & tự chạy lúc boot (delay nội bộ 10s) ===
# ---- CONFIG ----
$Servers      = @('8.8.8.8','8.8.4.4')   # đổi tuỳ ý, ví dụ Cloudflare: @('1.1.1.1','1.0.0.1')
$DelaySeconds = 10                        # chờ 10 giây sau khi khởi động rồi mới set DNS
$IgnoreRegex  = 'TAP|TUN|Hyper-V|VMware|VirtualBox|Npcap|Docker|WSL'  # bỏ qua NIC ảo/VPN/VM
$TaskName     = 'ForceDNS_8_8_8_8'

# ---- Build command chạy lúc boot (nhúng sẵn delay) ----
$serversLiteral = ($Servers | ForEach-Object { "'$_'" }) -join ','
$cmd = @"
`$ErrorActionPreference='SilentlyContinue';
Start-Sleep -Seconds $DelaySeconds;
`$servers=@($serversLiteral);
`$regex='$IgnoreRegex';
`$nics=Get-NetAdapter -Physical -EA SilentlyContinue;
if(-not `$nics){ `$nics=Get-NetAdapter -EA SilentlyContinue }
if('' -ne `$regex){ `$nics=`$nics | Where-Object { `$_.InterfaceDescription -notmatch `$regex } }
foreach(`$nic in `$nics){ try{ Set-DnsClientServerAddress -InterfaceIndex `$nic.IfIndex -ServerAddresses `$servers -EA Stop }catch{} }
ipconfig /flushdns | Out-Null
"@

# ---- Dọn task cũ & đăng ký lại ----
try { Unregister-ScheduledTask -TaskName $TaskName -Confirm:$false -EA SilentlyContinue } catch {}
$action    = New-ScheduledTaskAction -Execute 'powershell.exe' -Argument "-NoProfile -ExecutionPolicy Bypass -WindowStyle Hidden -Command `"$cmd`""
$trigger   = New-ScheduledTaskTrigger -AtStartup                 # không dùng -Delay để tương thích 5.1
$principal = New-ScheduledTaskPrincipal -UserId 'SYSTEM' -RunLevel Highest
$settings  = New-ScheduledTaskSettingsSet -StartWhenAvailable -ExecutionTimeLimit (New-TimeSpan -Minutes 5)
Register-ScheduledTask -TaskName $TaskName -Action $action -Trigger $trigger -Principal $principal -Settings $settings -Force | Out-Null

# ---- Áp dụng ngay lần đầu ----
powershell -NoProfile -ExecutionPolicy Bypass -Command $cmd

Write-Host "`nĐÃ ÉP DNS -> $($Servers -join ', ') và tạo task chạy sau $DelaySeconds giây kể từ lúc khởi động." -ForegroundColor Green
Get-ScheduledTask -TaskName $TaskName | Select-Object TaskName,State,LastRunTime
Get-DnsClientServerAddress -AddressFamily IPv4 | Select-Object InterfaceAlias,ServerAddresses
```

Kiểm tra nhanh: chạy

```powershell
Get-DnsClientServerAddress -AddressFamily IPv4 | Select InterfaceAlias,ServerAddresses
```

Bạn sẽ thấy mỗi card có hai server theo thứ tự `8.8.8.8, 8.8.4.4`. Windows sẽ ưu tiên cái đầu, gặp lỗi mới thử cái sau. Nếu đang trong môi trường domain/VPN có DNS nội bộ, cân nhắc giữ bộ lọc `$IgnoreRegex` như trên để khỏi “phá” phân giải tên nội bộ. Muốn mình thêm IPv6 (2001:4860:4860::8888/8844) hoặc chỉ áp dụng cho “Wi-Fi”/“Ethernet”, mình nhúng thêm giúp bạn được ngay.
