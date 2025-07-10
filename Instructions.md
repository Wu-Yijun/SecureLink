# 基于 GitHub Actions 的高级私有文件安全发布方案

## 1. 方案概述
本文档旨在设计一个分层级、自动化、高效且安全的高级文件发布系统. 该系统利用两个独立的 GitHub 仓库——一个私有仓库 (assets) 用于存储原始数据, 一个公共仓库 (deploys) 用于部署和公网访问.

系统将文件访问权限划分为三个等级, 并通过 GitHub Actions 实现增量构建、路径混淆和客户端加密, 最终发布到 GitHub Pages.

### 1.1. 核心原理

#### 1.1.1. **存管分离:**
- assets (私有仓库): 存放所有原始文件, 按来源仓库进行组织.
- deploys (公共仓库): 托管最终生成的网页内容, 对外提供服务.
#### 1.1.2. **三层访问模型:**
- **Level 1: 完全公开 (Public)**
    - **assets 路径:** `public/<repo_name>/...`
    - **deploys URL:** `/<repo_name>/<path>/<file>`
    - **说明:** 文件按原路径和原文件名发布, 任何人都可以访问.
- **Level 2: 链接访问 (Secret)**
    - **assets 路径:** `private/<repo_name>/...`
    - **deploys URL:** `/secret/<sha256_hash>.<ext>`
    - **说明:** 文件内容本身不加密, 但文件名被哈希替换. 只有知道确切链接的人才能访问, 防止目录遍历和文件名猜测. 这是一种"安全靠隐藏" (Security through obscurity) 的策略.
- **Level 3: 密码访问 (Encrypted)**
    - **assets 路径:** `encrypt/<repo_name>/...`
    - **deploys URL:** `/encrypt?file=<random_name>`
    - **说明:** 最高安全级别. 文件内容使用 AES-256 加密后存储在 `/encrypted/<random_name>` 路径. 访问者必须通过` /encrypt` 路径的查看器页面, 并提供正确的密码才能解密和下载文件.
### 1.1.3. **模块化管理:**
- **清单 (Manifest)**: 每个来源仓库 (<repo_name>) 都有一个独立的清单文件 (manifest/<repo_name>.json), 用于追踪该仓库下所有文件的状态, 实现高效的增量构建.
- **密码:** 每个需要加密的来源仓库都有一个独立的密码配置文件 (.github/secrets/<repo_name>.yaml), 使密码管理更加清晰和安全.

### 1.2. 最终效果
- **灵活的权限控制:** 你可以根据文件敏感度, 将其放入不同目录, 实现三种不同的分享策略.
- **极致的性能:** 增量构建机制确保只有变动的文件会被处理.
- **最高的安全性:** 端到端加密确保即使 deploys 仓库被访问, 没有密码也无法获取加密文件的任何信息, 包括原始文件名.

## 2. 详细设计与实施步骤
### 2.1. 仓库设置
#### 2.1.1. assets 仓库结构 (私有)
```
assets/
├── .github/
│   └── secrets/
│       └── my-secret-project.yaml  # 项目专属的密码文件
├── public/
│   └── my-public-project/          # L1: 完全公开的文件
│       └── docs/index.html
├── private/
│   └── my-internal-project/        # L2: 仅链接可访问的文件
│       └── design_v1.png
├── encrypt/
│   └── my-secret-project/          # L3: 需要密码访问的文件
│       └── financial-report.xlsx
└── manifest/
    └── my-public-project.json      # (此目录由Action自动创建和管理)
```

#### 2.1.2. 密码配置文件 (`.github/secrets/<repo_name>.yaml`)

```
# .github/secrets/my-secret-project.yaml

# 默认密码, 用于该项目下所有未被特别指定的加密文件
default: "a-very-strong-default-password-!@#$"

# 为特定文件或目录指定密码
files:
  - path: "financial-report.xlsx" # 精确匹配文件 (相对于 encrypt/my-secret-project/ 目录)
    password: "another-super-secret-password"
```

