# 技术文章

收集和撰写技术文章，记录学习与思考。

## 目录结构

```
technical-articles/
├── articles/          # 文章目录
│   ├── ai/           # AI / LLM 相关
│   ├── backend/      # 后端开发
│   ├── frontend/     # 前端开发
│   ├── devops/       # DevOps / 运维
│   └── tools/        # 工具使用
└── templates/         # 文章模板
```

## 文章列表

### AI / LLM
- [AI Agent 长期记忆管理实现](./articles/ai/AI-Agent-长期记忆管理实现.md) - 探讨 AI Agent 如何管理对话历史和上下文记忆
- [Agent 工具权限管理设计](./articles/ai/Agent-Tool-Permission-Design.md) - 介绍 Agent 工具权限管理架构的设计与实现
- [jojo-code 核心架构解析](./articles/ai/jojo-code-核心架构解析.md) - 解析 jojo-code 的整体架构设计和核心原理

### 后端开发
- [ ] 待添加

### 前端开发
- [ ] 待添加

### DevOps
- [ ] 待添加

### 工具
- [ ] 待添加

## 文章格式

每篇文章遵循以下结构：

```markdown
# 标题

> 一句话摘要

## 背景

## 正文

## 总结

## 参考资料
```

## 在线访问

本项目使用 GitHub Pages + Docsify 部署，无需构建，直接渲染 Markdown 文件。

**访问地址**: https://afine907.github.io/technical-articles/

### 本地预览（可选）

如果你想在本地预览，可以启动一个简单的 HTTP 服务器：

```bash
cd docs
python3 -m http.server 3000
# 访问 http://localhost:3000
```

或者使用 Docsify CLI：

```bash
npm i -g docsify-cli
docsify serve docs
```

## License

MIT
