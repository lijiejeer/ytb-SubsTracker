# ytb-SubsTracker

一、功能总览与关键点
系统包含三大模块：
登录与订阅列表管理页 /admin（带到期提醒、农历、分类筛选、测试通知等）
系统配置页 /admin/config（通知渠道、时区、通知时段、第三方 API Token）
YouTube 订阅页 /admin/youtube（频道订阅、轮询 RSS、KV 去重推送、日志查看）
存储
使用一个 KV 命名空间，绑定名为 SUBSCRIPTIONS_KV（存配置、订阅、YouTube 频道清单/状态/日志）
后端接口
订阅：/api/subscriptions… 系列
配置：/api/config
登录/登出：/api/login, /api/logout
第三方推送：/api/notify/{token}
YouTube：/api/yt/channels、/api/yt/cron、/api/yt/test、/api/yt/logs（新增）
YouTube 频道新增时，后端会解析频道名 feed.title 并连同 { ok, id, title } 返回前端，前端可立即提示，无需等待二次刷新
二、准备工作
Cloudflare 账号（Workers 可用）
一个 Workers 子域或自有域（用 *.workers.dev 或自定义 Route 都可）
Resend、Telegram、企业微信机器人、NotifyX、Bark 等至少一种通知渠道（可后配）
三、部署方式 A：Cloudflare Dashboard 直接部署
新建 Worker
进入 Cloudflare Dashboard → Workers & Pages → Create → “Create Worker”
将你的（index.js）全部内容粘贴覆盖保存（点 Save and deploy）
绑定 KV 命名空间
进入该 Worker → Settings → Bindings → KV Namespace → Add binding
Variable name 填：SUBSCRIPTIONS_KV
选择或新建一个命名空间（如 subs-db），保存
添加 Cron 定时任务
进入该 Worker → Triggers → Add Cron Trigger → 输入 cron 表达式（建议每 10 分钟一次）
例：*/10 * * * *
Note：定时任务会调用 scheduled() 钩子，执行订阅到期检查（以及你在代码中新增的 YouTube 巡检，如已加入）
配置 Route（可选）
如需自定义域名访问：在 Triggers → Routes 里添加你的域名路径路由（如 example.com/*），指向该 Worker
访问调试页
打开 https://你的-workers-子域/ 或 /debug
/ 会出现登录页
/debug 会显示 KV 绑定与配置读取等健康信息
四、部署方式 B：Wrangler CLI（本地/CI）
安装/登录
npm i -g wrangler
wrangler login
新建 wrangler.toml 在项目根目录创建 wrangler.toml
参考：
text


name = "subs-tracker"
main = "index.js"
compatibility_date = "2024-10-01"

[[kv_namespaces]]
binding = "SUBSCRIPTIONS_KV"
id = "<你的KV命名空间ID>"

[triggers]
crons = ["*/10 * * * *"]
发布
wrangler publish
五、首次初始化与后台配置
登录
打开 /（根路径）；初始账号密码在 KV 不存在时为 admin / password
登录后会写入 JWT cookie；如 401，检查浏览器同站策略与域名路径
系统配置 /admin/config
修改管理员用户名/密码并保存
设置时区 TIMEZONE（如 Asia/Shanghai），界面会按该时区显示和计算到期、发送时间
勾选启用的通知方式 ENABLED_NOTIFIERS 并填好对应凭据；逐一点“测试”验证通道通畅（Telegram/NotifyX/Webhook/企业微信机器人/Email/Bark）
配置通知时段 NOTIFICATION_HOURS（UTC），如“08, 12, 20”或“*”全天
如需第三方推送 API，点击“生成令牌”并保存；之后调用 /api/notify/{token}
订阅列表 /admin
添加订阅、测试通知、开启农历显示、分类筛选
YouTube 订阅 /admin/youtube
在输入框粘贴 UCID 或频道链接（支持 /channel/UC… 或 @handle 等，后端会解析 UCID）→ 点击“添加频道”
新增时后端立即返回 { ok, id, title }，并只记录最新一条为 last（不会推历史，避免刷屏）
点击“手动检测”或等待 Cron 定时检测；新视频将推送到已配置的通知渠道（通过 /api/yt/cron）
“最近日志”查看订阅更新记录（/api/yt/logs）
六、定时任务与执行频率
Cloudflare Cron 触发 scheduled()。建议 5–15 分钟一次，避免过密抓取 RSS
在代码中，YouTube 巡检函数 ytCheckAll(env) 应在 scheduled 中被调用（若你按建议添加了），否则只会跑到期提醒
注：KV 为最终一致性存储；定时批量写入后，读取可能有短暂延时是正常现象
七、第三方推送 API 用法（可选）
配置好 THIRD_PARTY_API_TOKEN 后，POST /api/notify/{token}
示例：
text


curl -X POST "https://你的域名或workers.dev/api/notify/<token>" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "外部系统告警",
    "content": "这是一条来自第三方系统的告警内容",
    "tags": ["外部系统", "重要"]
  }'
服务端会复用你在“系统配置”里启用的通知通道进行推送
八、常见排错
401 未授权：先访问 / 登录；确认浏览器未阻止 Cookie；同域路径访问 /admin、/api/* 都需要登录
KV 未绑定或名称错误：Worker Settings → Bindings 中必须将命名空间绑定为 SUBSCRIPTIONS_KV（名字要与代码一致）
Cron 未执行：检查 Triggers → Cron 是否已保存；查看 Worker Logs 是否有“定时任务触发”输出
通知收不到：
在 /admin/config 使用“测试”按钮逐个验证；
Webhook 模板/Headers JSON 是否语法正确；目标地址可用；企业微信机器人 Webhook 填写完整
YouTube 更新收不到：
在 /admin/youtube 点击“手动检测”触发 /api/yt/cron 验证；
查看 /api/yt/logs 是否记录“pushed …”；
多频道较多时 Cloudflare 边缘请求过快被限，代码中已 sleep 节流；适当放缓 Cron 频率
编码乱码（仅若你本地看到 “��” 之类乱码）：确保 index.js 保存为 UTF-8（无 BOM）
九、注意事项与配额
Workers 免费版有日请求、CPU 时间等限制；KV 有读写计费与最终一致性特性
邮件通过 Resend，域名需在 Resend 验证（邮件发不出多半是发件域未验证）
遵守当地法律与 YouTube/通知渠道使用条款；RSS 轮询不使用 API Key
十、备份与迁移（可选）
使用 wrangler kv:key list/get/put 导出/导入 KV 内容（config、subscriptions、yt_channels、yt:last:*、yt_logs 等）
迁移到新 Worker 时，保持 KV 命名空间复用或拷贝键值即可
附：接口与键位对照（便于自检）
KV 键
config：系统配置
subscriptions：订阅列表
yt_channels：[{id, title?}, …]
yt:last:UCID：每个频道的最后已推送视频 ID
yt_logs：最近 200 条日志
关键接口
/api/yt/channels GET/POST/DELETE，新频道 POST 返回 { ok, id, title }
/api/yt/cron，/api/yt/test，/api/yt/logs
/api/config，/api/subscriptions…，/api/notify/{token}
