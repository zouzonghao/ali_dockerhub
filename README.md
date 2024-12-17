# ali_dockerhub

## 1、阿里云创建仓库

https://cr.console.aliyun.com/

## 2、Github Action 搭建后端

### 新建仓库一个，添加4个环境变量

进入 Settings -> Secret and variables -> Actions -> New Repository secret  添加下面的四个值

- ALIYUN_NAME_SPACE -> 阿里云命名空间
- ALIYUN_REGISTRY_USER -> 用户名
- ALIYUN_REGISTRY_PASSWORD -> 密码
- ALIYUN_REGISTRY -> 仓库地址

### 创建一个 github token

https://github.com/settings/tokens?type=beta

### 给 Action 权限

用来写 ali_image.txt 查看所有可用镜像

进入 Settings -> general -> 最下面 Workflow permissions -> Read and write permissions √


#### 添加文件，路径为 .github/workeflows/docker.yaml

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
          if curl -s -o /dev/null -w "%{http_code}" -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
            https://api.github.com/repos/${{ github.repository }}/contents/.lock | grep -q 200; then
            echo "Lock file exists, waiting..."
            sleep 10
            exit 1
          else
            # 创建锁文件
            echo "Lock file does not exist, creating lock..."
            curl -X PUT -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
              -d '{"message":"Create lock file","content":"IyBsb2NrCg=="}' \
              https://api.github.com/repos/${{ github.repository }}/contents/.lock
            echo "Lock acquired."
          fi
        continue-on-error: true
        


      - name: Pull latest changes
        run: |
          git config --global user.name "github-actions[bot]"
          git config --global user.email "github-actions[bot]@users.noreply.github.com"
          git pull --rebase origin main
          
      - name: Append docker pull command to ali_image.txt
        run: |
          echo "docker pull crpi-85vdj2b099n8ocq5.cn-guangzhou.personal.cr.aliyuncs.com/images730/$target_docker_image" >> ali_image.txt
        env:
          target_docker_image: ${{ secrets.ALIYUN_REGISTRY }}/${{ secrets.ALIYUN_NAME_SPACE }}/${{ matrix.images.target }}
      
      - name: Commit local changes
        run: |
          git add ali_image.txt
          git commit -m "Update ali_image.txt with new docker pull command"


      - name: Push changes
        run: |
          git push

      - name: Get lock file SHA
        id: get_lock_sha
        run: |
          LOCK_SHA=$(curl -s -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
            https://api.github.com/repos/${{ github.repository }}/contents/.lock | jq -r '.sha')
          echo "::set-output name=sha::$LOCK_SHA"

      - name: Release lock using GitHub API
        if: always()
        run: |
          curl -X DELETE -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
            -d '{"message":"Release lock file", "sha": "${{ steps.get_lock_sha.outputs.sha }}"}' \
            https://api.github.com/repos/${{ github.repository }}/contents/.lock
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
                  const pullCommands = this.images.map(image => \`docker pull crpi-85vdj2b099n8ocq5.cn-guangzhou.personal.cr.aliyuncs.com/images730/\${image.target}\`);

                  // 更新消息
                  this.message = \`同步请求已发送，时间：\${formattedTime}<br>请执行以下拉取命令：<br>\${pullCommands}<br>\`;
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
```

</details>

### 添加环境变量

设置 -> 变量和机密

- GITHUB_TOKEN -> 新建一个 github token 
- REPO_OWNER -> github 用户名
- REPO_NAME -> github 仓库名



