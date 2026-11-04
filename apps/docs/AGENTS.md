# Geistdocs 智能体说明

本项目使用打包的 Geistdocs 架构。`@vercel/geistdocs` 包负责共享运行时行为；本应用负责本地内容、配置、适配器和站点特定路由。

AI 编码智能体编辑本项目时，请遵循以下说明。

## 架构

- 运行时功能来自 `@vercel/geistdocs`，包括文档页面渲染器、布局辅助函数、MDX 组件、搜索、Ask AI、Markdown 路由、代理辅助函数和源辅助函数。
- Geistdocs 通过文档化的 `@vercel/geistdocs/components/*` 路径暴露共享 UI 原语。模板代码应基于包，不要直接导入提供方内部实现。
- `@vercel/geistdocs` 负责 Ask AI 客户端、服务端路由行为和 AI SDK v6 运行时依赖。不要为了适配旧版应用级 `ai` 版本而 Fork 包内部实现。
- 本地文件是用户自有的适配器，应保持精简，并调用 `@vercel/geistdocs/*` 的公开导出。
- 不要为了做定制而把包内部实现复制进应用。优先配置适配器文件或升级 `@vercel/geistdocs`。
- 不要从 `@vercel/geistdocs/dist` 深层导入，也不要编辑 `node_modules/@vercel/geistdocs` 中的文件。
- 不要编辑生成目录，如 `.source/`、`.next/`、`node_modules/` 或构建产物。

## 面向智能体的包文档

- 包 API 行为不明确时，先阅读 `node_modules/@vercel/geistdocs/docs` 中已安装的包文档，不要猜测。
- 从 `node_modules/@vercel/geistdocs/docs/agents.md` 和 `node_modules/@vercel/geistdocs/docs/sitemap.md` 开始，定位相关的专题页面。
- 任务特定指引见 `node_modules/@vercel/geistdocs/docs/pages/*.md`；仅在需要包的整体背景时使用 `node_modules/@vercel/geistdocs/docs/llms.txt`。
- 这些包文档是只读的生成产物。不要编辑 `node_modules/@vercel/geistdocs` 下的文件；应修改本地适配器文件或更新包。

## 常见编辑目标

| 任务 | 编辑文件 |
| --- | --- |
| 配置站点标题、Logo、导航、GitHub 链接、AI 提示词、检索、建议、翻译、`basePath` 或 `siteId` | `geistdocs.tsx` |
| 新增或更新文档页面 | `content/docs/**/*.mdx` |
| 控制侧边栏顺序、分组和文件夹标签 | `content/docs/meta.json` |
| 为页面设置更短的导航标签 | 在页面 frontmatter 中设置 `navTitle` |
| 覆盖 MDX 组件 | `components/geistdocs/mdx-components.tsx` |
| 包裹站点 Provider、分析或全局客户端行为 | `components/geistdocs/provider.tsx` |
| 自定义文档布局外壳 | `components/geistdocs/docs-layout.tsx` |
| 配置 Fumadocs 源适配器或版本化文档 | `lib/geistdocs/source.ts` |
| 配置 Fumadocs 集合和源安全的 MDX 处理 | `source.config.ts` |
| 配置文档页面渲染器 | `app/[lang]/docs/[[...slug]]/page.tsx` |
| 配置 AI 可读的 Markdown 输出和恢复 | `app/[lang]/desktop/llms.txt/route.ts`、`app/[lang]/desktop/llms.mdx/[[...slug]]/route.ts`、`app/[lang]/desktop/sitemap.md/route.ts` |
| 配置聊天或搜索 API | `app/api/chat/route.ts`、`app/api/search/route.ts`、`app/api/search/export/route.ts` |
| 在 Geistdocs 路由前后添加请求处理 | `proxy.ts` |
| 编辑营销首页 | `app/[lang]/(home)/**` |
| 编辑共享样式 | `app/global.css`、`app/styles/geistdocs.css` |

## 内容规范

