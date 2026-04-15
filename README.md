# MasterGo 插件开发技能

一个用于开发 MasterGo 设计工具插件的 Qoder Skill，提供完整的项目初始化、开发、构建和发布指南。

## 简介

本 Skill 帮助开发者快速创建 MasterGo 插件，涵盖：
- 项目初始化和配置
- Manifest 配置
- 主线程代码开发
- UI 界面开发（Vue 3）
- 插件与 UI 之间的消息通信
- 图片导出功能
- 数据存储

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

### 2. 核心配置

创建 `manifest.json`：

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

### 3. 开发脚本

在 `package.json` 中添加：

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

### 4. 启动开发

```bash
npm run dev
```

### 5. 构建发布

```bash
npm run build
```

然后在 MasterGo 中导入：
**插件 → 开发者模式 → 创建/添加插件 → 选择 manifest.json**

## 核心功能

### 消息通信

```typescript
// 插件 -> UI
mg.ui.postMessage(data, "*")

// UI -> 插件
parent.postMessage(data, "*")
```

### 导出图片

```typescript
const bytes = await node.exportAsync({
  format: 'PNG',
  constraint: { type: 'SCALE', value: 2 }
})
```

### 数据存储

```typescript
await mg.clientStorage.setAsync('key', data)
const data = await mg.clientStorage.getAsync('key')
```

## 调试技巧

1. **重新运行插件**: `Cmd+Option+P` (Mac) / `Ctrl+Alt+P` (Windows)
2. **打开开发者工具**: 插件面板右键 → 检查元素
3. **查看控制台日志**: 在 main.ts 中使用 `console.log()`

## 相关资源

- [MasterGo 插件开发文档](https://mastergo.com/dev)
- [MasterGo 插件 API 参考](https://mastergo.com/dev/api)

## 许可证

MIT
