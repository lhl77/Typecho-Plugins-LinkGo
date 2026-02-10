# LinkGo 插件 — 使用与主题开发文档

这是 LinkGo（Typecho 外部链接中间跳转）插件的说明文档。本文档侧重于主题（跳转页模板）开发，帮助你定制跳转页样式与行为。

## 目标读者
- Typecho 主题/插件开发者
- 希望自定义跳转页面样式或新增主题的管理员

## 目录
- 简介
- 安装与启用
- 插件配置项说明
- 路由与 Action 说明
- 主题目录结构与约定
- 模板变量（模板中可用）
- 示例主题（最小化 template.php + style.css）
- 调试与常见问题
- 链接与支持

## 简介
LinkGo 会把站外链接替换为站内的中间跳转页面，防止用户误点并能展示安全提示。插件会在内容渲染阶段替换外链，也在部分主题无法拦截时通过 `afterRender` 或输出缓冲做兜底替换。

跳转页模板位于插件目录：
`LinkGo/page/themes/<ThemeName>/template.php` 与 `style.css`。

## 安装与启用
1. 把 `LinkGo` 文件夹放到 Typecho 的 `usr/plugins/` 目录下（或你项目的相应插件目录）。
2. 在 Typecho 管理后台 -> 插件，启用 LinkGo。
3. 启用后插件会尝试注册 `/go/[target]` 路由（不同 Typecho 版本可能差异），启用失败请手动刷新插件或在 Typecho 后台禁用再启用一次。

## 插件配置项说明
在后台插件配置页（`插件 -> LinkGo -> 配置`）可以设置：
- 跳转页站点标题（`siteTitle`）
- Logo 图片 URL（`logoUrl`）
- 起始年份（`startYear`，页脚）
- 跳转页主题（从 `page/themes` 子目录自动扫描）
- 外部链接是否在新标签打开（`openInNewTab`）

配置页包含一个卡片式的说明头部，并且在底部给出仓库与支持文档链接。

## 路由与 Action
- 插件注册的 Action 类：`LinkGo_Action`（位于 `LinkGo/LinkGo/Action.php`，请参考源码）。
- 路由格式：`/go/[target]`，其中 `target` 是 URL-safe base64 编码（`+`/`/` -> `-`/`_`，去掉尾部 `=`）。
- Action 会解析请求、恢复 base64 填充并 decode，最后验证为合法 URL（仅 http/https）。

如果你的站点无法使用路径形式，也可以通过查询参数 `?target=` 调用（插件同时支持多种来源解析）。

## 主题目录结构与约定
在插件目录下放置主题：
```
LinkGo/
  page/
    themes/
      Default/
        template.php
        style.css
      MyTheme/
        template.php
        style.css
```

约定：
- `template.php`：必须输出完整的跳转页 HTML（或至少渲染主体），并读取由 Action 或 loader 提供的变量。通常包含 `<head>`、样式、主体与页脚。插件也内置了一个简单 loader (`page/index.php`) 可把通用变量注入到模板。
- `style.css`：主题样式，应使用相对选择器限定在主题容器内，避免污染站点其他风格。

## 模板文件中可用的变量
在 `template.php` 中，以下变量通常可用（由插件 Action 或 loader 提供）：
- `$title` — 跳转页标题（通常为 `LinkGo` 或自定义）
- `$siteTitle` — 站点显示标题（来自配置）
- `$logoUrl` — 配置的 Logo URL
- `$url` — 解码后的目标 URL（完整）
- `$display_url` — 用于前端显示的简短目标（如去掉协议或做截断）
- `$displayYear` — 页脚要显示的年份（从 `startYear` 到当前年）
- `$themeName` — 当前主题名

注意：若你直接包含 `template.php`（例如本地测试），请确保这些变量存在或为其设置默认值，以防 PHP Notice。

## 推荐模板结构（示例摘录）
下面给出最小化的 `template.php` 示例（仅示意，实际模板应更完整并包含样式与无障碍属性）：

```php
<?php
// ... 由 Action 传入或自行准备变量
$title = $title ?? 'LinkGo';
$siteTitle = $siteTitle ?? '本站';
$logoUrl = $logoUrl ?? '';
$url = $url ?? '';
$displayYear = $displayYear ?? date('Y');
?>
<!doctype html>
<html lang="zh-CN">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <title><?php echo htmlspecialchars($title, ENT_QUOTES); ?> - <?php echo htmlspecialchars($siteTitle, ENT_QUOTES); ?></title>
  <link rel="stylesheet" href="<?php echo rtrim($siteUrl, '/'); ?>/usr/plugins/LinkGo/page/themes/<?php echo rawurlencode($themeName); ?>/style.css">
</head>
<body>
  <main>
    <h1>您即将前往</h1>
    <p><?php echo htmlspecialchars($url, ENT_QUOTES); ?></p>
    <a href="<?php echo htmlspecialchars($url, ENT_QUOTES); ?>">继续访问</a>
  </main>
  <footer>&copy; <?php echo htmlspecialchars($displayYear, ENT_QUOTES); ?> <?php echo htmlspecialchars($siteTitle, ENT_QUOTES); ?></footer>
</body>
</html>
```

示例 `style.css`（基础）

```css
/* 限定作用域 */
.lg-wrapper { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'PingFang SC', 'Hiragino Sans GB', 'Microsoft Yahei', sans-serif; }
.lg-wrapper .btn-primary { background: #3b82c4; color: #fff; padding: 10px 16px; border-radius: 10px; text-decoration: none; display:inline-block; }
```

## 可选：使用图标与外部资源
- 模板可加载 Font Awesome 或使用内联 SVG 图标；若你希望主题不依赖外网资源，建议把必要的 SVG 放在主题目录并内联。

## 无障碍与安全建议
- Continue 按钮应含有 `rel="noopener noreferrer"` 并在需要时使用 `target="_blank"`。
- 对用户展示的目标 URL 做截断展示并把完整 URL 放到 `title` 属性或可复制的输入框中，便于验证。

## 调试建议
- 插件在模板中会有少量 debug 输出（`console.log` 注入），如果看不到 `target` 参数，检查：
  - 是否正确启用插件并重新启用以注册路由；
  - Web 服务器是否把请求的 query/path 转发到 PHP（部分 rewrite 规则可能丢弃 query）；
  - 在 `Action.php` 中查看 `error_log()` 输出（插件在开发时会写入调试日志）。

## 常见问题
- 路由注册失败：尝试在后台禁用/启用插件；确认 Typecho 版本是否支持 `Helper::addRoute`。
- 模板变量为空：确保 Action/loader 正确包含模板（不要直接在 web 根目录单独调用 theme 文件而不通过插件 Action）。

## 贡献与支持
- 项目仓库： https://github.com/lhl77/Typecho-Plugins-LinkGo
- 使用文档（作者博客）： https://blog.lhl.one/artical/949.html

欢迎提交 Issue 或 Pull Request，主题模板与样式也非常欢迎贡献示例主题。

---

如果你希望，我可以：
- 在 `page/themes/` 下帮你生成一个新的示例主题目录（含完整 `template.php` 与 `style.css`），用于快速开始；
- 或者把 README 中的示例扩展为更详细的“主题 API”表格与调试步骤。告诉我你希望哪个方向。
