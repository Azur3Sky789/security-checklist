# 🚀 Ultimate VPS Setup & Optimization Checklist (For MT4/MT5)

คู่มือการรีดประสิทธิภาพ VPS สำหรับรันบอทเทรด (EA) ขั้นสูงสุด ลด Latency ประหยัดทรัพยากร และเพิ่มความปลอดภัยระดับ Dark Server

---

## 🛡️ Phase 1: Network & Security (Hardening & Stealth Mode)
*เน้นการซ่อนตัวตน ป้องกันการถูก Port Scan จากภายนอก และลด Latency ของเครือข่าย*

- [ ] **ติดตั้ง Tailscale:** โหลดและติดตั้ง Tailscale เพื่อสร้างท่อ VPN ส่วนตัว (Mesh Network)
    1. โหลดและติดตั้ง Tailscale ล็อกอินให้เรียบร้อย
    2. คลิกขวาที่ไอคอน Tailscale ตรงมุมขวาล่างของจอ (Taskbar)
    3. ไปที่ Preferences > ติ๊กถูกที่ Run unattended (เพื่อให้ Tailscale เปิดทันทีตอนบูตโดยไม่ต้องรอ Login)
    4. เข้าเว็บ admin.tailscale.com ไปที่แท็บ Machines > หาชื่อ VPS > กดจุด 3 จุดขวาสุด > เลือก Disable key expiry
- [ ] **ปิดพอร์ตสาธารณะเพื่อป้องกัน Port Scan (RDP Firewall):**
    1. เปิด `Windows Defender Firewall with Advanced Security`
    2. ไปที่ **Inbound Rules** > หา Rule ชื่อ `Remote Desktop - User Mode (TCP-In)`
    3. ดับเบิลคลิกไปที่แท็บ **Scope** > ตรง Remote IP address เลือก *These IP addresses*
    4. กด **Add** แล้วใส่ `IP Tailscale ของเครื่องคอมพิวเตอร์เรา` (เช่น `100.x.x.x`)
       *(วิธีหา IP: คลิกที่ไอคอน Tailscale ใน Taskbar ของเครื่องเรา แล้ว Copy IP มาใส่)*
    5. กด Apply (การทำแบบนี้จะทำให้ VPS ไม่ตอบสนองต่อการสแกนพอร์ตจาก Public IP ใดๆ ทั้งสิ้น)
- [ ] **ปลดล็อกคอขวด Network (TCP Tuning):** เปิด **PowerShell (Run as Administrator)** แล้วรัน 3 คำสั่งนี้ทีละบรรทัด:

    ```powershell
    # 1. ขยายท่อรับข้อมูลให้กว้างที่สุด (รับ Tick ทันทีไม่มีอั้น)
    netsh int tcp set global autotuninglevel=experimental

    # 2. กระจายโหลดให้ CPU ช่วยกันทำงาน
    netsh int tcp set global rss=enabled

    # 3. ลดขนาดข้อมูลขยะ (ปิดการส่ง Timestamps)
    netsh int tcp set global timestamps=disabled
    ```

---

## ⚙️ Phase 2: Windows OS Optimization (รีดพลัง CPU/RAM)
*รันสคริปต์นี้เพื่อปิดเซอร์วิสที่ไม่จำเป็นสำหรับการเทรดทั้งหมด*

