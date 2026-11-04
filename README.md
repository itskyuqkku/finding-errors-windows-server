ด้านล่างเป็นสคริปต์ตรวจสอบแบบครบชุดในก้อนเดียว คัดลอกไปวางใน **PowerShell ISE (Run as administrator)** แล้วกด **F5** ได้เลยครับ

**สคริปต์นี้ทำอะไรบ้าง**
- อ่านค่า PHP/IIS, extension, สิทธิ์ไฟล์, โค้ดช่วงที่น่าสงสัย และไฟล์ที่ถูกแก้ล่าสุด
- จำลองการรันแบบ command line และ**ยิง request จริงเข้า IIS จากในเครื่อง** เพื่อจับ error ที่เกิดตอนรันด้วยสิทธิ์ของ IIS (จุดนี้คือสิ่งที่ยังขาดอยู่)
- **ไม่แก้ php.ini และไม่แก้ไฟล์เว็บเดิม** สิ่งที่สร้างขึ้นมีแค่โฟลเดอร์ `D:\PHPLogs` กับไฟล์ทดสอบชั่วคราวหนึ่งไฟล์ ซึ่งสคริปต์จะลบให้เองตอนจบ ไฟล์ทดสอบนี้เปิดได้เฉพาะจากในเครื่อง Server และต้องใช้รหัสสุ่ม
- ซ่อนรหัสผ่านใน `Config.php` ให้อัตโนมัติ
- บันทึกผลทั้งหมดลง `D:\PHPLogs\diag_report.txt` แล้วเปิดใน Notepad ให้

