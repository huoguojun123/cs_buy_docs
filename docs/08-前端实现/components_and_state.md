# 组件化与状态管理

在现代前端开发中，组件化和状态管理是构建可扩展、可维护应用的两大基石。本项目基于Vue 3，充分利用其组合式API（Composition API）和Pinia实现了清晰的逻辑复用和状态分离。

## 1. 组件复用与封装策略

项目大量使用了组件化思想来提高代码的复用性和可维护性。组件策略主要分为两类：

### 1.1 基础组件封装 (Presentational Components)

这类组件主要负责UI展示，通常不包含复杂的业务逻辑。我们对项目中使用的UI库 `Element Plus` 的部分组件进行了二次封装，以达到以下目的：

-   **统一项目风格**：确保所有同类组件（如按钮、输入框）在整个应用中具有一致的外观和行为。
-   **简化API**：为组件预设常用的 `props`，减少在业务代码中的重复配置。
-   **增强功能**：在原有组件基础上增加新的功能，如为按钮增加默认的加载状态。

**示例：封装一个自定义加载按钮 `MyButton.vue`**
```vue
<!-- MyButton.vue -->
<template>
  <el-button :type="type" :loading="loading" @click="handleClick">
    <slot></slot>
  </el-button>
</template>

<script setup>
import { defineProps, defineEmits } from 'vue';

// 使用defineProps定义接收的属性
const props = defineProps({
  type: {
    type: String,
    default: 'primary' // 默认样式为主按钮
  },
  loading: {
    type: Boolean,
    default: false
  }
});

// 使用defineEmits声明抛出的事件
const emit = defineEmits(['click']);

// 封装事件逻辑
const handleClick = (event) => {
  // 可以在这里增加防抖等逻辑
  emit('click', event);
};
</script>
```

### 1.2 业务组件 (Container Components)

业务组件与具体的业务场景深度绑定，它们负责管理自身或子组件的状态，并处理与后端的数据交互。

-   **高内聚**：将特定业务的所有逻辑（UI、状态、API调用）封装在一个组件内。
-   **可复用**：在不同页面中可以方便地复用同一个业务场景。

**示例：商品卡片组件 `ProductCard.vue`**

这个组件封装了展示商品信息、加入购物车、跳转到详情页等所有相关逻辑。

```vue
<template>
  <div class="product-card">
    <img class="product-image" :src="product.image" :alt="product.name" @click="goToDetail"/>
    <div class="product-info">
      <h3 class="product-name">{{ product.name }}</h3>
      <p class="product-price">¥{{ product.price }}</p>
      <el-button type="primary" @click="addToCart">加入购物车</el-button>
    </div>
  </div>
</template>

<script setup>
import { defineProps } from 'vue';
import { useCartStore } from '@/store/modules/cart';
import { useRouter } from 'vue-router';

const props = defineProps({
  product: {
    type: Object,
    required: true
  }
});

const cartStore = useCartStore();
const router = useRouter();

// 调用Pinia Store中的action
const addToCart = () => {
  cartStore.addItem(props.product);
};

// 使用Vue Router进行导航
const goToDetail = () => {
  router.push(`/product/${props.product.id}`);
};
</script>
```

## 2. 全局状态管理 (Pinia)

对于跨组件、跨页面的共享状态，项目使用 `Pinia` 进行集中管理。Pinia以其简洁的API、完整的TypeScript支持和模块化的设计，成为了Vue 3生态的首选状态管理方案。

### 2.1 Store 结构

每个业务模块拥有独立的Store文件，使得状态的划分清晰明了。

**用户Store (`store/modules/user.js`)**

这个Store集中管理了用户认证（Token）和用户信息。

```javascript
import { defineStore } from 'pinia';
import { login as loginApi, getUserInfo } from '@/api/user';
import { localCache } from '@/utils/cache'; // 假设有缓存工具

export const useUserStore = defineStore('user', {
  // 核心状态：使用函数返回，避免跨请求污染
  state: () => ({
    token: localCache.get('token') || '',
    userInfo: null,
  }),
  // 计算属性：派生状态
  getters: {
    isLoggedIn: (state) => !!state.token,
    isAdmin: (state) => state.userInfo?.roles?.includes('ADMIN'),
    // 可以添加更复杂的getter，如用户头像
    userAvatar: (state) => state.userInfo?.avatar || 'default_avatar.png',
  },
  // 核心业务逻辑：异步操作和同步修改
  actions: {
    async login(credentials) {
      try {
        const response = await loginApi(credentials);
        this.token = response.data.token;
        localCache.set('token', this.token); // 持久化
        await this.fetchUserInfo(); // 登录后立即获取用户信息
        return true;
      } catch (error) {
        // 清理状态，防止脏数据
        this.logout();
        return false;
      }
    },
    async fetchUserInfo() {
      if (!this.token) return;
      try {
        const response = await getUserInfo();
        this.userInfo = response.data;
      } catch (error) {
         // Token可能已失效，执行登出
        this.logout();
      }
    },
    logout() {
      this.token = '';
      this.userInfo = null;
      localCache.remove('token');
    },
  },
});
```

### 2.2 在组件中使用Store

在组件中通过 `useUserStore()` Hook即可轻松访问和操作Store。

```vue
<script setup>
import { computed } from 'vue';
import { useUserStore } from '@/store/modules/user';

const userStore = useUserStore();

// 响应式地访问state和getters
const userName = computed(() => userStore.userInfo?.name);
const isLoggedIn = computed(() => userStore.isLoggedIn);

// 调用actions
const handleLogout = () => {
  userStore.logout();
  // 可选：跳转到登录页
  // router.push('/login');
};
</script>
```
Pinia的设计鼓励将业务逻辑（Actions）与UI（Components）分离，使得代码更易于测试和维护。 