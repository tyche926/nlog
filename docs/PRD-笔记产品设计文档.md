# 笔记产品完整设计方案

---

## 1. 产品概述

### 1.1 业务目标、目标用户、核心价值

**业务目标**：打造一款轻量级、响应式的 Web 端笔记管理平台，支持用户注册登录后创建、编辑、管理个人笔记，提供流畅的写作体验。

**目标用户**：
- 个人用户：日常记录、知识管理、待办事项
- 学生群体：课堂笔记、学习总结
- 职场人士：会议记录、工作日志

**核心价值**：
- 随时随地通过浏览器访问，无需安装客户端
- 响应式设计，适配桌面和平板设备
- 简洁高效的 Markdown 编辑体验，实时预览
- 安全的账户体系，数据云端存储不丢失

### 1.2 整体业务流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                          用户访问首页                                 │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │       是否已登录？              │
                    └───────────────────────────────┘
                         │                │
                      已登录            未登录
                         │                │
                         ▼                ▼
              ┌──────────────────┐  ┌──────────────────────┐
              │   笔记列表页       │  │     登录/注册页        │
              │  (我的笔记首页)    │  └──────────────────────┘
              └──────────────────┘           │
                         │           ┌───────┴───────┐
                         │           ▼               ▼
                         │   ┌────────────┐  ┌────────────┐
                         │   │  用户注册   │  │  用户登录   │
                         │   │ 校验→入库   │  │ 校验→JWT   │
                         │   └─────┬──────┘  └─────┬──────┘
                         │         └───────┬───────┘
                         │                 │ 成功
                         │                 ▼
                         │         ┌──────────────┐
                         │         │ 跳转笔记列表  │
                         │         └──────┬───────┘
                         │                │
                         ▼                ▼
              ┌──────────────────────────────────────┐
              │           笔记列表页                   │
              │  · 左侧：目录树（展开/折叠）            │
              │  · 右侧：当前目录下的笔记 (分页)        │
              │  · 支持搜索/排序/移入回收站              │
              │  · 右键目录：新建/重命名/移入回收站      │
              │  · 点击"新建笔记"或点击已有笔记          │
              │  · 顶部导航：回收站入口                 │
              └──────────────────────────────────────┘
                         │           │
                   点击已有笔记   点击新建笔记
                         │           │
                         ▼           ▼
              ┌──────────────────────────────────────┐
              │           笔记编辑页                   │
              │  · Markdown 编辑器 (左侧编辑)          │
              │  · 实时预览 (右侧渲染)                  │
              │  · 自动保存草稿 (每30秒)               │
              │  · 手动保存 / 返回列表                 │
              └──────────────────────────────────────┘
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
          保存成功    保存失败    返回列表
              │          │          │
              ▼          ▼          ▼
        更新列表    提示错误    确认丢弃
        留在编辑    留在编辑    返回列表
        页继续      页修改      页

回收站流程：
  笔记/目录 → 删除 → 移入回收站(deleted_at=NOW) → 30天倒计时
      回收站中 → 恢复(点击恢复,deleted_at=NULL) → 回到笔记列表
      回收站中 → 彻底删除 → 物理删除数据
      回收站中 → 超过30天 → 定时任务自动物理删除

异常流程：
  · Token 过期 → 自动刷新 Token，刷新失败 → 跳转登录页
  · 网络异常 → 提示"网络连接失败，请检查网络"
  · 服务端 500 → 提示"服务器繁忙，请稍后重试"
  · 并发编辑冲突 → 提示"笔记已被其他设备修改，请刷新后重试"（乐观锁）
  · 未登录访问受保护页面 → 重定向到登录页，登录成功后回跳原页面