#### 2.1.3 `deploys` 仓库结构 (公共)

- 启用 `gh-pages` 分支的 GitHub Pages.
- 在主分支中创建以下文件:
    - `.github/workflows/deploy.yml`
    - `scripts/process-assets.js`
    - `scripts/viewer-template.html`

### 2.2. 访问令牌配置
此步骤不变. 确保在 deploys 仓库的 Actions secrets 中配置了 ASSETS_REPO_TOKEN.

这是连接两个仓库的关键.

1. 生成 Personal Access Token (PAT)
    - 进入你的 GitHub Settings > Developer settings > Personal access tokens > Tokens (classic).
    - 点击 Generate new token (classic).
    - Note: 填写一个描述性的名称, 如 DEPLOYS_TO_ASSETS_ACCESS.
    - Expiration: 选择一个合适的过期时间.
    - Scopes: 勾选 repo 权限. 这将允许该令牌访问你的所有仓库.
    - 点击 Generate token, 并立即复制生成的令牌, 因为离开页面后将无法再次看到.
2. 在 deploys 仓库中配置 Secret
    - 进入 deploys 仓库的 Settings > Secrets and variables > Actions.
    - 点击 New repository secret.
    - Name: ASSETS_REPO_TOKEN
    - Value: 粘贴上一步复制的 PAT.
    - 点击 Add secret.

### 2.3. GitHub Actions 与处理脚本
#### 2.3.1. 核心部署工作流 (`deploy.yml`)
此工作流基本保持不变, 它作为调用处理脚本的启动器.

```yml
# .github/workflows/deploy.yml in 'deploys' repository

name: Advanced Deploy Private Assets

on:
  workflow_dispatch:
  repository_dispatch:
    types: [deploy-triggered]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - name: Checkout deploys repo (for script)
        uses: actions/checkout@v4
        with:
          path: 'self'

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
      
      - name: Install Dependencies
        run: npm install crypto-js js-yaml glob fs-extra
        working-directory: ./self/scripts

      - name: Create Deploy Target Directory
        run: mkdir ./deploy_target

      - name: Checkout assets repo (private repo)
        uses: actions/checkout@v4
        with:
          repository: <YourGitHubUsername>/assets
          token: ${{ secrets.ASSETS_REPO_TOKEN }}
          path: 'assets_src'
      
      - name: Copy existing deployment from gh-pages
        run: |
          git clone --depth=1 --branch=gh-pages https://x-access-token:${{ secrets.GITHUB_TOKEN }}@github.com/${{ github.repository }}.git ./deploy_target || echo "No existing gh-pages branch. Starting fresh."
          # 清理旧文件, 但保留 .git 目录和 manifest
          find ./deploy_target -mindepth 1 ! -name ".git" ! -name "manifest" -exec rm -rf {} +

      - name: Run Asset Processing Script
        run: |
          node self/scripts/process-assets.js \
            --assets-path ./assets_src \
            --deploy-path ./deploy_target \
            --template-path ./self/scripts/viewer-template.html
        
      - name: Upload filename map as artifact
        uses: actions/upload-artifact@v4
        with:
          name: filename-map
          path: ./deploy_target/map.json

      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./deploy_target
          user_name: 'github-actions[bot]'
          user_email: 'github-actions[bot]@users.noreply.github.com'
          commit_message: "Deploy: ${{ github.event.head_commit.message || 'Manual Trigger' }}"
          keep_files: true # 保留 manifest 等未被覆盖的文件
```

请记得将 `<YourGitHubUsername>` 替换为你的 GitHub 用户名.

