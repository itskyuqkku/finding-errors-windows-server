ด้านล่างเป็นคำสั่งสำหรับ Windows Server 2012 R2 เรียงตามลำดับที่ควรทำ ให้เปิด **PowerShell แบบ Run as Administrator** แล้วทำทีละขั้น ส่วน path อย่าง `C:\PHP` หรือ `D:\Web\...` ในตัวอย่างเป็นค่าสมมติ ขั้นที่ 0 จะช่วยหา path จริงของเครื่องคุณ

---

## ขั้นที่ 0: หาว่า PHP, php.ini และโฟลเดอร์เว็บอยู่ที่ไหน

```powershell
$appcmd = "$env:windir\system32\inetsrv\appcmd.exe"

# 1) ดูว่า IIS เรียก php-cgi.exe ตัวไหน และตั้ง PHPRC (ตำแหน่ง php.ini) ไว้หรือไม่
& $appcmd list config -section:system.webServer/fastCgi

# 2) หาโฟลเดอร์จริงของเว็บ (physicalPath)
& $appcmd list vdir

# 3) หา application pool ของเว็บนี้ (จะใช้ตอน recycle)
& $appcmd list app
& $appcmd list apppool
```

จากข้อ 1 จะได้ path ประมาณ `C:\PHP\php-cgi.exe` ให้ใช้ `php.exe` ในโฟลเดอร์เดียวกัน (สำคัญมาก เพราะเครื่องอาจมี PHP หลายเวอร์ชัน) แล้วตั้งตัวแปรไว้ใช้ต่อ:

```powershell
$php  = "C:\PHP\php.exe"     # แก้ให้ตรงกับผลข้อ 1
$site = "D:\Web\E-Administration\BackEnd\Program\Manage"   # แก้ให้ตรงกับผลข้อ 2 + path ย่อย
cd $site

& $php -v
& $php --ini                               # ดูว่าใช้ php.ini ไฟล์ไหน
& $php -i | findstr /i "error_log log_errors display_errors open_basedir"
```

> ถ้าผลจาก `appcmd` มี `PHPRC` ชี้ไปโฟลเดอร์อื่น แปลว่า IIS ใช้ php.ini คนละไฟล์กับที่ `php --ini` บอก ให้ยึดตาม PHPRC

---

## ขั้นที่ 1: ดู error log

### 1.1 ตรวจว่ามี log อยู่แล้วหรือยัง

```powershell
# ถ้า error_log มีค่า ให้เปิดดูท้ายไฟล์
Get-Content "C:\PHP\logs\php_errors.log" -Tail 50
Select-String -Path "C:\PHP\logs\php_errors.log" -Pattern "view100" | Select-Object -Last 20
```

### 1.2 ถ้ายังไม่มี log ให้เปิดใช้งาน

แก้ไฟล์ php.ini (ตัวที่ IIS ใช้) ให้มีค่าดังนี้:

```ini
log_errors = On
error_log = "C:\PHP\logs\php_errors.log"
display_errors = Off
error_reporting = E_ALL
```

จากนั้นสร้างโฟลเดอร์ ให้สิทธิ์ IIS เขียนได้ แล้ว recycle app pool:

```powershell
New-Item -ItemType Directory -Force "C:\PHP\logs"
icacls "C:\PHP\logs" /grant "IIS_IUSRS:(OI)(CI)M" /grant "IUSR:(OI)(CI)M"

& $appcmd recycle apppool /apppool.name:"ชื่อAppPoolจากขั้น0"
# หรือถ้าไม่แน่ใจ: iisreset   (เว็บทั้งเครื่องจะสะดุดประมาณ 10-30 วินาที)
```

แล้วเปิดดู log แบบ real-time ระหว่างที่ลองกดเปิด PDF ในเบราว์เซอร์:

```powershell
Get-Content "C:\PHP\logs\php_errors.log" -Wait -Tail 20
```

เปิด `https://personweb.kku.ac.th/E-Administration/BackEnd/Program/Manage/view100.php?FILE_ID=243631` แล้ว error จะขึ้นในหน้าต่าง PowerShell ทันที (กด Ctrl+C เพื่อหยุดดู)

### 1.3 ดู Event Viewer และ IIS log ประกอบ

