# Prisma Schema
## Overview
### 多文件Prisma Schema
```
prisma/
├── migrations
├── models
│   ├── posts.prisma
│   ├── users.prisma
│   └── ... other `.prisma` files
└── schema.prisma
```

## Models

### 定义models

```
model comments {
  // Fields
}

// 模型命名惯例（单数形式）并不总是匹配数据库中的表名。数据库中表/集合命名的常用方法是使用复数形式

// 通过使用 @@map 属性，在不重命名数据库底层注释表的情况下遵守命名规范：
model Comment {
  // Fields

  @@map("comments")
}

// @map列名或枚举值，并@@map枚举名
```

### 定义fields

#### Fields 字段速查表

| Name | Type | Scalar vs Relation | Type Modifier | Attributes |
|------|------|--------------------|---------------|------------|
| `id` | `Int` | Scalar | - | `@id` `@default(autoincrement())` |
| `email` | `String` | Scalar | - | `@unique` |
| `name` | `String` | Scalar | `?` | - |
| `role` | `Role` | Scalar (enum) | - | `@default(USER)` |
| `posts` | `Post` | Relation | `[]` | - |
| `profile` | `Profile` | Relation | `?` | - |

#### Scalar fields 
```
model Comment {
  id      Int    @id @default(autoincrement()) 
  title   String
  content String
}

model Tag {
  name String @id
}
```

#### Relation fields
```
model Post {
  id       Int       @id @default(autoincrement())
  // Other fields
  comments Comment[] // A post can have many comments
}

model Comment {
  id     Int
  // Other fields
  post   Post? @relation(fields: [postId], references: [id]) // A comment can have one post
  postId Int?
}
```

### 定义属性attributes
属性会修改场或模型块的行为。以下示例包含三个字段属性（@id、@default和@unique）和一个块属性（@@unique）
```
model User {
  id        Int     @id @default(autoincrement())
  firstName String
  lastName  String
  email     String  @unique
  isAdmin   Boolean @default(false)

  @@unique([firstName, lastName])
}
```

### 定义ID字段
在关系型数据库中，ID可以是单个字段，也可以基于多个字段。如果模型没有@id或@@id，你必须定义一个强制的@unique场或@@unique块。
在 MongoDB 中，ID 必须是一个单一字段，定义一个 @id 属性和一个 @map（“_id”）属性。
```
// 用户ID由id整数字段表示
model User {
  id      Int      @id @default(autoincrement()) 
  email   String   @unique
  name    String?
  role    Role     @default(USER)
  posts   Post[]
  profile Profile?
}

//用户ID由firstname和lastName字段组合表示
model User {
  firstName String
  lastName  String
  email     String  @unique
  isAdmin   Boolean @default(false)

  @@id([firstName, lastName]) 
}

//@unique字段作为唯一标识符
model User {
  email   String   @unique
  name    String?
  role    Role     @default(USER)
  posts   Post[]
  profile Profile?
}
```
### 定义默认值

```
model Post {
  id         Int        @id @default(autoincrement())
  createdAt  DateTime   @default(now()) 
  title      String
  published  Boolean    @default(false) 
  data       Json       @default("{ \"hello\": \"world\" }") 
  author     User       @relation(fields: [authorId], references: [id])
  authorId   Int
  categories Category[] @relation(references: [id])
}
``` 

### 定义唯一域

```
model User {
  id    Int     @id @default(autoincrement())
  email String  @unique
  name  String?
}

model Post {
  id         Int        @id @default(autoincrement())
  createdAt  DateTime   @default(now())
  title      String
  published  Boolean    @default(false)
  author     User       @relation(fields: [authorId], references: [id])
  authorId   Int
  categories Category[] @relation(references: [id])

  @@unique([authorId, title]) 
}
```

### 定义索引
通过模型上的@@index，在模型的一个或多个字段上定义索引
```
model Post {
  id      Int     @id @default(autoincrement())
  title   String
  content String?

  @@index([title, content])
}
```

### 定义枚举
```
model User {
  id    Int     @id @default(autoincrement())
  email String  @unique
  name  String?
  role  Role    @default(USER) 
}

enum Role { 
  USER 
  ADMIN 
} 
```

### 定义复合类型
复合类型目前仅在 MongoDB 上提供。

### 函数
```
model Post {
  id        Int      @id @default(autoincrement())
  createdAt DateTime @default(now())
}
```

### 类型定义
```
export type User = {
  id: number;
  email: string;
  name: string | null;
  role: string;
};
```


## Relations
关系是prisma schema中两个模型之间的连接

### 关系类型
- 一对一
- 一对多
- 多对多

### 一对一
一对一（1-1）关系指的是关系双方最多只能连接一条记录的关系

用@unique属性标记字段，以确保每个配置文件只有一个用户连接。在下面的例子中，用户字段引用了用户模型中的一个电子邮件字段，该字段标记为@unique属性：
```
model User {
  id      Int      @id @default(autoincrement())
  profile Profile?
}

model Profile {
  id     Int  @id @default(autoincrement())
  user   User @relation(fields: [userId], references: [id])
  userId Int  @unique // relation scalar field (used in the `@relation` attribute above)
}
```

#### 复合主键与复合唯一约束的绑定

```
model User {
  firstName String
  lastName  String
  profile   Profile?

  @@id([firstName, lastName])// <== 复合主键
}

model Profile {
  id            Int    @id @default(autoincrement())
  user          User   @relation(fields: [userFirstName, userLastName], references: [firstName, lastName])
  userFirstName String // relation scalar field (used in the `@relation` attribute above)
  userLastName  String // relation scalar field (used in the `@relation` attribute above)
  
  // 用这两个字段，去对接 User 表的 [firstName, lastName]
  @@unique([userFirstName, userLastName])// <== 复合唯一约束
}
```


### 一对多
在 Prisma 中，判断一个关系是不是“一对多”，核心只需要看两个关键点：
- 有没有 [] 数组符号（决定了谁是“一”，谁是“多”）。
- 外键字段上有没有 @unique 约束（决定了它是“一对多”还是“一对一”）。

```
model User {
  id    Int    @id @default(autoincrement())
  posts Post[]
}

model Post {
  id       Int  @id @default(autoincrement())
  author   User @relation(fields: [authorId], references: [id])
  authorId Int
}
```

```
model User {
  id    Int    @id @default(autoincrement())
  email String @unique // <-- add unique attribute
  posts Post[]
}

model Post {
  id          Int    @id @default(autoincrement())
  authorEmail String
  author      User   @relation(fields: [authorEmail], references: [email])
}
```

#### 基于“复合主键”建立的一对多关系

```
model User {
  firstName String
  lastName  String
  post      Post[]

  @@id([firstName, lastName])
}

model Post {
  id              Int    @id @default(autoincrement())
  author          User   @relation(fields: [authorFirstName, authorLastName], references: [firstName, lastName])
  authorFirstName String // relation scalar field (used in the `@relation` attribute above)
  authorLastName  String // relation scalar field (used in the `@relation` attribute above)
}
```

### 多对多

```

```