```

### 1.3 功能模块分解

| 一级模块 | 二级功能 | 页面路由 | 说明 |
|---------|---------|---------|------|
| 账户模块 | 用户注册 | `/register` | 邮箱+密码注册 |
|  | 用户登录 | `/login` | 邮箱+密码登录，支持"记住我" |
|  | 退出登录 | - | 清除Token，跳转登录页 |
| 目录模块 | 目录树展示 | 笔记列表页左侧栏 | 支持展开/折叠，右键菜单 |
|  | 新建目录 | 右键菜单 / 顶部按钮 | 弹窗输入名称，支持子目录 |
|  | 重命名目录 | 右键菜单 | 弹窗修改名称 |
|  | 移入回收站 | 右键菜单 | 确认后移入回收站，目录下笔记移入"未分类" |
| 笔记模块 | 笔记列表 | `/notes` + `?notebook_id=` | 按目录筛选展示，分页，搜索，排序 |
|  | 新建笔记 | `/notes/new` + `?notebook_id=` | Markdown编辑器，可指定目标目录 |
|  | 编辑笔记 | `/notes/:id/edit` | 加载已有笔记内容，可修改所属目录 |
|  | 笔记详情/预览 | `/notes/:id` | 只读渲染模式 |
|  | 移动笔记 | 笔记卡片拖拽 / 右键菜单 | 将笔记移入指定目录 |
| 回收站模块 | 回收站列表 | `/trash` | 展示已删除的笔记和目录，显示剩余天数 |
|  | 恢复 | 按钮 | 将笔记/目录恢复到原位置 |
|  | 彻底删除 | 按钮 | 物理删除，不可恢复 |
|  | 一键清空 | 按钮 | 彻底删除回收站内所有内容 |
|  | 30天自动清理 | 定时任务 | 每天凌晨3点清理超过30天的内容 |
| 通用 | 404页面 | `*` | 未匹配路由提示 |

---

## 2. 详细 PRD 需求文档

### 2.1 登录页 `/login`

**页面描述**：已注册用户通过邮箱和密码登录系统。

**操作步骤**：
1. 用户访问 `/login`
2. 输入邮箱和密码
3. 可选勾选"记住我"（7天免登录）
4. 点击"登录"按钮
5. 成功后跳转到笔记列表页（或回跳原页面）

**输入项**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| 邮箱 | email | 是 | 标准邮箱格式，最大255字符 |
| 密码 | password | 是 | 6-32位，支持特殊字符 |
| 记住我 | checkbox | 否 | 勾选后 refresh_token 有效期延长至7天 |

**输出展示**：
- 登录成功 → 前端存储 access_token + refresh_token，跳转 `/notes`
- 登录失败 → 在表单顶部显示红色错误提示

**验证规则**：

| 规则 | 提示文案 |
|------|---------|
| 邮箱格式不正确 | "请输入正确的邮箱地址" |
| 密码少于6位 | "密码长度不能少于6位" |
| 密码多于32位 | "密码长度不能超过32位" |
| 邮箱或密码为空 | "请输入邮箱地址" / "请输入密码" |
| 连续5次登录失败 | 锁定账户15分钟，提示"账户已被临时锁定，请15分钟后重试" |

**错误提示**：

| 错误码 | 提示文案 |
|--------|---------|
| 401 | "邮箱或密码错误" |
| 423 | "账户已被临时锁定，请{duration}分钟后重试" |
| 500 | "服务器繁忙，请稍后重试" |

**埋点需求**：
- 页面曝光：`page_view_login`
- 点击登录：`click_login_btn`
- 登录成功：`login_success`
- 登录失败：`login_fail`（含失败原因）

**权限控制**：已登录用户访问登录页 → 自动重定向到 `/notes`

---

### 2.2 注册页 `/register`

**页面描述**：新用户创建账户。

**操作步骤**：
1. 用户访问 `/register`
2. 输入昵称、邮箱、密码、确认密码
3. 点击"注册"按钮
4. 成功后自动登录并跳转到笔记列表页

**输入项**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| 昵称 | text | 是 | 2-20字符，支持中英文数字下划线 |
| 邮箱 | email | 是 | 标准邮箱格式 |
| 密码 | password | 是 | 6-32位 |
| 确认密码 | password | 是 | 必须与密码一致 |

**验证规则**：

| 规则 | 提示文案 |
|------|---------|
| 昵称格式不符 | "昵称为2-20位中英文、数字或下划线" |
| 邮箱格式不正确 | "请输入正确的邮箱地址" |
| 密码少于6位 | "密码长度不能少于6位" |
| 两次密码不一致 | "两次输入的密码不一致" |
| 邮箱已注册 | "该邮箱已被注册" |

**埋点需求**：
- 页面曝光：`page_view_register`
- 点击注册：`click_register_btn`
- 注册成功：`register_success`

**权限控制**：已登录用户访问 → 重定向到 `/notes`

---

### 2.3 笔记列表页 `/notes`（需登录）

**页面描述**：用户登录后的首页。左侧为目录树，右侧为当前选中目录下的笔记列表。

**布局结构**：
```
┌──────────────┬─────────────────────────────────────────┐
│   目录树      │   笔记列表区                              │
│  (240px)     │                                         │
│              │  [搜索框] [排序] [新建笔记]                 │
│  📁 工作      │                                         │
│    📄 周报     │  ┌─────────────────────────────┐       │
│    📄 需求     │  │ 笔记卡片1                   │       │
│  📁 学习      │  │ 摘要...  更新时间   [删除]     │       │
│  📁 个人      │  └─────────────────────────────┘       │
│              │  ┌─────────────────────────────┐       │
│  [+ 新建目录] │  │ 笔记卡片2                   │       │
│              │  └─────────────────────────────┘       │
│              │                                         │
│              │  [分页]                                  │
└──────────────┴─────────────────────────────────────────┘
```

**目录树交互**：
- 默认选中"全部笔记"（根节点），展示所有笔记
- 支持展开/折叠父目录，显示/隐藏子目录和子笔记数
- 右键菜单（或点击目录旁 `···` 图标）：新建子目录、重命名、删除
- 点击目录 → 右侧列表切换为该目录下的笔记（通过 `?notebook_id=` 路由参数）
- 目录右侧显示该目录下的笔记数量 badge

**操作步骤**：
1. 页面加载时请求目录树和默认目录下的笔记列表
2. 左侧目录树：点击目录切换笔记列表
3. 右侧列表：支持搜索、排序、翻页
4. 点击笔记卡片 → 进入编辑页
5. 点击"新建笔记" → 进入新建页（自动带入当前目录）
6. 点击删除 → 弹出确认弹窗 → 确认后删除

**输入项**：

| 操作 | 说明 |
|------|------|
| 搜索框 | 输入关键词，在当前目录下搜索笔记标题 |
| 排序切换 | 按更新时间 / 创建时间 排序 |
| 目录切换 | 左侧目录树，点击切换筛选范围 |
| 分页 | 底部页码，每页10条 |
| 新建按钮 | 跳转 `/notes/new?notebook_id={当前目录ID}` |
| 右键菜单 | 目录节点支持右键：新建子目录 / 重命名 / 删除 |

**输出展示**：
- 笔记卡片列表：标题、内容摘要（前100字符）、所属目录名、更新时间
- 空状态：若所选目录下无笔记，显示"该目录下还没有笔记"
- 若全部笔记为空，显示"还没有笔记，点击创建第一篇"
- 加载中：目录树和列表区分别展示骨架屏

**删除（移入回收站）确认弹窗**：
- 标题："移入回收站"
- 内容："确定要将笔记「{标题}」移入回收站吗？30天后将自动清理。"
- 按钮：「取消」「移入回收站」

**埋点需求**：
- 页面曝光：`page_view_note_list`
- 搜索笔记：`search_notes`
- 新建笔记：`click_create_note`
- 打开笔记：`click_note_card`（含note_id, notebook_id）
- 移入回收站：`trash_note`（含note_id）
- 排序切换：`sort_notes`
- 点击目录：`click_notebook`（含notebook_id）
- 展开/折叠目录：`toggle_notebook`
- 新建目录：`create_notebook`（含parent_id）
- 重命名目录：`rename_notebook`（含notebook_id）
- 删除目录：`delete_notebook`（含notebook_id）
- 移动笔记：`move_note`（含note_id, from_notebook_id, to_notebook_id）

**权限控制**：未登录 → 重定向到 `/login?redirect=/notes`

---

### 2.4 笔记新建/编辑页 `/notes/new` 和 `/notes/:id/edit`（需登录）

**页面描述**：Markdown 编辑器页面，左侧编辑区，右侧实时预览。

**操作步骤**：
1. 进入页面，编辑页加载已有笔记内容
2. 左侧 Markdown 编辑区输入内容
3. 右侧实时渲染预览
4. 点击"保存" → 调用保存接口
5. 自动保存：每30秒自动保存一次草稿（仅本地，不触发接口）
6. 点击"返回" → 若有未保存内容弹出确认

**输入项**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| 标题 | text | 是 | 1-200字符 |
| 内容 | textarea(Markdown) | 否 | 支持 Markdown 语法，最大10000字符 |
| 所属目录 | select | 否 | 下拉选择目录（树形），默认"未分类" |

**输出展示**：
- 右侧实时渲染 Markdown 为 HTML
- 保存成功 → 顶部绿色 Toast "保存成功"
- 保存失败 → 顶部红色 Toast 提示错误信息
- 自动保存 → 左下角显示"已自动保存 HH:mm:ss"

**验证规则**：

| 规则 | 提示文案 |
|------|---------|
| 标题为空 | "请输入笔记标题" |
| 标题超长 | "标题不能超过200字符" |
| 内容超长 | "内容不能超过10000字符" |

**离开确认弹窗**（有未保存内容时）：
- 标题："提示"
- 内容："你有未保存的内容，确定要离开吗？"
- 按钮：「留在页面」「离开页面」

**埋点需求**：
- 页面曝光：`page_view_note_editor`
- 开始编辑：`start_editing`
- 保存笔记：`save_note`（含note_id, is_new）
- 编辑器操作：`format_bold` / `format_italic` / `insert_image` 等

**权限控制**：未登录 → 重定向；非笔记所有者编辑别人笔记 → 403

---

### 2.5 笔记详情页 `/notes/:id`（需登录）

**页面描述**：只读模式查看笔记的 Markdown 渲染内容。

**操作步骤**：
1. 加载笔记内容，渲染为 HTML 展示
2. 提供"编辑"按钮跳转到编辑页

**输出展示**：
- 笔记标题 + 渲染后的正文
- 创建时间、更新时间
- 编辑按钮

**埋点需求**：
- 页面曝光：`page_view_note_detail`

---

### 2.6 目录管理（笔记列表页内嵌，需登录）

**功能描述**：在笔记列表页左侧栏以树形结构管理笔记目录（Notebook），支持多级嵌套。

**目录操作**：

#### 2.6.1 新建目录

**触发方式**：
- 点击目录树顶部 `[+ 新建目录]` 按钮 → 创建根级目录
- 右键某个目录 → 选择"新建子目录" → 在其下创建子目录

**弹窗输入项**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| 目录名称 | text | 是 | 1-30字符，支持中英文数字和常用符号 |

**验证规则**：

| 规则 | 提示文案 |
|------|---------|
| 名称为空 | "请输入目录名称" |
| 名称超长 | "目录名称不能超过30字符" |
| 同父目录下重名 | "该目录下已存在同名目录" |

**输出展示**：创建成功 → 目录树即时刷新，新目录自动展开并高亮

#### 2.6.2 重命名目录

**触发方式**：右键目录 → 选择"重命名"

**弹窗输入项**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| 新名称 | text | 是 | 1-30字符，同上述规则 |

**输出展示**：重命名成功 → 目录树即时更新

#### 2.6.3 移入回收站

**触发方式**：右键目录 → 选择"移入回收站"

**确认弹窗**：
- 标题："移入回收站"
- 内容："确定要将目录「{名称}」移入回收站吗？该目录下的 {N} 篇笔记将移入「未分类」。30天后将自动清理。"
- 按钮：「取消」「移入回收站」

**输出展示**：移入成功 → 目录树即时刷新，自动切换到"全部笔记"视图

**埋点需求**：
- 新建目录：`create_notebook`（含parent_id）
- 重命名目录：`rename_notebook`（含notebook_id）
- 移入回收站：`trash_notebook`（含notebook_id）

---

### 2.7 回收站页 `/trash`（需登录）

**页面描述**：集中管理所有被移入回收站的笔记和目录，支持恢复、彻底删除、一键清空，并展示每条记录的剩余保留天数。

**操作步骤**：
1. 页面加载时请求回收站列表（笔记+目录）
2. 每条记录显示：图标（📝笔记 / 📁目录）、名称、移入回收站时间、剩余天数倒计时
3. 支持按类型（笔记/目录）筛选
4. 点击"恢复" → 笔记/目录恢复到原位置
5. 点击"彻底删除" → 确认后物理删除，不可恢复
6. 点击"一键清空" → 确认后彻底删除回收站内所有内容

**输入项**：

| 操作 | 说明 |
|------|------|
| 类型筛选 | 全部 / 笔记 / 目录 三个Tab切换 |
| 搜索 | 按名称搜索回收站内的笔记/目录 |
| 分页 | 底部页码，每页20条 |

**输出展示**：

| 列 | 说明 |
|------|------|
| 类型 | 图标标识：📝笔记 / 📁目录 |
| 名称 | 笔记标题或目录名 |
| 移入时间 | `deleted_at` 格式化为 `YYYY-MM-DD HH:mm` |
| 剩余天数 | `30 - DATEDIFF(NOW(), deleted_at)` 天，≤3天标红预警 |
| 原位置 | 笔记显示原目录名；目录显示"根目录"或原父目录名 |
| 操作 | 「恢复」「彻底删除」按钮 |

**空状态**：回收站为空时，显示"回收站是空的" + "删除的笔记和目录会在这里保存30天"

**恢复确认**：
- 笔记恢复：笔记直接恢复到原目录下（若原目录已被删除，则恢复至"未分类"）
- 目录恢复：目录恢复到原父目录下（若原父目录已被删除，则恢复至根目录）

**彻底删除确认弹窗**：
- 标题："彻底删除"
- 内容："确定要彻底删除「{名称}」吗？此操作不可撤销。"
- 按钮：「取消」「彻底删除」

**一键清空确认弹窗**：
- 标题："一键清空回收站"
- 内容："确定要彻底删除回收站内所有内容吗？共 {N} 条记录将被永久删除，此操作不可撤销。"
- 按钮：「取消」「全部彻底删除」

**批量操作**（可选）：
- 列表左侧支持多选 Checkbox
- 选中后显示批量操作栏：「批量恢复」「批量彻底删除」

**埋点需求**：
- 页面曝光：`page_view_trash`
- 恢复笔记/目录：`restore_item`（含item_type, item_id）
- 彻底删除：`permanent_delete`（含item_type, item_id）
- 一键清空：`empty_trash`（含count）
- 批量操作：`batch_restore` / `batch_delete`
- 类型筛选：`filter_trash`

**权限控制**：未登录 → 重定向到 `/login?redirect=/trash`

---

## 3. 前端设计解决方案

### 3.1 技术栈选择

| 技术 | 版本 | 选型理由 |
|------|------|---------|
| React | 18.x | 生态成熟，社区活跃，Hooks 开发效率高 |
| TypeScript | 5.x | 类型安全，减少运行时错误，提升可维护性 |
| React Router | v6 | 声明式路由，支持嵌套路由和懒加载 |
| Zustand | 4.x | 轻量状态管理（<1KB），比 Redux 简洁，适合中小项目 |
| Ant Design | 5.x | 企业级UI组件库，表单/表格/弹窗开箱即用 |
| Vite | 5.x | 极速开发服务器，HMR 热更新，构建快 |
| Axios | 1.x | HTTP 请求库，拦截器支持 Token 刷新 |
| marked + highlight.js | - | Markdown 渲染 + 代码高亮 |
| dayjs | 1.x | 轻量日期处理（2KB），替代 moment.js |

**不使用 Uniapp / 小程序**：当前需求仅 Web 端，React + Vite 是最佳选择。后续若有移动端需求可通过 React Native 或 Capacitor 扩展。

### 3.2 页面路由规划与状态管理

**路由表**：

```typescript
// src/router/index.tsx
import { createBrowserRouter } from 'react-router-dom';
import { lazy, Suspense } from 'react';

