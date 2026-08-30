---
title: 内容仓库示例文章
slug: hello-content-sync
tags: [示例, 内容同步]
published: false
excerpt: 这篇是用来验证同步管线的示例，确认无误后可以直接删掉。
---

这篇文章用来验证「push → webhook → 入库 → 前台展示」整条链路。
`published` 现在是 `false`，所以它只会入库、不会出现在前台。
把它改成 `true` 再 push，就能在前台看到。

## 支持的写法

行内 `code`、**加粗**、*斜体*、[链接](https://blog.shepherdyun.com)。

### 代码高亮

代码块会用 Prism 渲染成 `.token.*` 类名，和前台 `globals.css` 里的主题配套：

```js
const greet = (name) => `hello, ${name}`
```

```java
public record Post(String slug, String title) {}
```

### 表格

| 字段 | 是否必填 | 说明 |
|---|---|---|
| `title` | 是 | 缺失则该篇同步失败并写入告警 |
| `slug` | 否 | 省略时取目录名 / 文件名 |
| `published` | 否 | 省略为 `false` |

### 图片

同目录下的图片用相对路径引用，同步时会自动上传到 OSS 并改写链接：

```markdown
![示意图](./diagram.png)
```

## 两种文章形态

- **单文件**：`posts/go/context-basics.md`，slug 取文件名 `context-basics`
- **文章目录**：`posts/go/context-basics/index.md`，slug 取目录名，随文图片放同目录

需要二级分类就再套一层目录，比如 `posts/java/spring/xxx.md`，
分类会自动串成「Java → Spring」的父子关系。


test
