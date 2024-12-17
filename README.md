# 从 Cloudflare worker 到 Github Action 再到阿里云 Docker

![demo](./demo.avif)



原理：前端 Cloudflare worker ，接收用户的参数，访问 Github Action api，激活 Action 下载镜像，同步到阿里云镜像仓库，然后将镜像拉取命令保存私库

## 1、阿里云创建仓库

https://cr.console.aliyun.com/

收集 4 个值
- 用户名
- 密码
- 仓库地址
- 命名空间

## 2、Github 【私库】

新建一个【私库】，专门用来存放生成的镜像列表，Action执行后，会在仓库生成一个 images_list.txt 文件

里面记载了所有同步到阿里云的镜像的拉取命令，方便以后查找

创建的时候，勾选生成 .md 文件

## 2、Github 【Action库】

新建一个公开的库，主要执行Action用，称为【Action库】

### 创建2个 github token

https://github.com/settings/tokens?type=beta

两个仓库，分别创建一个限权token:

- 【私库】token 需填入 Action 仓库的环境变量，
- 【Action库】 token 需填入 Cloudflare worker 的环境变量

也可以创建一个全局 token，替代这两个

### 在【Action库】添加6个环境变量

进入 Settings -> Secret and variables -> Actions -> New Repository secret  添加下面的

- ALIYUN_NAME_SPACE -> 阿里云命名空间
- ALIYUN_REGISTRY_USER -> 阿里云用户名
- ALIYUN_REGISTRY_PASSWORD -> 阿里云密码
- ALIYUN_REGISTRY -> 阿里云仓库地址
- S_REPO -> Github用户名/【私库】名
- S_TOKEN -> 【私库】token


### 添加文件，路径为 .github/workeflows/docker.yaml

<details>
	<summary>docker.yaml</summary>

```yaml
name: Sync Docker Image To Aliyun Repo By Api
on:
  repository_dispatch:
    types: sync_docker # 只有当 event_type 为 sync_docker 时触发
jobs:
  sync-task:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        images: ${{ github.event.client_payload.images }}
    steps:
      - uses: actions/checkout@v4
      - name: Sync ${{ matrix.images.source }} to ${{ matrix.images.target }}
        run: |
          docker pull --platform ${{ matrix.images.platform || 'linux/amd64' }} $source_docker_image
          docker tag $source_docker_image $target_docker_image
          docker login --username=${{ secrets.ALIYUN_REGISTRY_USER }} --password=${{ secrets.ALIYUN_REGISTRY_PASSWORD }} ${{ secrets.ALIYUN_REGISTRY }}
          docker push $target_docker_image
        env:
          source_docker_image: ${{ matrix.images.source }}
          target_docker_image: ${{ secrets.ALIYUN_REGISTRY }}/${{ secrets.ALIYUN_NAME_SPACE }}/${{ matrix.images.target }}

      - name: Acquire lock using GitHub API
        id: lock
        run: |
          # 检查锁文件是否存在
          if curl -s -o /dev/null -w "%{http_code}" -H "Authorization: token ${{ secrets.S_TOKEN }}" \
            https://api.github.com/repos/${{ secrets.S_REPO }}/contents/.lock | grep -q 200; then
            echo "Lock file exists, waiting..."
            sleep 10
            exit 1
          else
            # 创建锁文件
            echo "Lock file does not exist, creating lock..."
            curl -X PUT -H "Authorization: token ${{ secrets.S_TOKEN }}" \
              -d '{"message":"Create lock file","content":"IyBsb2NrCg=="}' \
              https://api.github.com/repos/${{ secrets.S_REPO }}/contents/.lock
            echo "Lock acquired."
          fi
        continue-on-error: true

      - name: Checkout private repository
        uses: actions/checkout@v4
        with:
          repository: ${{ secrets.S_REPO }} # 使用 secrets.S_REPO 动态指定私库
          token: ${{ secrets.S_TOKEN }}    # 使用 S_TOKEN 进行身份验证
          path: private-repo              # 指定拉取的私库存放路径




      - name: Append docker pull command to image_list.txt
        run: |
          # 去重处理
          cat private-repo/image_list.txt | awk '!seen[$0]++' > private-repo/image_list_temp.txt
          mv private-repo/image_list_temp.txt private-repo/image_list.txt  # 修复路径

          # 检查是否已经存在相同的 docker pull 命令
          if ! grep -Fxq "docker pull $target_docker_image" private-repo/image_list.txt; then
            # 如果不存在，则追加新的 docker pull 命令
            echo "docker pull $target_docker_image" >> private-repo/image_list.txt
          else
            echo "Docker pull command already exists in image_list.txt, skipping append."
          fi
        env:
          target_docker_image: ${{ secrets.ALIYUN_REGISTRY }}/${{ secrets.ALIYUN_NAME_SPACE }}/${{ matrix.images.target }}

      - name: Check if there are changes to commit
        id: check_changes
        run: |
          cd private-repo
          if [ -n "$(git status --porcelain)" ]; then
            echo "::set-output name=has_changes::true"
          else
            echo "::set-output name=has_changes::false"
          fi

      - name: Commit local changes
        if: steps.check_changes.outputs.has_changes == 'true'
        run: |
          cd private-repo
          git config --global user.name "github-actions[bot]"
          git config --global user.email "github-actions[bot]@users.noreply.github.com"
          git add image_list.txt  # 确保添加正确的文件
          git commit -m "Update image_list.txt with new docker pull command"

      - name: Fetch and merge remote changes
        if: steps.check_changes.outputs.has_changes == 'true'
        run: |
          cd private-repo
          git fetch origin main
          git merge origin/main

      - name: Push changes
        if: steps.check_changes.outputs.has_changes == 'true'
        run: |
          cd private-repo
          git push https://x-access-token:${{ secrets.S_TOKEN }}@github.com/${{ secrets.S_REPO }}.git HEAD:main

      - name: Get lock file SHA
        id: get_lock_sha
        run: |
          LOCK_SHA=$(curl -s -H "Authorization: token ${{ secrets.S_TOKEN }}" \
            https://api.github.com/repos/${{ secrets.S_REPO }}/contents/.lock | jq -r '.sha')
          echo "::set-output name=sha::$LOCK_SHA"

      - name: Release lock using GitHub API
        if: always()
        run: |
          curl -X DELETE -H "Authorization: token ${{ secrets.S_TOKEN }}" \
            -d '{"message":"Release lock file", "sha": "${{ steps.get_lock_sha.outputs.sha }}"}' \
            https://api.github.com/repos/${{ secrets.S_REPO }}/contents/.lock
          echo "Lock released."
```