- [ ] **รัน Optimization Script:** เปิด **PowerShell (Run as Administrator)** ก๊อปปี้โค้ดด้านล่างนี้ไปวางแล้วกด Enter:

    ```powershell
    # 1. ปิดบริการที่ไม่จำเป็นสำหรับ VPS เทรด (คืน RAM/CPU)
    Write-Host "Disabling unnecessary services..." -ForegroundColor Cyan
    $Services = @("SysMain", "TabletInputService", "WalletService", "PrintSpooler", "MapsBroker", "SensorService")
    foreach ($Service in $Services) {
        if (Get-Service $Service -ErrorAction SilentlyContinue) {
            # สั่งหยุดและซ่อน Error ถ้าเซอร์วิสดื้อ
            Stop-Service $Service -Force -ErrorAction SilentlyContinue
            # สั่ง Disable ไม่ให้เปิดเองตอนบูตเครื่อง
            Set-Service $Service -StartupType Disabled -ErrorAction SilentlyContinue
        }
    }

    # 2. ปิด Windows Update (ป้องกันเครื่องรีบูตเองกลางดึกตอนบอททำงาน)
    Write-Host "Disabling Windows Update to prevent auto-reboot..." -ForegroundColor Cyan
    Stop-Service wuauserv -Force -ErrorAction SilentlyContinue
    Set-Service wuauserv -StartupType Disabled -ErrorAction SilentlyContinue

    # 3. ปรับ Power Plan เป็น High Performance (ให้ CPU วิ่งเต็มสปีด)
    Write-Host "Setting Power Plan to High Performance..." -ForegroundColor Cyan
    powercfg /setactive 8c5e7fda-e8bf-4a96-9a85-a6e23a8c635c *>$null

    # 4. ปิด Privacy Features และ Telemetry (ลดการส่งข้อมูลกลับ Microsoft)
    Write-Host "Disabling Telemetry and Data Collection..." -ForegroundColor Cyan
    Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\DataCollection" -Name "AllowTelemetry" -Value 0 -ErrorAction SilentlyContinue
    Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\DataCollection" -Name "AllowTelemetry" -Value 0 -ErrorAction SilentlyContinue

    # 5. ปิด Windows Defender Real-time Monitoring (ลด CPU Usage ขั้นสุด)
    Write-Host "Optimizing Defender for low CPU overhead..." -ForegroundColor Cyan
    # เช็คก่อนว่าในเครื่องมี Defender ไหม ถ้าไม่มีให้ข้ามไปเลยจะได้ไม่ Error
    if (Get-Command Set-MpPreference -ErrorAction SilentlyContinue) {
        Set-MpPreference -DisableRealtimeMonitoring $true -ErrorAction SilentlyContinue
    } else {
        Write-Host " -> Defender module not found (Already removed by VPS provider). Skipping..." -ForegroundColor DarkGray
    }

    # 6. ตั้งค่า RDP ให้ปลอดภัยขึ้น (บังคับใช้ NLA)
    Write-Host "Hardening RDP settings..." -ForegroundColor Cyan
    Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name "UserAuthentication" -Value 1 -ErrorAction SilentlyContinue

    Write-Host "Hardening Complete! Please restart your VPS." -ForegroundColor Green
    ```

- [ ] **Restart VPS 1 รอบ:** เพื่อให้การตั้งค่า OS และ Network ทำงานสมบูรณ์

---

## 📈 Phase 3: MT4/MT5 Tuning (ให้โปรแกรมเบาที่สุด)
*ตั้งค่าเพื่อลดภาระการประมวลผลของตัวโปรแกรม MetaTrader*

- [ ] **จำกัดจำนวนแท่งเทียน:** ไปที่ `Tools` > `Options` > `Charts` ปรับค่า `Max bars in history` และ `Max bars in chart` ให้เหลือแค่ **2000 - 5000**
- [ ] **ปิดหน้าต่างขยะ:** ปิด `Market Watch`, `Navigator`, และ `Toolbox` ถ้าไม่ได้เฝ้าจอ (ลดภาระการเรนเดอร์ UI)
- [ ] **ปิดเสียงแจ้งเตือน:** ไปที่ `Tools` > `Options` > `Events` เอาติ๊กถูกที่ `Enable` ออกให้หมด

---

## 🔄 Phase 4: Stability & Automation (รัน 24/7 ไม่ต้องเฝ้า)
*เตรียมพร้อมรับมือไฟตก หรือ VPS รีบูตตัวเอง*

