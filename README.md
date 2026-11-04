ผลรอบนี้ทำให้รู้แล้วว่า **ระบบส่วนใหญ่ปกติดี** และปัญหาแคบลงเหลือจุดเดียวครับ

## สิ่งที่ยืนยันได้จากรายงาน

**1. ไฟล์ PDF, ฐานข้อมูล และ file server ทำงานปกติทั้งหมด**
ในหัวข้อ 9 ไฟล์ทดสอบที่รันผ่าน IIS แล้วเรียก `View100.php` ต่อ ได้ผลเป็น **PDF จริงขนาด 439,248 ไบต์** (`FIRST8=255044462d312e37` คือ `%PDF-1.7`) และ header ถูกต้องครบ ทั้ง `Content-type: application/pdf`, `Content-Length` และ `Cache-Control` แปลว่า
- ฐานข้อมูลค้น FILE_ID เจอ
- อ่านไฟล์จาก `//10.74.100.59/DOC_PD/Files/` ได้
- การเช็กสิทธิ์ตาม IP ผ่าน
- `SERVER_NAME` จะเป็น `personweb.kku.ac.th` หรือ `127.0.0.1` ก็ได้ผลเหมือนกัน จึงไม่ใช่ต้นเหตุ

**2. แต่ถ้าเรียก `view100.php` ตรง ๆ ผ่าน IIS ยังได้ 0 ไบต์และ `text/html`**
แปลว่าสคริปต์จบการทำงานก่อนถึงคำสั่ง `header()` ที่ส่ง PDF ปัญหาจึงอยู่ที่**ความต่างระหว่าง "เรียกตรง" กับ "เรียกผ่านไฟล์ทดสอบ"** ซึ่งมีอยู่ไม่กี่อย่าง:
- ไฟล์ทดสอบเปิด output buffering (`ob_start`)
- ไฟล์ทดสอบสั่ง `chdir()` ไปที่โฟลเดอร์ Manage ก่อน (`View100.php` ใช้ `require("../../System/...")` ซึ่ง PHP คิด path แบบนี้จาก current directory)
- ชื่อไฟล์ที่ถูกเรียกต่างกัน (เว็บเรียก `view100.php` ตัวเล็ก ส่วนไฟล์จริงชื่อ `View100.php`) และเครื่องนี้เปิด **WinCache** ไว้ ซึ่งแคชไฟล์ PHP ตาม path
- การตั้งค่า error log

**3. เจอสาเหตุที่ error ไม่เคยถูกบันทึก**
`C:\Windows\temp\PHP56_errors.log` ให้สิทธิ์ `IIS_IUSRS` แค่ `(S,RD)` คืออ่านได้อย่างเดียว app pool จึงเขียน log ไม่ได้ error ทุกอย่างบนเว็บจึงหายไปเงียบ ๆ ตั้งแต่ปี 2017

**4. เรื่องเล็กน้อย**
- ข้อความ `Get-History ... Cannot bind parameter 'Id'` เป็นความผิดพลาดของผมเอง ผมตั้งชื่อฟังก์ชันว่า `H` ซึ่งไปชนกับคำสั่งย่อของ `Get-History` ไม่กระทบผลตรวจ แค่หัวข้อไม่แสดง รอบนี้แก้แล้ว
- ในหัวข้อ 8 ตอนรันด้วยสิทธิ์ Administrator ได้ผล `ERROR : ...` แทน PDF น่าจะเป็นเพราะบัญชี Administrator เข้า share `10.74.100.59` ไม่ได้ ซึ่งไม่เกี่ยวกับปัญหาบนเว็บ

## สคริปต์รอบที่ 2: แยกทีละปัจจัยเพื่อหาตัวการ

สคริปต์นี้จะ:
- **ให้สิทธิ์ IIS เขียน error log ได้** ซึ่งเป็นการแก้ที่ควรทำอยู่แล้ว เพื่อให้เห็น error ในอนาคต
- สร้างไฟล์ทดสอบชั่วคราว 5 ไฟล์ แต่ละไฟล์เปลี่ยนแค่ปัจจัยเดียว แล้วยิง request เทียบกัน
- **recycle DefaultAppPool** หนึ่งครั้งเพื่อล้าง WinCache แล้วทดสอบซ้ำ เว็บทั้ง Default Web Site จะสะดุดไม่กี่วินาที
- ลบไฟล์ทดสอบทั้งหมดตอนจบ

