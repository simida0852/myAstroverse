---
title: 使用NestJS的API-02
slug: 使用NestJS的API-02
pubDate: 2022-05-30
tags:
  - 后端
  - Nestjs
  - 教程
description: 使用NestJS的API-02
author: xf
cover: src/images/cat-1.webp
coverAlt: Nestjs
category:
  - 后端
---

学习如何创建 API 的下一个重要事项是如何存储数据。在这篇文章中，我们将探讨如何使用 PostgreSQL 和 NestJS 来做到这一点。为了使数据库的管理更加方便，我们使用了一个名为 TypeORM 的对象关系映射（ORM）工具。为了更好地理解，我们还将研究一些 SQL 查询。通过这样做，我们可以掌握 ORM 给我们带来的优势。

你可以在这个仓库中找到以下所有代码。

<a name="6ed1b6f6"></a>
### 创建一个 PostgreSQL 数据库

使用 Docker 启动我们的开发是最直接的方法。这里我们使用的设置与 TypesScript Express 系列中的相同。

首先需要安装 Docker 和 Docker Compose。现在我们需要创建一个 docker-compose 文件并运行它。

```yaml
version: "3"
services:
  postgres:
    container_name: postgres
    image: postgres:latest
    ports:
    - "5432:5432"
    volumes:
    - /data/postgres:/data/postgres
    env_file:
    - docker.env
    networks:
    - postgres

  pgadmin:
    links:
    - postgres:postgres
    container_name: pgadmin
    image: dpage/pgadmin4
    ports:
    - "8080:80"
    volumes:
    - /data/pgadmin:/root/.pgadmin
    env_file:
    - docker.env
    networks:
    - postgres

networks:
  postgres:
    driver: bridge
```

上面配置的有用之处在于它还启动了一个 pgAdmin 控制台。这使我们有可能查看数据库的状态并与之交互。

为了提供我们的 Docker 容器使用的凭据，我们需要创建 `docker.env` 文件。你可能希望通过将其添加到 `.gitignore` 来跳过提交它。

```
POSTGRES_USER=admin
POSTGRES_PASSWORD=admin
POSTGRES_DB=nestjs
PGADMIN_DEFAULT_EMAIL=admin@admin.com
PGADMIN_DEFAULT_PASSWORD=admin
```

一旦以上所有设置完成，我们需要启动容器：

```
docker-compose up
```

<a name="3867e350"></a>
### 环境变量

运行我们的应用程序的一个关键事项是设置环境变量。通过使用它们来保存配置数据，我们可以使其易于配置。同时，也更容易防止敏感数据被提交到仓库。

在 Express TypeScript 系列中，我们使用了一个名为 dotenv 的库来注入我们的变量。在 NestJS 中，我们有一个我们可以在应用程序中使用的 `ConfigModule`。它在底层使用 dotenv。

```
npm install @nestjs/config
```

app.module.ts

```typescript
import { Module } from '@nestjs/common';
import { PostsModule } from './posts/posts.module';
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [PostsModule, ConfigModule.forRoot()],
  controllers: [],
  providers: []
})
export class AppModule {}
```

一旦我们在应用程序的根目录下创建了 `.env` 文件，NestJS 就会将它们注入到我们很快就会使用的 ConfigSerivice 中。

```
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_USER=admin
POSTGRES_PASSWORD=admin
POSTGRES_DB=nestjs
PORT=5000
```

<a name="bcbd3025"></a>
### 验证环境变量

在运行应用程序之前验证我们的环境变量是一个绝佳的想法。在 TypeScript Express 系列中，我们使用了一个名为 envalid 的库。

NestJS 内置的 `ConfigModule` 支持 `@hapi/joi`，我们可以使用它来定义验证模式。

```
npm install @hapi/joi @types/hapi__joi
```

app.module.ts

```typescript
import { Module } from '@nestjs/common';
import { PostsModule } from './posts/posts.module';
import { ConfigModule } from '@nestjs/config';
import * as Joi from '@hapi/joi';

@Module({
  imports: [
    PostsModule,
    ConfigModule.forRoot({
      validationSchema: Joi.object({
        POSTGRES_HOST: Joi.string().required(),
        POSTGRES_PORT: Joi.number().required(),
        POSTGRES_USER: Joi.string().required(),
        POSTGRES_PASSWORD: Joi.string().required(),
        POSTGRES_DB: Joi.string().required(),
        PORT: Joi.number()
      })
    })
  ],
  controllers: [],
  providers: []
})
export class AppModule {}
```

<a name="fe8eb24a"></a>
### 将 NestJS 应用程序与 PostgreSQL 连接

一旦我们的数据库运行起来，首先要做的事情就是定义我们的应用程序与数据库之间的连接。为此，我们使用 `TypeOrmModule`。