```powershell
# ===== KKU View100.php diagnostic =====
$php    = "C:\Program Files (x86)\PHP\v5.6\php.exe"
$ini    = "C:\Program Files (x86)\PHP\v5.6\php.ini"
$site   = "D:\WEB\E-Administration\BackEnd\Program\Manage"
$sys    = "D:\WEB\E-Administration\BackEnd\System"
$logd   = "D:\PHPLogs"
$report = "$logd\diag_report.txt"
$appcmd = "$env:windir\system32\inetsrv\appcmd.exe"
$tok    = [guid]::NewGuid().ToString('N').Substring(0,12)
$diag   = "$site\_diag100_$tok.php"

New-Item -ItemType Directory -Force $logd | Out-Null
icacls $logd /grant "IIS_IUSRS:(OI)(CI)M" /grant "IUSR:(OI)(CI)M" /grant "IIS AppPool\DefaultAppPool:(OI)(CI)M" | Out-Null
Remove-Item "$logd\diag100.log" -ErrorAction SilentlyContinue

function Mask($t) {
  $t = $t -replace '(?i)((pass|pwd|passwd|password|user|uid|username)\w*[''"]?\s*(=>|=|,)\s*)([''"])[^''"]*\4', '$1$4***$4'
  $t -replace '(?i)((_connect|_pconnect|new\s+PDO|odbc_connect)\s*\()[^)]*\)', '$1***masked***)'
}
function H($t) { "`r`n==================== $t ====================" }
function Req($url, $hostHdr) {
  try {
    $r = [System.Net.HttpWebRequest]::Create($url)
    if ($hostHdr) { $r.Host = $hostHdr }
    $r.AllowAutoRedirect = $false; $r.Timeout = 60000
    $resp = $r.GetResponse()
  } catch [System.Net.WebException] {
    $resp = $_.Exception.Response
    if (-not $resp) { return "REQUEST ERROR: $($_.Exception.Message)" }
  }
  $ms = New-Object IO.MemoryStream
  $resp.GetResponseStream().CopyTo($ms); $b = $ms.ToArray()
  $first = ''; $text = ''
  if ($b.Length -gt 0) {
    $first = [BitConverter]::ToString($b[0..([Math]::Min(7, $b.Length - 1))])
    $text  = [Text.Encoding]::UTF8.GetString($b, 0, [Math]::Min(300, $b.Length))
  }
  "STATUS=$([int]$resp.StatusCode) TYPE=$($resp.ContentType) BYTES=$($b.Length) FIRST8=$first`r`nTEXT=$text"
  $resp.Close()
}

& {
H "1. PHP / IIS"
& $php -c $ini -v 2>&1
& $php -c $ini -i 2>&1 | Select-String -Pattern '^(error_log|log_errors|display_errors|error_reporting|extension_dir|include_path|open_basedir|disable_functions|date.timezone) ='
& $appcmd list site
& $appcmd list config "Default Web Site/" -section:handlers | Select-String php

H "2. PHP extensions (look for 'Unable to load')"
& $php -c $ini -m 2>&1

H "3. Files, dates, permissions"
Get-Item "$site\View100.php", "$site\view.php", "$site\ViewDocFile100.php", "$sys\Config.php" |
  Format-Table Name, Length, LastWriteTime -AutoSize | Out-String
icacls "$site\View100.php"
icacls "$sys\Config.php"
icacls "C:\Windows\temp\PHP56_errors.log"

H "4. Files changed in last 90 days"
Get-ChildItem "D:\WEB\E-Administration" -Recurse -File -Include *.php, *.ini, *.inc, web.config -ErrorAction SilentlyContinue |
  Where-Object { $_.LastWriteTime -gt (Get-Date).AddDays(-90) } |
  Sort-Object LastWriteTime -Descending | Select-Object -First 40 |
  Format-Table LastWriteTime, Length, FullName -AutoSize | Out-String -Width 250

H "5. View100.php lines 1-60 (masked)"
$i = 0; Get-Content "$site\View100.php" -TotalCount 60 | ForEach-Object { $i++; "{0,4}: {1}" -f $i, (Mask $_) }

H "6. Config.php lines 70-130 (masked)"
$c = @(Get-Content "$sys\Config.php")
for ($n = 69; $n -lt [Math]::Min(130, $c.Count); $n++) { "{0,4}: {1}" -f ($n + 1), (Mask $c[$n]) }

H "7. include/require and UNC paths"
foreach ($f in "$site\View100.php", "$sys\Config.php") {
  "--- $f"
  Select-String -Path $f -Pattern 'include|require|\\\\' | ForEach-Object { "{0,4}: {1}" -f $_.LineNumber, (Mask $_.Line.Trim()) }
  Select-String -Path $f -Pattern '(include|require)(_once)?\s*\(?\s*[''"]([^''"]+)[''"]' -AllMatches |
    ForEach-Object { $_.Matches } | ForEach-Object {
      $p = $_.Groups[3].Value
      "  {0}  exists(from Manage)={1}  exists(from file dir)={2}" -f $p, (Test-Path (Join-Path $site $p)), (Test-Path (Join-Path (Split-Path $f) $p))
    }
}

H "8. CLI simulation (as Administrator)"
@'
<?php
error_reporting(E_ALL); ini_set('display_errors', 'stderr');
$_SERVER['SERVER_NAME'] = 'personweb.kku.ac.th'; $_SERVER['HTTP_HOST'] = 'personweb.kku.ac.th';
$_SERVER['REQUEST_METHOD'] = 'GET'; $_SERVER['REMOTE_ADDR'] = '127.0.0.1';
$_GET['FILE_ID'] = $_REQUEST['FILE_ID'] = '243631';
chdir('D:/WEB/E-Administration/BackEnd/Program/Manage');
ob_start();
register_shutdown_function(function () {
  $o = ''; while (ob_get_level() > 0) { $o .= ob_get_clean(); }
  fwrite(STDERR, "\nCLI_OUTPUT_BYTES=" . strlen($o) . " FIRST8=" . bin2hex(substr($o, 0, 8)) .
    "\nCLI_INCLUDED=" . implode(' ; ', get_included_files()) .
    "\nCLI_LAST_ERROR=" . print_r(error_get_last(), true) . "\n");
});
include 'View100.php';
'@ | Set-Content -Encoding ASCII "$env:windir\Temp\dbg100.php"
& $php -c $ini "$env:windir\Temp\dbg100.php" 2>&1 | Select-Object -Last 40

H "9. IIS simulation (real web context)"
@'
<?php
$log = 'D:/PHPLogs/diag100.log';
$ip = isset($_SERVER['REMOTE_ADDR']) ? $_SERVER['REMOTE_ADDR'] : '';
if (!in_array($ip, array('127.0.0.1', '::1')) || !isset($_GET['k']) || $_GET['k'] !== '__TOK__') { header('HTTP/1.1 404 Not Found'); exit; }
ini_set('log_errors', '1'); ini_set('error_log', $log); ini_set('display_errors', '0'); error_reporting(E_ALL);
$w = @file_put_contents($log, "\n=== " . date('c') . " pool=" . getenv('APP_POOL_ID') . " SERVER_NAME=" . $_SERVER['SERVER_NAME'] .
  " HTTP_HOST=" . $_SERVER['HTTP_HOST'] . " PHP=" . PHP_VERSION . " ini=" . php_ini_loaded_file() . "\n", FILE_APPEND);
if ($w === false) { echo "CANNOT_WRITE_LOG"; exit; }
$_GET['FILE_ID'] = $_REQUEST['FILE_ID'] = isset($_GET['FILE_ID']) ? $_GET['FILE_ID'] : '243631';
chdir(dirname(__FILE__));
ob_start();
register_shutdown_function(function () use ($log) {
  $o = ''; while (ob_get_level() > 0) { $o .= ob_get_clean(); }
  file_put_contents($log, "IIS_OUTPUT_BYTES=" . strlen($o) . " FIRST8=" . bin2hex(substr($o, 0, 8)) .
    "\nIIS_HEADERS=" . implode(' | ', headers_list()) .
    "\nIIS_INCLUDED=" . implode(' ; ', get_included_files()) .
    "\nIIS_LAST_ERROR=" . print_r(error_get_last(), true) . "\n", FILE_APPEND);
  if (!headers_sent()) { header_remove(); header('Content-Type: text/plain'); }
  echo "DIAG_DONE bytes=" . strlen($o);
});
include 'View100.php';
'@.Replace('__TOK__', $tok) | Set-Content -Encoding ASCII $diag

$base = "http://127.0.0.1/E-Administration/BackEnd/Program/Manage"
"--- diag via IIS, Host=personweb.kku.ac.th"
Req "$base/_diag100_$tok.php?k=$tok&FILE_ID=243631" "personweb.kku.ac.th"
"--- diag via IIS, Host=localhost"
Req "$base/_diag100_$tok.php?k=$tok&FILE_ID=243631" $null
"--- real View100.php via IIS"
Req "$base/view100.php?FILE_ID=243631" "personweb.kku.ac.th"
"--- real view.php via IIS (for comparison)"
Req "$base/view.php?FILE_ID=243631" "personweb.kku.ac.th"
"--- diag100.log"
if (Test-Path "$logd\diag100.log") { Get-Content "$logd\diag100.log" | ForEach-Object { Mask $_ } } else { "diag100.log NOT CREATED" }

H "10. Event log (last 3 days)"
Get-EventLog -LogName Application -After (Get-Date).AddDays(-3) -ErrorAction SilentlyContinue |
  Where-Object { $_.Source -match 'FastCGI|PHP|Application Error|WAS|W3SVC|IIS' } |
  Select-Object -First 15 | Format-List TimeGenerated, Source, EventID, Message | Out-String -Width 250

H "11. Cleanup"
Remove-Item $diag, "$env:windir\Temp\dbg100.php" -ErrorAction SilentlyContinue
"diag file removed: $(-not (Test-Path $diag))"
} *>&1 | Tee-Object -FilePath $report

notepad $report
```

