---
name: mastergo-plugin-dev
description: 开发 MasterGo 设计工具插件，包括项目初始化、manifest配置、主线程代码、UI界面开发、消息通信、图片导出、数据存储等功能。当用户需要创建 MasterGo 插件或修改现有插件功能时使用。
---

# MasterGo 插件开发

## 项目结构

```
my-plugin/
├── manifest.json          # 插件配置
├── package.json           # 依赖配置
├── vite.config.ts         # 构建配置
├── tsconfig.json          # TypeScript配置
├── lib/
│   └── main.ts           # 主线程代码（沙箱中运行）
├── ui/
│   ├── App.vue           # UI主组件
│   └── ui.ts             # UI入口
├── messages/
│   └── sender.ts         # 消息类型定义
└── dist/                 # 构建输出
    ├── index.html        # UI页面
    └── main.js           # 主线程脚本
```

## 快速开始

### 1. 初始化项目

```bash
# 创建目录
mkdir my-plugin && cd my-plugin

# 初始化 package.json
npm init -y

# 安装依赖
npm install vue@^3.2.31 @mastergo/plugin-typings@^2.2.0
npm install -D vite@^2.8.6 @vitejs/plugin-vue@^2.2.4 vite-plugin-singlefile@^0.7.1 cross-env@^7.0.3
```

### 2. 核心配置文件

**manifest.json** - 插件入口配置：
```json
{
  "name": "插件名称",
  "id": 1234567890,
  "api": "1.0.0",
  "main": "dist/main.js",
  "ui": "dist/index.html",
  "editor_type": ["canvas"],
  "permissions": ["currentuser"]
}
```

**vite.config.ts** - 双目标构建（UI + Main）：
```typescript
import { resolve } from 'path'
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import { viteSingleFile } from "vite-plugin-singlefile"

const target = process.env.TARGET

export default defineConfig(() => {
  const buildConfig = target === 'ui'
    ? {
        target: "esnext",
        assetsInlineLimit: 100000000,
        cssCodeSplit: false,
        rollupOptions: {
          inlineDynamicImports: true,
        },
      }
    : {
        lib: {
          entry: resolve(__dirname, './lib/main.ts'),
          name: 'myLib',
          formats: ['umd'],
          fileName: () => `main.js`
        },
      }

  return {
    plugins: [vue(), viteSingleFile()],
    build: { ...buildConfig, emptyOutDir: false },
    resolve: {
      alias: {
        "@lib": resolve(__dirname, './lib'),
        "@ui": resolve(__dirname, './ui'),
        "@messages": resolve(__dirname, './messages'),
      }
    },
  }
})
```

**package.json scripts**：
```json
{
  "scripts": {
    "dev": "npm run dev:ui & npm run dev:main",
    "dev:ui": "cross-env TARGET=ui NODE_ENV=development vite build --mode development -w",
    "dev:main": "cross-env TARGET=main NODE_ENV=development vite build --mode development -w",
    "build": "npm run build:ui && npm run build:main",
    "build:ui": "cross-env TARGET=ui vite build",
    "build:main": "cross-env TARGET=main vite build"
  }
}
```

### 3. 消息通信架构

**messages/sender.ts** - 定义消息类型：
```typescript
// 插件 -> UI
export enum PluginMessage {
  STYLE_ADDED = 'STYLE_ADDED',
  STYLES_LOADED = 'STYLES_LOADED',
  STYLE_UPDATED = 'STYLE_UPDATED',
  EXPORT_ERROR = 'EXPORT_ERROR',
}

// UI -> 插件
export enum UIMessage {
  ADD_STYLE = 'ADD_STYLE',
  LOAD_STYLES = 'LOAD_STYLES',
  UPDATE_STYLE = 'UPDATE_STYLE',
  DELETE_STYLE = 'DELETE_STYLE',
}

// 数据类型
export interface StyleItem {
  id: string;
  name: string;
  description: string;
  imageData: string;
  createdAt: number;
}

export const sendMsgToUI = (data: any) => {
  mg.ui.postMessage(data, "*")
}

export const sendMsgToPlugin = (data: any) => {
  parent.postMessage(data, "*")
}
```