```
npm install @nestjs/typeorm typeorm pg
```

为了保持我们的代码整洁，我建议创建一个数据库模块。

database.module.ts

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ConfigModule, ConfigService } from '@nestjs/config';

@Module({
  imports: [
    TypeOrmModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (configService: ConfigService) => ({
        type: 'postgres',
        host: configService.get('POSTGRES_HOST'),
        port: configService.get('POSTGRES_PORT'),
        username: configService.get('POSTGRES_USER'),
        password: configService.get('POSTGRES_PASSWORD'),
        database: configService.get('POSTGRES_DB'),
        entities: [
          __dirname + '/../**/*.entity.ts'
        ],
        synchronize: true
      })
    })
  ]
})
export class DatabaseModule {}
```

上面的 `synchronize` 标志非常重要。我们将在后面详细讨论它。

最重要的一点是我们使用了 `ConfigModule` 和 `ConfigService`。`useFactory` 方法能够访问环境变量，这要归功于提供的 `imports` 和 `inject` 数组。我们将在本系列的后续部分中详细讨论这些机制。

现在我们需要导入我们的 DatabaseModule。

app.module.ts

```typescript
import { Module } from '@nestjs/common';
import { PostsModule } from './posts/posts.module';
import { ConfigModule } from '@nestjs/config';
import * as Joi from '@hapi/joi';
import { DatabaseModule } from './database/database.module';

@Module({
  imports: [
    PostsModule,
    ConfigModule.forRoot({
      validationSchema: Joi.object({
        POSTGRES_HOST: Joi.string().required(),
        POSTGRES_PORT: Joi.number().required(),
        POSTGRES_USER: Joi.string().required(),
        POSTGRES_PASSWORD: Joi.string().required(),
        POSTGRES_DB: Joi.string().required(),
        PORT: Joi.number()
        })
        }),
        DatabaseModule
        ],
        controllers: [],
        providers: []
        })
export class AppModule {}
```

<a name="808e5669"></a>
### 实体

使用 TypeORM 时，最关键的概念是**实体**。它是一个映射到数据库表的类。我们使用 `@Entity()` 装饰器来创建它。

post.entity.ts
```typescript

import { Column, Entity, PrimaryGeneratedColumn } from 'typeorm';

@Entity()
  class Post {
    @PrimaryGeneratedColumn()
    public id: number;

    @Column()
    public title: string;

    @Column()
    public content: string;
  }

export default Post;
```

TypeORM 与 TypeScript 的良好集成是一个优点，因为它是用 TypeScript 编写的。我们可以使用各种装饰器来定义我们的列。

@PrimaryGeneratedColumn()<br />主键是用于在表中唯一标识行的列。虽然我们可能使用现有列并将其设为主键，但我们通常创建一个 `id` 列。通过选择 TypeORM 的 `PrimaryGeneratedColumn`，我们创建了一个自动生成值的整数主列。

@Column()<br />`@Column()` 装饰器将一个属性标记为列。使用它时，我们有两种可能的方法。

第一种方法是不显式传递列类型。这样做时，TypeORM 会使用我们的 TypeScript 类型来确定列类型。这是可能的，因为 NestJS 在底层使用了 reflect-metadata。

第二种方法是显式传递列类型，例如使用 `@Column('text')`。可用的列类型在不同数据库（如 MySQL 和 Postgres）之间有所不同。你可以在 TypeORM 文档中查找它们。

现在是讨论在 Postgres 中存储字符串的不同方式的合适时机。依赖 TypeORM 来确定字符串列的类型会导致使用“character varying”类型，也称为 varchar。

Varchar 与文本类型的列非常相似，但它给我们提供了限制字符串长度的可能性。从性能角度看，这两种类型是相同的。

<a name="89f8857a"></a>
### SQL 查询

在 pgAdmin 中，我们可以检查一个与 TypeORM 为我们做的事情等效的查询。

```sql
CREATE TABLE public.post
(
    id integer NOT NULL DEFAULT nextval('post_id_seq'::regclass),
    title character varying COLLATE pg_catalog."default" NOT NULL,
    content character varying COLLATE pg_catalog."default" NOT NULL,
    CONSTRAINT "PK_be5fda3aac270b134ff9c21cdee" PRIMARY KEY (id)
)
```

上面有几个有趣的事项需要注意：

使用 `@PrimaryGeneratedColumn()` 导致使用 int 列。它默认返回 `nextval` 函数的返回值，该函数返回唯一的 id。另一种方式是使用 `serial` 类型，这会使查询更简短，但在底层工作方式相同。

我们的实体有使用 COLLATE 的 varchar 列。排序规则用于指定排序顺序和字符分类。要查看我们的默认排序规则，我们可以运行这个查询：

```
SHOW LC_COLLATE
```

`en_US.utf8`

上面的值是在用于创建我们数据库的查询中定义的。默认情况下，它是 UTF8 和英语。

```
CREATE DATABASE nestjs
    WITH
    OWNER = admin
    ENCODING = 'UTF8'
    LC_COLLATE = 'en_US.utf8'
    LC_CTYPE = 'en_US.utf8'
    TABLESPACE = pg_default
    CONNECTION LIMIT = -1;
