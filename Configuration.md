# 创建精细粒度访问令牌 

不用进入任何库, 单击窗口右上角的头像即可

打开 `Github Settings`, 选择左侧栏目最下方 `Developer Settings`, 展开 `Personal access tokens` 并选择第一项 `Fine-grained tokens`.

创建一个细粒度的访问令牌, 需要填写如下内容:
- **Token name**: 标识名称, 如 `DEPLOYS_TO_ASSETS_ACCESS`
- **Description**: (可选)描述, 如 `DEPLOYS_TO_ASSETS_ACCESS, token for SecureLink to upload files to assets repo, and for deploys to fetch files from it.`
- **Owner**: 所有者, 默认就是你自己就可以, 如果你在组织里可以扩大访问范围
- **Expiration**: 有效期, 建议在 Development 阶段设置为 7 天(防止意外泄漏的风险), 等开发稳定后, 删除原来的, 再新建一个长期有效的
- **Repository access**: 令牌授权的库, 选择最后一个 **仅选中的库**, 再在下方的复选框中选择令牌目标库, 也就是 assets 库. *选中后下方会列出全部可访问的库*
- **Permissions**: 我们仅需要assets库的读写权限, 因此展开 **Repository permissions** , 然后将 **Contents** 的权限改为 **Read and write**, Metadata 的权限会自动改为 read-only.

最后点击 **Generate Token** 按钮即可创建一个令牌.

> ⚠️ NOTICE ⚠️
> 切记在关闭页面前**将密钥复制下来**, 然后仅保存到指定仓库的 Repository Secrets 中(流程如下), 不要粘贴到别处, 也不要忘记复制.

## 保存精细粒度访问令牌

我们仅将此令牌保存到 deploys 仓库的密钥中, 以供 Github Actions 访问而不会泄露.

打开 deploys 仓库, 上方的工具栏中选择 **Settings**, 展开 **Security** 区域下的 **Secrets and variables** 选单, 点击 **Actions** 以配置流水线密钥.

在第一列 **Secrets** 的 **Repository secrets** 中, 点击 **New repository secrets** 创建一个仓库密钥, 将我们刚才生成的令牌复制到下方的 `Secret *` 区, 上方的名称可填写 `DEPLOYS_TO_ASSETS_ACCESS`. 然后点击 **Add secret** 添加即可.

# 调度机制

**文件上传**: 外部仓库 \-\> 调用 `assetsHelper` 的可重用工作流 \-\> `assetsHelper` 使用专用令牌将文件写入 `assets` 仓库.  
**构建与部署**: 对 `assets` 仓库的 `push` 操作 \-\> 触发 `assets` 内部的部署工作流 \-\> 构建页面、上传构建产物 (Artifact) \-\> `actions/deploy-pages` 将产物发布到 GitHub Pages.  

我们可以通过这种方式保护 token 不被滥用, 且deploy-pages是自发完成的, 不会被外部脚本干扰

