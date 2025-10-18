Chuẩn bài “nhẹ nhàng tình cảm”: chỉ can thiệp **trong 1 phút đầu sau khi khởi động**, kiểm tra mỗi **10 giây**, **chỉ sửa khi thấy lệch**, xong là **thoát hẳn** (không lặp vĩnh viễn, không ghi Registry). Có thêm một cú trễ 10 giây cho mạng kịp lên.

Dán nguyên khối vào **PowerShell (Run as Administrator)**:

```powershell
# === SOFT-GUARD DNS: chỉ giữ 1 phút sau khi boot, sửa khi lệch, rồi thoát ===
# ---- CONFIG ----
$Servers              = @('8.8.8.8','8.8.4.4')   # đổi tuỳ ý (Cloudflare: @('1.1.1.1','1.0.0.1'))
$InitialDelaySeconds  = 10                        # chờ một chút cho mạng lên
$WindowSeconds        = 60                        # bảo vệ trong 1 phút đầu sau khi boot
$IntervalSeconds      = 10                        # kiểm tra mỗi 10s trong cửa sổ 1 phút
$TaskName             = 'ForceDNS_StartupWindow'

# ---- Lệnh sẽ chạy lúc boot (nhẹ, chỉ sửa khi lệch) ----
$serversLiteral = ($Servers | ForEach-Object { "'$_'" }) -join ','
$template = @'
$ErrorActionPreference='SilentlyContinue';
Start-Sleep -Seconds __INIT__;
$desired=@(__SERVERS__);
$deadline = (Get-Date).AddSeconds(__WIN__);
$interval = __INT__;
$changed  = $false

do {
  # Chỉ xử lý adapter đang Up (Wi-Fi/Ethernet/VPN/VM ... đang hoạt động)
  $up = Get-NetAdapter -EA SilentlyContinue | Where-Object { $_.Status -eq 'Up' }
  if ($up) {
    $clients = Get-DnsClient -AddressFamily IPv4 -EA SilentlyContinue | Where-Object {
      $up.InterfaceAlias -contains $_.InterfaceAlias
    }

    foreach ($c in $clients) {
      try {
        $cur = (Get-DnsClientServerAddress -InterfaceAlias $c.InterfaceAlias -AddressFamily IPv4 -EA SilentlyContinue).ServerAddresses
        if (-not $cur -or ($cur -join ',') -ne ($desired -join ',')) {
          # Thử cmdlet hiện đại trước
          try { Set-DnsClientServerAddress -InterfaceIndex $c.InterfaceIndex -ServerAddresses $desired -EA Stop }
          catch {
            # Nếu vẫn lệch, fallback netsh (chỉ cho alias đó)
            & netsh interface ipv4 set dns name="$($c.InterfaceAlias)" static $desired[0] primary | Out-Null
            for ($i=1; $i -lt $desired.Count; $i++) {
              & netsh interface ipv4 add dns name="$($c.InterfaceAlias)" address=$desired[$i] index=$($i+1) | Out-Null
            }
          }
          $changed = $true
        }
      } catch {}
    }
    if ($changed) { ipconfig /flushdns | Out-Null; $changed = $false }
  }

  if ((Get-Date) -ge $deadline) { break }
  Start-Sleep -Seconds $interval
} while ($true)
'@

$cmd = $template.
  Replace('__INIT__',[string]$InitialDelaySeconds).
  Replace('__WIN__',[string]$WindowSeconds).
  Replace('__INT__',[string]$IntervalSeconds).
  Replace('__SERVERS__',$serversLiteral)

# ---- Gỡ task cũ (nếu có) & đăng ký lại (AtStartup duy nhất) ----
try { Unregister-ScheduledTask -TaskName $TaskName -Confirm:$false -EA SilentlyContinue } catch {}
$action    = New-ScheduledTaskAction -Execute 'powershell.exe' -Argument "-NoProfile -ExecutionPolicy Bypass -WindowStyle Hidden -Command `"$cmd`""
$trigger   = New-ScheduledTaskTrigger -AtStartup               # không dùng -Delay để tương thích 5.1; delay nằm trong $cmd
$principal = New-ScheduledTaskPrincipal -UserId 'SYSTEM' -RunLevel Highest
$settings  = New-ScheduledTaskSettingsSet -StartWhenAvailable -ExecutionTimeLimit (New-TimeSpan -Minutes 5)
Register-ScheduledTask -TaskName $TaskName -Action $action -Trigger $trigger -Principal $principal -Settings $settings -Force | Out-Null

# ---- Áp dụng ngay lần đầu (khỏi chờ reboot) ----
powershell -NoProfile -ExecutionPolicy Bypass -Command $cmd

Write-Host "`nĐÃ cấu hình: DNS -> $($Servers -join ', ') | Task: $TaskName (AtStartup, bảo vệ $WindowSeconds s, kiểm tra mỗi $IntervalSeconds s)." -ForegroundColor Green
Get-ScheduledTask -TaskName $TaskName | Select-Object TaskName,State,LastRunTime
Get-DnsClientServerAddress -AddressFamily IPv4 | Select-Object InterfaceAlias,ServerAddresses
```

### Kiểm tra nhanh

```powershell
Get-DnsClientServerAddress -AddressFamily IPv4 | Select InterfaceAlias,ServerAddresses
```

### Gỡ bỏ (khi không cần nữa)

```powershell
Unregister-ScheduledTask -TaskName 'ForceDNS_StartupWindow' -Confirm:$false
```

Bạn có thể tinh chỉnh “độ dịu”: tăng `IntervalSeconds` (ít kiểm tra hơn) hoặc giảm `WindowSeconds` (ngắn hơn), nhưng với 10s/60s như trên, máy gần như không cảm nhận được overhead. Nếu sau này bạn dùng VPN nào “đổi DNS muộn hơn 1 phút”, mình có thể chuyển sang bản **event-driven** (chỉ chạy ngay khi mạng/VPN kết nối) — vẫn mềm mà chuẩn.