- 文档放在 `content/docs`，除非项目在 `lib/geistdocs/source.ts` 中添加了其他源。
- 每个新页面都要加入 `content/docs/meta.json`，以便显示在侧边栏。
- 文档页面使用 MDX frontmatter，至少包含 `title` 和 `description`。仅当导航标签需要与页面标题不同时，才设置 `navTitle`。
- 保持 slug 稳定，除非任务明确包含重定向或链接更新。
- 添加翻译内容时，遵循现有的语言区域后缀模式，如 `page.cn.mdx`。
- 页面需要给读者提供可复制到编码智能体的提示词时，使用 `CopyPrompt`。

## 路由与代理规范

- App Router 路由文件应保持为围绕包辅助函数的精简适配器，如 `createDocsPage`、`createChatRoute`、`createLlmsRoute` 和 `createProxy`。
- `next.config.ts` 的包装器保持为 `@vercel/geistdocs/next` 的 `createGeistdocs`。它组合 Fumadocs MDX 并生成 `createProxy` 使用的应用路由清单；不要直接用 `createMDX` 替换它。
- `next.config.ts` 中保留 `cacheComponents: true` 和 `partialPrefetching: true`。不要从 App Router 页面或路由处理器导出 `dynamic`、`revalidate` 或 `fetchCache`。
- 在 Server Component 中从 `next/root-params` 读取 `[lang]`。在 Route Handler 和 Server Action 中保留路由上下文 `params`。
- 应用自有的、指向完全静态文档页面的链接使用 `prefetch={true}`，避免导航停在通用路由外壳。
- `proxy.ts` 中的 `export const config` 保持为静态对象。Next.js 必须在构建时解析代理匹配器。
- 代理匹配器排除项只匹配 `/api` 和 `/api/...`，如 `api(?:/|$)`。不要排除 `api` 这类宽泛前缀，否则也会排除 `/api-reference` 等路由。
- 除非任务明确更改 AI 可读输出，否则保留 Markdown 协商。Geistdocs 为 `.md`、`.mdx` 和 `Accept: text/markdown` 请求提供 `/desktop/llms.txt` 和逐页 Markdown。
- 新增、删除或重命名 App Router 页面或路由后，重启 `next dev`，让 `createGeistdocs` 重新生成路由清单。
- 添加自定义代理行为时，优先使用 `createProxy` 的 `before`、`after`、`transformRewrite` 和 `markdownRoutes` 选项，不要替换代理。`transformRewrite` 用于必须在语言区域和 Markdown 解析之后应用的内部路由段。
- 根路径挂载的文档，或首页/应用路由与文档路由共存的站点，使用显式 `markdownRoutes`。
- 设置 `config.basePath` 时，源 URL、导航链接、`getPageUrl` 和 `markdownRoutes` 保持应用本地。Geistdocs 会单独推导公开的页面操作和 Markdown URL。
- 请求特定的页面访问使用 `createDocsPage({ canViewPage })` 及其返回的 `getPageTree(params)`。逐页 Markdown 使用一个同步的公开过滤器作为 `canViewPage`，llms.txt、sitemap、搜索、搜索导出和聊天路由使用 `filterPage`。
- Next.js base-path 应用在应用根路径提供文档时，在静态代理匹配器中显式包含 `"/"`。

## Ask AI 与 Vertex 代理规范

