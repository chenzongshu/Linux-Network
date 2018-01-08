
https://www.cnblogs.com/metoy/p/4320813.html

http://blog.csdn.net/u011456940/article/details/52634184

http://blog.chinaunix.net/uid-26495963-id-3279216.html


iptables 实际上就是一种包过滤型防火墙

# 四表五链

总体说来，iptables就是由“四表五链”组成。

其中表是按照对数据包的操作区分的，链是按照不同的Hook点来区分的，表和链实际上是netfilter的两个维度

## 四表

- `filter`：过滤，防火墙
- `nat` ： 网络地址转换（端口映射，地址映射等），
- `mangle`：拆解报文，作出修改，封装报文（即对数据包进行修改）
- `raw`： 关闭nat表上启用的链接追踪机制，为了提高效率使用的，raw本身的含义是指“原生的”、“未经过加工的”，符合raw表所对应规则的数据包将会跳过一些检查，这样就可以提高效率

优先级为 ： raw > mangle > nat > filter

## 五链

在设计iptables的时候,在内核空间中选择了5个位置,放置了钩子函数,被称为链,"chain"

    1.内核空间中：从一个网络接口进来，到另一个网络接口去的
    2.数据包从内核流入用户空间的
    3.数据包从用户空间流出的
    4.进入/离开本机的外网接口
    5.进入/离开本机的内网接口

- `PREROUTING` : 数据包进入路由之前
- `INPUT` : 目的地址为本机
- `FORWARD` : 实现转发
- `OUTPUT` : 原地址为本机，向外发送
- `POSTROUTING` : 发送到网卡之前

