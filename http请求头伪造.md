📝 Web安全基础：HTTP请求头伪造与来源欺骗
Tags: #Web安全 #CTF #BurpSuite #网络安全 #蓝桥杯备考 #网络工程师
Description: 总结在 Web 渗透与 CTF 竞赛中常见的 HTTP Header 伪造技巧，探讨客户端信任漏洞的利用与防御。

📌 1. 核心概念：为什么可以伪造？
HTTP 协议是一个无状态且明文的协议。Web 服务器在接收 HTTP 请求时，往往需要从请求头（Headers）中获取客户端的信息（如来源位置、IP地址、设备类型等）。
漏洞成因： 后端代码过度信任了来自客户端的输入。因为 HTTP 报文在到达服务器前可以被拦截修改（例如使用 Burp Suite 抓包），攻击者可以随意篡改 Header 字段，绕过服务器的逻辑校验。

🛠️ 2. 常见 Header 伪造场景与实战技巧
在 CTF Web 题和实际渗透测试中，最常遇到以下三种限制，都可以通过修改 Header 绕过：

🅰️ 场景一：来源限制 (Referer Spoofing)
服务器逻辑： 检查用户是从哪个链接跳转过来的（防盗链或特定的访问控制）。

目标 Header： Referer

实战表现： 页面提示 "Are you from google?" 或 "Must access from internally"。

Burp Suite 修改示例：

HTTP
GET /flag HTTP/1.1
Host: challenge.ctf.com
Referer: https://www.google.com   <-- 手动添加或修改此行
🅱️ 场景二：IP 限制 (IP Spoofing)
服务器逻辑： 限制只有特定 IP（如本地 127.0.0.1 或内网段）才能访问敏感接口。

目标 Header： X-Forwarded-For (XFF), Client-IP, X-Real-IP

实战表现： 页面提示 "Only local users can access" 或 "Your IP is not allowed"。

Burp Suite 修改示例：

HTTP
GET /admin HTTP/1.1
Host: challenge.ctf.com
X-Forwarded-For: 127.0.0.1       <-- 伪造经过代理的真实客户端IP
Client-IP: 127.0.0.1             <-- 备用手段
🅲 场景三：设备/浏览器限制 (User-Agent Spoofing)
服务器逻辑： 根据客户端设备类型返回不同页面，或者要求必须使用特定的内部浏览器。

目标 Header： User-Agent

实战表现： 页面提示 "Please use our internal browser" 或 "Only mobile devices allowed"。

Burp Suite 修改示例：

HTTP
GET /index.php HTTP/1.1
Host: challenge.ctf.com
User-智能体：Mozilla/5.0 (iPhone; CPU iPhone OS 14_0 like Mac OS X) AppleWebKit/605.1.15  <-- 伪装成苹果手机访问
🎯 3. 进阶拓展：其他可能被利用的 Header
除了上述“老三样”，在更复杂的题目或系统架构中，还可以关注以下头部：

Host: * 作用： 指定请求的域名。

利用： Host 碰撞漏洞、SSRF（服务器端请求伪造）绕过、密码重置邮件投毒。

接受-Language:

作用： 客户端期望的语言。

利用： 如果后台将此字段直接拼接到 SQL 语句或页面输出中，可能存在 SQL 注入或 XSS。

Cookie:

作用： 维持会话状态。

利用： 权限越权（修改 role=admin）、会话固定、伪造签名（如果加密算法存在缺陷）。