- 不设置 `GEISTDOCS_CHAT_PROXY_URL` 且不配置 `ai.eveAgent`，即使用默认 AI Gateway 路径。该模式下，`app/api/chat/route.ts` 调用 `createChatRoute` 时不传 `proxy` 选项，并在 AI SDK `streamText` 循环中使用本地文档搜索工具。
- 设置 `ai.retrieval: "mixedbread"` 和唯一的 `siteId`，让 Ask AI 使用 Mixedbread 语义检索。可见的搜索对话框仍使用本地 Orama 搜索。
- 从本消费站点仓库根目录创建 Mixedbread Store，此时 Vercel CLI 已链接到本站点的 Vercel 项目。切勿在 Geistdocs 包仓库或其他消费仓库中运行 `vercel integration add mixedbread`。
- `MXBAI_API_KEY` 和 `MXBAI_STORE_ID` 放在环境变量中。切勿放入 `geistdocs.tsx`，因为配置对客户端组件可见。
- 启用 Mixedbread 检索时，保留 `app/api/search/export/route.ts` 和 `next build && geistdocs search sync` 构建脚本。生产构建会增量同步聊天可见的 Markdown；本地和预览构建跳过远程同步，除非传入 `--allow-non-production`。
- Mixedbread 失败时回退到本地 Orama 检索。保持本地源定义和 `excludeFrom: [chat]` 可见性行为不变。
- 在 `geistdocs.tsx` 中设置 `ai.eveAgent: { url }`，改用托管的 eve 框架智能体回答 Ask AI。URL 通过配置对象传递，路由文件无需更改。eve 0.59+ 响应器在站点应于聊天客户端加载时准备空持久会话的情况下，设置 `prewarm: true`。请求默认使用每请求的 Vercel OIDC bearer token 认证；自定义认证通过 `createChatRoute` 的 `eveAgent` 选项传入仅服务端使用的请求头。切勿把认证材料放入 `geistdocs.tsx`。同时配置 `proxy` 和 eve 智能体时，路由创建会抛错。
- eve 智能体模式下，设置仅服务端使用的 `GEISTDOCS_CHAT_SECRET`（至少 32 字节，用 `openssl rand -base64 32` 生成）。它用于签名 httpOnly 的 `geistdocs_eve_session` cookie，将浏览器绑定到其 eve 会话（该 cookie 为 `SameSite=Strict`，因此聊天路由需与文档页面同站）；不设置时每轮都会新建 eve 会话，预准备请求会被拒绝。切勿加 `NEXT_PUBLIC_` 前缀或放入 `geistdocs.tsx`。
- `createChatRoute` 限制请求大小（默认 1,000,000 字节请求体、200 条消息、`pageContext.content` 200,000 字符；可用 `limits` 选项覆盖）。它不做速率限制，需用站点自身的速率限制包裹 `POST`。
- Geistdocs Ask AI 面向 AI SDK v6：`ai` v6 和 `@ai-sdk/react` v3。除非 `@vercel/geistdocs` 新版本更改，否则这些依赖保持生成的包版本。
- 应用在 Geistdocs 之外的产品代码使用 `ai` 或 `@ai-sdk/react` 时，单独迁移该应用代码，或让包管理器安装不同版本。不要为了匹配无关应用代码而降级 Geistdocs Ask AI。
- 仅当 Ask AI 应通过中央 Vertex 代理路由时，设置 `GEISTDOCS_CHAT_PROXY_URL`。值必须包含 `/vertex` 路由，如 `https://<geistdocs-platform-deployment>/vertex`。
- 不要在 Geistdocs 站点中添加 Vertex 凭据。中央平台代理在 `x-vercel-trusted-oidc-idp-token` 中转发 Vercel OIDC token；Vertex 部署应通过 Deployment Protection Trusted Sources 信任平台 Vercel 项目。
- 使用 Geistdocs 平台 `/vertex` 代理时，设置仅服务端使用的 `GEISTDOCS_CHAT_PROXY_TOKEN`。其值必须与平台部署的 `GEISTDOCS_VERTEX_PROXY_TOKEN` 一致，后者至少 32 字节。切勿用 `NEXT_PUBLIC_` 前缀暴露任一 token。
- `app/api/chat/route.ts` 保持为围绕 `createChatRoute` 的精简适配器。优先配置 `GEISTDOCS_CHAT_PROXY_URL` 和 `GEISTDOCS_CHAT_PROXY_TOKEN`，不要 Fork 包聊天路由。
- 自定义聊天客户端代码使用 `DefaultChatTransport.prepareSendMessagesRequest` 时，添加 `currentRoute` 等字段需在返回的 `body` 中保留 `messages`。返回自定义 `body` 会替换 AI SDK 默认请求体。

## 迁移规范

