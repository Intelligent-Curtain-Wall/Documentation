# 服务器前端部署文档

[toc]

## nuxt框架说明

### nuxt项目结构说明
本项目基于nuxt3构建，其目录结构如下：
```
CURTAINWALLWEB-FRONTEND/
├── api/
├── components/
├── composables/
├── layouts/
├── middleware/
├── pages/
├── public/
├── server/
├── utils/
├── app.vue
├── error.vue
├── nuxt.config.ts
├── package.json
├── tsconfig.json
└── tailwind.config.ts
```

#### pages-路由核心
作用：自动生成前端路由
Nuxt 会根据 pages/ 下的文件结构自动生成 URL 路径
示例：
```
pages/
├── index.vue        →  /
├── login.vue        →  /login
├── user/
│   └── info.vue     →  /user/info
```
不需要手写路由配置（无 vue-router 手动配置）
所有页面访问路径由该目录决定
接口联调、Nginx转发路径必须和这里一致
> 各模块的页面前端请在此处撰写

#### components——组件目录

存放可复用 UI 组件（如图表、卡片、按钮等）
- 进参与构建，不影响部署结构
- 最终会被打包进.output或dist中

#### layouts——页面布局

定义页面外壳（如导航栏、侧边栏），主要用于统一页面结构

#### middleware——路由拦截

- 用于页面访问前的逻辑处理（如登录校验），如JWT校验或者未登录跳转/login
- 若后端有鉴权，需与后端策略保持一致

#### composables——逻辑复用

- 类似 hooks（Vue3 Composition API）
- 存放请求封装、状态管理等

部署意义：
- 所有 API 请求通常封装在这里
- 接口地址（baseURL）一般在这里或 config 中定义

#### api——接口封装层

封装后端所有api请求
如
```
api/
├── user.ts
├── stain.ts
```

#### public——静态资源目录

- 存放不会被打包的静态文件
- 直接映射为网站根路径

会被原样复制到部署目录
常用于：
- 图片
- favicon
- 静态 JSON

#### server——NuxtServer

`server/` 目录用于编写 Nuxt 内置的服务端接口，其作用类似一个轻量级 Node.js 后端，可直接在前端项目中定义 API。

本项目中的目录结构如下：

```
server/
├── api/
│   ├── account/
│   │   └── login.post.ts
│   ├── device/
│   ├── monitor/
│   ├── segment/
├── middleware/
```

##### 功能说明

- `server/api/`：定义接口路由（自动映射为 `/api/*`）
- `*.post.ts`：表示 POST 请求接口
- `middleware/`：服务端中间件（请求拦截、鉴权等）

#####  在本项目中的作用

该目录主要用于：

- 本地开发阶段快速提供接口模拟
- 简单业务逻辑处理

但在本项目实际部署架构中：

- 主要后端服务由独立服务提供（Spring Boot / Python 等）
- 仅在开发阶段可以考虑使用

##### 注意事项

- 仅在以下方式下生效：

```
nuxt build + node .output/server/index.mjs
```

即：**需要 Node.js 运行 Nuxt 服务端**

- 若采用本项目当前部署方式：

```
npm run generate + Nginx 静态部署
```

则： `server/` 目录中的所有接口 **不会生效**

#### utils——工具函数

存放通用方法（时间处理、格式化等）

#### nuxt.config.ts——核心配置文件

nuxt.config.ts 是 Nuxt 项目的全局配置文件，用于统一管理前端应用的构建方式、模块依赖以及与后端服务的交互方式。

1. 运行模式

    ```
    ssr: false
    ```
    - 表示当前项目为 SPA（纯前端应用）
    - 部署结论：
      - 不需要 Node.js 运行环境
      - 只需构建后将静态文件部署到 Nginx
2. 接口代理（开发 vs 生产）
   ```
    nitro: {
        devProxy: {
            '/api': { target: 'http://8.159.143.133:8000' },
            '/predict': { target: 'http://47.102.208.89:8007' },
            '/oss': { target: 'http://8.159.143.133:9000' },
            '/crackdetection': { target: 'http://110.42.214.164:8001' }
        }
    }
   ```
   该配置仅在开发环境生效
   用于把前端请求转发到不同后端服务
   > 生产环境必须在 Nginx 中实现同样的转发规则，否则接口无法访问
3. 后端地址配置（环境变量）
   ```
    runtimeConfig: {
        public: {
            apiBase: process.env.NUXT_API_BASE_URL
        }
    }
   ```
   定义前端请求的基础 API 地址
   部署关键
   - 需要在服务器环境中设置：
     - ```NUXT_API_BASE_URL``` 
   - 或直接通过 Nginx 统一转发 /api
4. 静态资源路径（必须保证）
    ```
    app: {
        baseURL: '/',
        buildAssetsDir: '/_nuxt/',
    }
    ```
    部署关键点：
    - Nginx 必须允许访问：
      - ```/_nuxt/*```
    - 否则页面 JS / CSS 无法加载