const Login = lazy(() => import('@/pages/Login'));
const Register = lazy(() => import('@/pages/Register'));
const NoteList = lazy(() => import('@/pages/NoteList'));
const NoteEditor = lazy(() => import('@/pages/NoteEditor'));
const NoteDetail = lazy(() => import('@/pages/NoteDetail'));
const Trash = lazy(() => import('@/pages/Trash'));
const NotFound = lazy(() => import('@/pages/NotFound'));
const AuthGuard = lazy(() => import('@/components/AuthGuard'));

export const router = createBrowserRouter([
  {
    path: '/login',
    element: <Suspense fallback={<PageLoading />}><Login /></Suspense>,
  },
  {
    path: '/register',
    element: <Suspense fallback={<PageLoading />}><Register /></Suspense>,
  },
  {
    path: '/',
    element: <AuthGuard />,
    children: [
      { index: true, element: <Navigate to="/notes" replace /> },
      {
        // 笔记列表页 + 目录管理（内嵌左侧目录树）
        // 支持 ?notebook_id= 查询参数按目录筛选
        path: 'notes',
        element: <Suspense fallback={<PageLoading />}><NoteList /></Suspense>,
      },
      {
        // 新建笔记，支持 ?notebook_id= 指定目标目录
        path: 'notes/new',
        element: <Suspense fallback={<PageLoading />}><NoteEditor /></Suspense>,
      },
      {
        path: 'notes/:id',
        element: <Suspense fallback={<PageLoading />}><NoteDetail /></Suspense>,
      },
      {
        path: 'notes/:id/edit',
        element: <Suspense fallback={<PageLoading />}><NoteEditor /></Suspense>,
      },
      {
        // 回收站页面
        path: 'trash',
        element: <Suspense fallback={<PageLoading />}><Trash /></Suspense>,
      },
    ],
  },
  { path: '*', element: <NotFound /> },
]);
```

**状态管理（Zustand）**：

```typescript
// src/stores/authStore.ts
import { create } from 'zustand';

interface AuthState {
  accessToken: string | null;
  refreshToken: string | null;
  userInfo: { id: number; nickname: string; email: string } | null;
  setAuth: (access: string, refresh: string, user: any) => void;
  clearAuth: () => void;
  isAuthenticated: () => boolean;
}

export const useAuthStore = create<AuthState>((set, get) => ({
  accessToken: localStorage.getItem('access_token'),
  refreshToken: localStorage.getItem('refresh_token'),
  userInfo: JSON.parse(localStorage.getItem('user_info') || 'null'),
  setAuth: (access, refresh, user) => {
    localStorage.setItem('access_token', access);
    localStorage.setItem('refresh_token', refresh);
    localStorage.setItem('user_info', JSON.stringify(user));
    set({ accessToken: access, refreshToken: refresh, userInfo: user });
  },
  clearAuth: () => {
    localStorage.removeItem('access_token');
    localStorage.removeItem('refresh_token');
    localStorage.removeItem('user_info');
    set({ accessToken: null, refreshToken: null, userInfo: null });
  },
  isAuthenticated: () => !!get().accessToken,
}));
```

```typescript
// src/stores/noteStore.ts
import { create } from 'zustand';

interface NoteState {
  currentNote: { id?: number; title: string; content: string; notebook_id?: number } | null;
  isDirty: boolean;
  setCurrentNote: (note: any) => void;
  updateTitle: (title: string) => void;
  updateContent: (content: string) => void;
  markClean: () => void;
  resetNote: () => void;
}

