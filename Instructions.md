# **基于 GitHub Actions 的高级私有文件安全发布方案**

## **1. 方案概述**

本文档旨在设计一个**最高安全级别**的、分层级、自动化的文件发布系统. 该方案的核心是利用一个私有仓库 (`assets`) 直接构建和发布 GitHub Pages, 同时通过一个公共的"助手"仓库 (`assetsHelper`) 来安全地管理文件上传入口, 从而确保包括源码、文件结构和访问令牌在内的所有敏感信息都保持私有.

### **1.1. 核心原理**

1. **架构: 彻底的私有化部署**  
    * **`assets` (私有仓库)**: 系统的绝对核心. 它不仅存储所有原始文件, 还负责运行构建脚本、管理状态清单 (Manifest), 并通过 `actions/deploy-pages` 直接将构建产物发布到 GitHub Pages. 整个部署流程在私有环境中闭环完成.  
    * **`assetsHelper` (公共仓库)**: 扮演一个安全的"代理"或"跳板机". 它本身不存储任何敏感数据, 仅包含一个可重用的 GitHub Actions 工作流. 其唯一作用是接收来自其他仓库的文件上传请求, 并使用一个具有 `assets` 仓库写入权限的令牌, 安全地将文件存入 `assets` 仓库.  
    * **外部项目仓库 (调用方)**: 任何需要发布文件的项目, 只需调用 `assetsHelper` 的工作流即可, 无需直接接触 `assets` 仓库或其令牌.  
2. **工作流**  
    * **文件上传**: 外部仓库 \-\> 调用 `assetsHelper` 的可重用工作流 \-\> `assetsHelper` 使用专用令牌将文件写入 `assets` 仓库.  
    * **构建与部署**: 对 `assets` 仓库的 `push` 操作 \-\> 触发 `assets` 内部的部署工作流 \-\> 构建页面、上传构建产物 (Artifact) \-\> `actions/deploy-pages` 将产物发布到 GitHub Pages.  
3. **三层访问模型 (逻辑不变)**  
    - **Level 1: 完全公开 (Public)**
        * **assets 路径:** `public/<repo_name>/...`
        * **deploy URL:** `/<repo_name>/<path>/<file>`
        * **说明:** 文件按原路径和原文件名发布, 任何人都可以访问.
    - **Level 2: 链接访问 (Secret)**
        - **assets 路径:** `private/<repo_name>/...`
        - **deploy URL:** `/secret/<sha256_hash>.<ext>`
        - **说明:** 文件内容本身不加密, 但文件名被哈希替换. 只有知道确切链接的人才能访问, 防止目录遍历和文件名猜测. 这是一种"安全靠隐藏" (Security through obscurity) 的策略.
    - **Level 3: 密码访问 (Encrypted)**
        - **assets 路径:** `encrypt/<repo_name>/...`
        - **deploy URL:** `/encrypt?file=<random_name>`
        - **说明:** 最高安全级别. 文件内容使用 AES-256 加密后存储在 `/encrypted/<random_name>` 路径. 访问者必须通过` /encrypt` 路径的查看器页面, 并提供正确的密码才能解密和下载文件.

### **1.1.3. 模块化管理:**
- **清单 (Manifest)**: 每个来源仓库 (`<repo_name>`) 都有一个独立的清单文件 (`manifest/<repo_name>.json`), 用于追踪该仓库下所有文件的状态, 实现高效的增量构建.
- **密码:** 每个需要加密的来源仓库都有一个独立的密码配置文件 (`.github/secrets/<repo_name>.yaml`), 使密码管理更加清晰和安全.

### **1.2. 最终效果**

- **极致的安全**: 由于不再有公开的 `deploys` 仓库, 攻击者无法通过克隆 `gh-pages` 分支来窥探你的文件结构. 你的源文件、哈希文件名、加密文件等都存储在私有仓库中, 不会泄露.  
- **令牌隔离**: 具有高权限的 `ASSETS_REPO_TOKEN` 被严格限制在 `assetsHelper` 仓库的 Secrets 中, 其他项目仓库无法接触到它, 大大降低了令牌滥用的风险.  
- **灵活的权限控制:** 你可以根据文件敏感度, 将其放入不同目录, 实现三种不同的分享策略.
- **极致的性能:** 增量构建机制确保只有变动的文件会被处理.

## **2. 详细设计与实施步骤**

### **2.1. 仓库设置**

