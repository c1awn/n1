# N1盒子备忘录

## 写在前面
>基于Kernel 5.4.86-flippy-51+o

如果不需要端口转发，那主路由模式下，放弃openclash，选择ssr+；

如果需要端口转发，只能放弃N1主路由方案。因为**N1做主路由，只能转发路由器本身的端口；N1做旁路由，那端口转发是主路由的事** 。

---

## 1. 旁路由模式

如果已经刷入 "2020-*-*-N1_Openwrt_R20.12.26_amlogic-5.4.83-50+o-增强版" 类似固件，打算升级了，怎么做？

1. USB2.0的U盘刷入固件，因为N1接口是2.0，用3.0很大可能失败
2. 保证可以登录固件给的管理IP，比如上述固件默认项：OpenWrt 默认后台管理地址192.168.123.2 默认 用户名：root 密码：password。  
  链接：[【2022-08-05 】 N1盒子 Openwrt 固件,支持 在线升级！-斐讯无线路由器以及其它斐迅网络设备-恩山无线论坛 - Powered by Discuz!](https://www.right.com.cn/forum/thread-3160780-1-1.html)  
  由于不想动物理线路，手动修改路由器wan的地址段为192.168.123.1/24，即客户端网关为192.168.123.1，客户端比如电脑重新连接WiFi后获取192.168.123段的地址
3. U盘插入N1盒子，靠近hdmi的口，重启盒子，**盒子重启自动进入U盘**
4. 电脑打开后台管理地址192.168.123.2，登陆，配置： **重点**

- 接口的ipv4网关和DNS服务器都是路由器的192.168.123.1
- 防火墙加入规则iptables -t nat -I POSTROUTING -o eth0 -j MASQUERADE ，次规则对应下面的lan eth0
- lan口取消桥接，eth0 lan,同时新增一个wan口，取名wan,改网卡名称eth1，这里的wan口不要默认用eth0，就用dhcp，不需要配置IP地址，仅仅是新增一个wan口。  
  **nat规则不写会导致不间断丢包，wan口不新增会导致可以连外网不能访问国内**

5. 订阅，改密码，关闭无线接口
6. 建议试用一段时间，觉得新版本没问题再刷入emmc，在此期间就插着U盘使用
7. 防火墙自定义参考：

```
iptables -t nat -A PREROUTING -p udp --dport 53 -j REDIRECT --to-ports 53
iptables -t nat -A PREROUTING -p tcp --dport 53 -j REDIRECT --to-ports 53
iptables -I FORWARD -i zt44xcjxwe -j ACCEPT
iptables -I FORWARD -o zt44xcjxwe  -j ACCEPT
iptables -t nat -I POSTROUTING -o zt44xcjxwe -j MASQUERADE
#iptables -t nat -A PREROUTING -d 192.168.123.2 -p tcp --dport 4433 -j DNAT --to-destination 192.168.123.15:443 
#iptables -t nat -A POSTROUTING -s 0.0.0.0/0 -o br-lan -j SNAT --to 192.168.123.2
# iptables -t nat -A PREROUTING -p tcp --dport 22 -j DROP
# iptables -I FORWARD -d xx.xx.xx.xx -j DROP
# iptables -t nat -I POSTROUTING -o br-lan -j MASQUERADE
iptables -t nat -I POSTROUTING -o eth0 -j MASQUERADE
```

8. 如果使用openclash，建议redir-host模式，fakeip虽然更快但是对国内也有影响。同样的梯子，clash或者quanx可以访问Netflix，ssr plus插件不一定可以，这个应该和DNS和Netflix的分流IP段有关系，特别是DNS，故如果ssr+无法观看Netflix，可以尝试clash。
  

- smartdns自定义配置，此插件收效甚微

```
#/etc/smartdns/smartdns-domains.china.conf 加载中国域名文件，生成的脚本代码
#https://github.com/huifukejian/test/blob/master/update-china-list.sh
#定时脚本更新conf
conf-file /etc/smartdns/smartdns-domains.china.conf

#speed-check-mode tcp:443,tcp:80,ping
#下面的bind  :6053会和基本设置的6053冲突，需要改基本设置的端口，比如改成6054,。端口冲突会导致服务无法运行。
#root@N1:~# netstat -an|grep 605
#udp        0      0 0.0.0.0:6053            0.0.0.0:*                           
#udp        0      0 0.0.0.0:6054            0.0.0.0:* 
bind  :6053 -group china
bind  :5335 -group gfwlist

#缓存网站个数,可以适当增加
cache-size 512

#停用IPv6
force-AAAA-SOA yes

#过期域名缓存
serve-expired yes

#缓存持久化
cache-persist no

# 开启域名预获取
prefetch-domain yes

# 设置 TTL 最小值和最大值
rr-ttl-min 1800
rr-ttl-max 86400

#国内DNS
server-https https://doh.pub/dns-query -group china -exclude-default-group
server-https https://223.5.5.5/dns-query -group china -exclude-default-group
server 61.177.7.1:53    -group china    -exclude-default-group

#国外DNS
# 如果部分地区存在TCP阻断53端口的情况，可以尝试把国外DNS换成DOT/DOH，如下
#server-https https://1.1.1.1/dns-query    -group gfw    -exclude-default-group
#server-tls 8.8.4.4:853                    -group gfw    -exclude-default-group
server-tls 8.8.8.8:853 -group gfwlist
server-tls 8.8.4.4:853 -group gfwlist
server-tls 208.67.222.222:853 -group gfwlist
```

9.修改用户名root参考，改后重启。第六步不修改可能导致页面修改配置点保存后一直转圈“正在应用/修改”，虽然修改已经成功，很膈应。这个问题走了一些弯路，最开始以为是只读或者硬盘坏道导致，e2fsck 命令没有效果，检查坏块提示0坏块，所以和硬盘只读无关。

```
1.修改/etc/passwd
将root:x:0:0:root:/root:/bin/ash修改为username:x:0:0:root:/root:/bin/ash。
2.修改/etc/shadow
将第一条root:修改为username:
3.修改/usr/lib/lua/luci/controller/admin/index.lua
将page.sysauth = “root” 修改为page.sysauth = “username”。
4.修改/etc/config/rpcd
将option username 'root' 改成 option username 'username'
将option password '$p$root' 改成 option password '$p$username'
5./usr/lib/lua/luci/view/sysauth.htm  
<input class="cbi-input-user" type="text" name="luci_username" value="<%=duser%>" />
改为
<input class="cbi-input-user" type="text" name="luci_username" value="" />
6.修改/usr/lib/lua/luci/controller/admin/servicectl.lua
将entry({“servicectl”}, alias(“servicectl”, “status”)).sysauth = {“root”}修改为entry({“servicectl”}, alias(“servicectl”, “status”)).sysauth = {“username”}。
```

## 2. 主路由模式

主路由的优点：

- 端口转发等规则更加灵活
  
- 入站规则更加灵活
  

主路由的缺点：

- 端口转发可能失效，至少我没找到成功案例。即使在规则搭配了规则`iptables -t nat -I PREROUTING -i pppoe-wan -p tcp --dport 外网暴露端口号 -j DNAT --to-destination 内网ip:实际端口号`也无效。[参见 X86 openwrt的端口转发不起作用](https://www.right.com.cn/forum/thread-3157978-1-1.html) 和[N1 Openwrt端口转发群晖NAS失败](https://www.right.com.cn/forum/thread-2691645-1-1.html)
- nat loop可能有缺陷，[参见 lean版固件NAT 回环问题 · Issue #356 · stupidloud/nanopi-openwrt · GitHub](https://github.com/stupidloud/nanopi-openwrt/issues/356)
- 启用openclash等插件后，可能影响国内访问的速度。
- 主路由挂了，全家网络都受影响

主路由配置方法：

无vlan交换机，[N1作为主路由参考配置](https://www.right.com.cn/forum/thread-572715-1-1.html)，其中尤其注意的是iptables规则，是`iptables -t nat -I POSTROUTING -o pppoe-wan -j MASQUERADE`，非`“iptables -t nat -I POSTROUTING -o eth0 -j MASQUERADE”`。

**注意：eth0的动态伪装只适用旁路由模式，如果在主路由模式开启eth0动态伪装，那端口转发会失效。**

另，发现一个~~可能和iptables允许self入站的现象~~bug（nat loop缺陷，可能和固件有关）：同一个局域网的设备通过**公网**探测openwrt的22等端口是通的，导致我以为wan的入站拒绝是摆设，“特么全部暴露在公网了？”，其实探测端换一个公网环境再通过公网探测openwrt的端口就是timeout。

其他：

- 防火墙-区域-wan-转发选择“拒绝”，上面的“基本设置”的转发选择“接受”，那端口转发还是正常的，说明“基本设置”是一个全局策略。尾部的两个框需要勾选
  
- 防火墙-区域-lan，尾部的两个框不需要勾选，即ip动态伪装和mss钳制不需要
  

## 3. openclash的问题

openclash的dns总是有奇葩的问题，和模式无关。即使在“自定义上游dns”配置了当地运营商的dns，默认的dns全部丢弃，国内访问还是很慢。具体的指标参考：不开启openclash，打开贝壳网二手房主页的900ms-1.2s，开启openclash后3.5s-3.8s，差距近四倍。

| 模式  | 大缺陷 | 备注  |
| --- | --- | --- |
| redir-host | youtube可以打开，但是一直黑屏+转圈 | 怀疑是dns问题 |
| fake-ip | ddns通过请求ip.3322.net的ip错误 | 解决ddns的办法：dns高级设置自定义fakeip的filter域名 |

为了验证smartdns是否有帮助，在openclash的fakeIP模式下，openclash的dns设置如下：

- [ ] 勾选`本地DNS劫持`、`自定义上游DNS服务器`和`禁止Dnsmasq缓存DNS`
  
- [ ] 网络-dhcp/dns的`dns转发` 默认为空，因为`本地DNS劫持` 后会修改这里。同时，既然已经劫持本地dns，那理论上lan接口的dns配置可为空，也可以有值
  
- [ ] 自定义上游DNS服务器中的`Nameserver` 为smartdns，127.0.0.1的6053/udp
  
- [ ] smartdns的重定向选`无` ，dns填本地运营商和几个公共的dns
  

上述设置后，dns路线是先经过openclash，再通过自定义nameserver请求smartdns。****可结果不如人意，打开贝壳网二手房主页依旧是3.5s-3.8s，和没开启smartdns之前一样****。不过想一想，openclash的自定义dns不管是smartdns还是直接填写本地运营商，其实是一回事，因为smartdns配置的也是本地运营商等dns。

ssr+和openclash的对比：

- ssr+功能简单，分流规则太弱，~~同样的梯子，可能ssr+不能打开**Netflix**，而openclash可以打开。~~ ssr+版本更新后对Netflix分流已升级，高级设置可以定义在线的Netflix ip段，也可以自定义。
  
- openclash可能导致**国内网站肉眼可见的慢**或者**YouTube不能正常播放**，应该都和dns相关
