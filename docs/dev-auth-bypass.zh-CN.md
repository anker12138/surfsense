# SurfSense：开发模式免密登录（DEV_AUTH_BYPASS）说明

本文记录为开发调试目的所做的认证绕过修改，以及修改后从登录页进入聊天界面的完整流程。

---

## 一、修改背景

原始项目使用 fastapi-users 库进行账号密码登录，密码通过 bcrypt 哈希校验。开发阶段每次都需要注册账号、记住密码，繁琐且低效。

本次修改引入一个环境变量开关 `DEV_AUTH_BYPASS`：开启后，后端接受**任意 email + 任意密码**登录，若该 email 不存在则自动创建账号，无需手动注册。

---

## 二、涉及文件与改动内容

### 1. `surfsense_backend/.env`

```diff
  # Auth
  AUTH_TYPE=LOCAL
  REGISTRATION_ENABLED=TRUE
+ DEV_AUTH_BYPASS=true
```

- `DEV_AUTH_BYPASS=true`：启用开发免密模式
- 生产环境不设置此变量（或设为 `false`）即可恢复正常密码校验

---

### 2. `surfsense_backend/app/config/__init__.py`

```diff
      AUTH_TYPE = os.getenv("AUTH_TYPE")
      REGISTRATION_ENABLED = os.getenv("REGISTRATION_ENABLED", "TRUE").upper() == "TRUE"
+     # Development bypass: when true, any password is accepted (and unknown emails are auto-created)
+     DEV_AUTH_BYPASS = os.getenv("DEV_AUTH_BYPASS", "FALSE").upper() == "TRUE"
```

- 新增配置字段 `DEV_AUTH_BYPASS`，默认值 `False`，通过环境变量控制

---

### 3. `surfsense_backend/app/users.py`

**新增 import：**

```diff
- from fastapi_users import BaseUserManager, FastAPIUsers, UUIDIDMixin, models
+ from fastapi_users import BaseUserManager, FastAPIUsers, UUIDIDMixin, exceptions, models
  ...
  from app.config import config
  from app.db import User, get_user_db
+ from app.schemas import UserCreate
```

**在 `UserManager` 类中覆盖 `authenticate()` 方法：**

```python
async def authenticate(self, credentials) -> User | None:
    """In DEV_AUTH_BYPASS mode, accept any password and auto-create unknown accounts."""
    if config.DEV_AUTH_BYPASS:
        try:
            user = await self.get_by_email(credentials.username)
        except exceptions.UserNotExists:
            user = await self.create(
                UserCreate(
                    email=credentials.username,
                    password="dev-bypass-placeholder",
                    is_verified=True,
                ),
                safe=True,
            )
        return user
    return await super().authenticate(credentials)
```

**逻辑说明：**

| 情况 | 行为 |
|------|------|
| `DEV_AUTH_BYPASS=false`（默认） | 调用 `super().authenticate()`，走正常 bcrypt 校验 |
| `DEV_AUTH_BYPASS=true` + email 已存在 | 直接返回该用户，**跳过密码校验** |
| `DEV_AUTH_BYPASS=true` + email 不存在 | 自动创建账号（`is_verified=True`），然后返回 |

---

## 三、修改后的登录到聊天页面完整流程

### 3.1 流程图

```text
用户在登录页输入任意 email + 任意密码
        │
        ▼
POST /auth/jwt/login  (application/x-www-form-urlencoded)
        │
        ▼
UserManager.authenticate(credentials)
  ├─ DEV_AUTH_BYPASS=true
  │     ├─ email 已存在 → 直接返回 User（跳过密码校验）
  │     └─ email 不存在 → 自动 create() 账号 → 返回 User
  │
  └─ DEV_AUTH_BYPASS=false（正常模式）
        └─ bcrypt 哈希比对 → 通过则返回 User，失败返回 None → 400 错误
        │
        ▼
fastapi-users 生成 JWT Token（有效期 24h）
        │
        ▼
CustomBearerTransport.get_login_response(token)
  └─ AUTH_TYPE=LOCAL → 返回 JSON: { access_token, token_type }
        │
        ▼
前端 LocalLoginForm → loginMutationAtom 收到 token
        │
        ▼
跳转 /auth/callback?token=<JWT>
        │
        ▼
TokenHandler 组件
  ├─ 从 URL 读取 token
  ├─ 写入 localStorage["surfsense_bearer_token"]
  └─ 跳转 /dashboard（或登录前保存的目标路径）
        │
        ▼
/dashboard/layout.tsx
  └─ getBearerToken() 检查 token 存在 → 放行渲染
        │
        ▼
/dashboard/page.tsx
  └─ 展示 Search Space 列表，用户选择一个空间
        │
        ▼
/dashboard/[search_space_id]
  └─ 立即跳转 → /dashboard/[search_space_id]/chats
        │
        ▼
client-layout.tsx
  ├─ 读取 LLM 偏好配置
  ├─ 若 owner 且 onboarding 未完成 → /dashboard/[id]/onboard
  └─ onboarding 已完成 → 正常渲染
        │
        ▼
/dashboard/[search_space_id]/researcher/[[...chat_id]]
  └─ ✅ 真正的 AI 聊天使用界面
        useChat() 调用后端:
        POST {BACKEND_URL}/api/v1/chat
        Authorization: Bearer <JWT>
```

### 3.2 关键代码路径速查

| 步骤 | 文件 |
|------|------|
| 登录表单 | `surfsense_web/app/(home)/login/LocalLoginForm.tsx` |
| 登录 API 调用 | `surfsense_web/lib/apis/auth-api.service.ts` |
| Token 落地 localStorage | `surfsense_web/components/TokenHandler.tsx` |
| Dashboard 鉴权守卫 | `surfsense_web/app/dashboard/layout.tsx` |
| Search Space 选择 | `surfsense_web/app/dashboard/page.tsx` |
| Onboarding 检查 | `surfsense_web/app/dashboard/[search_space_id]/client-layout.tsx` |
| 聊天核心界面 | `surfsense_web/app/dashboard/[search_space_id]/researcher/[[...chat_id]]/page.tsx` |
| **后端认证入口（本次修改）** | `surfsense_backend/app/users.py` → `UserManager.authenticate()` |

---

## 四、如何关闭 DEV_AUTH_BYPASS

编辑 `surfsense_backend/.env`，将以下行删除或改为 `false`：

```env
DEV_AUTH_BYPASS=false
```

重启后端服务即可恢复正常密码校验，不影响已创建的账号数据。

---

## 五、注意事项

- `DEV_AUTH_BYPASS=true` 时，**任何人只要知道已有用户的 email 即可无密码登录该账号**，切勿在生产或联网环境中开启。
- 自动创建的账号 `is_verified=True`，直接具备完整权限，无需走邮件验证流程。
- 同一 email 无论输入什么密码，始终登录同一账号（密码字段被忽略）。