```powershell
# php-cgi.exe crash หรือ FastCGI error
Get-EventLog -LogName Application -Newest 100 |
  Where-Object { $_.Source -match "FastCGI|PHP|IIS|Application Error|W3SVC|WAS" } |
  Format-List TimeGenerated, Source, EventID, Message

# IIS access log (W3SVC1 คือ ID ของ site ซึ่งดูได้จาก appcmd list site)
Get-ChildItem "C:\inetpub\logs\LogFiles\W3SVC1" | Sort-Object LastWriteTime -Descending | Select-Object -First 1 |
  ForEach-Object { Select-String -Path $_.FullName -Pattern "view100" | Select-Object -Last 10 }
```

---

## ขั้นที่ 2: เช็ก syntax และรันสคริปต์ตรงจาก command line

```powershell
cd $site
& $php -l view100.php
```

ถ้าขึ้น `No syntax errors detected` ให้ลองรันจริงแบบเปิด error เต็มที่ **ขั้นนี้มักเจอสาเหตุทันที** เพราะจากการทดสอบ สคริปต์ตายก่อนอ่านพารามิเตอร์:

```powershell
& $php -d display_errors=1 -d error_reporting=-1 -f view100.php 2>&1 | Select-Object -First 40
```

สิ่งที่อาจเห็น เช่น:
- `Warning: require(xxx): failed to open stream` → ไปขั้นที่ 3
- `Fatal error: Call to undefined function mssql_connect()` / `sqlsrv_connect()` / `odbc_...` → extension หายหรือไม่ถูกโหลด ตรวจด้วย `& $php -m` และบรรทัด `extension=` ใน php.ini
- `Fatal error: Allowed memory size...` หรือ `open_basedir restriction` → ปรับค่าใน php.ini

> ข้อควรรู้: CLI กับ IIS อาจรันด้วยสิทธิ์ต่างกัน CLI รันด้วยสิทธิ์ Administrator ส่วน IIS รันด้วย identity ของ app pool ถ้า CLI ไม่เจอ error แต่ผ่าน IIS ยังว่างอยู่ ให้สงสัยเรื่อง **สิทธิ์ไฟล์** (ดูขั้นที่ 3.3)

---

## ขั้นที่ 3: ไล่ include/require

### 3.1 ดูว่า view100.php เรียกไฟล์อะไรบ้าง

```powershell
Select-String -Path view100.php -Pattern "include|require" | Format-Table LineNumber, Line -AutoSize -Wrap
```

### 3.2 เช็กว่าไฟล์เหล่านั้นมีอยู่จริงไหม (path แบบ relative จะคิดจากโฟลเดอร์ Manage)

```powershell
Select-String -Path view100.php -Pattern "(include|require)(_once)?\s*\(?\s*['""]([^'""]+)['""]" -AllMatches |
  ForEach-Object { $_.Matches } | ForEach-Object {
    $p = $_.Groups[3].Value
    [PSCustomObject]@{ File = $p; Exists = (Test-Path (Join-Path $site $p)) }
  } | Format-Table -AutoSize
```

ถ้ามีไฟล์ไหน `Exists = False` คือเจอต้นเหตุแล้ว ส่วน include ที่ใช้ตัวแปร เช่น `include($path."x.php")` ต้องเปิดไฟล์อ่านเองว่าตัวแปรนั้นมีค่าอะไร

จากนั้นรัน `php -l` กับไฟล์ที่ถูก include ทุกตัวด้วย:

```powershell
& $php -l "..\..\System\ชื่อไฟล์.php"
```

### 3.3 เช็กสิทธิ์ไฟล์ (สำหรับ identity ของ IIS)

```powershell
icacls view100.php
icacls "..\..\System"     # และโฟลเดอร์ของไฟล์ที่ถูก include
```

ต้องเห็น `IUSR` หรือ `IIS_IUSRS` (หรือ `IIS AppPool\ชื่อpool`) มีสิทธิ์ `(R)` หรือ `(RX)`

ถ้า PDF เก็บอยู่บน network share (เช่น `\\fileserver\docs`) ให้เช็กว่า app pool identity เข้าถึง share นั้นได้ ซึ่งมักเป็นต้นเหตุหลังเปลี่ยนรหัสผ่าน service account หรือย้าย file server

---

## ขั้นที่ 4: เทียบกับ view.php และหาไฟล์ที่ถูกแก้ล่าสุด