### 4. 主线程代码模板

**lib/main.ts**：
```typescript
import { UIMessage, PluginMessage, sendMsgToUI } from '@messages/sender'

mg.showUI(__html__)

// 获取选中元素
const selection = mg.document.currentPage.selection

// 导出图片
const bytes = await node.exportAsync({
  format: 'PNG',
  constraint: { type: 'SCALE', value: 2 }
})

// 数据存储
await mg.clientStorage.setAsync('key', data)
const data = await mg.clientStorage.getAsync('key')

// 监听UI消息
mg.ui.onmessage = (msg) => {
  const { type, data } = msg
  switch (type) {
    case UIMessage.ADD_STYLE:
      // 处理添加
      break
  }
}
```

### 5. UI代码模板

**ui/App.vue**：
```vue
<template>
  <div class="plugin-ui">
    <button @click="addStyle">添加</button>
    <div v-for="item in items" :key="item.id">
      {{ item.name }}
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { sendMsgToPlugin, UIMessage, PluginMessage } from '@messages/sender'

const items = ref([])

function addStyle() {
  sendMsgToPlugin({ type: UIMessage.ADD_STYLE })
}

// 监听插件消息
window.onmessage = (event) => {
  const { type, data } = event.data
  switch (type) {
    case PluginMessage.STYLE_ADDED:
      items.value.unshift(data)
      break
  }
}

onMounted(() => {
  sendMsgToPlugin({ type: UIMessage.LOAD_STYLES })
})
</script>
```

## 常用API

### 获取选中元素
```typescript
const selection = mg.document.currentPage.selection
if (selection.length > 0) {
  const node = selection[0]
}
```

### 导出图片
```typescript
const exportSettings = {
  format: 'PNG', // 'PNG' | 'JPG' | 'WEBP' | 'SVG' | 'PDF'
  constraint: { type: 'SCALE', value: 2 }
}
const bytes = await node.exportAsync(exportSettings)
```

### Uint8Array转Base64
```typescript
function arrayBufferToBase64(buffer: Uint8Array): string {
  const bytes = new Uint8Array(buffer)
  const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/'
  let result = ''
  let i = 0
  while (i < bytes.length) {
    const b1 = bytes[i++], b2 = bytes[i++] || 0, b3 = bytes[i++] || 0
    const bitmap = (b1 << 16) | (b2 << 8) | b3
    result += chars.charAt((bitmap >> 18) & 63)
    result += chars.charAt((bitmap >> 12) & 63)
    result += i - 2 < bytes.length ? chars.charAt((bitmap >> 6) & 63) : '='
    result += i - 1 < bytes.length ? chars.charAt(bitmap & 63) : '='
  }
  return result
}
```

### 数据存储
```typescript
// 异步存储
await mg.clientStorage.setAsync('key', value)
const value = await mg.clientStorage.getAsync('key')
await mg.clientStorage.deleteAsync('key')
```

### 通知提示
```typescript
mg.notify('操作成功', { type: 'success', timeout: 3000 })
```

## 调试技巧

1. **重新运行插件**: `Cmd+Option+P` (Mac) / `Ctrl+Alt+P` (Windows)
2. **打开开发者工具**: 插件面板右键 → 检查元素
3. **查看控制台日志**: 在 main.ts 中使用 `console.log()`

## 常见问题

| 问题 | 解决方案 |
|------|---------|
| `mg.currentSelection` 不存在 | 使用 `mg.document.currentPage.selection` |
| `btoa` 未定义 | 使用自定义 base64 编码函数 |
| 导出图片卡住 | 检查选中元素是否有效，添加 try-catch |
| 旧数据不兼容 | 添加数据迁移函数处理新字段 |

## 构建与发布

```bash
# 开发模式
npm run dev

# 生产构建
npm run build

# 导入 MasterGo
# 插件 -> 开发者模式 -> 创建/添加插件 -> 选择 manifest.json
```
