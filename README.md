# autional-cn/cdn

`cdn.autional.cn` 的内容源。**这个仓的 `ui/` 目录是生成物，不要手工编辑。**

## 它是什么

设计系统的**运行期资产**。存在的理由只有三条，每条都对应 npm 覆盖不到的场景：

1. **Go 服务 demo 不是 Node 构建**，装不了 npm 包，只能靠 `<link href="https://cdn...">` 消费设计系统。
2. **站点图标（favicon / logo）在此之前完全没有交付通道** —— 14 个门户各自手工放置，已经漂移。
3. **字体走同一个 URL 才能跨站命中缓存** —— 14 个门户只下载一次，而不是 14 次。

## 它不是什么

**不是构建期代码的分发通道。** `tokens/index.js`、`tailwind-preset`、`antd-theme`、React 组件
必须在构建时被 bundler 解析，CDN 的 `<script>` 语义解决不了这个问题。那些走 npm：`@autional-cn/*`。

## 铁律

1. **路径版本化且不可变。** `/ui/v<version>/` 下的内容是永久缓存（`max-age=31536000, immutable`）。
   内容一变就换版本号。**不要**原地改一个已发布版本的文件 —— 全世界的缓存都不会知道。
2. **跟随最新版要读 `/ui/latest.json`**（`max-age=300`）。它是唯一的可变指针。
3. **CORS 必须保持 `*`。** 跨域 `@font-face` 缺 `Access-Control-Allow-Origin` 会**静默回退**到系统字体，
   症状是「字体偶尔不对」这种极难排查的问题。`vercel.json` 里那条全局头不能删。
4. **字体要用 `crossorigin`。** `<link rel="preload" as="font" ... crossorigin>`，否则预加载会失效并重复下载。

## 怎么更新

```bash
# 在 autional-cn/ui 仓里
pnpm build:cdn            # 默认输出到 ../cdn
cd ../cdn && git add -A && git commit -m "..." && git push
```

推送到 `main` 即触发 Vercel 生产部署。

## 结构

```
ui/latest.json               当前版本指针（短 TTL）
ui/v<version>/manifest.json  每个文件的 path / bytes / sha384
ui/v<version>/tokens.css     预编译令牌（含 @font-face）
ui/v<version>/primitives.css
ui/v<version>/fonts/
ui/v<version>/icons/         favicon 套件 + 生成的 site.webmanifest
ui/v<version>/logo/
```

`site.webmanifest` 是**生成**的而非拷贝来的：原始那份用相对路径，搬到 CDN 上就是死链；
且它的 `theme_color` 取自设计系统令牌，不是手填的字面量。

## 已知遗留：白色填充路径（U66 #12）

逐个查过 `assets/logo` 与 `assets/favicon` 的全部变体，各自含有的填充色是：

| 文件 | 填充色 | 白底可用？ |
|---|---|---|
| `logo/logo-mark.svg` | `#93BFDE` `#FFFFFF` `#223A58` `#E8B440` | ⚠️ 白色那笔会隐形 |
| `logo/logo-mark.color.svg` | 同上 | ⚠️ 同上 |
| `logo/logo-mark.dark.svg` | `#93BFDE` `#FFFFFF` `#E8B440` | ⚠️ **同样含白色**，它是给深色底用的 |
| `logo/logo-mark.mono-white.svg` | `#FFFFFF` | ❌ 白底完全不可见（本就不该用于白底） |
| `logo/logo-mark.mono-black.svg` | `#000000` | ✅ 唯一在白底上确定安全的变体 |
| `icons/favicon.svg` | `#0A2B47` `#93BFDE` `#FFFFFF` `#223A58` `#E8B440` | ⚠️ 同 logo-mark |

**容易搞错的一点**：`logo-mark.dark.svg` 是「深色主题版本」（给深底用），不是「深色描边版本」，
它同样含白色填充，**不能**当作白底方案。

所以当前没有「白底彩色」变体。要修的话有两条路，都需要设计决策：
① 把白色那一笔替换成一个在白底上可见的颜色，产出 `logo-mark.on-light.svg`；
② 或者确认白色描边本就是为深色底设计的，白底一律用 `mono-black`。

在定下来之前，**白底场景请用 `mono-black`**。见 `autional-cn/ui` 的 `docs/logo-favicon-kit.md`。
