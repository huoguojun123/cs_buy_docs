# API交互与路由控制

前端应用的核心是与后端的数据交互和页面间的导航。本项目通过封装Axios实现了可靠的API调用，并利用Vue Router的导航守卫实现了精细的权限控制。

## 1. API调用与错误处理

所有与后端API的通信都通过一个封装的Axios实例来完成，这提供了统一处理请求和响应的能力。

### 1.1 Axios实例封装 (`utils/request.js`)

我们创建了一个`utils/request.js`文件，它导出了一个配置完备的Axios实例。

**核心功能**:
-   **统一配置**: 设置了基础URL (`baseURL`) 和请求超时时间 (`timeout`)。
-   **请求拦截器**: 在每个请求发送前，自动附加认证所需的`Authorization`头（JWT Token）。
-   **响应拦截器**: 对所有响应进行预处理，包括处理业务错误码和HTTP状态码。

```javascript
import axios from 'axios';
import { ElMessage } from 'element-plus';
import { useUserStore } from '@/store/modules/user';

// 创建axios实例
const service = axios.create({
  // 从环境变量中读取基础API路径
  baseURL: import.meta.env.VITE_APP_BASE_API, 
  timeout: 10000, // 请求超时时间
});

// 请求拦截器 (Request Interceptor)
service.interceptors.request.use(
  (config) => {
    // Pinia的Store实例必须在函数内部获取，以确保是当前活动的Store
    const userStore = useUserStore();
    if (userStore.isLoggedIn) {
      config.headers['Authorization'] = `Bearer ${userStore.token}`;
    }
    return config;
  },
  (error) => {
    console.error('Request Error:', error); // for debug
    return Promise.reject(error);
  }
);

// 响应拦截器 (Response Interceptor)
service.interceptors.response.use(
  (response) => {
    const res = response.data;
    // 假设成功的业务码为 200 或 '200'
    if (res.code !== 200) {
      ElMessage.error(res.message || '业务错误，请联系管理员');
      return Promise.reject(new Error(res.message || 'Error'));
    }
    // 如果业务成功，则直接返回响应体中的数据
    return res.data;
  },
  (error) => {
    // 处理HTTP网络错误
    if (error.response) {
      switch (error.response.status) {
        case 401:
          // Token失效或未认证
          ElMessage.error('认证失败，请重新登录');
          useUserStore().logout(); // 使用Store的action执行登出
          window.location.href = '/login'; // 跳转到登录页
          break;
        case 403:
          ElMessage.error('没有权限访问该资源');
          break;
        case 500:
          ElMessage.error('服务器内部错误');
          break;
        default:
          ElMessage.error(error.message);
      }
    } else {
       ElMessage.error('网络连接异常');
    }
    return Promise.reject(error);
  }
);

export default service;
```

### 1.2 API模块化 (`api/`)

在`api`目录下，我们根据业务模块（如`item.js`, `user.js`）来组织API请求函数。

**示例 `api/item.js`**:
```javascript
import request from '@/utils/request';

/**
 * 获取商品详情
 * @param {string} id - 商品ID
 * @returns Promise
 */
export function getItemDetail(id) {
  return request({
    url: `/item/${id}`,
    method: 'get',
  });
}

/**
 * 分页搜索商品
 * @param {object} params - 查询参数，如 { page: 1, size: 10, keyword: '手机' }
 * @returns Promise
 */
export function searchItems(params) {
  return request({
    url: '/item/search',
    method: 'get',
    params, // GET请求的参数会附加到URL上
  });
}
```

## 2. 前端路由与权限控制

项目使用`vue-router`进行路由管理，并通过**导航守卫**实现了完善的权限控制。

### 2.1 路由配置与元信息

在`router/index.js`中，我们为需要保护的路由添加`meta`元信息。

```javascript
// router/routes.js
export const routes = [
  { path: '/login', component: () => import('@/views/login/index.vue') },
  { path: '/403', component: () => import('@/views/error/403.vue') },
  {
    path: '/',
    component: Layout, // 主布局
    children: [
      { path: 'home', component: () => import('@/views/home/index.vue') },
      { 
        path: 'dashboard', 
        component: () => import('@/views/dashboard/index.vue'),
        // meta字段用于定义权限需求
        meta: { 
          requiresAuth: true,    // 需要登录
          roles: ['ADMIN', 'EDITOR'] // 需要是'ADMIN'或'EDITOR'角色
        }
      },
      {
        path: 'profile',
        component: () => import('@/views/profile/index.vue'),
        meta: { requiresAuth: true } // 只需要登录，不限角色
      }
    ],
  },
];
```

### 2.2 全局导航守卫 (`beforeEach`)

`router.beforeEach`是实现路由权限控制的核心。在每次路由跳转之前，它都会被触发，检查用户的认证状态和角色权限。

```javascript
// router/permission.js
import router from './router';
import { useUserStore } from '@/store/modules/user';
import { ElNotification } from 'element-plus';

router.beforeEach(async (to, from, next) => {
  const userStore = useUserStore();

  const needsAuth = to.meta.requiresAuth;

  // 如果目标路由需要认证
  if (needsAuth) {
    // 检查用户是否已登录 (通过Token判断)
    if (userStore.isLoggedIn) {
      // 如果已登录，但没有用户信息，说明是刷新页面或刚登录
      // 此时需要异步获取用户信息
      if (!userStore.userInfo) {
        try {
          await userStore.fetchUserInfo();
        } catch (error) {
           // 获取用户信息失败（例如Token已在后端过期）
           ElNotification.error('用户信息获取失败，请重新登录');
           next(`/login?redirect=${to.path}`); // 重定向到登录页
           return;
        }
      }
      
      // 检查角色权限
      const requiredRoles = to.meta.roles;
      const userRoles = userStore.userInfo?.roles || [];
      const hasPermission = requiredRoles ? requiredRoles.some(role => userRoles.includes(role)) : true;

      if (hasPermission) {
        next(); // 有权限，放行
      } else {
        next('/403'); // 无权限，跳转到403页面
      }
    } else {
      // 未登录，重定向到登录页，并携带目标路径
      next(`/login?redirect=${to.path}`);
    }
  } else {
    // 目标路由不需要认证，直接放行
    next();
  }
});
```
这种设计将权限逻辑与页面组件解耦，使得权限规则的管理和修改变得非常集中和方便。 