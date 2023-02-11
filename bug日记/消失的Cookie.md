## Set-Cookie标头无法自动保存，保存的Cookie也无法自动发送

### 基本信息

- **时间**：2023-2-11

- **项目名称**：Open Project

- **语言**：Java, Javascript

- **持续时间**：五天左右

### BUG特征

- **问题描述**：在 localhost:3000 和 localhost:8080 分别运行着前端和后端程序，在前端发送登录请求成功之后，服务器会正确发送带有 Set-Cookie 标头的响应，前端能收到 `200 OK` 、请求 body 和 header 中的其它内容，但 Set-Cookie 标头无法被浏览器自动保存
- **尝试过的解决方法**：
  - 使用 Postman 进行调试，发现 Postman 可以正确地自动保存 Cookie
  - 怀疑是 Chrome 浏览器设置的问题，在浏览器选项中开启「允许所有 Cookie」，又用 Edge 浏览器访问网页，发现问题仍然存在
  - 回滚代码到之前的版本，发现使用 jwt token 的方式登录是能成功的，自从切换到 Cookie 之后就无法登录
  - 取消 Cookie 中的 HttpOnly flag，修改 Cookie 名称（去掉下划线、横杠等字符）都不起作用


### 解决方案

- **原因分类**：跨域策略失败（same-origin policy failed）
- **详细原因**：端口不同也会被视作不同域（protocol，domain，port 三者都相同才视为 same-origin），即使设置了 CORS filter 能够接收到一般的请求体内容，但 Cookie 被视作敏感信息仍会分开储存
- **解决过程**：使用 Nginx 做反向代理将 3000 端口和 8080 端口都统一代理到 80 端口，靠 URI 区分：一开始考虑在 Docker 中使用 Nginx，但无法访问到 3000 和 8080 端口显示 `502 Bad Gateway`（存疑）；后来直接使用 Windows 上的 Nginx 服务，使用 `start nginx` 来启动（不要双击 nginx.exe 直接运行，会导致修改配置文件后 reload 无效，需要在任务管理器中手动关掉所有 nginx 进程）

### 总结反思

- **预防措施**：一次性问题，已经一劳永逸解决；今后写前后端分离业务使用 Cookie 鉴权需要首先配置好反向代理服务，防止跨域出错

- **BUG自闭程度**：高 | 一度怀疑是 Chrome 浏览器出问题了

- **BUG引起损失**：浪费时间，五天怎么也想不明白，写项目信心挺受打击



**附**：Nginx 配置文件（nginx.conf）

```nginx
events {}

http {
    server {
        listen 80;
        server_name Open Project Gateway Server;

        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $remote_addr;

        location ~ ^/api {
            rewrite  ^/api/(.*) /$1 break;
            proxy_pass http://localhost:8080;
        }

        location ~ ^/ {
            proxy_pass http://localhost:3000;
        }
    }
}
```