export const useNoteStore = create<NoteState>((set) => ({
  currentNote: null,
  isDirty: false,
  setCurrentNote: (note) => set({ currentNote: note, isDirty: false }),
  updateTitle: (title) => set((s) => ({
    currentNote: s.currentNote ? { ...s.currentNote, title } : null,
    isDirty: true,
  })),
  updateContent: (content) => set((s) => ({
    currentNote: s.currentNote ? { ...s.currentNote, content } : null,
    isDirty: true,
  })),
  markClean: () => set({ isDirty: false }),
  resetNote: () => set({ currentNote: null, isDirty: false }),
}));
```

```typescript
// src/stores/notebookStore.ts
import { create } from 'zustand';

interface Notebook {
  id: number;
  name: string;
  parent_id: number;
  sort_order: number;
  note_count: number;
  children: Notebook[];
}

interface NotebookState {
  tree: Notebook[];
  selectedId: number | null;
  expandedIds: Set<number>;
  loading: boolean;
  setTree: (tree: Notebook[]) => void;
  setSelectedId: (id: number | null) => void;
  toggleExpand: (id: number) => void;
  setLoading: (loading: boolean) => void;
}

export const useNotebookStore = create<NotebookState>((set) => ({
  tree: [],
  selectedId: null,
  expandedIds: new Set(),
  loading: false,
  setTree: (tree) => set({ tree }),
  setSelectedId: (id) => set({ selectedId: id }),
  toggleExpand: (id) => set((s) => {
    const next = new Set(s.expandedIds);
    next.has(id) ? next.delete(id) : next.add(id);
    return { expandedIds: next };
  }),
  setLoading: (loading) => set({ loading }),
}));
```

### 3.3 全局公共组件列表

| 组件名 | 用途 | Props |
|--------|------|-------|
| `AuthGuard` | 路由守卫，校验登录状态 | children |
| `PageLoading` | 路由懒加载时的全屏Loading | - |
| `AppLayout` | 全局布局（Header + 内容区） | children |
| `NoteCard` | 笔记列表卡片 | note: NoteItem, onDelete, onClick, onMove |
| `NotebookTree` | 目录树组件（支持展开/折叠/右键/拖拽） | tree, selectedId, onSelect, onContextMenu |
| `NotebookContextMenu` | 目录右键菜单（新建/重命名/删除） | notebook, x, y, onClose, onCreate, onRename, onDelete |
| `CreateNotebookModal` | 新建/重命名目录弹窗 | open, title, defaultValue, onOk, onCancel |
| `MoveNoteModal` | 移动笔记到目录弹窗（目录树选择器） | open, noteId, onOk, onCancel |
| `TrashTable` | 回收站列表表格（含分页/筛选/多选） | dataSource, loading, onRestore, onDelete, onBatchAction |
| `DaysRemaining` | 剩余天数标签（≤3天标红） | deletedAt |
| `TrashToolbar` | 回收站工具栏（类型筛选Tab + 搜索 + 一键清空按钮） | activeTab, onTabChange, onSearch, onEmpty, trashedCount |
| `EmptyState` | 空状态占位图 | description, actionText, onAction |
| `ErrorBoundary` | React错误边界 | children, fallback |
| `ConfirmModal` | 通用确认弹窗 | open, title, content, onOk, onCancel |
| `SearchInput` | 带防抖的搜索输入框 | value, onChange, placeholder |
| `MarkdownEditor` | Markdown编辑器封装 | value, onChange, placeholder |
| `MarkdownPreview` | Markdown渲染预览 | content |
| `Toast` | 轻提示 | message, type: 'success'\|'error'\|'info' |

### 3.4 交互规范

**加载状态**：
- 页面级：全屏居中 Spin + "加载中..."
- 组件级：骨架屏（Skeleton）模拟内容占位
- 按钮级：loading 属性 + 禁用点击

**空状态**：
- 笔记列表为空 → 居中展示插图 + "还没有笔记" + "创建第一篇" 按钮
- 搜索结果为空 → "未找到相关笔记"

**错误状态**：
- API 请求失败 → 根据错误码显示对应提示
- 网络断开 → 顶部固定 Banner "网络连接已断开，正在尝试重连..."

**表单交互**：
- 输入框失焦时触发校验
- 提交按钮校验通过前 disabled
- 密码输入框支持显示/隐藏切换

**响应式断点**：
- ≥1024px：双栏布局（左侧导航 + 右侧内容）
- 768-1023px：单栏布局，侧边栏折叠
- <768px：移动端布局，编辑页全屏

### 3.5 第三方依赖

```json
{
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "react-router-dom": "^6.23.0",
    "zustand": "^4.5.0",
    "antd": "^5.17.0",
    "@ant-design/icons": "^5.3.0",
    "axios": "^1.7.0",
    "marked": "^12.0.0",
    "highlight.js": "^11.9.0",
    "dayjs": "^1.11.11",
    "dompurify": "^3.1.0"
  },
  "devDependencies": {
    "typescript": "^5.4.0",
    "vite": "^5.2.0",
    "@vitejs/plugin-react": "^4.2.0",
    "@types/react": "^18.3.0",
    "@types/react-dom": "^18.3.0"
  }
}
```

**说明**：
- `dompurify`：渲染 Markdown 前做 XSS 过滤
- `marked`：Markdown 解析
- `highlight.js`：代码块语法高亮
- 不上传/不打印/不涉及证书签名 → 无需上传组件

---

## 4. 后端架构设计

### 4.1 架构模式

**选择：Go + Gin 单体架构**

**选型理由**：
1. 项目已位于 `gopath`，团队技术栈为 Go
2. Gin 是 Go 生态最成熟的 Web 框架，性能优异（比 SpringBoot 吞吐量高3-5倍）
3. 笔记产品功能边界清晰，无复杂业务拆分需求，单体架构足够
4. 单体架构部署简单，运维成本低，后续可按模块拆分为微服务

### 4.2 分层结构

```
nlog/
├── cmd/
│   └── server/
│       └── main.go              # 入口文件
├── internal/
│   ├── config/
│   │   └── config.go            # 配置加载（Viper）
│   ├── middleware/
│   │   ├── auth.go              # JWT 认证中间件
│   │   ├── cors.go              # CORS 跨域
│   │   ├── logger.go            # 请求日志
│   │   ├── ratelimit.go         # API 限流
│   │   ├── recovery.go          # Panic 恢复
│   │   └── request_id.go        # 请求追踪 ID
│   ├── controller/
│   │   ├── auth_controller.go   # 登录/注册/登出
│   │   ├── note_controller.go   # 笔记 CRUD
│   │   ├── notebook_controller.go # 目录 CRUD
│   │   └── trash_controller.go  # 回收站（列表/恢复/彻底删除/清空）
│   ├── service/
│   │   ├── auth_service.go      # 认证业务逻辑
│   │   ├── note_service.go      # 笔记业务逻辑
│   │   ├── notebook_service.go  # 目录业务逻辑
│   │   └── trash_service.go     # 回收站业务逻辑
│   ├── model/
│   │   ├── user.go              # 用户模型
│   │   ├── note.go              # 笔记模型
│   │   └── notebook.go          # 目录模型
│   ├── repository/
│   │   ├── user_repo.go         # 用户数据访问
│   │   ├── note_repo.go         # 笔记数据访问
│   │   ├── notebook_repo.go     # 目录数据访问
│   │   └── trash_repo.go        # 回收站数据访问
│   ├── dto/
│   │   ├── request/             # 请求 DTO
│   │   └── response/            # 响应 DTO
│   ├── pkg/
│   │   ├── jwt/                 # JWT 工具
│   │   ├── hash/                # 密码哈希
│   │   ├── validator/           # 参数校验
│   │   └── errcode/             # 统一错误码
│   └── router/
│       └── router.go            # 路由注册
├── migrations/
│   └── 001_init.sql             # 数据库迁移脚本
├── docs/
│   └── PRD-笔记产品设计文档.md
├── Dockerfile
├── docker-compose.yml
├── go.mod
├── go.sum
└── .env.example
```

### 4.3 全局中间件

**中间件执行顺序**：RequestID → Recovery → Logger → CORS → RateLimit → Auth

```go
// internal/middleware/auth.go - JWT 认证
func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        if token == "" || !strings.HasPrefix(token, "Bearer ") {
            c.AbortWithStatusJSON(401, gin.H{"code": 401, "msg": "未登录或Token已过期"})
            return
        }
        claims, err := jwt.ParseAccessToken(token[7:])
        if err != nil {
            c.AbortWithStatusJSON(401, gin.H{"code": 401, "msg": "Token无效或已过期"})
            return
        }
        // 校验 Redis 中 Token 是否在白名单（支持多设备登录管理）
        exists, _ := redisClient.Exists(ctx, "token:"+claims.TokenID).Result()
        if exists == 0 {
            c.AbortWithStatusJSON(401, gin.H{"code": 401, "msg": "该设备已在其他位置登录"})
            return
        }
        c.Set("user_id", claims.UserID)
        c.Set("user_email", claims.Email)
        c.Next()
    }
}
```

```go
// internal/middleware/ratelimit.go - 令牌桶限流
func RateLimitMiddleware(rate int) gin.HandlerFunc {
    return func(c *gin.Context) {
        // 使用 Redis 实现滑动窗口限流，key = "ratelimit:{ip}:{path}"
        // 默认 100次/分钟
        allowed, _ := redisClient.Eval(ctx, luaScript, ...).Bool()
        if !allowed {
            c.AbortWithStatusJSON(429, gin.H{"code": 429, "msg": "请求过于频繁，请稍后重试"})
            return
        }
        c.Next()
    }
}
```

### 4.4 缓存策略

| 场景 | Key 设计 | 过期时间 | 说明 |
|------|---------|---------|------|
| 在线设备管理 | `token:{token_id}` | 与 accessToken 一致(15min) | 存储 user_id，管理多设备登录 |
| 用户登录失败计数 | `login_fail:{email}` | 15min | 5次失败后锁定 |
| 目录树缓存 | `notebook_tree:{user_id}` | 30min | 用户的目录树结构（JSON），目录增删改时主动失效 |
| 笔记详情缓存 | `note:{note_id}` | 10min | 读多写少，加速访问 |
| API限流计数 | `ratelimit:{ip}:{api}` | 1min | 滑动窗口计数器 |
| 密码重置验证码 | `reset_code:{email}` | 5min | 6位数字验证码 |

### 4.5 定时任务与异步队列

**定时任务**（cron）：
- 清理过期Token记录：每天凌晨2点执行
- 清理登录失败计数：每小时执行
- **回收站自动清理**：每天凌晨3点执行 —— 物理删除 `deleted_at < NOW() - INTERVAL 30 DAY` 的笔记和目录

**回收站清理任务实现**：
```go
// internal/cron/trash_cleanup.go
func TrashCleanupJob(db *gorm.DB) {
    // 清理超过30天的笔记
    db.Unscoped().Where("deleted_at < ?", time.Now().AddDate(0, 0, -30)).Delete(&model.Note{})
    // 清理超过30天的目录
    db.Unscoped().Where("deleted_at < ?", time.Now().AddDate(0, 0, -30)).Delete(&model.Notebook{})
    log.Info("回收站自动清理完成")
}
```

**异步队列**：
- 当前阶段无需消息队列（业务简单，同步处理即可）
- 若后续增加笔记分享/协作功能，引入 Redis Stream 或 RabbitMQ

### 4.6 安全方案

**JWT 双 Token 多设备登录方案**：

```
┌──────────────────────────────────────────────────────────────────┐
│                     JWT 双Token 多设备登录方案                     │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 用户登录成功后，服务端生成：                                   │
│     - access_token (JWT, 有效期 15min, 载荷: user_id,token_id)   │
│     - refresh_token (UUID, 有效期 7天, 载荷: user_id,token_id)   │
│                                                                  │
│  2. 服务端 Redis 存储映射：                                       │
│     - token:{token_id} → {"user_id":1, "device":"Chrome/Windows" │
│       "created_at": timestamp}                                   │
│     - user_tokens:{user_id} → Set{token_id1, token_id2, ...}    │
│                                                                  │
│  3. access_token 过期 → 前端自动用 refresh_token 换新             │
│     POST /api/v1/auth/refresh → 返回新的双Token                  │
│     refresh_token 也过期 → 跳转登录页                             │
│                                                                  │
│  4. 多设备管理：                                                  │
│     - 查询 user_tokens:{user_id} 获取所有设备                     │
│     - 踢出某设备：删除 token:{token_id}，该设备的后续请求返回401  │
│                                                                  │
│  5. "记住我" 功能：                                               │
│     - 勾选 → refresh_token 7天 + 本地 localStorage 存储          │
│     - 未勾选 → refresh_token 24小时                              │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**其他安全措施**：