1. **assets 仓库 (私有)**  
    - **结构**:  
      ```yml
      assets/ (私有)  
      ├── .github/  
      │   ├── secrets/  
      │   │   └── my-secret-project.yaml  # 项目专属的密码文件  
      │   └── workflows/  
      │       └── deploy.yml              # 核心部署工作流  
      ├── public/                         # 公开路径
      ├── private/                        # 链接访问路径
      ├── encrypt/                        # 加密文件路径
      ├── secrets/                        # 链接访问文件存储
      ├── encrypted/                      # 加密文件存储
      ├── manifest/                       # 存储状态清单, 由Action自动管理  
      └── scripts/  
          ├── process-assets.js           # 核心处理脚本  
          └── package.json                # 脚本依赖
      ```
    - **Settings > Pages**: 在 `Build and deployment` 部分, 将 Source 设置为 `GitHub Actions`.  
2. **assetsHelper 仓库 (公共)**  
    - **结构**:  
      ```yml
      assetsHelper/ (公共)  
      └── .github/  
          └── workflows/  
              └── upload.yml              # 可重用的上传工作流
      ```

    - **Settings > Secrets and variables > Actions**:  
      - 点击 `New repository secret`.  
      - **Name**: `ASSETS_REPO_TOKEN`  
      - **Value**: 粘贴一个拥有对 `assets` 仓库 `write` 权限的 Personal Access Token (PAT).  
3. **外部项目仓库 (例如 `my-project`)**  
   * 无需配置任何 Secret.  
   * 仅需一个调用 `assetsHelper` 工作流的 Action 文件.

### **2.2. GitHub Actions 与处理脚本**

#### **2.2.1. `assetsHelper` 的可重用上传工作流 (`upload.yml`) **

这个工作流是外部世界与私有 `assets` 仓库沟通的唯一桥梁, 已加入目标路径安全校验.
```yml
# assetsHelper/.github/workflows/upload.yml

name: Reusable Asset Uploader

on:
  workflow_call:
    inputs:
      source:
        description: |
          (from_repo=false)Source artifact name or
          (from_repo=true)The directory from the source repository to sync
        required: true
        type: string
      from_repo:
        description: 'Whether to use the source from the calling repo (true/false)'
        required: false
        default: false
        type: boolean
      type:
        description: 'Asset type (public, private, encrypt)'
        required: false
        default: public
        type: string
      destination:
        description: 'The target subdirectory in assets repo (e.g., {type}/<repo_name>/{destination})'
        required: false
        default: ./
        type: string
      
jobs:
  sync-to-assets:
    runs-on: ubuntu-latest
    steps:
      - name: Validate destination path and type
        run: |
          DEST="${{ inputs.destination }}"
          TYPE="${{ inputs.type }}"

          # Validate type
          if [[ "$TYPE" != "public" && "$TYPE" != "private" && "$TYPE" != "encrypt" ]]; then
            echo "❌ Error: type must be one of 'public', 'private', or 'encrypt'"
            exit 1
          fi

          # Check for path traversal
          if [[ "$DEST" == *".."* ]]; then
            echo "❌ Error: destination must not contain '..'"
            exit 1
          fi

          echo "✅ Destination and type are valid"

      - name: Compute full destination path
        id: path
        run: |
          DEST_PATH="${{ inputs.type }}/${{ github.event.repository.name }}/${{ inputs.destination }}"
          echo "dest_path=$DEST_PATH" >> "$GITHUB_OUTPUT"

      - name: Checkout source repository (the calling repo) If needed
        if: ${{ inputs.from_repo == 'true' }}
        uses: actions/checkout@v4
        with:
          repository: ${{ github.event.repository.full_name }}
          path: source_repo

      - name: Extract artifact If not from repo
        if: ${{ inputs.from_repo == 'false' }}
        uses: actions/download-artifact@v4
        with:
          name: ${{ inputs.source }}
          repository: ${{ github.event.repository.full_name }}
          path: source_repo

      - name: Checkout assets repository
        uses: actions/checkout@v4
        with:
          repository: Wu-Yijun/assets # TODO: 替换为你的用户名
          token: ${{ secrets.ASSETS_REPO_TOKEN }}
          path: 'assets_repo'

      - name: Sync files to assets
        run: |
          target_dir="assets_repo/${{ inputs.destination_dir }}"
          source_dir="source_repo/${{ inputs.source_dir }}"
          echo "Syncing from ${source_dir} to ${target_dir}"
          mkdir -p "${target_dir}"
          rsync -av --delete "${source_dir}/" "${target_dir}/"

      - name: Commit and push to assets
        run: |
          cd assets_repo
          git config --global user.name 'github-actions[bot]'
          git config --global user.email 'github-actions[bot]@users.noreply.github.com'
          git add .
          if git diff --staged --quiet; then
            echo "No changes to commit."
          else
            git commit -m "Sync from ${{ github.repository }}: ${{ inputs.destination_dir }}"
            git push
          fi
```

#### **2.2.2. `assets` 的核心部署工作流 (`deploy.yml`)**

