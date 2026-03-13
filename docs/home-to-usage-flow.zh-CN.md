# SurfSense 前端：从首页到真正使用界面的源码流程

本文基于 `surfsense_web` 源码，梳理本地访问 `http://localhost:3000` 后，用户从首页点击 **Get Started / Sign In** 到进入可实际使用的业务界面的完整代码路径。

---

## 1. 首页入口（localhost:3000）

### 1.1 首页路由
- 文件：`surfsense_web/app/(home)/page.tsx`
- 该页面渲染 `HeroSection`、功能区块等首页内容。

### 1.2 Get Started 按钮
- 文件：`surfsense_web/components/homepage/hero-section.tsx`
- Hero 区域按钮：
  - `Link href="/login"`
  - 文案：`Get Started`

### 1.3 Sign In 按钮
- 文件：`surfsense_web/components/homepage/navbar.tsx`
- 桌面端与移动端都存在：
  - `Link href="/login"`
  - 文案：`Sign In`

> 结论：从首页进入系统的两个核心按钮，都会跳转到 `/login`。

---

## 2. 登录页分流（/login）

### 2.1 登录页主逻辑
- 文件：`surfsense_web/app/(home)/login/page.tsx`
- 核心逻辑：读取环境变量 `NEXT_PUBLIC_FASTAPI_BACKEND_AUTH_TYPE`。
  - 如果是 `GOOGLE`：渲染 `GoogleLoginButton`
  - 否则：渲染 `LocalLoginForm`

### 2.2 Google 登录分支
- 文件：`surfsense_web/app/(home)/login/GoogleLoginButton.tsx`
- 点击按钮后：
  1. 调用后端：`GET {NEXT_PUBLIC_FASTAPI_BACKEND_URL}/auth/google/authorize`
  2. 获取 `authorization_url`
  3. `window.location.href = authorization_url` 跳转到 OAuth 授权链路

### 2.3 本地账号密码登录分支
- 入口文件：`surfsense_web/app/(home)/login/LocalLoginForm.tsx`
- 调用链：
  1. `LocalLoginForm` 提交表单
  2. 调用 `loginMutationAtom`（`surfsense_web/atoms/auth/auth-mutation.atoms.ts`）
  3. `loginMutationAtom` 调用 `authApiService.login`（`surfsense_web/lib/apis/auth-api.service.ts`）
  4. `authApiService.login` 请求后端 `POST /auth/jwt/login`
  5. 成功后前端跳转：`/auth/callback?token=<access_token>`

---

## 3. 登录回调与 Token 落地（/auth/callback）

### 3.1 回调页
- 文件：`surfsense_web/app/auth/callback/page.tsx`
- 渲染 `TokenHandler`，参数中指定：
  - `redirectPath="/dashboard"`
  - `tokenParamName="token"`
  - `storageKey="surfsense_bearer_token"`

### 3.2 TokenHandler 行为
- 文件：`surfsense_web/components/TokenHandler.tsx`
- 核心流程：
  1. 从 URL 参数读取 token
  2. 写入 `localStorage`（`surfsense_bearer_token`）
  3. 读取并清理“登录前想访问的路径”（`getAndClearRedirectPath`）
  4. 若存在历史目标路径则回跳，否则跳 `/dashboard`

### 3.3 认证工具函数
- 文件：`surfsense_web/lib/auth-utils.ts`
- 关键能力：
  - `getBearerToken` / `setBearerToken`
  - `redirectToLogin`
  - `authenticatedFetch`（401 时统一触发回登录页）

---

## 4. Dashboard 第一层入口（/dashboard）

### 4.1 Dashboard Layout 的鉴权
- 文件：`surfsense_web/app/dashboard/layout.tsx`
- 页面挂载时执行：
  1. `getBearerToken()` 检查本地 token
  2. 无 token 则 `redirectToLogin()`
  3. 有 token 才放行渲染子页面

### 4.2 Dashboard 列表页
- 文件：`surfsense_web/app/dashboard/page.tsx`
- 页面主要功能：
  - 拉取用户 Search Spaces（`useSearchSpaces`）
  - 让用户选择一个空间进入
- 典型入口链接：`/dashboard/${space.id}/documents`（在空间卡片上）

> 这一步是“进入具体业务空间”的门户页，不是最终聊天使用页。

---

## 5. 进入具体 Search Space 后的分支

### 5.1 `/dashboard/[search_space_id]` 的立即跳转
- 文件：`surfsense_web/app/dashboard/[search_space_id]/page.tsx`
- 挂载后直接：
  - `router.push(`/dashboard/${search_space_id}/chats`)`

### 5.2 Search Space 级别布局与 Onboarding 判断
- 布局文件：`surfsense_web/app/dashboard/[search_space_id]/layout.tsx`
- 客户端布局：`surfsense_web/app/dashboard/[search_space_id]/client-layout.tsx`
- 关键逻辑（在 `client-layout.tsx`）：
  1. 读取 LLM 偏好配置（`useLLMPreferences`）
  2. 读取用户在该空间的权限（`useUserAccess`）
  3. 如果是 owner 且 onboarding 未完成，则强制跳转：
     - `/dashboard/${searchSpaceId}/onboard`

### 5.3 Onboarding 页面
- 文件：`surfsense_web/app/dashboard/[search_space_id]/onboard/page.tsx`
- 功能：
  - 配置 LLM
  - 可跳转 Team / Sources / Start Chatting
- 当检测到初始即已完成 onboarding 时，会回跳 `/dashboard/${searchSpaceId}`。

---

## 6. 真正的“使用界面”在哪里？

从当前代码看，**真正执行 AI 对话与检索的核心界面**是：

- `surfsense_web/app/dashboard/[search_space_id]/researcher/[[...chat_id]]/page.tsx`

该页使用 `useChat` 调用后端：
- API：`{NEXT_PUBLIC_FASTAPI_BACKEND_URL}/api/v1/chat`
- 携带 Bearer Token
- 处理消息流、会话状态、连接源等

因此，业务上的“真正使用界面”可定义为 **Researcher Chat 页面**。

---

## 7. 一图总结（路由流程）

```text
/ (Home)
  └─ 点击 Get Started / Sign In
      └─ /login
          ├─ GOOGLE: /auth/google/authorize -> OAuth -> /auth/callback?token=...
          └─ LOCAL:  /auth/jwt/login -> /auth/callback?token=...
              └─ TokenHandler 写入 localStorage
                  └─ /dashboard (或登录前保存的目标路径)
                      └─ 选择 search space
                          └─ /dashboard/[search_space_id]
                              └─ /dashboard/[search_space_id]/chats
                                  ├─ 若 owner 且未完成 LLM onboarding
                                  │   └─ /dashboard/[search_space_id]/onboard
                                  └─ 进入 researcher 使用页
                                      └─ /dashboard/[search_space_id]/researcher/[[...chat_id]]
```

---

## 8. 关键结论

1. 首页两个入口按钮都统一走 `/login`。
2. 登录成功后统一通过 `/auth/callback` 落 token，再进入 `/dashboard`。
3. `middleware.ts` 当前是空透传，不参与登录拦截。
4. 实际鉴权由前端页面逻辑（`dashboard/layout.tsx` + `auth-utils.ts`）完成。
5. 真正承载 AI 问答能力的核心页面是 `researcher/[[...chat_id]]/page.tsx`。
