# weChatOldExe

微信 Windows 版安装包的**自动归档仓库**。灵感与流程来自 [Rodert/wechat-win-versions](https://github.com/Rodert/wechat-win-versions)，按自己的需求重写了一版。

每天（北京时间 15:00）自动检查微信官网有没有出新版本，有就下载、算 SHA256、把安装包发到本仓库的 GitHub Releases，并重新生成 `releases.json` 供下载页读取。

## 它能做什么

- **自动追踪新版**：GitHub Actions 定时跑，无需人工干预。
- **记录真实版本号**：官网文件名可能是 `WeChatWin_4.1.9.exe`，但安装包内部其实是 `4.1.9.30`。脚本会解包读内部版本号，避免被文件名误导。
- **SHA256 留档**：每个版本都记了官方原包的 SHA256，以后能验证手上的包是不是原版。
- **下载页**：`index.html` 通过 GitHub Pages 展示版本列表、发布时间、SHA256 和下载直链。

## 相比原项目的改动

原项目**每天都会把 250MB 安装包整个下载下来**再比 SHA256，纯属浪费 Actions 流量和时间。这里加了一步 **HEAD 预检**：

1. 先发 HEAD 请求拿 `Last-Modified` + `Content-Length`；
2. 和上一次 release 记录的这两个值比对；
3. 都没变 → 直接退出，**不下那 250MB**；
4. 变了才下载，再走原来的 SHA256 去重流程。

其余逻辑（官网直链抠取、7z 解包取内部版本号、去重、发 Release）和原项目一致。

## 目录结构

```
weChatOldExe/
├── .github/workflows/
│   ├── destversion.yml   # 每日检查 + 发 Release + 重建 releases.json
│   └── pages.yml         # 把 index.html 部署到 GitHub Pages
├── scripts/
│   └── destVersionRelease.sh   # 核心脚本（仓库无关，换仓库可直接复用）
├── index.html            # 黑底绿字下载页，读 ./releases.json
├── releases.json         # 由 Actions 自动生成/更新，别手改
└── .gitignore
```

## 工作原理

```
cron 每天 UTC 07:00（北京 15:00）
  └─ scripts/destVersionRelease.sh
       1. 抓 https://pc.weixin.qq.com/ 抠 dldir1v6.qq.com 的安装包直链
       2. HEAD 预检（Last-Modified + Content-Length），没变就退出
       3. 下载 exe → sha256sum
       4. 和上一个 release 的 SHA256 比：一样 = 没新版，退出
       5. 7z 解包读内部真实版本号
       6. 同版本已存在且 SHA256 一致 → 退出
       7. 落地 exe + .sha256 说明 → gh release create / upload
  └─ github-script 重新生成 releases.json → 提交
  └─ pages.yml 把 index.html 发到 GitHub Pages
```

手动触发：仓库 **Actions → Wechat Dest Version → Run workflow**，可填一个自定义下载链接（留空则从官网自动抓）。

## 如何启用

1. 把本仓库 push 到 GitHub（见下文）。
2. **开启 Pages**：`Settings → Pages → Build and deployment → Source` 选 **GitHub Actions**。之后 `index.html` 一更新就会自动部署，地址形如 `https://feiwenqiang.github.io/weChatOldExe/`。
3. 等第一次 cron（或手动 Run workflow）跑完，Releases 里出现第一个版本，`releases.json` 被更新，下载页就有内容了。

> 不用配任何 secret：`GITHUB_TOKEN` 是 GitHub 自带的，工作流里直接用它发 Release 和推 `releases.json`。

## 注意事项

- **只往前追，不回溯**：仓库建好之后才开始记录版本。微信 4.1.6 之前的历史版本（尤其 3.x）这里拿不到，需要自己另找来源手动传。
- **存储**：每个安装包约 200~300MB，时间久了 Releases 会累积不少体积。GitHub 对公开仓库的 Release 存储没有硬性收费上限，但数量很大时建议定期清理不再需要的中间版本（保留各大版本末版即可）。
- **版权**：安装包版权归腾讯所有，本仓库仅作个人存档 / 兼容测试用途，请勿用于商业再分发。
- **依赖官网结构**：脚本靠正则抠 `dldir1v6.qq.com/weixin/Universal/Windows/*.exe`。微信如果改下载域名，抓取会失败退出（不会误发），届时来改这条正则即可。
- **无 LICENSE**：沿用原项目做法，未添加许可证。如需开放协议可自行添加。