5. 前端部署人员须知（重点）
   - 这是一个 纯前端项目（SPA）
   - 构建后只需部署静态文件（无后端依赖）
   - 所有 /api、/predict 等请求必须在 Nginx 中做反向代理
   - 必须保证 /_nuxt/ 静态资源路径正常访问

### nuxt破解

由于nuxt框架的高级UI组件库需要授权码（付费），所以本小组使用了破解的方法，具体可参考前端仓库中的```破解nuxt.md```文件。

## 前端网页说明

### 页面结构说明（基于pages目录）
本项目前端页面基于 Nuxt 的 pages/ 目录自动生成路由，所有网页访问路径均由该目录结构决定。

典型结构如下：
```
pages/
├── index.vue
├── login.vue
├── Usermanage.vue
├── crack/
│   └── CrackDetection.vue
├── vibration/
│   └── abnormal.vue
```
### 页面访问路径说明

Nuxt 会根据文件路径自动映射为 URL：

| 文件路径                      | 访问地址             | 功能说明     |
| ----------------------------- | -------------------- | ------------ |
| `pages/index.vue`             | `/`                  | 系统首页     |
| `pages/login.vue`             | `/login`             | 用户登录页面 |
| `pages/user/info.vue`         | `/user/info`         | 用户信息页面 |
| `pages/crack/detect.vue`      | `/crack/detect`      | 裂缝检测页面 |
| `pages/vibration/monitor.vue` | `/vibration/monitor` | 振动监测页面 |

部署重点：
- 所有页面访问必须通过前端入口（Nginx）
- 不允许直接访问 .vue 文件
- 路由由前端控制（SPA）

### 页面加载流程（部署视角）

用户访问网页时的流程如下：

```浏览器 → Nginx → index.html → JS加载 → 路由匹配 → 页面渲染```
具体过程：

1. 用户访问任意路径（如 /login）
2. Nginx 返回统一的 index.html
3. 前端 JS 加载完成
4. Nuxt 根据 URL 匹配 pages/ 中对应页面
5. 渲染对应 Vue 页面

### 页面与后端交互
前端页面通过 API 与后端进行数据交互，典型请求路径如下：
```
/login        → /api/account/login
/crack/detect → /crackdetection
/vibration    → /predict 或 /history
```
请求流程：
```
前端页面 → Axios请求 → Nginx → 后端服务
```
部署要求：
- 必须在 Nginx 中配置：
  - /api
  - /predict
  - /history
  - /oss
  - /crackdetection

否则页面功能无法正常使用。

### 前端资源说明

前端页面依赖以下资源：

- JS 文件（打包后位于 /_nuxt/）
- CSS 样式文件
- 图片资源（来自 /public 或 OSS）

### 总结（部署重点）

1. 页面路径由 pages/ 自动生成
2. 所有页面通过 index.html 统一加载（SPA）
3. 页面功能依赖后端 API，必须配置反向代理
4. 静态资源路径 /_nuxt/ 必须正常访问

## 静态打包与部署说明

### 前端静态打包

在前端项目根目录执行：
```
npm install
npm run generate
```
执行完成后，Nuxt 会生成静态站点文件，可直接用于部署。

### 打包产物说明
本项目打包后的目录如下：
```/.output/public```
目录结构示例：
```
integrated-frontend/
├── index.html
├── 200.html
├── 404.html
├── _nuxt/
├── assets/
├── login/
├── vibration/
├── crackdetect/
├── resilience/
├── userManage/
└── ...
```
各文件/目录作用如下：
- index.html：前端应用入口页面
- 200.html：SPA 路由兜底页面（关键）
- 404.html：错误页面
- _nuxt/：打包后的 JS、CSS 等核心资源
- 各业务目录（如 login/、vibration/）：预生成的页面路径

### 部署方式
将打包生成的静态文件上传/移动至服务器目录：
```/var/www/integrated-frontend```
该目录用于存放 Integrated 前端主应用的静态资源，例如：
```
index.html
_nuxt/
assets/
login/
vibration/
crackdetect/
resilience/
```
在 Nginx 中，主前端通过以下配置指向该目录：
```
location / {
        root /var/www/integrated-frontend;
        index index.html;
        try_files $uri $uri/ /index.html;
    }
```
其中：
- root /var/www/integrated-frontend; 表示主前端静态文件所在目录
- index index.html; 表示默认入口文件
- try_files $uri $uri/ /index.html; 用于处理前端路由刷新问题

同时，本项目同时部署了多个前端（如 dataset 模块），需分别配置对应路径。

### Nginx配置（核心）

本项目使用 Nginx 同时完成两类工作：

- 托管前端静态资源
- 将不同路径的接口请求转发到对应后端服务

#### 编辑配置文件：

