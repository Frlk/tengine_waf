Tengine waf
=====

Tengine自nginx发展而来，是源自淘宝的开源项目。
基于tengine的防攻击模块，最初我尝试了mod-security，但有一个bug，
在大并发的时候狂吃内存，直至拖垮应用，不知道这个问题现在解决没有。
后来转向ngx_lua_waf（感谢loveshell），并在此基础上做了改良，
觉得效果不错，就推荐给大家。

一、部署
====
部署简单，拿来就能用。整个工程是基于centos6.x，已经编译好了。

如果是其他系统，需要重新编译一遍，参考loveshell的编译步骤。

```txt
(1) 下载 到/usr/local/ 目录
(2) chown –R nobody:nobody  /usr/local/tengine
(3) cp –r /usr/local/tengine/lib/*  /usr/lib64/
(4) 配置 /usr/local/tengine/conf/server/ 站点
    参照模板，比普通的vhosts配置只在location下面多了一行 access_by_lua_file
(5) cd /usr/local/tengine/sbin && ./nginx
```

二、WEB 防护
=======
规则目录为tengine/conf/wafconf/，提供了多维度的web防护策略，有以下几个文件：
```
user-agent      匹配拦截恶意的user-agent
url             匹配拦截恶意的网页路径
args			  匹配拦截恶意的GET请求参数
post            匹配拦截恶意的post请求参数
cookie          匹配拦截恶意的cookie 请求
whitetip         ip白名单
whiteurl         网页路径白名单
blockip          ip黑名单
```

整个配置文件规则为文本格式，简单易懂

每条规则采用正则匹配的方式，例如下面whiteip的规则表示

把来自本机和192.168.100网段的请求加入白名单
```
#whiteip
127.0.0.1
^192\.168\.100\.
```


三、CC防护
=====
防cc攻击，也有很多方法，但大都功能比较少。Tengine waf 的目的就是要做功能强大，支持多样的场景需求。

配置文件路径tengine/conf/wafconf/denycc ， 一条规则有3部分组成：

第一部分为token， token最多由3项组成，中间用“+”拼接。每一项可以是以下值：
```
ip              表示客户端ip
domain          表示域名，有多个域名反代的时候可以区分
uri             表示uri页面路径
uri:/xx/yy.php  表示匹配xx/yy.php 的 uri
GetParam:xxx    表示GET参数xxx的值
PostParam:xxx   表示POST参数xxx的值
CookieParam:xxx 表示Cookie参数xxx的值
header:imei     表示http头参数imei的值
```

第二部分为频率阀值，前面表示次数，后面表示时间（单位秒），例如30/60 表示： 60秒内请求达到30次

第三部分为封禁时间，单位为秒。

示例一：
```
ip+uri 60/60  3600
```
如上规则表示为：如果来自同一个ip同一页面的请求次数超过每分钟60次，则对其封禁1小时。

示例二：

假如我不需要对所有页面去做流控，只需要对特定页面做限制即可。

比如说登录页面/logon.php 和 短信验证码页面 /sms/send.php，

防止暴力破解和短信炸弹，则配置可以为：

```
ip+uri:logon.php   60/60  1800
ip+uri:/sms/       10/60  3600
```

首先允许配置多条规则，其次uri采取正则匹配的形式，/sms/ 则匹配了所有路径含有/sms/的页面。

示例三：

假如网站架构不一样，比如api结构的，gateway?api=logon&则token 可以为 ip+GetParam:api

示例四：

假如我想根据用户维护来做限制，禁止某些用户操作太频繁，首选要取得用户标识，

比如GetParam:userid 或者CookieParam:sessionid，再根据实际情况去组合规则



四、 Q & A
----
4.1  为什么是centos, 为什么部署路径是/usr/local/

因为我这是已经编译好的版本，反正我待过的几家互联网公司用的基本都是centos。

也不用担心/usr目录不够大，因为日志目录可以改，和nginx一样 


4.2 有没有控制开关

有开关，在conf/config.lua里, 有6个开关，分别对应各个配置文件功能是否开启。

```
urlMatch="on"
cookieMatch="on"
postMatch="on"
whiteurlMatch="on"
whiteipMatch="on"
blackipMatch="on"
denycc="on"
```

还有一个总开关，默认关闭，当开启了只进行检测，不进行拦截，若是匹配到规则了会记录到攻击日志中
```
OnlyCheck="off"
```

如果不想用这个防攻击模块了，也很简单，把conf/server/ 目录下

站点的配置文件去掉 access_by_lua_file  这一行即可


4.3  某个ip被封禁了为什么还有请求过来

注意这里实行的是宽封禁，即封禁也需要满足token条件，假如规则为ip+uri:/abcd/ 命中被封禁了，

则只封这个ip并带有abcd目录的访问，这个ip访问其他页面仍旧不封禁。所以默认策略下token参数越多，

封禁力度就越小。 建议读下代码，想改也很容易。




如发现bug，欢迎随时联系caobinbupt@foxmail.com，或加我微信号 lanzbupt

参考:
https://github.com/loveshell/ngx_lua_waf