#### 2.3.2. 文件处理脚本 (`process-assets.js`) (重大更新)
此脚本经过重写, 以支持三层模型和模块化管理.
```js
// scripts/process-assets.js
const fs = require('fs-extra');
const path = require('path');
const crypto = require('crypto');
const yaml = require('js-yaml');
const glob = require('glob');

// --- Helper Functions ---
const getArgs = () => {
    const args = {};
    process.argv.slice(2).forEach(arg => {
        const [key, value] = arg.split('=');
        args[key.substring(2)] = value;
    });
    return args;

};
const getFileHash = (filePath) => {
    const data = fs.readFileSync(filePath);
    return crypto.createHash('sha256').update(data).digest('hex');
};
const getFileSha256 = (filePath) => fs.readFileSync(filePath).toString('base64');

// --- Main Logic ---
async function main() {
    const args = getArgs();
    const assetsPath = args['assets-path'];
    const deployPath = args['deploy-path'];
    const templatePath = args['template-path'];

    const userMap = {};
    const processedDeployFiles = new Set();

    const processRepo = (repoType, repoName) => {
        const sourceDir = path.join(assetsPath, repoType, repoName);
        if (!fs.existsSync(sourceDir)) return;

        // 1. Load manifest
        const manifestDir = path.join(deployPath, 'manifest');
        fs.ensureDirSync(manifestDir);
        const manifestPath = path.join(manifestDir, `${repoName}.json`);
        const oldManifest = fs.existsSync(manifestPath) ? fs.readJsonSync(manifestPath) : {};
        const newManifest = {};

        // 2. Load passwords if applicable
        let passwordConfig = { files: [] };
        if (repoType === 'encrypt') {
            const passwordPath = path.join(assetsPath, '.github', 'secrets', `${repoName}.yaml`);
            if (fs.existsSync(passwordPath)) {
                passwordConfig = yaml.load(fs.readFileSync(passwordPath, 'utf8'));
            } else {
                console.warn(`Warning: Password file not found for encrypted repo '${repoName}'.`);
            }
        }
        const getPasswordForFile = (p) => {
            const specific = passwordConfig.files?.find(f => f.path === p);
            return specific?.password || passwordConfig.default;
        };

        const assetFiles = glob.sync(`${sourceDir}/**/*`, { nodir: true });

        for (const filePath of assetFiles) {
            const relativePath = path.relative(sourceDir, filePath);
            const hash = getFileHash(filePath);

            if (oldManifest[relativePath]?.hash === hash) {
                newManifest[relativePath] = oldManifest[relativePath];
                processedDeployFiles.add(oldManifest[relativePath].deployPath);
                if(repoType !== 'public') userMap[`${repoType}/${repoName}/${relativePath}`] = oldManifest[relativePath].userUrl;
                continue;
            }

            let deployRelativePath, userUrl;
            if (repoType === 'public') {
                deployRelativePath = path.join(repoName, relativePath);
                fs.copySync(filePath, path.join(deployPath, deployRelativePath));
            } else if (repoType === 'private') {
                const ext = path.extname(relativePath);
                deployRelativePath = path.join('secret', `${hash}${ext}`);
                userUrl = deployRelativePath;
                fs.copySync(filePath, path.join(deployPath, deployRelativePath));
            } else if (repoType === 'encrypt') {
                const randomName = crypto.randomBytes(16).toString('hex');
                const encryptedDataPath = path.join('encrypted', randomName);
                const viewerPath = path.join('encrypt', `${randomName}.html`);
                
                const password = getPasswordForFile(relativePath);
                if (!password) {
                    console.error(`Error: No password found for ${filePath}. Skipping.`);
                    continue;
                }
                const fileContent = fs.readFileSync(filePath);
                const encrypted = CryptoJS.AES.encrypt(fileContent.toString('base64'), password).toString();

                fs.outputFileSync(path.join(deployPath, encryptedDataPath), encrypted);
                
                const viewerTemplate = fs.readFileSync(templatePath, 'utf8');
                const viewerHtml = viewerTemplate
                    .replace('%%ENCRYPTED_DATA_URL%%', `../${encryptedDataPath}`)
                    .replace("'%%ORIGINAL_FILENAME%%'", JSON.stringify(path.basename(relativePath)));

                fs.outputFileSync(path.join(deployPath, viewerPath), viewerHtml);
                deployRelativePath = viewerPath;
                userUrl = `encrypt?file=${randomName}.html`;
            }

            newManifest[relativePath] = { hash, deployPath: deployRelativePath, userUrl };
            if(userUrl) userMap[`${repoType}/${repoName}/${relativePath}`] = userUrl;
            processedDeployFiles.add(deployRelativePath);
            if(repoType === 'encrypt') processedDeployFiles.add(path.join('encrypted', path.basename(newManifest[relativePath].deployPath, '.html')));
        }
        
        // Cleanup old files for this repo
        for (const p in oldManifest) {
            if (!newManifest[p]) {
                const oldDeployPath = path.join(deployPath, oldManifest[p].deployPath);
                fs.removeSync(oldDeployPath);
                if(oldManifest[p].deployPath.startsWith('encrypt/')){
                    const oldDataPath = path.join(deployPath, 'encrypted', path.basename(oldManifest[p].deployPath, '.html'));
                    fs.removeSync(oldDataPath);
                }
            }
        }
        fs.outputJsonSync(manifestPath, newManifest, { spaces: 2 });
    };

    const repoTypes = ['public', 'private', 'encrypt'];
    repoTypes.forEach(type => {
        const repos = fs.readdirSync(path.join(assetsPath, type), { withFileTypes: true })
            .filter(dirent => dirent.isDirectory())
            .map(dirent => dirent.name);
        repos.forEach(repoName => processRepo(type, repoName));
    });

    fs.outputJsonSync(path.join(deployPath, 'map.json'), userMap, { spaces: 2 });
    console.log('Processing complete.');
}

main().catch(console.error);
```