วางใน **Script Pane (พื้นที่สีขาว)** ของ ISE แล้วกด **F5** เหมือนรอบที่แล้ว

```powershell
# ===== KKU View100.php diagnostic #2 (bisect) =====
$php    = "C:\Program Files (x86)\PHP\v5.6\php.exe"
$site   = "D:\WEB\E-Administration\BackEnd\Program\Manage"
$phplog = "C:\Windows\temp\PHP56_errors.log"
$report = "D:\PHPLogs\diag2_report.txt"
$appcmd = "$env:windir\system32\inetsrv\appcmd.exe"
$tok    = [guid]::NewGuid().ToString('N').Substring(0,10)
$base   = "http://127.0.0.1/E-Administration/BackEnd/Program/Manage"
$fid    = "243631"

function Section($t) { "`r`n==================== $t ====================" }
function Req($url) {
  try {
    $r = [System.Net.HttpWebRequest]::Create($url)
    $r.Host = "personweb.kku.ac.th"
    $r.AllowAutoRedirect = $false; $r.Timeout = 60000
    $resp = $r.GetResponse()
  } catch [System.Net.WebException] {
    $resp = $_.Exception.Response
    if (-not $resp) { return "REQUEST ERROR: $($_.Exception.Message)" }
  }
  $ms = New-Object IO.MemoryStream
  $resp.GetResponseStream().CopyTo($ms); $b = $ms.ToArray()
  $text = ''
  if ($b.Length -gt 0) { $text = [Text.Encoding]::UTF8.GetString($b, 0, [Math]::Min(400, $b.Length)) }
  if ($text.StartsWith('%PDF')) { $text = '(PDF OK)' }
  "STATUS=$([int]$resp.StatusCode) TYPE=$($resp.ContentType) BYTES=$($b.Length)`r`nTEXT=$text"
  $resp.Close()
}

$files = [ordered]@{
  "t_inc"   = "<?php include 'View100.php';"
  "t_ob"    = "<?php ob_start(); include 'View100.php';"
  "t_chdir" = "<?php chdir(dirname(__FILE__)); include 'View100.php';"
  "t_env"   = @'
<?php
if (!in_array($_SERVER['REMOTE_ADDR'], array('127.0.0.1', '::1'))) { header('HTTP/1.1 404 Not Found'); exit; }
header('Content-Type: text/plain');
echo "cwd=" . getcwd() . "\n";
echo "dir=" . dirname(__FILE__) . "\n";
echo "SCRIPT_FILENAME=" . $_SERVER['SCRIPT_FILENAME'] . "\n";
echo "config_exists_relative=" . var_export(file_exists('../../System/Config.php'), true) . "\n";
echo "user=" . getenv('USERNAME') . " pool=" . getenv('APP_POOL_ID') . "\n";
echo "wincache=" . (extension_loaded('wincache') ? 'loaded' : 'no') . " fc=" . ini_get('wincache.fcenabled') . " oc=" . ini_get('wincache.ocenabled') . "\n";
echo "error_log=" . ini_get('error_log') . " writable=" . var_export(is_writable(ini_get('error_log')), true) . "\n";
echo "request_order=" . ini_get('request_order') . " variables_order=" . ini_get('variables_order') . "\n";
echo "output_buffering=" . ini_get('output_buffering') . " user_ini=" . ini_get('user_ini.filename') . "\n";
'@
}

& {
Section "0. Allow IIS to write PHP error log"
icacls $phplog /grant "IIS_IUSRS:(M)"
$before = @(Get-Content $phplog).Count

Section "1. Create test files"
Copy-Item "$site\View100.php" "$site\_t_copy_$tok.php"
foreach ($k in $files.Keys) { $files[$k] | Set-Content -Encoding ASCII "$site\_${k}_$tok.php" }
Get-ChildItem "$site\_t_*_$tok.php" | Select-Object -ExpandProperty Name

Section "2. Tests (before recycle)"
"--- T1 real view100.php (lowercase, as the site uses)";  Req "$base/view100.php?FILE_ID=$fid"
"--- T2 real View100.php (capital V)";                    Req "$base/View100.php?FILE_ID=$fid"
"--- T3 real view100.php without FILE_ID (expect 'FILE ID IS BLANK')"; Req "$base/view100.php"
"--- T4 exact copy of View100.php under new name";        Req "$base/_t_copy_$tok.php?FILE_ID=$fid"
"--- T5 wrapper: include only";                           Req "$base/_t_inc_$tok.php?FILE_ID=$fid"
"--- T6 wrapper: ob_start + include";                     Req "$base/_t_ob_$tok.php?FILE_ID=$fid"
"--- T7 wrapper: chdir + include";                        Req "$base/_t_chdir_$tok.php?FILE_ID=$fid"
"--- T8 environment";                                     Req "$base/_t_env_$tok.php"

Section "3. New PHP error log lines (before recycle)"
Get-Content $phplog | Select-Object -Skip $before
$before = @(Get-Content $phplog).Count

Section "4. Recycle DefaultAppPool (clears WinCache) and retest"
& $appcmd recycle apppool /apppool.name:"DefaultAppPool"
Start-Sleep -Seconds 5
"--- T1 again";  Req "$base/view100.php?FILE_ID=$fid"
"--- T3 again";  Req "$base/view100.php"
"--- T4 again";  Req "$base/_t_copy_$tok.php?FILE_ID=$fid"

Section "5. New PHP error log lines (after recycle)"
Get-Content $phplog | Select-Object -Skip $before

Section "6. web.config / .user.ini on the path"
foreach ($d in "D:\WEB", "D:\WEB\E-Administration", "D:\WEB\E-Administration\BackEnd", "D:\WEB\E-Administration\BackEnd\Program", $site) {
  foreach ($n in "web.config", ".user.ini") {
    $p = Join-Path $d $n
    if (Test-Path $p) { "--- $p"; Get-Content $p }
  }
}

Section "7. View100.php lines 61-110"
$v = @(Get-Content "$site\View100.php")
for ($n = 60; $n -lt [Math]::Min(110, $v.Count); $n++) { "{0,4}: {1}" -f ($n + 1), $v[$n] }

Section "8. Cleanup"
Remove-Item "$site\_t_*_$tok.php" -ErrorAction SilentlyContinue
"test files left: $(@(Get-ChildItem "$site\_t_*_$tok.php" -ErrorAction SilentlyContinue).Count)"
} *>&1 | Tee-Object -FilePath $report

notepad $report
```

