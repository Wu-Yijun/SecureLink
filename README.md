# 安全链接：基于 GitHub Actions 的高级私有文件发布方案 
[(English) SecureLink: Advanced Private File Publishing with GitHub Actions](./README.en.md)

## 💡 方案概述

SecureLink 是一个分层级、自动化、高效且安全的高级文件发布系统。它巧妙地利用两个独立的 GitHub 仓库——一个私有仓库（`assets`）用于存储原始数据，一个公共仓库（`deploys`）用于部署和公网访问，实现了文件内容的灵活发布与严格的权限控制 。

## ✨ 核心特性

本方案将文件访问权限划分为三个等级，并通过 GitHub Actions 实现增量构建、路径混淆和客户端加密，最终发布到 GitHub Pages，确保您的私有文件得到安全管理和发布 。

### 🛡️ 三层访问模型

* **Level 1: 完全公开 (Public)** 
    * 文件按原路径和原文件名发布，任何人都可以访问，适用于公开文档或内容.
    * `assets` 路径: `public/<repo_name>/...` 
    * `deploys` URL: `/<repo_name>/<path>/<file>` 

* **Level 2: 链接访问 (Secret)** 
    * 文件内容本身不加密，但文件名被 SHA256 哈希替换，防止目录遍历和文件名猜测。只有知道确切链接的人才能访问 。
    * `assets` 路径: `private/<repo_name>/...` 
    * `deploys` URL: `/secret/<sha256_hash>.<ext>` 

* **Level 3: 密码访问 (Encrypted)** 
    * 最高安全级别。文件内容使用 AES-256 加密后存储。访问者必须通过查看器页面并提供正确的密码才能解密和下载文件，确保即使 `deploys` 仓库被访问，没有密码也无法获取加密文件的任何信息，包括原始文件名。
    * `assets` 路径: `encrypt/<repo_name>/...` 
    * `deploys` URL: `/encrypt?file=<random_name>` 

### ⚙️ 模块化管理

* **存管分离**：`assets` 私有仓库用于存放所有原始文件，按来源仓库组织；`deploys` 公共仓库托管最终生成的网页内容，对外提供服务。
* **清单 (Manifest)**：每个来源仓库都有一个独立的清单文件 (`manifest/<repo_name>.json`)，用于追踪文件状态，实现高效增量构建 。
* **密码管理**：每个需要加密的来源仓库都有独立的密码配置文件 (`.github/secrets/<repo_name>.yaml`)，使密码管理更加清晰和安全 。

## 🚀 如何使用

1.  **准备文件**：根据所需安全级别，将文件放入 `assets` 仓库的 `public/`, `private/`, 或 `encrypt/` 下对应的项目文件夹中 。
2.  **配置密码**：如果添加了 `encrypt` 目录下的文件，请确保在 `assets` 仓库的 `.github/secrets/` 中有对应的 `<repo_name>.yaml` 密码文件 。
3.  **触发部署**：推送更新到 `assets` 仓库，或手动运行 `deploys` 仓库的 GitHub Actions 工作流 。
4.  **获取链接**：根据文件类型从 `map.json` 或直接通过 GitHub Pages URL 获取访问链接。

## ⚠️ 安全注意事项

* **客户端加密**：加解密过程完全在用户的浏览器中进行，原始文件和密码不会在网络中明文传输（用户在 URL 中主动提供密码的情况除外） 。
* **URL 中的密码**：在 URL 中直接附加 `&key=` 虽然方便，但密码会明文出现在浏览器历史记录、服务器日志中，安全性较低。仅在风险可控的情况下使用。更好的方式是让用户手动输入密码 。
