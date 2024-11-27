---
title: 使用NestJS的API-01
slug: 使用NestJS的API-01
pubDate: 2022-05-30
tags:
  - 后端
  - Nestjs
  - 教程
description: 使用NestJS的API-01
author: xf
cover: src/images/cat.webp
coverAlt: Nestjs
category:
  - 后端
---
## 使用NestJS的API #1. 控制器、路由和模块结构

NestJS是一个用于构建Node.js应用程序的框架。它在某种程度上是有主见的，并迫使我们遵循它对一个应用程序应该是什么样子的看法。这可能被视为一件好事，帮助我们保持整个应用程序的一致性，并迫使我们遵循良好的做法。

NestJS默认使用Express.js的引擎。如果你熟悉我的[TypeScript Express系列](https://wanago.io/2018/12/03/typescript-express-tutorial-routing-controllers-middleware/)，并且你很喜欢它，那么你很有可能也会喜欢NestJS。另外，Express框架的知识也会派上用场。
<!-- ![Screenshot-from-2020-05-09-20-51-51](media/16805199395570/Screenshot-from-2020-05-09-20-51-51.png) -->
根据[Risingstars](https://risingstars.js.org/2019/en/#section-nodejs-framework)的数据，Nest是2019年Github上排名最快的Node.js技术

一个重要的说明是，[NestJS的文档](https://docs.nestjs.com/)是全面的，你会从查找它中受益。在这里，我们试图将这些知识按顺序排列，但有时我们也会链接到官方文档。我们还参考了Express框架来强调使用NestJS的优势。为了从这篇文章中获益更多，一些Express的经验可能是有用的，但不是必须的。

> 如果你想了解Node.js的核心，我建议你看看[Node.js TypeScript系列](https://wanago.io/2019/02/11/node-js-typescript-modules-file-system/)。它涵盖了诸如流、事件循环、多进程和工人线程的多线程等主题。此外，了解如何在没有任何框架的情况下创建API，如Express和NestJS，会让我们更加欣赏它们。

## 开始使用NestJS

最直接的入门方式是克隆官方TypeScript启动库。Nest 是用 TypeScript 构建的，并且完全支持它。你可以使用JavaScript来代替，但在这里我们专注于TypeScript。

```
git clone git@github.com:nestjs/typescript-starter.git
```

在上述资源库中，值得关注的是tsconfig.json文件。我强烈建议添加 `alwaysStrict` 和 `noImplicitAny` 选项
上面的资源库包含了最基本的软件包。我们也得到了基本的文件类型，让我们开始吧。
> 这个系列的所有代码都可以在这个资源库中找到。希望它以后可以作为NestJS的模板，有一些内置的功能。它是一个官方typecript-starter的分叉。请随意给它们都打上一颗星。
>
## 控制器 (Controllers)

控制器处理传入的请求并向客户端返回响应。`typescript-starter`资源库包含了我们的第一个控制器。让我们来创建一个更强大的控制器：
posts.controller.ts

```javascript
import { Body, Controller, Delete, Get, Param, Post, Put } from '@nestjs/common';
import PostsService from './posts.service';
import CreatePostDto from './dto/createPost.dto';
import UpdatePostDto from './dto/updatePost.dto';
 
@Controller('posts')
export default class PostsController {
  constructor(
    private readonly postsService: PostsService
  ) {}
 
  @Get()
  getAllPosts() {
    return this.postsService.getAllPosts();
  }
 
  @Get(':id')
  getPostById(@Param('id') id: string) {
    return this.postsService.getPostById(Number(id));
  }
 
  @Post()
  async createPost(@Body() post: CreatePostDto) {
    return this.postsService.createPost(post);
  }
 
  @Put(':id')
  async replacePost(@Param('id') id: string, @Body() post: UpdatePostDto) {
    return this.postsService.replacePost(Number(id), post);
  }
 
  @Delete(':id')
  async deletePost(@Param('id') id: string) {
    this.postsService.deletePost(Number(id));
  }
}
```

post.interface.ts

```
export interface Post {
  id: number;
  content: string;
  title: string;
}
```

我们可以注意到的第一件事是，NestJS经常使用装饰器。为了标记一个类为控制器，我们使用@Controller()装饰器。我们向它传递一个可选的参数。它作为控制器内所有路线的路径前缀。

## 路由 (Routing)

在上述控制器中，与路由相连的下一组装饰器是@Get(), @Post(), Delete(), 和@Put()。它们告诉Nest为HTTP请求的特定端点创建一个处理程序。上述控制器创建了以下一组端点：

```
GET /posts
```

> 返回所有的文章

```
GET /posts/{id}
```

> 返回一个具有给定id的文章

```
POST /posts
```

> 创建一个新的文章

```
PUT /posts/{id}
```

> 更新一个具有给定id的文章

```
DELETE /posts/{id}
```

> 删除一个具有给定id的文章

默认情况下，NestJS以**200 OK**状态代码响应，但POST的**201 Created**除外。我们可以用`@HttpCode()`装饰器[轻松地改变它](https://docs.nestjs.com/controllers#status-code)。
当我们实现一个API时，我们经常需要引用一个特定的元素。我们可以用路由参数来做到这一点。它们是特殊的URL段，用于捕获在其位置上指定的值。要创建一个路由参数，我们需要在其名称前加上`：`符号。

提取路由参数值的方法是使用`@Param()`装饰器。多亏了它，我们可以在我们的路由处理程序的参数中访问它。
> 我们可以使用一个可选的参数来指代一个特定的参数，例如`@Param('id')`。否则，我们就可以访问带有所有参数的`params`对象。

由于路由参数是字符串，而我们的id是数字，我们需要首先转换参数。
> 我们还可以使用**管道**来转换路由参数。管道是NestJS的内置功能，我们将在后面介绍。

## 访问一个请求的主体

当我们在上面的控制器中处理POST和PUT时，我们也需要访问一个请求的主体。通过这样做，我们可以用它来填充我们的数据库。
NestJS提供了一个@Body()装饰器，让我们可以轻松访问主体。就像在TypeScript Express系列中，我们引入了数据传输对象（DTO）的概念。它定义了请求中发送的数据的格式。它既可以是一个接口，也可以是一个类，但使用后者会给我们更多的可能性，我们将在后面进行探讨。
createPost.dto.ts

```
class CreatePostDto {
  content: string;
  title: string;
}
```

updatePost.dto.ts

```
class UpdatePostDto {
  id: number;
  content: string;
  title: string;
}
```

## 处理函数的参数

让我们再来研究一下处理程序函数的参数。

```
async replacePost(@Body() post: UpdatePostDto, @Param('id') id: string) {
  return this.postsService.replacePost(Number(id), post);
}
```

通过使用方法参数装饰器，我们告诉Nest将特定的参数注入我们的方法中。NestJS是围绕着依赖性注入和反转控制的概念建立的。当我们通过各种功能时，我们会在很多方面进行阐述。
>依赖性注入是实现反转控制的技术之一。如果你想了解更多关于IoC的信息，请查看《将SOLID原则应用于你的TypeScript代码》。

一个重要的注意点是，颠倒它们的顺序会产生相同的结果，这在一开始可能看起来是违反直觉的。

```
async replacePost(@Param('id') id: string, @Body() post: UpdatePostDto) {
  return this.postsService.replacePost(Number(id), post);
}
```

## NestJS相对于Express的优势

NestJS给了我们很多开箱即用的东西，并希望我们使用控制器来设计我们的API。另一方面，Express.js给我们留下了更多的灵活性，但没有给我们配备这样的工具来维护我们代码的可读性。
我们可以自由地用Express来实现控制器。我们在[TypeScript Express系列](https://wanago.io/2018/12/03/typescript-express-tutorial-routing-controllers-middleware/)中做到了这一点。

```
import { Request, Response, Router } from 'express';
import Controller from '../../interfaces/controller.interface';
import PostsService from './posts.service';
import CreatePostDto from './dto/createPost.dto';
import UpdatePostDto from './dto/updatePost.dto';
 
export default class PostsController implements Controller {
  private path = '/posts';
  public router = Router();
  private postsService = new PostsService();
 
  constructor() {
    this.intializeRoutes();
  }
 
  intializeRoutes() {
    this.router.get(this.path, this.getAllPosts);
    this.router.get(`${this.path}/:id`, this.getPostById);
    this.router.post(this.path, this.createPost);
    this.router.put(`${this.path}/:id`, this.replacePost);
  }
 
  private getAllPosts = (request: Request, response: Response) => {
    const posts = this.postsService.getAllPosts();
    response.send(posts);
  }
 
  private getPostById = (request: Request, response: Response) => {
    const id = request.params.id;
    const post = this.postsService.getPostById(Number(id));
    response.send(post);
  }
 
  private createPost = (request: Request, response: Response) => {
    const post: CreatePostDto = request.body;
    const createdPost = this.postsService.createPost(post);
    response.send(createdPost);
  }
 
  private replacePost = (request: Request, response: Response) => {
    const id = request.params.id;
    const post: UpdatePostDto = request.body;
    const replacedPost = this.postsService.replacePost(Number(id), post);
    response.send(replacedPost);
  }
 
  private deletePost = (request: Request, response: Response) => {
    const id = request.params.id;
    this.postsService.deletePost(Number(id));
    response.sendStatus(200);
  }
}[](https://wanago.io/2018/12/03/typescript-express-tutorial-routing-controllers-middleware/)
```

上面，我们可以看到在纯Express中创建的一个类似的控制器。这里有一些明显的不同。

首先，我们需要自己处理控制器的路由。我们没有这样方便的装饰器，可以依靠它来为我们做这件事。NestJS的工作方式有点类似于为Java编写的Spring框架。

在[TypeScript Express系列](https://wanago.io/2018/12/03/typescript-express-tutorial-routing-controllers-middleware/)中，我们使用一个Application类，将路由附加到应用程序。

```

class Application {
  // ...
  private initializeControllers(controllers: Controller[]) {
    controllers.forEach((controller) => {
      this.app.use('/', controller.router);
    });
  }
  // ...
}
```

NextJS的另一大优势是它为我们提供了一种处理[**Request**](https://docs.nestjs.com/controllers#request-object)和[**Response**](https://docs.nestjs.com/controllers#appendix-library-specific-approach)对象的优雅方式。像`@Body()`和`@Params()`这样的装饰器有助于提高我们代码的可读性。

Nest所提供的最有用的东西之一是它如何处理响应。我们的路由处理程序可以返回原始类型（例如，字符串）、承诺，甚至是RxJS可观察流。我们不需要每次都手动处理，并使用 `response.send` 函数。NestJS还可以让我们的应用程序轻松地处理错误，我们将在本系列接下来的部分中探讨。

>当使用NestJS时，我们也可以直接操作请求和响应对象。不过，自己处理响应使我们失去了NestJS的一些优势。
在纯Express和NestJS中，我们处理依赖关系的方式也有区别。
在上面的Express控制器中，我们直接在`PostController`中创建一个新的`PostService`。不幸的是，这打破了SOLID原则中的**依赖倒置原则**。其中一个问题是会给编写测试带来一些麻烦。
另一方面，NestJS通过实现依赖性注入，非常关心对依赖性倒置原则的遵守。

## 服务 (Services)

`typescript-starter`资源库也包含了我们的第一个服务。服务的一项工作是将业务逻辑与控制器分开，使其更简洁，更便于测试。让我们为我们的帖子创建一个简单的服务。
posts.service.ts

```
import { HttpException, HttpStatus, Injectable } from '@nestjs/common';
import CreatePostDto from './dto/createPost.dto';
import Post from './post.interface';
import UpdatePostDto from './dto/updatePost.dto';
 
@Injectable()
export default class PostsService {
  private lastPostId = 0;
  private posts: Post[] = [];
 
  getAllPosts() {
    return this.posts;
  }
 
  getPostById(id: number) {
    const post = this.posts.find(post => post.id === id);
    if (post) {
      return post;
    }
    throw new HttpException('Post not found', HttpStatus.NOT_FOUND);
  }
 
  replacePost(id: number, post: UpdatePostDto) {
    const postIndex = this.posts.findIndex(post => post.id === id);
    if (postIndex > -1) {
      this.posts[postIndex] = post;
      return post;
    }
    throw new HttpException('Post not found', HttpStatus.NOT_FOUND);
  }
 
  createPost(post: CreatePostDto) {
    const newPost = {
      id: ++this.lastPostId,
      ...post
    }
    this.posts.push(newPost);
    return newPost;
  }
 
  deletePost(id: number) {
    const postIndex = this.posts.findIndex(post => post.id === id);
    if (postIndex > -1) {
      this.posts.splice(postIndex, 1);
    } else {
      throw new HttpException('Post not found', HttpStatus.NOT_FOUND);
    }
  }
}
```

虽然上述逻辑是直截了当的，但那里有几句值得注意的话。
我们使用内置的`HttpException`类来抛出NestJS能够理解的错误。当我们抛出`HttpException('Post not found', HttpStatus.NOT_FOUND)`时，它会被传播到**全局异常过滤器**，并且一个适当的响应被发送到客户端。我们将在本系列的后续部分中更多地探讨这个话题。
<!-- ![Screenshot-from-2020-05-10-17-39-24](media/16805199395570/Screenshot-from-2020-05-10-17-39-24.png) -->
`@Injectable()` 装饰器告诉Nest，这个类是一个[provider](https://docs.nestjs.com/providers)。由于这一点，我们可以将其添加到一个**模块**中。

## 模块(Modules)

我们使用模块来组织我们的应用程序。我们的PostController和PostService是密切相关的，属于同一个应用领域。因此，把它们放在一个模块中是合适的。

通过这样做，我们按功能来组织我们的代码。这在我们的应用程序成长过程中特别有用。
posts.module.ts

```

import { Module } from '@nestjs/common';
import PostsController from './posts.controller';
import PostsService from './posts.service';
 
@Module({
  imports: [],
  controllers: [PostsController],
  providers: [PostsService],
})
export class PostsModule {}
```

此外，每个应用程序都需要一个根模块。它是Nest在构建应用程序时的一个起点。
app.module.ts

```

import { Module } from '@nestjs/common';
import { PostsModule } from './posts/posts.module';
 
@Module({
  imports: [PostsModule],
  controllers: [],
  providers: [],
})
export class AppModule {}
```

该模块包含：

* imports
    导入的模块--由于在我们的AppModule中导入了PostsModule，NestJS使用了它。
* controllers
    用于实例化的控制器
* providers
    要实例化的提供者 - 它们至少可以在这个模块中使用。
* exports
    在其他模块中可用的提供者的一个子集

## 摘要

通过上述所有操作，我们的src目录最终变成了这样：

```
├── src
│   ├── app.module.ts
│   ├── main.ts
│   └── posts
│       ├── dto
│       │   ├── createPost.dto.ts
│       │   └── updatePost.dto.ts
│       ├── post.interface.ts
│       ├── posts.controller.ts
│       ├── posts.module.ts
│       └── posts.service.ts
```

在这篇文章中，我们刚刚开始使用Nest。我们已经知道了什么是Controller，以及如何在我们的应用程序中处理基本的路由。我们还简单地触及了Services和Modules的主题。在本系列的后续部分中，我们将花相当多的时间讨论NestJS的应用结构。

以上所有的知识只是NestJS的冰山一角。希望它能让你相信，这个框架是值得研究的，因为它提供了很多价值。关于Nest提供的功能有很多可说的，如整齐的错误处理和依赖性注入。我们还将研究PostgreSQL数据库以及如何通过ORM和SQL语句使用它。

你可以在这个系列中期待它，还有更多，请继续关注