| 措施 | 实现 |
|------|------|
| API 限流 | Redis 滑动窗口，100次/分钟/IP，登录接口20次/分钟/IP |
| 数据脱敏 | 日志中密码、Token 自动脱敏为 `***` |
| SQL注入防护 | 使用 GORM 参数化查询，禁止拼接 SQL |
| XSS 防护 | 前端 DOMPurify 过滤 Markdown 渲染输出 |
| HTTPS | Nginx 反向代理强制 HTTPS |
| 密码存储 | bcrypt 加盐哈希（cost=12） |
| CORS | 仅允许白名单域名，生产环境严格限制 |
| 请求大小限制 | Body 最大 1MB |

---

## 5. 完整数据库设计

### 5.1 数据库选型

**MySQL 8.0 + InnoDB 引擎**

**选型理由**：
- 笔记产品为典型的关系型数据场景（用户-笔记一对多）
- MySQL 生态成熟，运维工具丰富
- InnoDB 支持行级锁、事务、外键约束
- 当前数据规模预估10万级用户，MySQL 完全胜任

### 5.2 数据表设计

#### 表1：`users` 用户表

```sql
CREATE TABLE `users` (
  `id`              BIGINT UNSIGNED  NOT NULL AUTO_INCREMENT       COMMENT '用户ID',
  `nickname`        VARCHAR(50)      NOT NULL                      COMMENT '昵称',
  `email`           VARCHAR(255)     NOT NULL                      COMMENT '邮箱',
  `password_hash`   VARCHAR(255)     NOT NULL                      COMMENT 'bcrypt密码哈希',
  `avatar_url`      VARCHAR(500)     DEFAULT ''                    COMMENT '头像URL',
  `status`          TINYINT          NOT NULL DEFAULT 1            COMMENT '状态:1正常 2禁用',
  `last_login_at`   DATETIME         DEFAULT NULL                  COMMENT '最后登录时间',
  `created_at`      DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `updated_at`      DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted_at`      DATETIME         DEFAULT NULL                  COMMENT '软删除时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_email` (`email`),
  KEY `idx_status_created` (`status`, `created_at`),
  KEY `idx_deleted_at` (`deleted_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='用户表';
```

#### 表2：`notebooks` 目录表

```sql
CREATE TABLE `notebooks` (
  `id`              BIGINT UNSIGNED  NOT NULL AUTO_INCREMENT       COMMENT '目录ID',
  `user_id`         BIGINT UNSIGNED  NOT NULL                      COMMENT '所属用户ID',
  `parent_id`       BIGINT UNSIGNED  NOT NULL DEFAULT 0            COMMENT '父目录ID，0表示根级目录',
  `name`            VARCHAR(50)      NOT NULL                      COMMENT '目录名称',
  `sort_order`      INT UNSIGNED     NOT NULL DEFAULT 0            COMMENT '排序权重，越小越靠前',
  `note_count`      INT UNSIGNED     NOT NULL DEFAULT 0            COMMENT '该目录下的笔记数量（冗余计数）',
  `status`          TINYINT          NOT NULL DEFAULT 1            COMMENT '状态:1正常 2回收站',
  `created_at`      DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `updated_at`      DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted_at`      DATETIME         DEFAULT NULL                  COMMENT '移入回收站时间(NULL=未删除)',
  PRIMARY KEY (`id`),
  KEY `idx_user_parent_status` (`user_id`, `parent_id`, `status`),
  KEY `idx_user_name` (`user_id`, `name`),
  KEY `idx_deleted_at` (`deleted_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='笔记目录表';
```

#### 表3：`notes` 笔记表

```sql
CREATE TABLE `notes` (
  `id`              BIGINT UNSIGNED  NOT NULL AUTO_INCREMENT       COMMENT '笔记ID',
  `user_id`         BIGINT UNSIGNED  NOT NULL                      COMMENT '所属用户ID',
  `notebook_id`     BIGINT UNSIGNED  NOT NULL DEFAULT 0            COMMENT '所属目录ID，0表示未分类',
  `title`           VARCHAR(200)     NOT NULL DEFAULT ''           COMMENT '笔记标题',
  `content`         TEXT             NOT NULL                      COMMENT '笔记内容(Markdown)',
  `content_html`    MEDIUMTEXT       NOT NULL                      COMMENT '渲染后的HTML(缓存)',
  `summary`         VARCHAR(200)     DEFAULT ''                    COMMENT '内容摘要(前100字符)',
  `status`          TINYINT          NOT NULL DEFAULT 1            COMMENT '状态:1正常 2回收站',
  `version`         INT UNSIGNED     NOT NULL DEFAULT 1            COMMENT '乐观锁版本号',
  `created_at`      DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `updated_at`      DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted_at`      DATETIME         DEFAULT NULL                  COMMENT '移入回收站时间(NULL=未删除)',
  PRIMARY KEY (`id`),
  KEY `idx_user_notebook_status_updated` (`user_id`, `notebook_id`, `status`, `updated_at`),
  KEY `idx_user_title` (`user_id`, `title`),
  KEY `idx_deleted_at` (`deleted_at`),
  FULLTEXT KEY `ft_title_content` (`title`, `content`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='笔记表';
```

#### 表4：`refresh_tokens` 刷新令牌表（可选，也可纯 Redis）

```sql
CREATE TABLE `refresh_tokens` (
  `id`              BIGINT UNSIGNED  NOT NULL AUTO_INCREMENT       COMMENT '记录ID',
  `user_id`         BIGINT UNSIGNED  NOT NULL                      COMMENT '用户ID',
  `token_id`        VARCHAR(64)      NOT NULL                      COMMENT 'Token唯一标识',
  `token_hash`      VARCHAR(255)     NOT NULL                      COMMENT 'refresh_token哈希',
  `device_info`     VARCHAR(255)     DEFAULT ''                    COMMENT '设备信息',
  `expires_at`      DATETIME         NOT NULL                      COMMENT '过期时间',
  `created_at`      DATETIME         NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_token_id` (`token_id`),
  KEY `idx_user_id` (`user_id`),
  KEY `idx_expires_at` (`expires_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='刷新令牌表';
```

### 5.3 数据库设计说明

**分表策略**：
- 当前数据量级不需要分库分表
- 若用户量超过100万，可考虑按 `user_id` 对 `notes` 表进行水平分表

**大字段处理**：
- `content_html` 使用 `MEDIUMTEXT`（最大16MB），支持大笔记
- 列表接口不返回 `content` 和 `content_html`，仅返回 `summary`
- 详情/编辑接口才返回完整内容

**索引优化说明**：
- `notes` 表联合索引 `(user_id, notebook_id, status, updated_at)` 覆盖按目录筛选的列表查询最常用场景
- `notebooks` 表联合索引 `(user_id, parent_id, status)` 支持目录树构建查询（一次查用户所有目录后在内存中构建树）
- `FULLTEXT` 全文索引支持标题+内容搜索，小数据量也可用 `LIKE` 降级
- `users` 表 `uk_email` 保证邮箱唯一，支持登录查询

**目录树构建逻辑**：
- 目录采用邻接表模型（`parent_id`），查询时一次性加载用户所有目录
- 在应用层（Go / 前端均可）递归构建树结构，无需多次数据库查询
- `note_count` 冗余字段避免每次查询子目录时 COUNT 笔记表，提升性能
- 笔记创建/删除/移动时，通过事务同步更新对应目录的 `note_count`

**回收站机制**：
- `deleted_at` 字段即回收站时间戳，`NULL` 表示正常，非 `NULL` 表示在回收站中
- 删除操作（笔记/目录）不会物理删除数据，仅设置 `deleted_at = NOW()`，`status = 2`
- 所有正常查询（列表/详情/目录树）默认过滤 `deleted_at IS NULL`
- 回收站查询使用 `Unscoped()` 获取 `deleted_at IS NOT NULL` 的数据
- 恢复操作：`deleted_at = NULL, status = 1`
- 彻底删除：`DELETE FROM table` 物理删除
- 30天自动清理通过 `idx_deleted_at` 索引高效定位过期数据

---

## 6. RESTful API 文档

### 6.1 通用约定

**Base URL**：`http://localhost:8080/api/v1`

**请求头**：

| Header | 值 | 说明 |
|--------|-----|------|
| Content-Type | application/json | 请求体JSON格式 |
| Authorization | Bearer {access_token} | JWT认证令牌（需登录接口） |
| X-Request-ID | UUID | 请求追踪ID（可选） |

**通用响应格式**：

```json
{
  "code": 0,
  "msg": "success",
  "data": {},
  "request_id": "uuid-string"
}
```

**通用错误码**：

| code | 说明 |
|------|------|
| 0 | 成功 |
| 400 | 参数错误 |
| 401 | 未认证 / Token 过期 |
| 403 | 无权限 |
| 404 | 资源不存在 |
| 409 | 版本冲突（乐观锁） |
| 429 | 请求过于频繁 |
| 500 | 服务器内部错误 |

---

### 6.2 认证模块 API

#### POST /api/v1/auth/register — 用户注册

**Request Body**：
```json
{
  "nickname": "张三",
  "email": "zhangsan@example.com",
  "password": "Abc@123456"
}
```

**Response 201**：
```json
{
  "code": 0,
  "msg": "注册成功",
  "data": {
    "user": {
      "id": 1,
      "nickname": "张三",
      "email": "zhangsan@example.com"
    },
    "access_token": "eyJhbGciOiJIUzI1NiIs...",
    "refresh_token": "d4f3a2b1-c6e5-4f7a-8b9c-0d1e2f3a4b5c",
    "expires_in": 900
  }
}
```

**Error Responses**：
```json
// 400 - 参数校验失败
{ "code": 400, "msg": "昵称不能为空" }

// 400 - 邮箱已注册
{ "code": 400, "msg": "该邮箱已被注册" }
```

---

#### POST /api/v1/auth/login — 用户登录

**Request Body**：
```json
{
  "email": "zhangsan@example.com",
  "password": "Abc@123456",
  "remember_me": true
}
```

**Response 200**：
```json
{
  "code": 0,
  "msg": "登录成功",
  "data": {
    "user": {
      "id": 1,
      "nickname": "张三",
      "email": "zhangsan@example.com"
    },
    "access_token": "eyJhbGciOiJIUzI1NiIs...",
    "refresh_token": "a1b2c3d4-e5f6-7a8b-9c0d-1e2f3a4b5c6d",
    "expires_in": 900
  }
}
```

**Error Responses**：
```json
// 401 - 邮箱或密码错误
{ "code": 401, "msg": "邮箱或密码错误" }

// 423 - 账户锁定
{ "code": 423, "msg": "账户已被临时锁定，请15分钟后重试" }
```

---

#### POST /api/v1/auth/refresh — 刷新Token

**Request Body**：
```json
{
  "refresh_token": "a1b2c3d4-e5f6-7a8b-9c0d-1e2f3a4b5c6d"
}
```

**Response 200**：
```json
{
  "code": 0,
  "msg": "令牌刷新成功",
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIs...",
    "refresh_token": "f9e8d7c6-b5a4-3210-fedc-ba9876543210",
    "expires_in": 900
  }
}
```

---

#### POST /api/v1/auth/logout — 退出登录

**Headers**：`Authorization: Bearer {access_token}`

**Response 200**：
```json
{
  "code": 0,
  "msg": "退出成功",
  "data": null
}
```

---

### 6.3 目录模块 API

#### GET /api/v1/notebooks — 获取目录树

**Headers**：`Authorization: Bearer {access_token}`

**Response 200**：
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "tree": [
      {
        "id": 1,
        "name": "工作",
        "parent_id": 0,
        "sort_order": 0,
        "note_count": 5,
        "children": [
          {
            "id": 3,
            "name": "周报",
            "parent_id": 1,
            "sort_order": 0,
            "note_count": 3,
            "children": []
          }
        ]
      },
      {
        "id": 2,
        "name": "学习",
        "parent_id": 0,
        "sort_order": 1,
        "note_count": 2,
        "children": []
      }
    ]
  }
}
```

---

#### POST /api/v1/notebooks — 创建目录

**Headers**：`Authorization: Bearer {access_token}`

**Request Body**：
```json
{
  "name": "读书笔记",
  "parent_id": 2
}
```

**Response 201**：
```json
{
  "code": 0,
  "msg": "创建成功",
  "data": {
    "id": 4,
    "name": "读书笔记",
    "parent_id": 2,
    "sort_order": 0,
    "created_at": "2026-06-10T16:00:00+08:00"
  }
}
```

**Error**：
```json
// 400 - 同父目录下重名
{ "code": 400, "msg": "该目录下已存在同名目录" }
```

---

#### PUT /api/v1/notebooks/:id — 重命名目录

**Headers**：`Authorization: Bearer {access_token}`

**Request Body**：
```json
{
  "name": "技术学习"
}
```

**Response 200**：
```json
{
  "code": 0,
  "msg": "重命名成功",
  "data": {
    "id": 2,
    "name": "技术学习",
    "updated_at": "2026-06-10T17:00:00+08:00"
  }
}
```

---

#### PUT /api/v1/notebooks/:id/sort — 目录排序（拖拽调整）

**Headers**：`Authorization: Bearer {access_token}`

**Request Body**：
```json
{
  "sort_order": 0,
  "parent_id": 0
}
```

**Response 200**：
```json
{
  "code": 0,
  "msg": "排序成功",
  "data": null
}
```

---

#### DELETE /api/v1/notebooks/:id — 移入回收站

**Headers**：`Authorization: Bearer {access_token}`

**说明**：目录移入回收站时，其下的所有笔记 `notebook_id` 置为 0（未分类），子目录保持不变。

**Response 200**：
```json
{
  "code": 0,
  "msg": "已移入回收站",
  "data": {
    "moved_notes_count": 3
  }
}
```

**Error**：
```json
// 404
{ "code": 404, "msg": "目录不存在" }
```

---

### 6.4 笔记模块 API

#### GET /api/v1/notes — 获取笔记列表

**Headers**：`Authorization: Bearer {access_token}`

**Query Parameters**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| page | int | 否 | 页码，默认1 |
| page_size | int | 否 | 每页条数，默认10，最大50 |
| notebook_id | int | 否 | 目录ID筛选，0=未分类，不传=全部笔记 |
| keyword | string | 否 | 搜索关键词 |
| sort_by | string | 否 | 排序字段:`updated_at`(默认) / `created_at` |
| sort_order | string | 否 | 排序方向:`desc`(默认) / `asc` |

**Response 200**：
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "list": [
      {
        "id": 1,
        "title": "会议记录",
        "summary": "今天讨论了项目进度...",
        "notebook_id": 1,
        "notebook_name": "工作",
        "created_at": "2026-06-10T10:00:00+08:00",
        "updated_at": "2026-06-10T14:30:00+08:00"
      }
    ],
    "pagination": {
      "page": 1,
      "page_size": 10,
      "total": 25,
      "total_pages": 3
    }
  }
}
```

---

#### GET /api/v1/notes/:id — 获取笔记详情

**Headers**：`Authorization: Bearer {access_token}`

**Response 200**：
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "id": 1,
    "title": "会议记录",
    "content": "# 今日会议\n- 讨论项目进度\n- 确定下周计划",
    "content_html": "<h1>今日会议</h1><ul><li>讨论项目进度</li></ul>",
    "notebook_id": 1,
    "notebook_name": "工作",
    "version": 3,
    "created_at": "2026-06-10T10:00:00+08:00",
    "updated_at": "2026-06-10T14:30:00+08:00"
  }
}
```

**Error**：
```json
// 404
{ "code": 404, "msg": "笔记不存在" }
// 403
{ "code": 403, "msg": "无权访问该笔记" }
```

---

#### POST /api/v1/notes — 创建笔记

**Headers**：`Authorization: Bearer {access_token}`

**Request Body**：
```json
{
  "title": "新笔记标题",
  "content": "# 这是我的笔记内容\n支持 Markdown 语法",
  "notebook_id": 0
}
```

**Response 201**：
```json
{
  "code": 0,
  "msg": "创建成功",
  "data": {
    "id": 2,
    "title": "新笔记标题",
    "notebook_id": 0,
    "version": 1,
    "created_at": "2026-06-10T15:00:00+08:00"
  }
}
```

---

#### PUT /api/v1/notes/:id — 更新笔记

**Headers**：`Authorization: Bearer {access_token}`

**Request Body**：
```json
{
  "title": "更新后的标题",
  "content": "# 更新后的内容",
  "notebook_id": 1,
  "version": 1
}
```

**Response 200**：
```json
{
  "code": 0,
  "msg": "更新成功",
  "data": {
    "id": 2,
    "title": "更新后的标题",
    "notebook_id": 1,
    "version": 2,
    "updated_at": "2026-06-10T16:00:00+08:00"
  }
}
```

**Error**：
```json
// 409 - 版本冲突
{ "code": 409, "msg": "笔记已被其他设备修改，请刷新后重试" }
```

---

#### PUT /api/v1/notes/:id/move — 移动笔记到其他目录

**Headers**：`Authorization: Bearer {access_token}`

**Request Body**：
```json
{
  "notebook_id": 3
}
```

**Response 200**：
```json
{
  "code": 0,
  "msg": "移动成功",
  "data": {
    "id": 2,
    "notebook_id": 3,
    "updated_at": "2026-06-10T18:00:00+08:00"
  }
}
```

---

#### DELETE /api/v1/notes/:id — 移入回收站

**Headers**：`Authorization: Bearer {access_token}`

**说明**：笔记移入回收站（软删除），设置 `deleted_at = NOW()`。30天后未恢复将被自动清理。

**Response 200**：
```json
{
  "code": 0,
  "msg": "已移入回收站",
  "data": null
}
```

---

### 6.5 回收站 API

#### GET /api/v1/trash — 获取回收站列表

**Headers**：`Authorization: Bearer {access_token}`

**Query Parameters**：

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| page | int | 否 | 页码，默认1 |
| page_size | int | 否 | 每页条数，默认20，最大50 |
| type | string | 否 | 类型筛选:`note`(笔记) / `notebook`(目录)，不传=全部 |
| keyword | string | 否 | 按名称搜索 |

**Response 200**：
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "list": [
      {
        "type": "note",
        "id": 5,
        "title": "被删除的会议记录",
        "notebook_name": "工作",
        "deleted_at": "2026-05-20T10:00:00+08:00",
        "remaining_days": 19
      },
      {
        "type": "notebook",
        "id": 3,
        "name": "旧项目",
        "parent_name": "工作",
        "deleted_at": "2026-06-01T14:00:00+08:00",
        "remaining_days": 9
      }
    ],
    "pagination": {
      "page": 1,
      "page_size": 20,
      "total": 2,
      "total_pages": 1
    }
  }
}
```