```

此外，我们的 `CREATE TABLE` 查询对我们的 id 施加了约束，以确保它们始终是唯一的。

`PK_be5fda3aac270b134ff9c21cdee` 是上述约束的名称，是生成的。

<a name="c270fc6f"></a>
### 仓库

通过**仓库**，我们可以管理特定的实体。仓库有多个函数用于与实体交互。我们再次使用 `TypeOrmModule` 来访问它。

posts.module.ts

```typescript
import { Module } from '@nestjs/common';
import PostsController from './posts.controller';
import PostsService from './posts.service';
import Post from './post.entity';
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [TypeOrmModule.forFeature([Post])],
  controllers: [PostsController],
  providers: [PostsService]
})
export class PostsModule {}
```

现在，在我们的 `PostsService` 中，我们可以注入 Posts 仓库。

```typescript
import { InjectRepository } from '@nestjs/typeorm';

constructor(
  @InjectRepository(Post)
  private postsRepository: Repository<PostEntity>
) {}
```

<a name="dc5d07a1"></a>
### 查找

使用 `find` 函数，我们可以获取多个元素。如果我们不提供任何选项，它返回所有元素。

```typescript
getAllPosts() {
  return this.postsRepository.find();
}
```

要获取单个元素，我们使用 `findOne` 函数。通过提供一个数字，我们表明我们想要一个具有特定 id 的元素。如果结果是 undefined，这意味着元素未找到。

```typescript
async getPostById(id: number) {
  const post = await this.postsRepository.findOne(id);

  if (post) {
    return post;
  }
  throw new HttpException('Post not found', HttpStatus.NOT_FOUND);
}
```

<a name="d9ac9228"></a>
### 创建

使用 `create` 函数，我们可以实例化一个新的 Post。之后我们可以使用 `save` 函数将我们的新实体填充到数据库中。

```typescript
async createPost(post: CreatePostDto) {
  const newPost = await this.postsRepository.create(post);
  await this.postsRepository.save(newPost);
  return newPost;
}
```

<a name="8347a927"></a>
### 修改

要修改现有元素，我们可以使用 `update` 函数。之后我们会使用 `findOne` 函数返回修改后的元素。

```typescript
async updatePost(id: number, post: UpdatePostDto) {
  await this.postsRepository.update(id, post);
  const updatedPost = await this.postsRepository.findOne(id);
  if (updatedPost) {
    return updatedPost;
  }
  throw new HttpException('Post not found', HttpStatus.NOT_FOUND
```
);<br />}

重要的是，它接受一个部分实体，所以它的作用类似于 PATCH 而非 PUT。如果你想了解更多关于 PUT 与 PATCH 的区别（尽管是用 MongoDB），可以查看 TypeScript Express 教程 #15：在 MongoDB 中使用 PUT 与 PATCH 的区别。

<a name="2f4aaddd"></a>
### 删除

要删除给定 id 的元素，我们可以使用 `delete` 函数。

```typescript
async deletePost(id: number) {
  const deleteResponse = await this.postsRepository.delete(id);
  if (!deleteResponse.affected) {
    throw new HttpException('Post not found', HttpStatus.NOT_FOUND);
  }
}
```

查看 DELETE 命令的文档，我们可以看到我们可以访问被删除元素的计数。这个数据在 `affected` 属性中可用。如果它等于零，我们可以假设该元素不存在。

<a name="f3bfa7fc"></a>
### 处理异步错误

NestJS 控制器非常好地处理了异步错误。

```typescript
@Get(':id')
getPostById(@Param('id') id: string) {
  return this.postsService.getPostById(Number(id));
}
```

如果 `getPostById` 函数抛出一个错误，NestJS 会自动捕获它并解析。使用纯 Express 时，我们需要自己来做这个：

```typescript
getAllPosts = async (request: Request, response: Response, next: NextFunction) => {
  const id = request.params.id;
  try {
    const post = await this.postsService.getPostById(id);
    response.send(post);
  } catch (error) {
    next(error);
  }
}
```

<a name="25f9c7fa"></a>
### 总结

在这篇文章中，我们介绍了将我们的 NestJS 应用程序与 PostgreSQL 数据库连接的基础知识。我们不仅使用了 TypeORM，还研究了一些 SQL 查询。NestJS 和 TypeORM 内置了许多功能，准备就绪可供使用。在本系列的后续部分，我们将更深入地探讨这些功能，敬请期待！