- [ ] **ตั้งค่า Auto-Login Windows:**
    1. กด `Win + R` พิมพ์ `netplwiz` แล้ว Enter
    2. เอาติ๊กถูกตรง *"Users must enter a user name and password..."* ออก
    3. ใส่รหัสผ่าน Administrator ยืนยัน (ระวัง: ทำขั้นตอนนี้หลังจากลง Tailscale และบล็อกพอร์ตแล้วเท่านั้น)
- [ ] **ตั้งโปรแกรมให้เปิดเอง (Startup):**
    1. กด `Win + R` พิมพ์ `shell:startup` แล้ว Enter
    2. สร้าง Shortcut ของ MT4/MT5 มาวางไว้ในโฟลเดอร์นี้
- [ ] ตั้งค่า Auto Clear RAM (สคริปต์คืนพื้นที่หน่วยความจำอัตโนมัติ): เปิด PowerShell (Run as Administrator) แล้วรันคำสั่งด้านล่างนี้ (คำสั่งนี้จะสร้างไฟล์ C:\ClearRAM.ps1 และตั้ง Task ให้รันแบบซ่อนหน้าต่างทุกๆ 1 ชั่วโมงโดยอัตโนมัติด้วยสิทธิ์ SYSTEM):

    ```powershell
    # 1. สร้างไฟล์สคริปต์ ClearRAM.ps1 ที่ไดรฟ์ C:\
    $ScriptContent = @'
    # สคริปต์คืนพื้นที่ RAM (Clear Working Set) ให้กับ VPS
    Get-Process | Where-Object {$_.Name -notmatch "System|Idle|csrss|smss"} | ForEach-Object {
        try {
            [System.GC]::Collect()
            [System.GC]::WaitForPendingFinalizers()
            $_.MaxWorkingSet = [IntPtr]((Get-Process -Id $_.Id).MaxWorkingSet)
        } catch {}
    }
    '@
    Set-Content -Path "C:\ClearRAM.ps1" -Value $ScriptContent -Encoding UTF8
    
    # 2. สร้าง Scheduled Task ให้รันทุกๆ 1 ชั่วโมง (ซ่อนหน้าต่าง & สิทธิ์สูงสุด)
    $Action = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-ExecutionPolicy Bypass -WindowStyle Hidden -File C:\ClearRAM.ps1"
    $Trigger = New-ScheduledTaskTrigger -Once -At (Get-Date).AddMinutes(1) -RepetitionInterval (New-TimeSpan -Hours 1)
    $Principal = New-ScheduledTaskPrincipal -UserId "SYSTEM" -LogonType ServiceAccount -RunLevel Highest
    Register-ScheduledTask -TaskName "AutoClearRAM" -Action $Action -Trigger $Trigger -Principal $Principal -Force
    
    Write-Host "Auto Clear RAM Task Scheduled Successfully! It will run every 1 hour." -ForegroundColor Green
    ```

---

## 🧹 Phase 5: Verification & Maintenance (ตรวจสอบความพร้อม)
*เช็คความเรียบร้อยก่อนปล่อยบอทรันจริง*

- [ ] **ตรวจสอบความเร็วเน็ตเวิร์ค (Latency):**
    1. คลิกขวาที่แถบสัญญาณมุมขวาล่างของ MT5 > กด `Rescan servers`
    2. ตรวจสอบว่า Access Point เชื่อมต่อไปยัง Server ที่ปิงต่ำที่สุด (ควร < 5ms)
- [ ] **เช็ค IP ปลายทาง:** เปิด CMD รันคำสั่ง `netstat -ano | findstr ":443"` เพื่อดูว่า MT5 คุยกับ IP โซนที่ต้องการจริงๆ หรือไม่
- [ ] **เคลียร์ขยะสม่ำเสมอ:** ลบไฟล์ในโฟลเดอร์ `MQL5/Logs` และ `Logs` เดือนละครั้ง เพื่อไม่ให้ไฟล์ Log ใหญ่เกินจนหน่วง Disk I/O
