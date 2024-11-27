---
title: 使用NestJS的API-07
slug: 使用NestJS的API-07
pubDate: 2022-05-30
tags:
  - 后端
  - Nestjs
  - 教程
description: 使用NestJS的API-07
author: xf
cover: src/images/cat-7.webp
coverAlt: Nestjs
category:
  - 后端
---
当我们构建一个应用程序时，我们会创建许多实体。这些实体通常以某种方式相互关联，定义这些关系是设计数据库的一个重要部分。在本文中，我们将探讨在Postgres数据库上下文中什么是关系，以及如何使用TypeORM和NestJS来处理这些关系。

关系型数据库已经存在很长时间了，并且在处理结构化数据方面表现出色。它们通过将数据组织成表格并将这些表格相互链接来实现此目的。运行各种SQL查询时，我们可以连接表格并提取有意义的信息。有几种不同类型的关系，今天我们将通过使用示例来逐一介绍它们。

我们还在TypeScript Express系列中介绍过这个话题。下面的文章作为我们可以从那里获得的信息的回顾。这次我们还将更深入地查看TypeORM生成的SQL查询。

你可以在这个仓库中找到本系列的所有代码。

**一对一关系**

在一对一关系中，第一个表中的一行只在第二个表中有一个匹配的行，反之亦然。

最直接的例子可能是添加一个地址实体。

```typescript
// users/address.entity.ts

import { Column, Entity, PrimaryGeneratedColumn } from 'typeorm';

@Entity()
class Address {
  @PrimaryGeneratedColumn()
  public id: number;

  @Column()
  public street: string;

  @Column()
  public city: string;

  @Column()
  public country: string;
}

export default Address;
```

假设一个地址只能链接到一个用户。同时，一个用户不能有多于一个的地址。

为了实现上述目标，我们需要一个一对一关系。使用TypeORM时，我们可以通过装饰器轻松创建它。

```typescript
// users/user.entity.ts

import { Column, Entity, JoinColumn, OneToOne, PrimaryGeneratedColumn } from 'typeorm';
import { Exclude } from 'class-transformer';
import Address from './address.entity';

@Entity()
class User {
  @PrimaryGeneratedColumn()
  public id: number;

  @Column({ unique: true })
  public email: string;

  @Column()
  public name: string;

  @Column()
  @Exclude()
  public password: string;

  @OneToOne(() => Address)
  @JoinColumn()
  public address: Address;
}

export default User;
```

上面我们使用了`@OneToOne()`装饰器。它的参数是一个返回我们想要建立关系的实体类的函数。

第二个装饰器`@JoinColumn()`表示`User`实体拥有关系。这意味着`User`表的行包含`addressId`列，可以存储地址的id。我们只在关系的一侧使用它。

我们可以查看pgAdmin来检查TypeORM是如何创建所需关系的。

上面我们可以看到，`addressId`是一个常规的整数列。它上面有一个约束，表明我们放入`addressId`列的任何值都需要与`address`表中的某个id匹配。

上述可以不使用`CONSTRAINT`关键字来简化。

```sql
CREATE TABLE user (
  // ...
  addressId integer REFERENCES address (id)
)
```

`ON UPDATE NO ACTION`和`ON DELETE NO ACTION`是默认行为。它们表明如果我们尝试删除或更改当前正在使用的地址的id，Postgres将会抛出错误。

`MATCH SIMPLE`指的是当我们使用多于一个列作为外键时的情况。它意味着我们允许其中一些列为null。

**反向关系**

目前我们的关系是单向的。这意味着关系的一方只有关于另一方的信息。我们可以通过创建一个反向关系来改变这一点。这样做我们可以使User和Address之间的关系变为双向。

为了创建反向关系，我们需要使用`@OneToOne`并提供一个持有关系另一方的属性。

