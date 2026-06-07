# 企业网络工程师学习手册

这是一份面向企业网络工程师的系统学习文档，目标是帮助学习者逐步具备以下能力：

- 理解企业网络的基础通信原理。
- 熟练配置交换机、路由器、防火墙等常见网络设备。
- 能够规划 VLAN、IP 地址、路由、安全策略和出口网络。
- 能够设计中小企业、园区网、数据中心、总部-分支互联等典型网络架构。
- 能够按照工程流程完成调研、设计、实施、割接、验收和运维。

## 浏览方式

- 总目录：[DIRECTORY.md](DIRECTORY.md)
- 章节文件：[`chapters/`](chapters/)
- 附录文件：[`appendices/`](appendices/)
- 模板文件：[`templates/`](templates/)

## mdBook 部署

本手册支持使用 mdBook 构建为静态站点。章节中使用了 Mermaid 图，因此需要同时安装 `mdbook-mermaid`。

### 安装依赖

```sh
mdbook --version
mdbook-mermaid --version
```

如果尚未安装 `mdbook-mermaid`，可参考项目说明安装：

```sh
cargo install mdbook-mermaid
```

如果部署环境没有 Cargo，也可以从 `mdbook-mermaid` 的 GitHub Releases 下载对应平台的预编译二进制，并确保 `mdbook-mermaid` 在 `PATH` 中。

`mdbook-mermaid` 项目地址：

```text
https://github.com/badboy/mdbook-mermaid
```

### 初始化 Mermaid 资源

首次使用或升级 `mdbook-mermaid` 后，在本目录执行：

```sh
mdbook-mermaid install .
```

该命令会确认 `book.toml` 中的 Mermaid 预处理器配置，并复制 `mermaid.min.js`、`mermaid-init.js` 到当前目录。

### 本地预览

```sh
mdbook serve
```

默认预览地址通常为：

```text
http://localhost:3000
```

### 构建静态站点

```sh
mdbook build
```

构建产物输出到：

```text
book/
```

部署到 GitHub Pages、Nginx、对象存储或其他静态站点平台时，发布 `book/` 目录即可。

### GitHub Pages 自动部署

仓库已提供 GitHub Actions 工作流：

```text
.github/workflows/deploy-pages.yml
```

该工作流会在推送到 `main` 或 `master` 分支时自动：

1. 安装 mdBook 和 `mdbook-mermaid`。
2. 运行 `mdbook-mermaid install enterprise-network-engineer-guide`。
3. 构建 `enterprise-network-engineer-guide/book/`。
4. 将构建产物发布到 GitHub Pages。

在 GitHub 仓库中需要确认：

```text
Settings -> Pages -> Build and deployment -> Source -> GitHub Actions
```

也可以在 Actions 页面手动运行 `Deploy mdBook to GitHub Pages` 工作流。

## 推荐学习顺序

1. 先学习第 1 部分和第 2 部分，建立网络通信基础。
2. 再学习交换机、路由、防火墙，掌握设备配置能力。
3. 然后学习企业网络架构设计，把零散技术组织成完整方案。
4. 最后通过综合实验和项目文档模板训练工程交付能力。

## 实验建议

建议至少准备一种实验环境：

- 华为方向：eNSP 或支持 VRP 的实验环境。
- H3C 方向：HCL。
- Cisco 方向：Packet Tracer、GNS3 或 EVE-NG。
- 综合实验：优先使用 EVE-NG 或 GNS3，便于混合交换机、防火墙和服务器系统。

学习时不要只看命令，要同时理解三个问题：

- 为什么要这么设计？
- 这条配置改变了设备的什么行为？
- 出故障时应该从哪里验证？
