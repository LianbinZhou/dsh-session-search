# dsh-session-search

给 DSH（DeepSeek Harness）的 Agent 装上「回忆」：一个 `session_search` 工具，让它能翻遍**所有历史会话**，而不只是当前这一个。

## 它解决什么

模型默认只看得到当前会话。用户说「我们上次聊过那个事」的时候，它要么猜，要么让用户重说一遍。

这个插件注册一个 `session_search` 工具，把整个会话库变成可查询的资产：

| 传参 | 模式 | 返回 |
|---|---|---|
| `query` | **search** | 跨全部会话全文检索，按相关度排序，返回命中会话 + 摘要 |
| `sessionId` | **read** | 那个会话的对话正文，逐条标注 user / assistant |
| 都不传 | **browse** | 最近的会话列表 |

## 两个刻意的设计取舍

这两条都是第一版跑出来太吵之后改的：

1. **只搜对话消息，不搜工具输出。** 工具结果里全是文件路径和目录列表，一个按主题命名的文件夹会让每个目录项都「命中」这个词，把真正的结果淹掉。
2. **摘要以命中位置居中，而不是从开头截。** 精确扫描返回的是整段文本，只显示开头的话，那条真正匹配的句子反而看不见。

## 全文索引怎么开

DSH 内置行 `session-query-sqlite` 默认是 `openAt: never` + 内存索引——`ctx.sessionQuery` 可用，但 `searchSessions` 会以 `SESSION_QUERY_SEARCH_DISABLED` 失败，工具于是退化成精确扫描。

本插件的 `cordis.patch.yml` 覆盖这一行，打开持久化索引：

```yaml
- id: session-query-sqlite
  config:
    path: 'C:\Users\<你>\.dsh\storages\session-search.db'   # ← 改成你自己的路径
    openAt: first-search
```

`first-search` 把 `node:sqlite` 的导入和索引句柄推迟到第一次搜索，DSH 启动时不会吵（最多一条 `node:sqlite` 的 ExperimentalWarning）。

索引是**派生产物，不碰会话存档本身**，删掉会自动重建。

## 安装

1. 把整个目录放到你喜欢的位置
2. 在目标 profile 的 `package.json` 里加一行依赖：

   ```json
   "@local/dsh-session-search": "link:<这个目录的绝对路径>"
   ```

   > 包名 `@local/...` 是本地约定，没有发布到 npm。想换成自己的作用域，改 `package.json` 的 `name` 和 `cordis.patch.yml` 里的 `name` 两处即可。

3. 在同一个 `package.json` 的 `dsh.profile.bundles` 数组里加上 `@local/dsh-session-search`
4. 在该 profile 目录跑一次 `pnpm install`
5. 重启 DSH

> ⚠️ 改 bundle 配置会触发 HMR 热重载。**不要在正在使用的会话里 toggle 这个 bundle**——重载窗口期内 `sessionController` 会被销毁重建，可能让该 profile 里正在跑的会话历史加载失败。要么在另一个 profile 里改，要么改完直接重启。

## 为什么零依赖

它只用 `ctx.tools.register` 和 `ctx.sessionQuery` 两个公开入口，不 import `@deepseek-ai/dsh-tools` 或任何 DSH 内部包——profile 里安装的 bundle 解析不到 harness 安装目录内部的东西。

所以整个插件就是一个 `index.js`，没有 `node_modules`，没有构建步骤，没有依赖树。

## 文件

| 文件 | 作用 |
|---|---|
| `index.js` | 全部实现：工具注册、三种模式、结果渲染 |
| `cordis.patch.yml` | 打开持久化全文索引 + 插入工具行 |
| `package.json` | 包声明与 `dsh.bundle` 入口 |

## 许可

MIT

---

[![](https://img.shields.io/badge/powered_by-dsh-4D6BFE?style=flat-square&logo=deepseek&logoColor=white)](https://github.com/deepseek-ai/deepseek-harness)
