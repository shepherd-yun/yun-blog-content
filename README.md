# yun-blog-content

[blog.shepherdyun.com](https://blog.shepherdyun.com) 的文章源。
在这里用 Markdown 写文章，push 之后 webhook 会自动同步到博客后端。

## 目录规范

```
posts/
├── java/
│   ├── _category.yml          # 分类元信息
│   ├── jvm-gc.md              # 单文件文章，slug = jvm-gc
│   └── spring-boot-3/         # 文章目录
│       ├── index.md           # 正文入口，slug = spring-boot-3
│       └── arch.png           # 随文图片，相对路径引用
└── java/spring/               # 再套一层就是二级分类
    └── aop.md
```

- 只有 `posts/` 下的 `.md` 会被同步，其他文件一概忽略。
- 目录层级即分类层级；`posts/` 根目录下的散文章归入「未分类」。
- 含 `index.md` 的目录是**文章目录**，不算分类。
- 相对路径图片会上传到 OSS 并改写链接；绝对 URL 原样保留。
  OSS key 取文件内容的 sha256，同一张图重复引用只上传一次。

## 现有分类

| 目录 | 名称 | 说明 |
|---|---|---|
| `java` | Java | Java 与 JVM 生态 |
| `js` | JavaScript | JavaScript / TypeScript 与前端 |
| `go` | Go | Go 语言与工程实践 |
| `sql` | SQL | 关系型数据库与 SQL |
| `redis` | Redis | Redis 与缓存 |
| `web3` | Web3 | 区块链与 Web3 |
| `ai` | AI | AI 与大模型应用 |
| `tools` | 工具 | 开发工具与效率 |

新增分类＝新建目录 + 写一个 `_category.yml`（省略也行，那就用目录名当分类名）。

## frontmatter

```yaml
---
title: Spring Boot 3 升级踩坑        # 必填，缺失则该篇同步失败并写入告警
slug: spring-boot-3-upgrade         # 省略则用目录名 / 文件名
tags: [Java, Spring]                 # 也可以写成单个字符串
published: true                      # 省略为 false（草稿只入库、不在前台出现）
publishedAt: 2026-08-30              # 省略则首次发布时取当前时间
excerpt: 摘要                         # 省略则从正文纯文本截取 160 字
cover: ./cover.png                   # 相对路径会一并上传 OSS
metaTitle: ...                       # 以下 SEO 字段均可省略
metaDescription: ...
metaKeywords: ...
canonicalUrl: ...
---
```

## 同步语义

| 这里发生的事 | 博客上的结果 |
|---|---|
| 新增 / 修改 `.md` | 按 slug 新建或更新 |
| 内容一字未改 | 跳过，不写库 |
| 移动到别的分类目录 | 分类跟着变，**浏览量 / 点赞 / 评论全部保留** |
| 删除 `.md` | 前台下架，数据和统计不删 |
| 两篇文章用了同一个 slug | 该篇同步失败并告警，不影响其他文章 |

单次 push 最多处理 100 篇；超出时会告警，需要在管理后台点一次「从仓库同步」补齐。

## 注意

- **slug 决定文章 URL，一旦发布就别再改**，改了等于换了一篇新文章、旧链接会失效。
- 移动目录是安全的（统计不丢），改 slug 不是。
- 删除是软下架，数据还在库里；真要彻底删除得去管理后台操作。