```typescript
// users/address.entity.ts

import { Column, Entity, OneToOne, PrimaryGeneratedColumn } from 'typeorm';
import User from './user.entity';

@Entity()
class Address {
  @PrimaryGeneratedColumn()
  public id: number;

  @Column()
  public street: string;

  @Column()
  public city: string;

  @Column()
  public country: string;

  @OneToOne(() => User, (user: User) => user.address)
  public user: User;
}

export default Address;
```

关键的一点是，反向关系有点是一个抽象概念，它不会在数据库中创建任何额外的列。

存储关系的双方信息可能很有用。我们可以轻松地关联双方，例如获取带有用户的地址。

如果我们希望我们的相关实体始终被包含，我们可以将我们的关系设置为`eager`。

现在，每当我们获取用户时，我们也会得到他们的地址。关系的只有一侧可以是`eager`。

**保存相关实体**

目前我们需要分别保存用户和地址，这可能不是最方便的方式。相反，我们可以打开`cascade`选项。这样我们可以在保存用户的同时保存地址。

**一对多和多对一关系**

`一对多`和`多对一`是一种关系，其中第一个表中的一行可以链接到第二个表的多行。第二个表中的行只能链接到第一个表的一行。

上述是实现帖子和用户的非常合适的关系，我们在本系列的前几部分已经定义过了。假设一个用户可以创建多个帖子，但是一个帖子只有一个作者。

```typescript
// users/user.entity.ts

import { Column, Entity, JoinColumn, OneToMany, OneToOne, PrimaryGeneratedColumn } from 'typeorm';
import { Exclude } from 'class-transformer';
import Address from './address.entity';
import Post from '../

posts/post.entity';

@Entity()
class User {
  @PrimaryGeneratedColumn()
  public id: number;

  @Column({ unique: true })
  public email: string;

  @Column()
  public name: string;

  @Column()
  @Exclude()
  public password: string;

  @OneToOne(() => Address, {
    eager: true,
    cascade: true
  })
  @JoinColumn()
  public address: Address;

  @OneToMany(() => Post, (post: Post) => post.author)
  public posts: Post[];
}

export default User;
```

通过使用`@OneToMany()`装饰器，一个用户可以链接到多个帖子。我们也需要定义这种关系的另一方。

```typescript
// posts/post.entity.ts

import { Column, Entity, ManyToOne, PrimaryGeneratedColumn } from 'typeorm';
import User from '../users/user.entity';

@Entity()
class Post {
  @PrimaryGeneratedColumn()
  public id: number;

  @Column()
  public title: string;

  @Column()
  public content: string;

  @Column({ nullable: true })
  public category?: string;

  @ManyToOne(() => User, (author: User) => author.posts)
  public author: User;
}

export default Post;
```

多篇帖子可以与一个用户关联，感谢`@ManyToOne()`装饰器。

我们在本系列的第三部分实现了认证。当在我们的API中创建一个帖子时，我们可以访问关于已认证用户的数据。我们需要使用它来确定帖子的作者。

```typescript
@Post()
@UseGuards(JwtAuthenticationGuard)
async createPost(@Body() post: CreatePostDto, @Req() req: RequestWithUser) {
  return this.postsService.createPost(post, req.user);
}

async createPost(post: CreatePostDto, user: User) {
  const newPost = await this.postsRepository.create({
    ...post,
    author: user
  });
  await this.postsRepository.save(newPost);
  return newPost;
}
```

如果我们想返回带有作者的帖子列表，现在可以很容易地做到。

```typescript
getAllPosts() {
  return this.postsRepository.find({ relations: ['author'] });
}

async getPostById(id: number) {
  const post = await this.postsRepository.findOne(id, { relations: ['author'] });
  if (post) {
    return post;
  }
  throw new PostNotFoundException(id);
}

async updatePost(id: number, post: UpdatePostDto) {
  await this.postsRepository.update(id, post);
  const updatedPost = await this.postsRepository.findOne(id, { relations: ['author'] });
  if (updatedPost) {
    return updatedPost;
  }
  throw new PostNotFoundException(id);
}
```