</details>

## 3、Cloudflare worker 前端

### 新建一个 worker，将下面的代码填入

<details>
	<summary>worker.js</summary>

```js
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
  if (request.method === 'GET') {
    try {
      const vueScript = await fetch('https://unpkg.com/vue@3/dist/vue.global.prod.js').then(r => r.text());
      const tailwindCSS = await fetch('https://cdn.jsdelivr.net/npm/tailwindcss@2.2.19/dist/tailwind.min.css').then(r => r.text());

      const appTemplate = `
        <div class="min-h-screen bg-gradient-to-r from-pink-100 to-blue-100 flex items-center justify-center">
          <div class="bg-white shadow-lg rounded-lg p-8 max-w-xl w-full">
            <h1 class="text-3xl font-bold text-center text-gray-800 mb-6">Docker 镜像同步</h1>
            <div v-for="(image, index) in images" :key="index" class="border border-gray-200 rounded-lg p-6 mb-6 bg-white shadow-sm">
              <h2 class="text-xl font-semibold text-gray-700 mb-4">镜像 {{ index + 1 }}</h2>
              <div class="mb-4">
                <label class="block text-gray-700 text-sm font-bold mb-2">来源镜像（例：vaultwarden/server:1.26.0）:</label>
                <input type="text" v-model="image.source" class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-400">
              </div>
              <div class="mb-4">
                <label class="block text-gray-700 text-sm font-bold mb-2">CPU架构:</label>
                <select v-model="image.platform" class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-400">
                  <option value="linux/amd64">linux/amd64</option>
                  <option value="linux/arm64">linux/arm64</option>
                  <option value="linux/arm/v7">linux/arm/v7</option>
                </select>
              </div>
              <div class="mb-4">
                <label class="block text-gray-700 text-sm font-bold mb-2">目标镜像（例：bitwarden:AMD64_1.26.0）:</label>
                <input type="text" v-model="image.target" class="w-full px-4 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-400">
              </div>
              <button @click="removeImage(index)" type="button" class="w-full bg-red-400 hover:bg-red-500 text-white font-bold py-2 px-4 rounded-lg transition duration-300">删除</button>
            </div>
            <div class="flex justify-between">
              <button @click="addImage" type="button" class="bg-blue-400 hover:bg-blue-500 text-white font-bold py-2 px-4 rounded-lg transition duration-300">添加镜像</button>
              <button @click="syncImages" type="button" class="bg-green-400 hover:bg-green-500 text-white font-bold py-2 px-4 rounded-lg transition duration-300">同步镜像</button>
            </div>
            <div v-if="message" class="mt-6 p-4 bg-gray-100 rounded-lg" :class="messageClass">
              <p class="text-left overflow-auto text-gray-800" v-html="message"></p>
            </div>
          </div>
        </div>
      `;

      const html = `<!DOCTYPE html>
      <html>
      <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Docker Image Sync</title>
        <style>${tailwindCSS}</style>
      </head>
      <body class="bg-gray-50">
        <div id="app"></div>
        <script>${vueScript}</script>
        <script>
          const { createApp, h } = Vue;

          const App = {
            data() {
              return {
                repoOwner: '${REPO_OWNER}', // 从环境变量读取
                repoName: '${REPO_NAME}', // 从环境变量读取
                images: [{ source: '', target: '', platform: 'linux/amd64' }],
                message: null,
                messageClass: null
              };
            },
            methods: {
              addImage() {
                this.images.push({ source: '', target: '', platform: 'linux/amd64' });
              },
              removeImage(index) {
                this.images.splice(index, 1);
              },
              async syncImages() {
                if (!this.githubToken) {
                  this.message = 'GitHub Token未配置';
                  this.messageClass = 'bg-red-100 text-red-600';
                  return;
                }
                if (this.images.some(item => !item.source || !item.target)) {
                  this.message = '请填写完整的镜像信息';
                  this.messageClass = 'bg-red-100 text-red-600';
                  return;
                }
                try {
                  const response = await fetch(
                    \`https://api.github.com/repos/\${this.repoOwner}/\${this.repoName}/dispatches\`,
                    {
                      method: 'POST',
                      headers: {
                        Accept: 'application/vnd.github.v3+json',
                        Authorization: \`token \${this.githubToken}\`,
                        'Content-Type': 'application/json'
                      },
                      body: JSON.stringify({
                        event_type: 'sync_docker',
                        client_payload: {
                          images: this.images,
                          message: 'github action sync'
                        }
                      })
                    }
                  );
                  if (!response.ok) {
                    const errorData = await response.json();
                    throw new Error(\`HTTP error \${response.status}: \${errorData.message || response.statusText}\`);
                  }

                  // 获取当前时间
                  const now = new Date();
                  const formattedTime = \`\${now.getFullYear()}-\${(now.getMonth() + 1).toString().padStart(2, '0')}-\${now.getDate().toString().padStart(2, '0')} \${now.getHours().toString().padStart(2, '0')}:\${now.getMinutes().toString().padStart(2, '0')}:\${now.getSeconds().toString().padStart(2, '0')}\`;

                  // 生成拉取命令
                  const pullCommands = this.images.map(image => \`docker pull ${ALIYUN_REGISTRY}/${ALIYUN_NAME_SPACE}/\${image.target}\`).join('<br><br>');

                  // 更新消息
                  this.message = \`同步请求已发送，时间：\${formattedTime}<br>稍等30S~60S后，请执行以下拉取命令：<br><br>\${pullCommands}<br>\`;
                  this.messageClass = 'bg-green-100 text-green-600';
                } catch (error) {
                  console.error("Error:", error);
                  this.message = \`同步请求失败： \${error.message}\`;
                  this.messageClass = 'bg-red-100 text-red-600';
                }
              }
            },
            computed: {
              githubToken() {
                return '${GITHUB_TOKEN}'; // 从环境变量读取
              },
              imageTargets() {
                return this.images.map(image => {
                  const sourceParts = image.source.split('/');
                  const imageName = sourceParts.length > 1 ? sourceParts[sourceParts.length - 1] : image.source;
                  let platformSuffix = image.platform.split('/')[1].toUpperCase();
                  if (platformSuffix == "ARM"){
                    platformSuffix = "ARM_V7"
                  }
                  return \`\${imageName}_\${platformSuffix}\`;
                });
              }
            },
            watch: {
              imageTargets(newTargets) {
                this.images.forEach((image, index) => {
                  image.target = newTargets[index];
                });
              }
            },
            template: \`${appTemplate}\`
          };

          const app = createApp(App);
          app.mount('#app');
        </script>
      </body>
      </html>`;

      return new Response(html, {
        headers: { 'Content-Type': 'text/html;charset=UTF-8' },
      });
    } catch (error) {
      return new Response(`Error: ${error.message}`, { status: 500 });
    }
  }
  return new Response("Method Not Allowed", { status: 405 });
}
```

</details>

### 添加环境变量

设置 -> 变量和机密

- GITHUB_TOKEN -> github【Action库】token
- REPO_OWNER -> github 用户名
- REPO_NAME -> github【Action库】
- ALIYUN_NAME_SPACE -> 阿里云仓库命名空间
- ALIYUN_REGISTRY -> 阿里云仓库地址

