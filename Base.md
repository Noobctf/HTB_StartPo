# Base
###### tags: `HTB_Starting Point`

1. 枚舉(Enumeration)
* 使用Ping指令確認VPN可以正常連線到靶機
```
ping -c 2 10.10.10.48
```
從上圖我們可以知道TTL=63，因此我們可以推測目標靶機的作業系統為Linux

* 使用nmap進行快速端口掃描，查看哪些端口狀態是開放的
	* -n:不使用DNS遞迴查詢
	* -vv:結果可視化
	* --open:只看有開放的端口
	* -T4:速度較快(0~5)
	* -p-(1-65535)
	* -oA(儲存結果)
```
sudo nmap -n -vv --open -T4  -p- -oA /nmap/Base 10.10.10.48
``` 
可以發現只有22/80Port是開啟的狀態

* 針對22/80Port，再執行NSE Scripts掃描
```
sudo nmap -sC -sV -oA /nmap/Base_Deatails 10.10.10.48 -p22,80
```

* 開啟Burp以及Browser檢查一下80Port



* Login介面是我們較感興趣的地方，嘗試枚舉一下，會看到一個有趣的畫面

* 我們將login.php.swp載到桌面，用cat觀察，會發現有很多亂碼字符，改用strings查看可以僅輸出該文件中的可讀字符串，不過內容似乎也被顛倒了，可以使用| tac，將內容反轉!
```
strings login.php.swp |tac
```

* 可以發現這裡 strcmp 被賦予一個空數組來與存儲的密碼進行比較，因此它將返回 null 並且在 PHP 中， == 運算符僅檢查變量的值是否相等。 因此 NULL 的值在這裡等於 0。 避免這種弱點； 應該使用 === 而不是 ==。

* 現在我們啟動Burp Suite 並嘗試捕獲登錄請求。 我們收到了post請求。 為了利用這個漏洞，修改請求體如下。👇👇然後關閉攔截。
* 為了知道上傳後的檔案會去哪裡，我們使用dirsearch來掃描


```
sudo dirsearch -u http://10.10.10.48/ -w /usr/share/SecLists/Discovery/Web-Content/big.txt
```
```
/var/www/html/login
ls -al
cat config.php
```

```
cat /etc/passwd
```
```
sudo /usr/bin/find /bin -exec /bin/bash \;
```

`cat /root/root.txt```