```powershell
# วันที่แก้ไขของไฟล์หลัก
Get-Item view.php, view100.php, ViewDocFile100.php | Format-Table Name, Length, LastWriteTime

# เทียบเนื้อหาสองไฟล์ทีละบรรทัด
fc.exe /n view.php view100.php

# ไฟล์ทั้งเว็บที่ถูกแก้ใน 60 วันที่ผ่านมา (ปรับ path ไปที่ root ของเว็บ)
Get-ChildItem "D:\Web\E-Administration" -Recurse -File -Include *.php,*.ini,*.inc |
  Where-Object { $_.LastWriteTime -gt (Get-Date).AddDays(-60) } |
  Sort-Object LastWriteTime -Descending | Format-Table LastWriteTime, FullName -AutoSize

# Windows Update ล่าสุด (เผื่อมีอัปเดตก่อนปัญหาเริ่ม)
Get-HotFix | Sort-Object InstalledOn -Descending | Select-Object -First 10
```

ถ้ามีไฟล์ที่ถูกแก้ตรงกับช่วงที่เริ่มเปิด PDF ไม่ได้ ไฟล์นั้นคือผู้ต้องสงสัยอันดับแรก ถ้ามีแบ็กอัป ให้เทียบด้วย `fc.exe /n` ได้เลย

---

## ขั้นที่ 5: แสดง error บนหน้าเว็บชั่วคราว (ทำเมื่อขั้น 1–4 ยังไม่เจอ)

**ต้องแบ็กอัปก่อน:**

```powershell
Copy-Item view100.php "view100.php.bak_$(Get-Date -Format yyyyMMdd_HHmm)"
```

เปิด `view100.php` ด้วย Notepad++ (อย่าใช้ Notepad ธรรมดา เพราะอาจใส่ BOM จนสคริปต์ส่ง PDF พังเพิ่ม) แล้วใส่โค้ดต่อจาก `<?php` บรรทัดแรก:

```php
<?php
if (isset($_GET['dbg']) && $_GET['dbg'] === 'kku-debug-7731') {   // เปลี่ยนเป็นคีย์ลับของคุณเอง
    ini_set('display_errors', 1);
    error_reporting(E_ALL);
}
register_shutdown_function(function () {
    $e = error_get_last();
    if ($e) {
        file_put_contents('C:\\PHP\\logs\\view100_fatal.log',
            date('c') . " {$e['type']} {$e['message']} @ {$e['file']}:{$e['line']}\r\n", FILE_APPEND);
    }
});
```

เหตุผลที่ทำแบบนี้:
- **ใช้คีย์ลับ** เพราะเว็บนี้เป็นเว็บสาธารณะ ไม่ควรให้คนทั่วไปเห็น error ซึ่งอาจเผย path หรือรหัสฐานข้อมูล ส่วนการกรองด้วย IP ใช้ไม่ได้ เพราะ nginx อยู่ด้านหน้า ทุก request จะเห็นเป็น IP ของ nginx
- **`register_shutdown_function`** จับ fatal error ได้แม้ไม่แสดงบนจอ และเขียนลงไฟล์แยก

ทดสอบโดยเปิด:
`https://personweb.kku.ac.th/E-Administration/BackEnd/Program/Manage/view100.php?FILE_ID=243631&dbg=kku-debug-7731`

แล้วดูไฟล์ log:

```powershell
Get-Content "C:\PHP\logs\view100_fatal.log" -Tail 20
```

> ข้อจำกัด: ถ้า error เป็น **syntax error ในไฟล์ view100.php เอง** วิธีนี้จะไม่ช่วย เพราะ PHP parse ไฟล์ไม่ผ่านตั้งแต่ก่อนรันบรรทัดแรก แต่กรณีนั้นขั้นที่ 2 (`php -l`) จะเจอไปแล้ว

**เมื่อแก้เสร็จ ให้คืนไฟล์เดิมหรือลบโค้ด debug ออก:**

```powershell
Copy-Item "view100.php.bak_YYYYMMDD_HHMM" view100.php -Force
```

---

ถ้ารันขั้นที่ 2 หรือดู log แล้วได้ข้อความ error ออกมา ส่งข้อความนั้นมาได้เลยครับ (ปิดบังรหัสผ่านหรือ IP ภายในก่อน) ผมจะช่วยวิเคราะห์ต่อว่าต้องแก้ตรงไหน