```
vim /etc/nginx/sites-available/proxy
```

#### 完整配置

```
server {
    listen 80;
    server_name 你的服务器IP;

    root /var/www/curtainwall-frontend;
    index index.html;

    # 前端路由（SPA核心）
    location / {
        try_files $uri $uri/ /index.html;
    }

    # =========================
    # 后端接口代理（必须配置）
    # =========================

    # 用户认证服务
    location /api/ {
        proxy_pass http://8.159.143.133:8000/;
        proxy_set_header Host $host;
    }

    # AI推理服务
    location /predict/ {
        proxy_pass http://47.102.208.89:8007/;
        proxy_set_header Host $host;
    }

    # 历史数据
    location /history/ {
        proxy_pass http://47.102.208.89:8007/;
        proxy_set_header Host $host;
    }

    # OSS存储
    location /oss/ {
        proxy_pass http://8.159.143.133:9000/;
        proxy_set_header Host $host;
    }

    # 裂缝检测
    location /crackdetection/ {
        proxy_pass http://110.42.214.164:8001/;
        proxy_set_header Host $host;
    }

    # 静态资源（Nuxt打包产物）
    location /_nuxt/ {
        alias /var/www/curtainwall-frontend/_nuxt/;
    }
}
```

##### Integrated 主前端配置

```
location / {
    root /var/www/integrated-frontend;
    index index.html;
    try_files $uri $uri/ /index.html;
}
```
该配置用于访问主前端页面。用户访问服务器根路径时，Nginx 会从 /var/www/integrated-frontend 中读取静态文件。

##### Dataset 前端配置

```
location /dataset/ {
    alias /var/www/dataset-frontend/;
    try_files $uri $uri/ /dataset/index.html;
}
```
/dataset/ 是另一个独立前端模块，使用 alias 映射到：
```/var/www/dataset-frontend/```
同时，Dataset 的 CSS、JS、图片和字体资源也分别配置了独立路径：
```
location /dataset/css/ {
    alias /var/www/dataset-frontend/css/;
}

location /dataset/js/ {
    alias /var/www/dataset-frontend/js/;
}

location /dataset/png/ {
    alias /var/www/dataset-frontend/png/;
}

location /dataset/fonts/ {
    alias /var/www/dataset-frontend/fonts/;
}
```

##### 后端接口代理配置

Nginx 根据不同路径前缀，将请求转发到不同后端服务。

1. 用户认证后端

```
location /api/ {
    rewrite ^/api/(.*)$ /$1 break;
    proxy_pass http://localhost:8000;
    include /etc/nginx/proxy_params;
}
```
该配置表示访问 /api/xxx 时，会去掉 /api/ 前缀后转发到本机 8000 端口的后端服务。

2. AI 推理服务

```
location /predict/ {
    rewrite ^/predict/(.*)$ /$1 break;
    proxy_pass http://47.102.208.89:8007;
    include /etc/nginx/proxy_params;
}
```
用于转发 AI 推理相关请求。

3. 历史数据服务

```
location /history/ {
    rewrite ^/history/(.*)$ /$1 break;
    proxy_pass http://47.102.208.89:8007;
    include /etc/nginx/proxy_params;
}
```
用于转发历史记录相关请求。

4.  OSS 服务

```
location /oss/ {
    rewrite ^/oss/(.*)$ /$1 break;
    proxy_pass http://8.159.143.133:9000;
    include /etc/nginx/proxy_params;
}
```
用于转发文件访问或对象存储相关请求。

5.  裂缝检测服务

```
location /crackdetection/ {
    rewrite ^/crackdetection/(.*)$ /$1 break;
    proxy_pass http://110.42.214.164:8001;
    include /etc/nginx/proxy_params;
}
```
用于转发裂缝检测相关请求。

### 启动与验证

```
sudo nginx -t
sudo systemctl restart nginx
```

修改 Nginx 配置后，先检查配置是否正确：
```nginx -t```
若显示配置正常，则重启 Nginx：
```若显示配置正常，则重启 Nginx：```
访问服务器地址：
```http://8.159.143.133```

部署人员需要验证以下内容：

1. 主前端页面是否正常加载
2. /dataset/ 页面是否可正常访问
3. 刷新页面后是否仍能正常显示
4. 登录、推理、历史数据、OSS、裂缝检测等接口是否可正常调用

### 常见问题

#### 刷新页面后出现 404

原因：前端是 SPA 应用，直接刷新子路由时，Nginx 可能找不到对应物理文件。
解决方式：
```try_files $uri $uri/ /index.html;```

#### 接口请求失败

原因可能包括：

- 后端服务未启动
- Nginx 代理路径配置错误
- rewrite 后路径与后端接口不匹配
- proxy_pass 地址或端口错误

解决方式：

- 使用 curl 直接访问后端服务
- 检查 Nginx error log
- 检查对应服务端口是否开放