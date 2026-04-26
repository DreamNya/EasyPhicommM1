# 斐讯悟空 M1 停止 WiFi 图标闪烁、调整亮度简单解决方案

个人折腾记录

## 最低需求

纯软件方案，无需硬件拆机刷机

-   安卓手机 `用于将悟空M1设备连接上WiFi`
-   能改 hosts 的路由器 `用于重定向斐讯服务器请求`
-   Linux 服务器 `用于接收设备请求（有其他可替代方案）`

# 正文

## 1. 校准时间

### 1.1 使用安卓 EasyLink 将悟空 M1 设备连接上 WiFi

参考 [斐讯悟空 M1 使用 EasyLink 连接 WiFi 校对时间](https://mrhao.net/archives/134/)

**\*成功后 WiFi 图标应该不再显示红叉，改为开始闪烁**

## 2. 停止 WiFi 图标闪烁

### 2.1 原理

```text
经抓包后发现，悟空M1连接上WiFi后会请求2个服务器
1.请求阿里云校准时间
2.TCP请求斐讯服务器aircat.phicomm.com:9000上传实时数据

由于众所周知的原因，斐讯服务器已经关闭，故请求不通因此会导致WiFi图标不断闪烁
为了使WiFi图标停止闪烁，我们可以使用重定向请求+伪造服务器的手段，欺骗设备
```

### 2.2 修改路由器 hosts

```text
【Linux服务器地址】 aircat.phicomm.com
```

_服务器无论是本地或远程，只要路由器能连上即可_
**改完后重启路由器使其生效**

### 2.3 Docker 启动容器

以下方案二选一

#### netcat
```bash
docker run -d -p 9000:9000 --name=m1-server --restart always subfuzion/netcat -vl 9000
```
参考 [群晖 NAS 解决悟空 M1 的 WiFi 图标红叉/闪烁](https://www.bilibili.com/opus/984763897121603592)

#### socat
```bash
docker run -d \
  --name=m1-server \
  --restart always \
  -p 9000:9000 \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  alpine sh -c "apk add --no-cache socat && socat -u TCP-LISTEN:9000,reuseaddr,fork,keepalive,tcp-keepidle=10,tcp-keepintvl=5,tcp-keepcnt=3 OPEN:/dev/null"
```
由Gemini优化，解决了netcat方案可能遇到网络波动导致设备无法正常重连的情况

### 2.4 原理

```text
悟空M1 TCP请求需持续连接，约每15秒HeartBeat一次，如此设备会认为通信正常，WiFi图标不再闪烁
netcat接收请求后无需返回任何内容（一旦返回内容可能会导致连接中断）
*如无Docker环境，建立持久TCP连接亦可
```

**\*成功后 WiFi 图标应该不再闪烁**
**（若出现闪烁 4 下+10 秒不闪烁，说明设置有误，TCP 连接中断）**

## 3. Docker 远程调整设备亮度

_（如果有斐讯 K2P 路由器可直接用官改固件控制，无需使用 Docker 手动调整）_

### 3.1 原理

```text
悟空M1 TCP请求HeartBeat时若收到指定JSON响应，可根据响应内容进行设置
包括亮度、定时等功能
```

参考 [斐讯 M1 空气检测器独立控制方法](https://iytc.net/wordpress/?p=4150)

### 3.2 关闭原容器

```bash
docker stop m1-server
```

### 3.3 建立新容器

```bash
docker run -d -p 9000:9000 --name=brightness alpine/socat   tcp-l:9000,fork,reuseaddr exec:'printf "\xaaO\x01\xf2E\x119\x8f\x0b\x00\x00\x00\x00\x00\x00\x00\x00\xb0\xf8\x93\x11T/\x007\x00\x00\x02{"brightness":"【亮度】","type":2}\xff#END#"'
```

_【亮度】改为所需亮度，仅可输入 0、25、50 三种值，初始值为 50_
如需定时调整亮度，需要通过脚本实现，本文不讨论

**\*成功表现：WiFi 图标停止闪烁 8~10 秒后，设备整体亮度才会调整到对应值，然后继续闪烁 4 下**

### 3.4 移除新容器，重启旧容器

```bash
docker stop brightness
docker rm brightness
docker start m1-server
```

# 完结撒花

如需脚本控制可额外参考  
[斐讯 M1 空气检测器独立控制方法](https://iytc.net/wordpress/?p=4150)  
[Phicomm 悟空 M1 服务器,数据可写入 MYSQL,前端使用 Flask 框架](https://github.com/fenggenet/PhicommM1_Server)