- 从 Fumadocs 或自定义 Geist 文档站点迁移时，先盘点 `source.config.ts`、路由文件、`middleware.ts` 或 `proxy.ts`、`public/llms.txt`、OG 路由、Tailwind CSS 配置和必需的环境变量，再开始编辑。
- 公开仓库使用 `@vercel/geistdocs`。仅在可访问受限 `@vercel/geistcn` 和 `@vercel/geistcn-assets` 包的内部环境中使用 `@vercel/geistdocs-private`。
- 私有 npm 凭据不要放入源文件；通过本地 npm 认证、Vercel 项目设置和 GitHub Actions secrets 配置。
- 盘点应用直接使用 `ai` 和 `@ai-sdk/react` 的情况。包自有的 Ask AI 使用 AI SDK v6；本地 AI SDK 代码需与 Geistdocs 路由适配器分开迁移。
- 在 `source.config.ts` 中从 `@vercel/geistdocs/source-config` 导入源配置辅助函数。不要在源配置中导入运行时组件入口点。
- 将现有 `middleware.ts` 行为移入 `createProxy({ before })` 或 `createProxy({ after })` 钩子。
- 使用 `createLlmsRoute` 时删除 `public/llms.txt`，否则静态文件会遮蔽 App Router 路由。
- 仅当应用包含 Geistdocs OG 路由时，才在 `createDocsPage` 中设置 `openGraph.images`，否则覆盖元数据以避免 `/og/...` URL 失效。
- 保留生成的包特定 Next.js 配置。私有模板包含其所需的额外 Geistcn 转译设置。
- 将 `geistFontClasses`、`tailwind` 和 `tailwind-preflight` 应用到根 `<html>` 元素。
- 导入所选 Geistdocs 包的样式表，并保留其生成的 Tailwind 源配置。
- 为仅生产环境变量添加本地回退值，使迁移构建不依赖生产密钥。

## 包更新

- 使用 `pnpm exec geistdocs update` 更新基于包的 Geistdocs 项目。
- `geistdocs update` 更新 `@vercel/geistdocs` 依赖，不会覆盖本地适配器文件。
- 在 canary 项目上，`geistdocs update` 跟随当前 `canary` dist-tag。更新活跃迁移前，先检查并测试该版本。
- 提交更新前，审查依赖变更并运行验证命令。

## 命令

- 启动开发：`pnpm dev`
- 生产构建：`pnpm build`
- 启动构建后的应用：`pnpm start`
- 安装依赖后重新生成 Fumadocs 输出：`pnpm postinstall`
- 更新 Geistdocs：`pnpm exec geistdocs update`
- 如已配置翻译，运行翻译：`pnpm translate`
- 明确需要时，在生产环境外同步 Mixedbread Store：`pnpm exec geistdocs search sync --allow-non-production`

## 验证

- 更改路由、配置、源设置、MDX 组件或包版本后，运行 `pnpm build`。
- 视觉布局、导航或 MDX 渲染有变化时，运行 `pnpm dev` 并打开更改的页面。
- 更改内容路由或代理行为时，检查 `/desktop` 和 AI 可读路由，如 `/desktop/llms.txt` 或页面级 `.md` URL。
- 更改 Mixedbread 检索或源配置后，确认 `/api/search/export` 为静态生成。
- 确认源文件中未添加密钥。本地值使用 `.env.local`，并使其不进入 Git。

<!-- BEGIN:nextjs-agent-rules -->

# 这不是你熟悉的 Next.js

此版本有破坏性变更——API、约定和文件结构都可能与你的训练数据不同。写任何代码前，先阅读 `node_modules/next/dist/docs/` 中的相关指南（从本文件所在目录解析；在 monorepo 中，`next` 包可能从仓库根目录不可见）。注意弃用通知。

此块由 `next dev` 写入并重新添加——可在 `node_modules/next/dist/server/lib/generate-agent-files.js` 验证。仅从 diff 中删除它只会重新产生未提交的更改；随工作一起提交可保持工作树干净。

<!-- END:nextjs-agent-rules -->