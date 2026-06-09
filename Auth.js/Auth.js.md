# Auth\.js

# 安装

## 安装包

```TypeScript
pnpm add next-auth@beta
```

## 设置环境

```TypeScript
npx auth secret
```

## 配置

Next\.js

1. 首先在应用根创建一个新的auth\.ts文件

```TypeScript
import NextAuth from "next-auth"
 
export const { handlers, signIn, signOut, auth } = NextAuth({
  providers: [],
})
```

2. Add a Route Handler under

```JavaScript
import { handlers } from "@/auth" // Referring to the auth.ts we just created
export const { GET, POST } = handlers
```

3. 添加可选代理以保持会话存活，每次调用时会话都会更新

```TypeScript
export { auth as proxy } from "@/auth"
```

## 设置认证方法



---

# 认证

Auth\.js 支持四种主要的认证范式

## OAuth

### 在GitHub仪表盘注册OAuth App

在GitHub注册OAuth应用时，它们都会要求输入应用的回调URL。

**Next\.js Callback URL: **`[origin]/api/auth/callback/github`

许多运营商一次只允许注册一个回拨URL。因此，如果想为开发和生产环境提供活跃的 OAuth 配置，需要在 GitHub 仪表盘注册第二个 OAuth 应用来管理其他环境。

### 设置环境变量

客户端ID和客户端SECRET添加到应用环境文件中：

```Plain Text
AUTH_GITHUB_ID={CLIENT_ID}
AUTH_GITHUB_SECRET={CLIENT_SECRET}
```

### 设置提供商

让我们在Auth\.js配置中启用GitHub作为登录选项。需要从包中导入GitHub provider，并传递到我们之前在Auth\.js配置文件中设置的提供者数组：

```JavaScript
import NextAuth from "next-auth"
import GitHub from "next-auth/providers/github"
 
export const { handlers, signIn, signOut, auth } = NextAuth({
  providers: [GitHub],
})
```

添加 NextAuth 返回到`api/auth/[...NextAuth]/route.ts` 文件，这样Auth\.js就能对任何收到的请求运行。

```TypeScript
import { handlers } from "@/auth"
export const { GET, POST } = handlers
```

### 添加登录按钮

```JavaScript

"use client"

import { signIn } from "next-auth/react"
 
export default function SignIn() {
  return <button onClick={() => signIn("github")}></button>
}

```

## Magic Links \(Email Provider\)

该登录机制始于用户在登录表单中提供电子邮件地址。然后，一个验证令牌会发送到所提供的邮箱地址。用户有24小时内点击邮件正文中的链接“消费”该令牌并注册账户，否则验证令牌将失效，必须申请新的。

电子邮件提供商可以同时使用JSON Web Tokens和数据库会话，无论选择哪种方式，都必须配置数据库，这样Auth\.js才能保存验证令牌，并在用户尝试登录时查找。没有数据库，无法启用电子邮件服务提供商。

### Resend Setup 重送机制

#### 数据库适配器

无密码登录需要数据库，因为需要存储验证令牌。

#### 设置环境变量

格式像上面示例那样，Auth\.js会自动识别这些信息。如果需要，也可以给环境变量用不同的名称，但那样就得手动把它们传给提供者。

```TypeScript
AUTH_RESEND_KEY=abc123
```

#### 设置提供商

在Auth\.js配置中启用重送作为登录选项。需要从包中导入Resend提供者，并传递到我们之前在Auth\.js配置文件中设置的提供者数组：

```JavaScript
import NextAuth from "next-auth"
import Resend from "next-auth/providers/resend"
 
export const { handlers, auth, signIn, signOut } = NextAuth({
  providers: [Resend],
})
```

#### 添加登录按钮

```JavaScript
"use client"
import { signIn } from "next-auth/react"
 
export function SignIn() {
  const resendAction = (formData: FormData) => {
    signIn("resend", formData)
  }
 
  return (
    <form action={resendAction}>
      <label htmlFor="email-resend">
        Email
        <input type="email" id="email-resend" name="email" />
      </label>
      <input type="submit" value="Signin with Resend" />
    </form>
  )
}
```

## Credentials 资历（略）

要设置Auth\.js任何外部认证机制，或者使用传统的用户名/邮箱和密码流程，可以使用凭证提供者。该提供商旨在将输入登录表单中的任何凭据（即用户名/密码，但不限于此）转发到认证服务。