## วิธีใช้

1. เปิด **Windows PowerShell ISE** ด้วย **Run as administrator**
2. ถ้าไม่เห็นช่องเขียนสคริปต์ด้านบน กด **Ctrl+R**
3. วางสคริปต์ทั้งหมดลงในช่องด้านบน แล้วกด **F5**
4. รอประมาณ 1–2 นาที อาจมีข้อความสีแดงบางบรรทัด ไม่ต้องตกใจ ถือเป็นส่วนหนึ่งของผลตรวจ
5. เมื่อ Notepad เปิดขึ้นมา กด **Ctrl+A** แล้ว **Ctrl+C** จากนั้นวางกลับมาในแชทนี้ได้เลย
   - ก่อนส่ง ให้กวาดตาดูอีกรอบว่าไม่มีรหัสผ่านหลุดมา สคริปต์ซ่อนให้แล้วแต่อาจไม่ครบทุกรูปแบบ
   - ถ้าข้อความยาวเกินจะวาง ไฟล์อยู่ที่ `D:\PHPLogs\diag_report.txt` ส่งเป็นภาพทีละช่วงได้ โดยเน้นหัวข้อ **8, 9 และ 10**

**ถ้าสคริปต์หยุดกลางทางเพราะ error** ให้ถ่ายภาพข้อความ error สีแดงช่วงแรกมา ส่วนไฟล์ทดสอบชั่วคราว ลบเองได้ด้วยคำสั่งนี้:

```powershell
Remove-Item D:\WEB\E-Administration\BackEnd\Program\Manage\_diag100_*.php
```

พอได้ผลกลับมา ผมจะบอกได้ว่าพังที่ไฟล์ไหน บรรทัดไหน และต้องแก้อย่างไรครับ
