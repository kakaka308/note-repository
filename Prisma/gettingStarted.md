# 创建一个新项目
1. 创建一个新项目
```
mkdir hello-prisma
cd hello-prisma

// 初始化 TypeScript 项目：
pnpm init
pnpm add typescript tsx @types/node --save-dev
pnpm add -D typescript @types/node
pnpm tsc --init
``` 

2. 安装所需的依赖
```
pnpm add prisma @types/pg --save-dev
pnpm add @prisma/client @prisma/adapter-pg pg dotenv
```

3. 配置ESM支持
```
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "bundler",
    "target": "ES2023",
    "strict": true,
    "esModuleInterop": true,
    "ignoreDeprecations": "6.0"
  }
  // 保持原有配置不变，添加补充
}
```

4. 初始化prisma
```
pnpm dlx prisma
pnpm dlx prisma init --datasource-provider postgresql --output ../generated/prisma
```
.env
```
DATABASE_URL="postgresql://username:password@localhost:5432/mydb?schema=public"
```

5. 定义数据模型
```
generator client {
  provider = "prisma-client"
  output   = "../generated/prisma"
}

datasource db {
  provider = "postgresql"
}

model User { 
  id    Int     @id @default(autoincrement()) 
  email String  @unique
  name  String?
  posts Post[]
} 

model Post { 
  id        Int     @id @default(autoincrement()) 
  title     String
  content   String?
  published Boolean @default(false) 
  author    User    @relation(fields: [authorId], references: [id]) 
  authorId  Int
} 
```

6. 创建并应用首次迁移
```
 // 创建数据库表
pnpm dlx prisma migrate dev --name init
```

```
// 生成Prisma客户端
pnpm dlx prisma generate
```

7. 实例化 PrismaClient
```
// lib/prisma.ts
import "dotenv/config";
import { PrismaPg } from "@prisma/adapter-pg";
import { PrismaClient } from "../generated/prisma/client";

const connectionString = `${process.env.DATABASE_URL}`;

const adapter = new PrismaPg({ connectionString });
const prisma = new PrismaClient({ adapter });

export { prisma };
```

8. 写下第一个查询
```
// script.ts
import { prisma } from "./lib/prisma";

async function main() {
  // Create a new user with a post
  const user = await prisma.user.create({
    data: {
      name: "Alice",
      email: "alice@prisma.io",
      posts: {
        create: {
          title: "Hello World",
          content: "This is my first post!",
          published: true,
        },
      },
    },
    include: {
      posts: true,
    },
  });
  console.log("Created user:", user);

  // Fetch all users with their posts
  const allUsers = await prisma.user.findMany({
    include: {
      posts: true,
    },
  });
  console.log("All users:", JSON.stringify(allUsers, null, 2));
}

main()
  .then(async () => {
    await prisma.$disconnect();
  })
  .catch(async (e) => {
    console.error(e);
    await prisma.$disconnect();
    process.exit(1);
  });
  ```
  Run the script:
  ```bash
  pnpm dlx tsx script.ts
  ```

  9. Prisma Studio
  ```
  npx prisma studio
  ```

