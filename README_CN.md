# aistudio-api 的 Render 部署适配文件

这个压缩包是**适配补丁/部署配置**，不是上游项目源码。它保留上游仓库的 Dockerfile、Web 管理台、Cookie 导入流程和签名子进程，不重写业务逻辑。

上游仓库：<https://github.com/DeconstructedCube/aistudio-api>

## 选择部署模式

- 免费测试：使用 `render.yaml`。没有持久磁盘，服务空闲 15 分钟会休眠；实例重启或重新部署时，`/app/data` 中的 Cookie、账号配置和统计文件可能丢失，需要重新导入。
- 长期使用：将 `render-paid.yaml` 改名为 `render.yaml` 后再提交。它使用当前最低付费 Web Service 方案和 1GB 持久磁盘，Cookie 与账号数据保存在 `/app/data`。按 Render 当前公开价格估算，计算资源约 US$7/月，1GB 磁盘约 US$0.25/月；实际账单以 Render 控制台为准。

## 部署步骤

1. 在 GitHub 打开上游仓库，点击 **Fork**，将仓库复制到你自己的 GitHub 账号。不要把 Cookie、`data/`、`.env` 或任何账号凭据提交到 GitHub。
2. 下载本适配包，将选定的 `render.yaml` 放到你 Fork 仓库的根目录（与 `Dockerfile`、`main.py` 同级）。免费模式直接用本包里的 `render.yaml`；持久化模式则把 `render-paid.yaml` 改名为 `render.yaml`。
3. 将文件提交并推送到你的 Fork 仓库。
4. 登录 <https://dashboard.render.com/>，选择 **New → Blueprint**，连接你的 GitHub，选择刚才 Fork 的仓库。
5. Render 读取 `render.yaml` 后会要求填写 `AISTUDIO_WEB_PASSWORD`。设置一个强随机密码（建议至少 32 个随机字符），不要使用示例密码，也不要把它写进仓库。
6. 点击部署。首次 Docker 构建会编译上游内置的 WAA signer，构建时间可能比普通 Python 网页应用长。完成后在 Render 的 Events/Logs 页面确认服务启动成功。
7. 打开 Render 分配的 `https://你的服务名.onrender.com/health`。能得到健康检查响应后，再打开 `https://你的服务名.onrender.com/` 进入管理后台，并用 `AISTUDIO_WEB_PASSWORD` 登录。
8. **先设置 API Key，再导入 Cookie**：进入管理后台的「系统设置」→「API Key 管理」，创建一枚独立 API Key。上游默认 `api_keys: []`，在没有配置 API Key 时，Gemini API 路由会免密开放；`AISTUDIO_WEB_PASSWORD` 只负责管理后台登录，不能替代 API Key。创建 API Key 后再继续。
9. 在后台的账号管理页导入你自己的 AI Studio Cookie。Cookie 等同于登录会话凭据，只能通过你自己的 HTTPS 管理后台录入；不要发给他人或放进 GitHub。
10. SillyTavern 等客户端选择 **Google Gemini** 协议：
   - API Base URL：`https://你的服务名.onrender.com`
   - 如果客户端强制要求版本路径：`https://你的服务名.onrender.com/v1beta`
   - API Key：填写刚创建的 API Key。

## 本适配处理了什么

- 上游 Dockerfile 的默认启动命令把端口写死为 `8080`。Render Web Service 默认接收 `10000` 端口，因此 Blueprint 覆盖容器命令为 `python3 main.py server --port 10000`。
- 服务仍由上游应用绑定 `0.0.0.0`，并继续使用它自己的 `/health` 健康检查。
- Cookie/账号数据、配置、统计和轮换状态统一指向 `/app/data`。
- 管理后台密码通过 Render Secret 环境变量注入，不写进仓库。

## 重要限制和安全提醒

- Render 免费服务会在 15 分钟无流量后休眠，唤醒通常需要约一分钟；免费服务可能随时重启，且不支持持久磁盘。免费模式适合试运行，不适合作为稳定的长期反代。
- 上游把 Cookie、账号注册信息、配置及状态写到 `/app/data`。没有持久磁盘时，这些本地文件无法保证跨重启保留。要保留 Cookie，使用付费模式并挂载 `/app/data` 持久磁盘。
- Render 分配的域名是公网地址。务必设置强 `AISTUDIO_WEB_PASSWORD` 保护管理后台。
- **API Key 与管理密码是两回事**。上游默认 `api_keys: []` 时 API 路由免密开放，因此部署后应立即登录后台，在「系统设置」→「API Key 管理」创建 API Key，再导入 Cookie、连接客户端。创建 API Key 之前不要把服务地址发给任何人，也不要先导入 Cookie。不要把 API Key 截图公开或交给他人；若怀疑泄露，在后台撤销并重新创建。
- 不要启用请求报文转储（`AISTUDIO_DUMP_REQUESTS` / `AISTUDIO_DUMP_FILE`），除非确实在排障且确认日志不会暴露隐私。
- 使用网页 Cookie 方式可能受 Google 登录会话、账号安全策略、服务条款及上游网页接口变化影响；部署成功不等于 Google 保证兼容或持续可用。

## 常见排错

- **Port scan / no open ports**：确认 Blueprint 中 `PORT` 与 `AISTUDIO_PORT` 都是 `10000`，且 `dockerCommand` 没被删掉。
- **健康检查失败**：查看 Logs，确认最后运行命令是 `python3 main.py server --port 10000`，再访问 `/health`。
- **后台登录提示鉴权失败**：检查 Render 环境变量 `AISTUDIO_WEB_PASSWORD` 是否已设置，保存后重新部署。
- **重启后账号列表为空**：免费模式没有持久磁盘；需要重新导入 Cookie，或改用带 `/app/data` 磁盘的付费模式。
- **构建失败**：检查上游仓库最近提交是否修改了 Dockerfile、Go/Zig 构建依赖或 WAA signer 构建脚本；本适配文件只负责 Render 端口和数据目录配置。