![](http://www.linuxidc.com/upload/2012_08/120807094039061.gif)

> 	对于filter来讲一般只能做在3个链上：INPUT ，FORWARD ，OUTPUT
对于nat来讲一般也只能做在3个链上：PREROUTING ，OUTPUT ，POSTROUTING
而mangle则是5个链都可以做：PREROUTING，INPUT，FORWARD，OUTPUT，POSTROUTING

# 命令

iptables [-t 表名] 命令选项 ［链名］ ［条件匹配］ ［-j 目标动作或跳转］

- 表名、链名用于指定 iptables命令所操作的表和链，
- 命令选项用于指定管理iptables规则的方式（比如：插入、增加、删除、查看等；
- 条件匹配用于指定对符合什么样条件的数据包进行处理；
- 目标动作或跳转用于指定数据包的处理方式（比如允许通过、拒绝、丢弃、跳转（Jump）给其它链处理。

## iptables命令的管理控制选项

```
-A 在指定链的末尾添加（append）一条新的规则
-D 删除（delete）指定链中的某一条规则，可以按规则序号和内容删除
-I 在指定链中插入（insert）一条新的规则，默认在第一行添加
-R 修改、替换（replace）指定链中的某一条规则，可以按规则序号和内容替换
-L 列出（list）指定链中所有的规则进行查看
-E 重命名用户定义的链，不改变链本身
-F 清空（flush）
-N 新建（new-chain）一条用户自己定义的规则链
-X 删除指定表中用户自定义的规则链（delete-chain）
-P 设置指定链的默认策略（policy）
-Z 将所有表的所有链的字节和数据包计数器清零
-n 使用数字形式（numeric）显示输出结果
-v 查看规则表详细信息（verbose）的信息
-V 查看版本(version)
-h 获取帮助（help）
```

## 防火墙处理数据包的四种方式

- ACCEPT 允许数据包通过
- DROP 直接丢弃数据包，不给任何回应信息
- REJECT 拒绝数据包通过，必要时会给数据发送端一个响应的信息。
- LOG在/var/log/messages文件中记录日志信息，然后将数据包传递给下一条规则

## 防火墙规则保存

iptables-save把规则保存到文件中，再由目录rc.d下的脚本（/etc/rc.d/init.d/iptables）自动装载

```
iptables-save > /etc/sysconfig/iptables

或者
service iptables save
```

## 常用策略

1.拒绝进入防火墙的所有ICMP协议数据包

```
iptables -I INPUT -p icmp -j REJECT
``` 

2.允许防火墙转发除ICMP协议以外的所有数据包

```
iptables -A FORWARD -p ! icmp -j ACCEPT
说明：使用“！”可以将条件取反。
```
 
3.拒绝转发来自192.168.1.10主机的数据，允许转发来自192.168.0.0/24网段的数据

```
iptables -A FORWARD -s 192.168.1.11 -j REJECT 
iptables -A FORWARD -s 192.168.0.0/24 -j ACCEPT
说明：注意要把拒绝的放在前面不然就不起作用了啊。
```
 
4.丢弃从外网接口（eth1）进入防火墙本机的源地址为私网地址的数据包

```
iptables -A INPUT -i eth1 -s 192.168.0.0/16 -j DROP 
iptables -A INPUT -i eth1 -s 172.16.0.0/12 -j DROP 
iptables -A INPUT -i eth1 -s 10.0.0.0/8 -j DROP
```

5.封堵网段（192.168.1.0/24），两小时后解封。

```
# iptables -I INPUT -s 10.20.30.0/24 -j DROP 
# iptables -I FORWARD -s 10.20.30.0/24 -j DROP 
# at now 2 hours at> iptables -D INPUT 1 at> iptables -D FORWARD 1
说明：这个策略咱们借助crond计划任务来完成，就再好不过了。
[1]   Stopped     at now 2 hours
```
 

6.只允许管理员从202.13.0.0/16网段使用SSH远程登录防火墙主机。

```
iptables -A INPUT -p tcp --dport 22 -s 202.13.0.0/16 -j ACCEPT 
iptables -A INPUT -p tcp --dport 22 -j DROP
说明：这个用法比较适合对设备进行远程管理时使用，比如位于分公司中的SQL服务器需要被总公司的管理员管理时。
```
 

7.允许本机开放从TCP端口20-1024提供的应用服务。

```
iptables -A INPUT -p tcp --dport 20:1024 -j ACCEPT 
iptables -A OUTPUT -p tcp --sport 20:1024 -j ACCEPT
```

8.允许转发来自192.168.0.0/24局域网段的DNS解析请求数据包。

```
iptables -A FORWARD -s 192.168.0.0/24 -p udp --dport 53 -j ACCEPT 
iptables -A FORWARD -d 192.168.0.0/24 -p udp --sport 53 -j ACCEPT
```

9.禁止其他主机ping防火墙主机，但是允许从防火墙上ping其他主机

```
iptables -I INPUT -p icmp --icmp-type Echo-Request -j DROP 
iptables -I INPUT -p icmp --icmp-type Echo-Reply -j ACCEPT 
iptables -I INPUT -p icmp --icmp-type destination-Unreachable -j ACCEPT
```

10.禁止转发来自MAC地址为00：0C：29：27：55：3F的和主机的数据包

```
iptables -A FORWARD -m mac --mac-source 00:0c:29:27:55:3F -j DROP
说明：iptables中使用“-m 模块关键字”的形式调用显示匹配。咱们这里用“-m mac –mac-source”来表示数据包的源MAC地址。
```
 
11.允许防火墙本机对外开放TCP端口20、21、25、110以及被动模式FTP端口1250-1280

```
iptables -A INPUT -p tcp -m multiport --dport 20,21,25,110,1250:1280 -j ACCEPT
说明：这里用“-m multiport –dport”来指定目的端口及范围
```
 
12.禁止转发源IP地址为192.168.1.20-192.168.1.99的TCP数据包。

```
iptables -A FORWARD -p tcp -m iprange --src-range 192.168.1.20-192.168.1.99 -j DROP
说明：此处用“-m –iprange –src-range”指定IP范围。
```
 
13.禁止转发与正常TCP连接无关的非—syn请求数据包。

```
iptables -A FORWARD -m state --state NEW -p tcp ! --syn -j DROP
说明：“-m state”表示数据包的连接状态，“NEW”表示与任何连接无关的，新的嘛！
```
 
14.拒绝访问防火墙的新数据包，但允许响应连接或与已有连接相关的数据包

```
iptables -A INPUT -p tcp -m state --state NEW -j DROP 
iptables -A INPUT -p tcp -m state --state ESTABLISHED,RELATED -j ACCEPT
说明：“ESTABLISHED”表示已经响应请求或者已经建立连接的数据包，“RELATED”表示与已建立的连接有相关性的，比如FTP数据连接等。
```
 
15.只开放本机的web服务（80）、FTP(20、21、20450-20480)，放行外部主机发住服务器其它端口的应答数据包，将其他入站数据包均予以丢弃处理。

```
iptables -I INPUT -p tcp -m multiport --dport 20,21,80 -j ACCEPT 
iptables -I INPUT -p tcp --dport 20450:20480 -j ACCEPT 
iptables -I INPUT -p tcp -m state --state ESTABLISHED -j ACCEPT 
iptables -P INPUT DROP
```
