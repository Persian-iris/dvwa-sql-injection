# dvwa-sql-injection

**📌 项目简介**

本项目记录在 DVWA（Damn Vulnerable Web Application）靶场中，手工完成 SQL 注入漏洞的检测与利用全过程。通过实战理解 SQL 注入原理、利用方式及防御方法。

**🛠 实验环境**

虚拟机：VMware 16 Pro + Kali Linux

靶场：DVWA (Damn Vulnerable Web Application)

Web 服务：Apache 2.4 + MySQL (MariaDB) + PHP 8.2

工具：浏览器、Burp Suite（可选）

**🔧 环境搭建要点**

在 Kali 中安装 LAMP 环境：sudo apt install -y apache2 mariadb-server php php-mysqli php-gd php-xml

将 DVWA 源码克隆到 /var/www/html/DVWA

配置数据库连接：config/config.inc.php 中设置数据库用户为 root，密码留空（根据实际修改）

解决 MySQL 认证问题：执行 ALTER USER 'root'@'localhost' IDENTIFIED BY '';

访问 http://127.0.0.1/DVWA/setup.php 初始化数据库

**🔍 实验步骤（安全级别：Low）**

1. 确认注入点
输入 1 正常返回；输入 1' 报错，确认存在 SQL 注入漏洞。

![代码](screenshots/sql_injection/1.png)

![代码](screenshots/sql_injection/1结果.png)

![1'](screenshots/sql_injection/1'.png)

![1'报错](screenshots/sql_injection/报错.png)

2. 判断字段数

使用 ORDER BY 猜测原查询的列数：

text
1' ORDER BY 1#   → 正常
1' ORDER BY 2#   → 正常
1' ORDER BY 3#   → 报错
得出字段数为 2。

![字段](screenshots/sql_injection/行数.png)

![报错](screenshots/sql_injection/报错2.png)

3. 获取当前数据库名和用户
4. 
text
1' UNION SELECT database(), user()#
返回：dvwa 和 root@localhost。

![字段](screenshots/sql_injection/database.png)

4. 获取所有表名
   
text
1' UNION SELECT table_name, table_schema FROM information_schema.tables WHERE table_schema='dvwa'#
得到表：guestbook, users。

![字段](screenshots/sql_injection/table.png)

5. 获取 users 表的列名
   
text
1' UNION SELECT column_name, data_type FROM information_schema.columns WHERE table_name='users'#
关键列：user, password。

![字段](screenshots/sql_injection/column.png)

6. 导出用户名和密码
   
text
1' UNION SELECT user, password FROM users#

![字段](screenshots/sql_injection/union.png)


（密码哈希可通过在线 MD5 解密得到明文，例如 admin 的密码为 password）

**📝 实验总结**

漏洞成因：应用程序直接将用户输入拼接到 SQL 查询中，未做任何过滤或参数化处理。

利用条件：原查询返回的列数需与 UNION 后的查询一致（此处为 2 列）。

风险：可窃取数据库中所有表的数据，甚至通过 INTO OUTFILE 写入 WebShell。

🛡 防御方法

使用参数化查询（预编译语句），示例（PHP PDO）：

```
php

$stmt = $pdo->prepare("SELECT * FROM users WHERE id = ?");
$stmt->execute([$id]);
```

**🔗 后续计划**

完成 DVWA 的 XSS、文件上传、盲注等模块

尝试 PortSwigger 的 SQL 注入实验室



**📂 附件**

详细操作截图（见 screenshots 文件夹）

所有使用的 SQL 注入 payload 见 [payload.txt](payload.txt)


# XSS Payloads (DVWA Low)

**1 XSS (Reflected) —— 反射型**

原理：输入的内容被立即拼接进页面并返回，未经过滤，导致 JavaScript 代码在受害者浏览器执行。

操作步骤：

在左侧导航栏点击 XSS (Reflected)。

在输入框中输入普通文本，如 hello，点击 “Submit”，观察页面显示 hello。

现在输入一个 JavaScript 弹窗 payload：

```
html
<script>alert('XSS')</script>
```
点击提交。如果成功，浏览器会弹出一个对话框，显示 “XSS”。

尝试另一种 payload（用于绕过简单的过滤，但在 Low 级别通常直接生效）：

```
html
<img src=x onerror=alert(1)>
```
同样会触发弹窗。

![XSS](screenshots/xss/1.png)

思考：反射型 XSS 通常需要诱使用户点击恶意链接，例如：
``
text
http://127.0.0.1/DVWA/vulnerabilities/xss_r/?name=<script>alert('XSS')</script>
```
可以复制这个链接到浏览器地址栏试试（注意 URL 编码问题，但 DVWA 会自动处理）。

![XSS](screenshots/xss/XSS.png)


**2 XSS (Stored) —— 存储型**
原理：输入的内容被保存到数据库，每次访问页面时都会执行，影响所有访客。

操作步骤：

点击左侧 XSS (Stored)，进入留言板页面。

在 “Name” 和 “Message” 框中先输入普通内容（如 test / test message），提交，观察留言显示。

在 “Message” 框中输入以下 payload：

```
html
<script>alert('Stored XSS')</script>
```
提交。页面会立即弹窗，并且这条留言被保存。刷新页面，弹窗再次出现，说明每次加载页面都会执行。

![XSS](screenshots/xss/刷新前.png)

![XSS](screenshots/xss/xss显示.png)

![XSS](screenshots/xss/刷新后.png)

尝试窃取 Cookie 的 payload（展示危害）：

```
html
<script>alert(document.cookie)</script>
```
提交后，弹窗会显示当前页面的 Cookie（如果有）。

![XSS](screenshots/xss/cookie.png)

注意：如果浏览器阻止了弹窗（如 Chrome 会限制非用户触发的 alert），可以换用 console.log('xss')，然后按 F12 打开开发者工具查看控制台输出。
