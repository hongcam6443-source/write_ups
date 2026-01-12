# Recon:
`22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.7 (Ubuntu Linux; protocol 2.0)`
`80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))`
第一个攻击向量：22端口跑的是ssh，可以尝试弱口令爆破，不过这一般不是第一选择
第二个攻击向量：80,http服务，理所当然我们要去看看：
![[gamezone-port80.png]]
# Access:
这是个游戏登录网站，一般情况来说，服务端会把所有玩家用户的信息储存在一个database里面，当你输入你的用户名和密码的时候，服务器会使用sql（structured query language）语句进行查询，就像这样：
`SELECT * FROM users WHERE username = :username AND password := password`
那如果我们把sql语句作为username或者password的值输入进去，就会截断原本的sql命令执行我们的命令，比如我们在username输入`' or 1=1 -- -`，然后password留空，然后我们的sql语句就会变成这样：
`SELECT * FROM users WHERE username = '' or 1=1 -- -' AND password := ''`
==SELECT * FROM users WHERE username = ''==语句在这里就闭合了，紧接着一个or判断，1=1恒定成立， --忽略后面所有内容，所以我们不需要已知任何用户名和密码就可以登录
![[gamezone-portal.png]]
这是一个portal.php，相当与直接站在了database的面前，可以直接执行查询。那我们就可以通过sql injection的手段获得数据库的全部内容，这一点我们可以通过自动化工具--sqlmap来实现：
我们先用burpsuite抓一个包出来储存在本地：
![[hackpark-request.png]]
然后我们使用kali内置的sqlmap直接开始：
`sqlmap -r request.txt --dbms=mysql --dump`
结果我们可以看到两个表：
![[gamezone-post.png]]![[gamezone-users.png]]
这下我们就拿到用户的密码哈希了，下面就是本地破解了
![[gamezone-crack.png]]
既然有密码了，我们可以直接使用ssh获得目标机器的shell：
![[gamezone-shell.png]]
`agent47@gamezone:~$ ss -tulpn`
我们来检查所有的socket通信：
`Netid  State      Recv-Q Send-Q Local Address:Port               Peer Address:Port`              
`udp    UNCONN     0      0              *:10000                      *:*`                  
`udp    UNCONN     0      0              *:68                         *:*`                  
`tcp    LISTEN     0      80     127.0.0.1:3306                       *:*`                  
`tcp    LISTEN     0      128            *:10000                      *:*`                  
`tcp    LISTEN     0      128            *:22                         *:*`                  
`tcp    LISTEN     0      128           :::80                        :::*`                  
`tcp    LISTEN     0      128           :::22                        :::*`                  除去我们nmap发现的port80和port2，还有port3306，不过他只监听本地回环，所以我们扫不到他。但还有和port10000和port68，他们都显示*，面向所有连接，但因为port是udp连接，我们在使用nmap扫描的时候没有发udp探测包，所以没扫出来，但port10000就不一样了，他是tcp通信，而且面向所有连接，但我们还是没有探测到，那就说明他被一个外部的防火墙拦截了，如果我们想看到这个端口运行的服务，那就要使用ssh port forwarding：
![[gamezone-port-forwarding.png]]
接下来我们直接在本地浏览器访问localhost:10000就可以看到内容了，因为目标port10000的内容已经被转发到我们port10000上来了：
![[gamezone-webmin.png]]![[gamezone-cms.png]]
接下来既然我们知道了webmin的具体版本号，我们可以尝试使用metasploit来进行提权：
![[gamezone-payload.png]]
这里有几点需要注意，首先我们要保持之前的ssh tunnel开启，因为有防火墙阻挡所以要让我们的payload发往我们本地的port10000，然后通过ssh从目标的内部冒出来。所以我们需要把RHOST 设置为本地回环：127.0.0.1。还有就是目标使用的是hhtp协议，所以ssl也要设置为False。
![[gamezone-root.png]]
