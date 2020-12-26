# N1盒子备忘录，旁路由模式(N1和其他设备都挂在路由器的WAN口)
如果已经刷入  "2020-\*-\*-N1_Openwrt_R20.12.26_amlogic-5.4.83-50+o-增强版"  类似固件，打算升级了，怎么做？
1. USB2.0刷入固件
2. 保证可以登录固件给的管理IP，比如上述固件默认项：OpenWrt 默认后台管理地址192.168.123.2  默认 用户名：root  密码：password。  
链接：https://www.right.com.cn/forum/thread-3160780-1-1.html  
由于不想连线，手动修改路由器wan的地址段为192.168.123.1，即客户端网关为192.168.123.1，客户端比如电脑重新获取192.168.123段的地址
3. U盘插入N1盒子，靠近hdmi的口，重启盒子，**盒子重启自动进入U盘**
4. 电脑打开后台管理地址192.168.123.2，登陆，配置：  **重点**
- 接口的ipv4网关和DNS服务器都是路由器的192.168.123.1   
- 防火墙加入规则iptables -t nat -I POSTROUTING -o eth0 -j MASQUERADE  
以上两条缺一不可，不然要么是N1无法连外网，要么无法解析DNS导致不能更新订阅
5. 订阅，改密码，关闭无线接口 
6. 建议试用一段时间，觉得新版本没问题再刷入emmc，在此期间就插着U盘使用
