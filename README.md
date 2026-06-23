# Twikoo 评论系统的 Cloudflare 部署文档

这是 [Twikoo](https://twikoo.js.org/intro.html) 评论系统的 Cloudflare 部署方案。相比 Vercel/Netlify + MongoDB 等其他部署方式，本方案大幅改善了冷启动延迟（从 `6s` 降至 `<0.5s`）。性能提升主要得益于对 Cloudflare Workers 的大量优化，以及 HTTP 服务器与数据库（Cloudflare D1）之间的集成环境。

## 部署步骤

1. 安装 npm 依赖包：
   ```shell
   npm install
   ```

2. 由于 Cloudflare Workers 免费套餐对打包体积有严格的 1MiB 限制，需要手动删除一些包以将打包体积控制在限制范围内。由于 Cloudflare Workers 的 Node.js [兼容性问题](#已知限制)，这些包实际上无法使用。
   ```shell
   echo "" > node_modules/jsdom/lib/api.js
   echo "" > node_modules/tencentcloud-sdk-nodejs/tencentcloud/index.js
   echo "" > node_modules/nodemailer/lib/nodemailer.js
   ```

3. 登录你的 Cloudflare 账户：
   ```shell
   npx wrangler login
   ```

4. 创建 Cloudflare D1 数据库并设置表结构：
   ```shell
   npx wrangler d1 create twikoo
   ```

5. 从上一步的输出中复制 `database_name` 和 `database_id` 两行内容，粘贴到 `wrangler.toml` 文件中，替换原有的值。

6. 执行 D1 数据库表结构初始化：
   ```shell
   npx wrangler d1 execute twikoo --remote --file=./schema.sql
   ```

7. 创建 Cloudflare R2 存储桶：
   ```shell
   npx wrangler r2 bucket create twikoo
   ```

8. 将 R2 的域名更新到 `wrangler.toml` 文件中，替换 `R2_PUBLIC_URL` 的值。

9. 部署 Cloudflare Worker：
   ```shell
   npx wrangler deploy --minify
   ```

10. 如果一切顺利，你会在命令行中看到类似 `https://twikoo-cloudflare.<你的用户名>.workers.dev` 的地址。访问该地址，如果部署成功，浏览器中会显示类似以下内容：
    ```
    {"code":100,"message":"Twikoo 云函数运行正常，请参考 https://twikoo.js.org/frontend.html 完成前端的配置","version":"1.6.33"}
    ```

11. 配置前端时，将第 9 步中的地址（包含 `https://` 前缀）作为 `twikoo.init` 的 `envId` 字段值。

> 自动部署：详见[博客文章](https://blog.mingy.org/2024/12/hexo-add-twikoo/)

## 已知限制

由于 Cloudflare Workers 仅[部分兼容](https://developers.cloudflare.com/workers/runtime-apis/nodejs/) Node.js，本 Twikoo Cloudflare 部署方案存在以下功能限制：

1. 无法使用环境变量（`process.env.XXX`）控制应用行为。
2. 无法集成腾讯云。
3. 无法基于 IP 地址定位（`@imaegoo/node-ip2region` 包的兼容性问题）。
4. 由于 `jsdom` 包的兼容性问题，无法使用 `dompurify` 清理评论内容。取而代之，我们使用 [`xss`](https://www.npmjs.com/package/xss) 包进行 XSS 过滤。
5. 本部署不规范化 `/some/path/` 与 `/some/path` 之间的 URL 路径。这是因为在 Cloudflare D1 SQL 查询中统一这两种路径比较困难。如果你的网站同一页面可能同时存在带斜杠和不带斜杠的路径，可以在 `twikoo.init` 中显式设置 `path` 字段。
6. 图片上传使用了 Cloudflare R2 存储。
7. 由于使用了 [axios-cf-worker](https://github.com/wuzhengmao/axios-cf-worker)，`pushoo.js` 功能正常。

## 配置邮件通知

由于 `nodemailer` 包的兼容性问题，通过 SMTP 发送通知邮件的集成方式无法直接使用。本 Worker 通过 SendGrid 的 HTTPS API 支持邮件通知。如需启用 SendGrid 邮件集成，请按以下步骤操作：

1. 确保你拥有可用的 SendGrid 账户（SendGrid 提供免费套餐，每天可发送 100 封邮件）或 MailChannels 账户（每月免费发送 3000 封邮件），并创建一个 API 密钥。
2. 在配置中设置以下字段：
   * `SENDER_EMAIL`：发件人邮箱地址。需在 SendGrid 中验证。
   * `SENDER_NAME`：发件人显示名称。
   * `SMTP_SERVICE`：`SendGrid`。
   * `SMTP_USER`：填写任意非空值。
   * `SMTP_PASS`：API 密钥。
3. 可选：设置其他配置值以自定义通知邮件的显示样式。
4. 在配置页面点击“发送测试邮件”按钮，确保集成正常工作。
5. 在邮件服务商处，确保收到的邮件不会被归类为垃圾邮件。