[Credentials](https://authjs.dev/getting-started/authentication/credentials) （Auth\.js官网对credentials的解释）

## WebAuthn 通行秘钥

### 安装依赖

```TypeScript
pnpm add @simplewebauthn/server@9.0.3 @simplewebauthn/browser@9.0.1v
```

### 应用所需的模式迁移

通行密钥提供者需要一个额外的表，称为认证器

```TypeScript
-- CreateTable
CREATE TABLE "Authenticator" (
    "credentialID" TEXT NOT NULL,
    "userId" TEXT NOT NULL,
    "providerAccountId" TEXT NOT NULL,
    "credentialPublicKey" TEXT NOT NULL,
    "counter" INTEGER NOT NULL,
    "credentialDeviceType" TEXT NOT NULL,
    "credentialBackedUp" BOOLEAN NOT NULL,
    "transports" TEXT,
    PRIMARY KEY ("userId", "credentialID"),
    CONSTRAINT "Authenticator_userId_fkey" FOREIGN KEY ("userId") REFERENCES "User" ("id") ON DELETE CASCADE ON UPDATE CASCADE
);
 
 
-- CreateIndex
CREATE UNIQUE INDEX "Authenticator_credentialID_key" ON "Authenticator"("credentialID");
```

### 更新Auth\.js配置

```JavaScript
import Passkey from "next-auth/providers/passkey"
import { PrismaAdapter } from "@auth/prisma-adapter"
import { PrismaClient } from "@prisma/client"
 
const prisma = new PrismaClient()
 
export default {
  adapter: PrismaAdapter(prisma),
  providers: [Passkey],
  experimental: { enableWebAuthn: true },
}
```

### 自定义页面

如果使用自定义登录页面，可以使用 next\-auth signIn 功能，通过以下代码发起 WebAuthn 注册和登录流程。使用 WebAuthn 的 signIn 功能时，还需要安装 @simplewebauth/浏览器对等依赖功能。

```JavaScript
"use client"
 
import { useSession } from "next-auth/react"
import { signIn } from "next-auth/webauthn"
 
export default function Login() {
  const { data: session, update, status } = useSession()
 
  return (
    <div>
      {status === "authenticated" ? (
        <button onClick={() => signIn("passkey", { action: "register" })}>
          Register new Passkey
        </button>
      ) : status === "unauthenticated" ? (
        <button onClick={() => signIn("passkey")}>Sign in with Passkey</button>
      ) : null}
    </div>
  )
}
```

---

# 数据库适配器

Auth\.js集成默认会将会话保存在 Cookie 中。建立数据库是可选的。如果想在自己的数据库中持久化用户信息，或者想实现某些流程，就需要使用数据库适配器。

数据库适配器是我们用来连接Auth\.js与数据库的桥梁。

## 官方适配器

### pg

https://authjs\.dev/getting\-started/adapters/pg

### Prisma

https://authjs\.dev/getting\-started/adapters/prisma

---

# Session Managment

## 登录和登出处理

### 登录

登录用户时，确保至少设置一种认证方式。需要构建一个按钮，调用Auth\.js框架包中的登录函数。

```JavaScript
"use client"
import { signIn } from "next-auth/react"
 
export function SignIn() {
  return <button onClick={() => signIn()}>Sign In</button>
}
```

也可以把某个提供商传给登录功能，登录功能会尝试直接登录该服务商。否则，当你在应用中点击这个按钮时，用户将被重定向到已配置的登录页面。如果没有设置自定义登录页面，用户将被重定向到默认登录页面 /\[basePath\]/signin。

认证完成后，用户将被重定向到他们开始登录的页面。如果你希望用户登录后被重定向到其他地方（\.i\.如 /dashboard），可以通过登录选项中的目标 URL 作为 redirectTo 来实现。

```JavaScript
"use client"
import { signIn } from "next-auth/react"
 
export function SignIn() {
  return (
    <button onClick={() => signIn("github", { redirectTo: "/dashboard" })}>
      Sign In
    </button>
  )
}
```

### 登出

```JavaScript
"use client"
import { signOut } from "next-auth/react"
 
export function SignOut() {
  return <button onClick={() => signOut()}>Sign Out</button>
}
```

## 获取会话　Get Session

一旦用户登录，通常需要获取会话对象以某种方式使用数据。一个常见的用例是展示他们的头像或其他用户信息。

```JavaScript
import { auth } from "../auth"
 
export default async function UserAvatar() {
  const session = await auth()
 
  if (!session?.user) return null
 
  return (
    <div>
      <img src={session.user.image} alt="User Avatar" />
    </div>
  )
}
```

## Protecting Resources

保护路由通常可以通过检查该会话，并在找不到活跃会话时采取行动，比如将用户重定向到登录页面，或简单返回401：未认证响应。

### pages

使用从 `NextAuth（）` 返回并导出auth\.ts或配置文件中的 auth 函数，auth\.js 配置文件来获取会话对象。

```JavaScript
import { auth } from "@/auth"
 
export default async function Page() {
  const session = await auth()
  if (!session) return <div>Not authenticated</div>
 
  return (
    <div>
      <pre>{JSON.stringify(session, null, 2)}</pre>
    </div>
  )
}
```

### API 路由

在Next\.js中，可以用认证函数来包装API路由处理器。请求参数上会有一个认证密钥，可以检查会话是否有效。

```TypeScript
import { auth } from "@/auth"
import { NextResponse } from "next/server"
 
export const GET = auth(function GET(req) {
  if (req.auth) return NextResponse.json(req.auth)
  return NextResponse.json({ message: "Not authenticated" }, { status: 401 })
})
```

### Next\.js Proxy

```JavaScript
import { auth } from "@/auth"
 
export const proxy = auth((req) => {
  if (!req.auth && req.nextUrl.pathname !== "/login") {
    const newUrl = new URL("/login", req.nextUrl.origin)
    return Response.redirect(newUrl)
  }
})
```

```JavaScript
import NextAuth from "next-auth"
 
export const { auth, handlers } = NextAuth({
  callbacks: {
    authorized: async ({ auth }) => {
      // Logged in users are authenticated, otherwise redirect to login page
      return !!auth
    },
  },
})
```

## 自定义页面

```JavaScript
import { NextAuth } from "next-auth"
import GitHub from "next-auth/providers/github"
 
// Define your configuration in a separate variable and pass it to NextAuth()
// This way we can also 'export const config' for use later
export const config = {
  providers: [GitHub],
  pages: {
    signIn: "/login",
  },
}
 
export const { signIn, signOut, handle } = NextAuth(config)
```