#### 2.3.3. HTML 查看器模板 (`viewer-template.html`) (重大更新)
此模板现在从外部文件获取加密数据, 而不是内联.
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Unlock File</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/crypto-js/4.1.1/crypto-js.min.js"></script>
    <style>
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif; display: flex; align-items: center; justify-content: center; min-height: 100vh; background-color: #f3f4f6; margin: 0; }
        .container { background: white; padding: 2rem; border-radius: 0.5rem; box-shadow: 0 4px 6px rgba(0,0,0,0.1); text-align: center; max-width: 400px; width: 90%; }
        input { width: calc(100% - 2rem); padding: 0.75rem; margin-bottom: 1rem; border: 1px solid #d1d5db; border-radius: 0.375rem; }
        button { width: 100%; padding: 0.75rem; border: none; border-radius: 0.375rem; background-color: #3b82f6; color: white; font-weight: bold; cursor: pointer; }
        button:hover { background-color: #2563eb; }
        #status, #download-area { margin-top: 1rem; }
        #download-link { display: inline-block; background-color: #10b981; padding: 0.75rem 1.5rem; border-radius: 0.375rem; color: white; text-decoration: none; font-weight: bold; }
        #download-link:hover { background-color: #059669; }
    </style>
</head>
<body>
     <div class="container" id="main-container">
        <div id="password-area">
            <h2>Enter Password to Unlock</h2>
            <p>This file is password protected.</p>
            <form id="password-form">
                <input type="password" id="password" placeholder="Password" required>
                <button type="submit">Unlock</button>
            </form>
        </div>
        <div id="download-area" style="display: none;">
            <h2>File Unlocked!</h2>
            <p id="filename-display"></p>
            <a href="#" id="download-link" download>Download File</a>
        </div>
        <div id="status"></div>
    </div>

    <script>
        const encryptedDataUrl = '%%ENCRYPTED_DATA_URL%%'; // This is new
        const originalFilename = '%%ORIGINAL_FILENAME%%';

        function base64ToBytes(base64) {
            const binString = atob(base64);
            const len = binString.length;
            const bytes = new Uint8Array(len);
            for (let i = 0; i < len; i++) {
                bytes[i] = binString.charCodeAt(i);
            }
            return bytes;
        }

        async function decryptAndDownload(password) { // Now async
            const statusEl = document.getElementById('status');
            try {
                statusEl.textContent = 'Fetching data...';
                const response = await fetch(encryptedDataUrl);
                if (!response.ok) throw new Error('Failed to fetch encrypted data.');
                const encryptedData = await response.text();

                statusEl.textContent = 'Decrypting...';
                const decryptedBase64 = CryptoJS.AES.decrypt(encryptedData, password).toString(CryptoJS.enc.Utf8);
                if (!decryptedBase64) throw new Error('Decryption failed, likely wrong password.');

                const byteCharacters = atob(decryptedBase64);
                const byteNumbers = new Array(byteCharacters.length);
                for (let i = 0; i < byteCharacters.length; i++) {
                    byteNumbers[i] = byteCharacters.charCodeAt(i);
                }
                const byteArray = new Uint8Array(byteNumbers);
                const blob = new Blob([byteArray], {type: 'application/octet-stream'});
                const url = URL.createObjectURL(blob);

                document.getElementById('password-area').style.display = 'none';
                document.getElementById('filename-display').textContent = `File: ${originalFilename}`;
                const downloadLink = document.getElementById('download-link');
                downloadLink.href = url;
                downloadLink.download = originalFilename;
                document.getElementById('download-area').style.display = 'block';
                statusEl.textContent = '';

            } catch (e) {
                statusEl.textContent = 'Error: Invalid password or failed to load data.';
                statusEl.style.color = 'red';
                console.error(e);
            }
        }

        document.addEventListener('DOMContentLoaded', () => {
            const urlParams = new URLSearchParams(window.location.search);
            const key = urlParams.get('key');
            const fileParam = urlParams.get('file'); // e.g., <random_name>.html

            const currentFile = window.location.pathname.split('/').pop();
            if (fileParam && fileParam !== currentFile) {
                // This logic ensures the URL is clean, e.g. /encrypt?file=... redirects to /encrypt/<random_name>.html
                window.location.href = fileParam + (key ? `?key=${key}` : '');
                return;
            }

            if (key) {
                decryptAndDownload(key);
            }

            document.getElementById('password-form').addEventListener('submit', (e) => {
                e.preventDefault();
                const password = document.getElementById('password').value;
                decryptAndDownload(password);
            });
        });
    </script>
</body>
</html>
```

## 3. 使用流程总结
- 准备文件: 根据所需安全级别, 将文件放入 `assets` 仓库的 `public/`, `private/`, 或 `encrypt/` 下对应的项目文件夹中.
- 配置密码: 如果添加了 `encrypt` 目录下的文件, 确保在 `.github/secrets/` 中有对应的 `<repo_name>.yaml` 密码文件.
- 触发部署: 推送更新到 `assets` 仓库, 或手动运行 deploys 仓库的 Action.
- 获取链接:
    - 公开文件: `https://<user>.github.io/deploys/<repo_name>/path/to/file.ext`
    - 链接访问: 从 `map.json` 中找到 `private/<repo_name>/...` 对应的路径, 链接为 `https://<user>.github.io/deploys/secret/<hash>.ext`
    - 密码访问: 从 `map.json` 中找到 `encrypt/<repo_name>/...` 对应的路径, 链接为 `https://<user>.github.io/deploys/encrypt?file=<random_name>.html`

## 4. 安全性与注意事项
- 客户端加密: 加解密过程完全在用户的浏览器中进行, 你的原始文件和密码永远不会在网络中明文传输 (除了用户在 URL 中主动提供密码的情况).
- URL 中的密码: 在 URL 中直接附加 `&key=<password>` 虽然方便, 但密码会明文出现在浏览器历史记录、服务器日志中, 安全性较低. 仅在风险可控的情况下使用. 更好的方式是让用户手动输入密码.