这是整个系统的引擎, 在 `assets` 仓库内部运行.

```yml
# assets/.github/workflows/deploy.yml

name: Build and Deploy to GitHub Pages

on:
  push:
    branches:
      - main # 或你的主分支
  workflow_dispatch:

permissions:
  contents: write # 允许提交 manifest 更新
  pages: write    # 允许部署到 Pages
  id-token: write # 允许 OIDC 认证

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
      
      - name: Install Dependencies
        working-directory: ./scripts
        run: npm install

      - name: Run Asset Processing Script
        run: |
          node scripts/process-assets.js \
            --assets-path . \
            --deploy-path ./_site \
            --template-path ./scripts/viewer-template.html

      - name: Commit updated manifests
        run: |
          git config --global user.name 'github-actions[bot]'
          git config --global user.email 'github-actions[bot]@users.noreply.github.com'
          git add manifest/
          if git diff --staged --quiet; then
            echo "Manifests are up-to-date."
          else
            git commit -m "Update manifests [skip ci]"
            git push
          fi
      
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v2
        with:
          path: './_site'

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v2
```

#### **2.2.3. 文件处理脚本 (`process-assets.js`)**

此脚本的逻辑与之前版本基本相同, 存放于 `assets/scripts/` 目录. 它现在从 `assets` 仓库的 `manifest/` 目录读取和写入状态, 并将最终产出放入 `_site` 文件夹.

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


#### **2.2.4. HTML 查看器模板 (`viewer-template.html`)**

此模板也与上一版相同, 存放于 `assets/scripts/`.

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


## **3. 使用流程总结**

1. **准备文件 (外部项目)**: 在你的项目 (如 `my-project`) 中, 创建一个 `.github/workflows/publish.yml` 文件.  
   这个工作流可以包含构建步骤. 例如, 你可以先运行一个编译命令, 将生成的文件放入一个临时目录 (如 `to_deploy`), 然后再调用 `assetsHelper` 的工作流来上传这个目录的内容.  
    ```yml
    # my-project/.github/workflows/publish.yml
    name: Build and Publish Docs
    on:
      push:
        branches: [ main ]
    jobs:
      call-uploader:
        runs-on: ubuntu-latest
        steps:
          - name: Checkout code
            uses: actions/checkout@v4

          - name: Create deployment directory
            run: |
              # 这是一个示例构建步骤. 实际项目中可能是 `npm run build` 等命令.
              mkdir -p ./to_deploy
              echo "Generated at $(date)" > ./to_deploy/index.html
              # 假设你有一个 docs 目录需要复制
              cp -r ./docs/* ./to_deploy/

          - name: Call Reusable Uploader
            uses: <YourGitHubUsername>/assetsHelper/.github/workflows/upload.yml@main
            with:
              source_dir: 'to_deploy' # 上传刚刚创建的目录
              destination_dir: 'public/my-project' # 在assets仓库中的目标位置
    ```
2. **触发上传**: 推送代码到 `my-project` 的 `main` 分支, 将自动触发 `publish` 工作流.  
3. **自动部署**: `assetsHelper` 接收到文件后, 会将其推送到 `assets` 仓库. `assets` 仓库的 `push` 事件会触发 `deploy.yml`, 完成所有处理和到 GitHub Pages 的部署.  
4. **获取链接**: 流程与之前相同, 通过 `map.json` (在 Action 的 `filename-map` 构件中) 获取非公开文件的链接.
    - 公开文件: `https://<user>.github.io/assets/<repo_name>/path/to/file.ext`
    - 链接访问: 从 `map.json` 中找到 `private/<repo_name>/...` 对应的路径, 链接为 `https://<user>.github.io/assets/secret/<hash>.ext`
    - 密码访问: 从 `map.json` 中找到 `encrypt/<repo_name>/...` 对应的路径, 链接为 `https://<user>.github.io/assets/encrypt?file=<random_name>.html`

## **4. 安全性与注意事项**

* **坚不可摧的隐私**: 这是目前最安全的方案. 由于部署源是私有仓库的 Action 产物, 外部用户永远无法访问到你的文件结构或源代码.  
* **路径注入防护**: `assetsHelper` 的 `upload.yml` 工作流现在会严格校验 `destination_dir` 参数, 确保其必须以 `public/`, `private/` 或 `encrypt/` 开头, 且不包含路径遍历字符 (`..`). 这可以有效防止某个项目恶意或错误地修改其他项目的文件.  
* **`[skip ci]`**: 在 `deploy.yml` 中提交 manifest 时使用 `[skip ci]` 至关重要, 它能有效防止因提交操作而再次触发自身, 避免无限循环.  
* **关注点分离**: `assetsHelper` 负责"收", `assets` 负责"存"和"发", 职责清晰, 易于维护.