如果我们查看数据库，可以看到使用`ManyToOne()`装饰器的关系一方存储了外键。

这意味着帖子存储了作者的id，而不是反过来。

**多对多关系**

之前我们在帖子中添加了一个名为`category`的属性。让我们进一步阐述。

我们希望能够定义跨帖子可重用的分类。我们还希望一个帖子能够属于多个分类。

上述是多对多关系。这发生在第一个表中的一行可以链接到第二个表的多行，反之亦然。

```typescript
// categories/category.entity.ts

import { Column, Entity, PrimaryGeneratedColumn } from 'typeorm';

@Entity()
class Category {
  @PrimaryGeneratedColumn()
  public id: number;

  @Column()
  public name: string;
}

export default Category;

// posts/post.entity.ts

import { Column, Entity, JoinTable, ManyToMany, ManyToOne, PrimaryGeneratedColumn } from 'typeorm';
import User from '../users/user.entity';
import Category from '../categories/category.entity';

@Entity()
class Post {
  @PrimaryGeneratedColumn()
  public id: number;

  @Column()
  public title: string;

  @Column()
  public content: string;

  @Column({ nullable: true })
  public category?: string;

  @ManyToOne(() => User, (author: User) => author.posts)
  public author: User;

  @ManyToMany(() => Category)
  @JoinTable()
  public categories: Category[];
}

export default Post;
```

当我们使用`@ManyToMany()`和`@JoinTable()`装饰器时，TypeORM设置了一个额外的表。这样，无论是帖子还是分类表都不存储关于关系的数据。

```sql
CREATE TABLE public.post_categories_category
(
    "postId" integer NOT NULL,
    "categoryId" integer NOT NULL,
    CONSTRAINT "PK_91306c0021c4901c1825ef097ce" PRIMARY KEY ("postId", "categoryId"),
    CONSTRAINT "FK_93b566d522b73cb8bc46f7405bd" FOREIGN KEY ("postId")
        REFERENCES public.post (id) MATCH SIMPLE
        ON UPDATE NO ACTION
        ON DELETE CASCADE,
    CONSTRAINT "FK_a5e63f80ca58e7296d5864bd2d3" FOREIGN KEY ("categoryId")
        REFERENCES public.category (id) MATCH SIMPLE
        ON UPDATE NO ACTION
        ON DELETE CASCADE
)
```

上面我们可以看到，我们新的`post_categories_category`表使用了由`postId`和`categoryId`组合成的主键。

我们也可以使多对多关系双向化。不过要记得，只能在关系的一侧使用`JoinTable`装饰器。

```typescript
@ManyToMany(() => Category, (category: Category) => category.posts)
@JoinTable()
public categories: Category[];

@ManyToMany(() => Post, (post: Post) => post.categories)
public posts: Post[];
```

通过上述操作，我们现在可以轻松地获取带有其帖子的分类。

```typescript
getAllCategories() {
  return this.categoriesRepository.find({ relations: ['posts'] });
}

async getCategoryById(id: number) {
  const category = await this.categoriesRepository.findOne(id, { relations: ['posts'] });
  if (category) {
    return category;
  }
  throw new CategoryNotFoundException(id);
}

async updateCategory(id: number, category: UpdateCategoryDto) {
  await this.categoriesRepository.update(id, category);
  const updatedCategory = await this.categoriesRepository.findOne(id, { relations: ['posts'] });
  if (updatedCategory) {


    return updatedCategory;
  }
  throw new CategoryNotFoundException(id);
}
```

**总结**

这次我们介绍了在使用NestJS、Postgres和TypeORM时如何创建关系。它包括一对一、一对多和多对多关系。我们为它们提供了各种选项，如`cascade`和`eager`。我们还查看了TypeORM创建的SQL查询，以更好地理解它是如何工作的。