---

#### POST /api/v1/trash/restore — 恢复（批量）

**Headers**：`Authorization: Bearer {access_token}`

**Request Body**：
```json
{
  "items": [
    { "type": "note", "id": 5 },
    { "type": "notebook", "id": 3 }
  ]
}
```

**Response 200**：
```json
{
  "code": 0,
  "msg": "恢复成功",
  "data": {
    "restored_count": 2,
    "failed_count": 0
  }
}
```

**Error**：
```json
// 400 - 部分恢复失败（如原目录已被删除）
{
  "code": 400,
  "msg": "部分项目恢复失败",
  "data": {
    "restored_count": 1,
    "failed_count": 1,
    "failed_items": [
      { "type": "note", "id": 5, "reason": "笔记内容已被清空" }
    ]
  }
}
```

**恢复规则**：
- 笔记恢复：`deleted_at = NULL, status = 1`，若原目录仍在 → 恢复到原 `notebook_id`，若原目录也已被删除 → 恢复到 `notebook_id = 0`（未分类）
- 目录恢复：`deleted_at = NULL, status = 1`，若原父目录仍在 → 恢复原 `parent_id`，若原父目录已被删除 → `parent_id = 0`（根目录）

---

#### DELETE /api/v1/trash/items — 彻底删除（批量）

