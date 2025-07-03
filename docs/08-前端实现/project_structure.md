# 前端项目结构

前端项目 `e-business-front` 遵循了典型的现代Vue项目结构，旨在实现高度的模块化和可维护性。使用 `Vite` 作为构建工具，确保了快速的开发服务器启动和高效的热模块替换（HMR）。

## 目录与文件解析

以下是项目的主要目录结构及其职责说明：

```
e-business-front/
├── public/              # 静态资源，不会被Vite处理
│   └── index.html       # HTML入口文件
├── src/
│   ├── api/             # API请求模块
│   ├── assets/          # 样式、图片、字体等静态资源
│   ├── components/      # 全局可复用组件
│   │   ├── SvgIcon/     # SVG图标组件
│   │   └── Shared/      # 其他共享业务组件
│   ├── layout/          # 整体页面布局组件
│   ├── router/          # Vue Router的路由配置
│   ├── store/           # Pinia状态管理模块
│   │   ├── modules/     # 按业务划分的Store模块
│   │   └── index.js     # Pinia实例初始化与插件
│   ├── utils/           # 通用工具函数，如request.js
│   ├── views/           # 页面级组件（路由组件）
│   │   ├── home/
│   │   ├── login/
│   │   └── product/
│   ├── App.vue          # Vue应用的根组件
│   └── main.js          # 项目入口文件
├── .env.development     # 开发环境的环境变量配置
├── .env.production      # 生产环境的环境变量配置
├── package.json         # 项目依赖与脚本配置
└── vite.config.js       # Vite构建配置文件
```

### 关键目录详解

-   **`src/api/`**: 这是数据请求的核心封装层。它通常会根据后端的微服务或业务领域（如 `user.js`, `item.js`, `order.js`）来组织文件。每个文件内部封装了具体的API调用函数，使得业务逻辑层（Views或Store）可以清晰地调用后端接口，而无需关心具体的请求细节。

-   **`src/components/`**: 存放不与特定页面强绑定的、可在多处复用的UI组件。这包括对第三方UI库（如Element Plus）的二次封装，以统一风格或增强功能，也包括项目自身设计的通用业务组件（如价格显示器、用户头像等）。

-   **`src/layout/`**: 定义了应用的整体视觉框架。通常包含一个主布局文件，其中定义了页眉（Header）、侧边栏（Sidebar）、主内容区（Main Content）和页脚（Footer）的结构。路由系统中的页面级组件会被渲染到主内容区中。

-   **`src/router/`**: 存放所有`vue-router`的配置。这包括路由表的定义、路由模式的选择（History或Hash），以及最重要的——**导航守卫**。导航守卫（`beforeEach`）是实现登录验证和页面权限控制的核心。

-   **`src/store/`**: Pinia状态管理的中心。`modules`文件夹下存放了不同业务领域的状态切片，如用户信息（`user.js`）、购物车（`cart.js`）等。这种模块化的结构使得全局状态清晰、易于维护。

-   **`src/views/`**: 存放与路由直接对应的页面组件。每个`view`通常代表一个完整的页面，它会组合`components`中的通用组件，并通过`api`层获取数据，最终构成用户所见的界面。

-   **`vite.config.js`**: Vite的配置文件，功能强大。在这里可以配置代理服务器（解决开发环境下的跨域问题）、定义路径别名（如`@/`指向`src/`）、配置插件（如组件按需引入）等。 