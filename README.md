# wp-rss
一个WordPress的rrs管理插件，己验证子比主题可用。代码由AI生成。
# RSS 管理（fgw-rss-manager）v1.1.1

WordPress 插件：一键开关控制系统 RSS / Atom 订阅源，可拦截、关闭、重定向全部或部分端点，作用所有主题，后台开关即时生效。

## 变更记录

### v1.1.1（修复致命缺陷）
- **致命修复**：`intercept_feed` / `intercept_format` 原先对**非 feed 请求**也会执行拦截，导致关闭总开关后全站普通页面（首页、文章页）被重定向或 404。现仅在当前请求确为 feed 端点时介入，抽出统一 `deny()` 处理。
- **致命修复**：后台 `save_settings` / `ajax_toggle` 中 `self::OPT` 为未定义常量（`FGWRS_Admin` 无 OPT），进后台即 PHP 致命错误。改用 `FGWRS_Core::OPT`。
- **严重修复**：后台资源加载 hook 名误写为 `toplevel_page_`，而菜单挂在「设置」下（`settings_page_`），导致后台 CSS/JS 永不加载、开关点击无效。已修正。
- **文档修复**：README 中的 option 名统一为代码实际值 `fgwrss_settings`（此前误写 `fgw_rss_settings`）。

### v1.1.0（对齐子比 zibll 真实端点）
- 修复 404 分支死代码：「不重定向、仅 404」选项现在真正输出 404 状态码 + HTML
- 修复输出格式开关形同虚设：新增 `intercept_format` + `current_format`，关闭某格式后请求被真正拦截
- 端点标签改为真实路由写法（/comments/feed/、/date/2026/09/feed、?author=1&feed=rss2 等）
- 新增论坛帖 CPT Feed 识别（`?post_type=forum_post` → `feed_forum`）
- 新增搜索 Feed 识别（`?s=xx&feed=rss2` → `feed_search`）

## 端点清单（默认全开）

| 端点 | 说明 | 对应开关 |
|------|------|----------|
| /feed/、/feed | 主 Feed | feed_posts |
| /feed/rss2/、/feed/atom/ | RSS 2.0 / Atom | feed_posts + 格式开关 |
| ?feed=rss2、?feed=atom、?feed=rss0.92 | 参数形式 | feed_posts |
| /rss | 短入口（301 → /feed） | feed_posts |
| /394.html/feed/ | 单文章 Feed | feed_single |
| /feed/?post_type=forum_post | 论坛帖 CPT Feed | feed_forum |
| /category/jc/feed/、/category/uncategorized/feed/ | 分类 Feed（站点分类仅 jc 与 uncategorized；「加密工具」是标签非分类） | feed_category |
| ?tag=html&feed=rss2 | 标签 Feed | feed_tag |
| ?author=1&feed=rss2 | 作者 Feed | feed_author |
| /comments/feed/ | 评论 Feed | feed_comments |
| /date/2026/09/feed/、?m=202609&feed=rss2 | 日期归档 Feed | feed_archive |
| /?s=xx&feed=rss2 | 搜索 Feed（注：子比搜索页带 feed 参数会报 400，可借此拦截） | feed_search |
| /wp-sitemap.xml、/wp-json/... | XML 站点地图、REST API | 不受本插件控制 |

## 开关选项（后台 · 设置 · RSS 管理）

- 总控：一键启用 / 禁用全站 RSS
- 每类端点独立开关（可重定向到首页或仅 404）
- 输出格式：RSS 2.0 / Atom / RDF（RSS 0.92），可分别禁用
- 前端行为：页头 `<link rel="alternate">` 自动发现标签、禁用端点的响应方式（重定向 / 404）
- 管理员访问被禁用的 feed 仍可见，便于排查；仅对访客生效

## 安装

1. 上传整个 `fgw-rss-manager/` 目录到 `wp-content/plugins/`，或直接安装 zip
2. 在后台启用插件
3. 进入「设置 → RSS 管理」配置开关；切换即时生效并自动保存
4. 卸载安全：`uninstall.php` 会清除插件数据（option `fgwrss_settings`）

## 结构

```
fgw-rss-manager/
├── fgw-rss-manager.php          # 主文件：常量定义、加载
├── uninstall.php                # 卸载清理
├── includes/
│   ├── class-fgwrs-core.php     # 核心逻辑：端点识别、拦截、重定向、格式控制
│   └── class-fgwrs-admin.php    # 后台 UI、Ajax 即时开关
└── assets/
    ├── admin.js                 # 前端交互（开关联动）
    └── admin.css                # 后台样式
```

## 技术要点

- 识别端点：`detect_feed_type()` 基于 `is_feed()` + `$_GET`（feed、feed_type、post_type、s）并辅以条件标签（is_single/is_category/is_tag/is_author/is_date/is_comment_feed），覆盖子比 zibll 全部端点
- 拦截时机：`template_redirect`（intercept_feed priority 5、intercept_format priority 6）；先确认是 feed 请求再判定，非 feed 请求零干预
- 格式控制：`current_format()` 解析 `?feed_type` / `?feed` / PATH_INFO（/feed/<fmt>/），映射为 rss/atom/rdf
- 性能：常量表匹配，零额外数据库查询；开关存 option `fgwrss_settings`，AJAX 直接 `update_option`，无需刷新
- 兼容：所有主题 / 插件共存；仅依赖 WP 核心 API；PHP 7.4+ / WP 5.0+
- 安全：后台操作校验 nonce（`fgwrs_nonce`）；设置键白名单校验