**Headers**：`Authorization: Bearer {access_token}`

**Request Body**：
```json
{
  "items": [
    { "type": "note", "id": 5 },
    { "type": "notebook", "id": 3 }
  ]
}
```

**Response 200**：
```json
{
  "code": 0,
  "msg": "已彻底删除",
  "data": {
    "deleted_count": 2
  }
}
```

---

#### DELETE /api/v1/trash/empty — 一键清空回收站

**Headers**：`Authorization: Bearer {access_token}`

**Response 200**：
```json
{
  "code": 0,
  "msg": "回收站已清空",
  "data": {
    "deleted_notes_count": 5,
    "deleted_notebooks_count": 2
  }
}
```

**说明**：彻底删除当前用户在回收站中的所有笔记和目录，此操作不可撤销。执行需二次确认（前端弹窗）。

---

## 7. 部署与容器化

### 7.1 前端 Dockerfile

```dockerfile
# Dockerfile.frontend
# =================== 多阶段构建 ===================

# --- Stage 1: Build ---
FROM node:20-alpine AS builder
WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci --production=false

COPY . .
RUN npm run build

# --- Stage 2: Nginx ---
FROM nginx:1.27-alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**Nginx 配置**：

```nginx
# nginx.conf
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    # Gzip 压缩
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml image/svg+xml;
    gzip_min_length 1024;
    gzip_comp_level 6;

    # SPA 路由回退
    location / {
        try_files $uri $uri/ /index.html;
    }

    # API 代理到后端
    location /api/ {
        proxy_pass http://backend:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # 静态资源缓存
    location ~* \.(js|css|png|jpg|jpeg|gif|svg|ico|woff|woff2)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

### 7.2 后端 Dockerfile

```dockerfile
# Dockerfile.backend
# =================== 多阶段构建 ===================

# --- Stage 1: Build ---
FROM golang:1.22-alpine AS builder

RUN apk add --no-cache git ca-certificates tzdata

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-s -w" -o /server ./cmd/server

# --- Stage 2: Runtime ---
FROM alpine:3.20

RUN apk add --no-cache ca-certificates tzdata curl
ENV TZ=Asia/Shanghai

WORKDIR /app
COPY --from=builder /server .
COPY --from=builder /app/.env.example .env

EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD curl -f http://localhost:8080/api/v1/health || exit 1

CMD ["./server"]
```

### 7.3 docker-compose.yml

```yaml
# docker-compose.yml
version: '3.8'

services:
  # ========== MySQL ==========
  mysql:
    image: mysql:8.0
    container_name: nlog-mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:-Root@123456}
      MYSQL_DATABASE: nlog
      MYSQL_USER: nlog_user
      MYSQL_PASSWORD: ${MYSQL_PASSWORD:-Nlog@2026}
      TZ: Asia/Shanghai
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
      - ./migrations:/docker-entrypoint-initdb.d
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --default-authentication-plugin=mysql_native_password
    networks:
      - nlog-network
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ========== Redis ==========
  redis:
    image: redis:7-alpine
    container_name: nlog-redis
    restart: always
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD:-Redis@2026}
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - nlog-network
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ========== Backend ==========
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile.backend
    image: nlog-backend:latest
    container_name: nlog-backend
    restart: always
    environment:
      APP_PORT: 8080
      APP_ENV: production
      DB_HOST: mysql
      DB_PORT: 3306
      DB_USER: nlog_user
      DB_PASSWORD: ${MYSQL_PASSWORD:-Nlog@2026}
      DB_NAME: nlog
      REDIS_HOST: redis
      REDIS_PORT: 6379
      REDIS_PASSWORD: ${REDIS_PASSWORD:-Redis@2026}
      JWT_SECRET: ${JWT_SECRET:-change-me-in-production}
      JWT_ACCESS_EXPIRE: 15m
      JWT_REFRESH_EXPIRE: 168h
      CORS_ORIGINS: ${CORS_ORIGINS:-http://localhost}
    ports:
      - "8080:8080"
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - nlog-network

  # ========== Frontend ==========
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.frontend
    image: nlog-frontend:latest
    container_name: nlog-frontend
    restart: always
    ports:
      - "80:80"
    depends_on:
      - backend
    networks:
      - nlog-network

volumes:
  mysql_data:
    driver: local
  redis_data:
    driver: local

networks:
  nlog-network:
    driver: bridge
```

### 7.4 服务器资源估算

| 组件 | CPU | 内存 | 磁盘 | 说明 |
|------|-----|------|------|------|
| MySQL | 2核 | 4GB | 50GB SSD | 数据存储，预留增长空间 |
| Redis | 1核 | 2GB | 10GB | 缓存 + Session |
| Backend | 2核 | 2GB | 10GB | Go二进制约15MB |
| Frontend (Nginx) | 1核 | 512MB | 5GB | 静态资源 |
| **总计** | **6核** | **8.5GB** | **75GB** | 建议 4核8GB 云服务器起步 |

**并发承载能力估算**：
- 单实例 Go + Gin：约 5000 QPS（简单查询）~ 2000 QPS（含渲染）
- Redis 缓存命中率 80%+ 时，可承载 1000 并发用户
- 数据库连接池 50-100 连接，支撑日活 1万用户
- 水平扩展：后端无状态，可 Nginx 反向代理多实例

---

## 附录：环境变量示例

```bash
# .env.example
APP_PORT=8080
APP_ENV=development

DB_HOST=127.0.0.1
DB_PORT=3306
DB_USER=nlog_user
DB_PASSWORD=Nlog@2026
DB_NAME=nlog

REDIS_HOST=127.0.0.1
REDIS_PORT=6379
REDIS_PASSWORD=Redis@2026

JWT_SECRET=your-jwt-secret-key-change-in-production
JWT_ACCESS_EXPIRE=15m
JWT_REFRESH_EXPIRE=168h

CORS_ORIGINS=http://localhost:5173,http://localhost
LOG_LEVEL=info
```

---

> **文档版本**：v1.2  
> **编写日期**：2026-06-10  
> **更新记录**：
>   - v1.1（2026-06-10）：增加目录管理功能  
>   - v1.2（2026-06-10）：增加回收站功能（30天自动清理、手动恢复/彻底删除/一键清空）  
> **适用范围**：研发团队（前端 + 后端 + 测试 + 运维）