## วิธีอ่านผล (สำหรับดูคร่าว ๆ ก่อนส่งมา)

| ถ้าผลเป็นแบบนี้ | แปลว่า |
|---|---|
| T1 กลับมา `(PDF OK)` **หลัง recycle** | WinCache แคชไฟล์ผิด แก้ได้ด้วยการ recycle และปรับค่า WinCache |
| T2 ได้ PDF แต่ T1 ได้ 0 ไบต์ | ปัญหาเรื่องตัวพิมพ์เล็ก/ใหญ่ของชื่อไฟล์ร่วมกับ WinCache |
| T4 (สำเนา) ได้ PDF แต่ T1 ได้ 0 ไบต์ | มีบางอย่างผูกกับชื่อ/path `view100.php` โดยเฉพาะ เช่น แคช หรือ config |
| T5 ได้ 0 แต่ T7 ได้ PDF | ปัญหาเรื่อง current directory |
| T5 ได้ 0 แต่ T6 ได้ PDF | ปัญหาเรื่อง output buffering |
| หัวข้อ 3 หรือ 5 มี `Fatal error` / `Warning` | ข้อความนั้นคือต้นเหตุโดยตรง |

ส่งรายงาน `diag2_report.txt` มาได้เลยครับ โดยเฉพาะหัวข้อ **2 ถึง 5** แล้วผมจะสรุปวิธีแก้ที่ตรงจุดให้
