---
title: 使用NestJS的API-06
slug: 使用NestJS的API-06
pubDate: 2022-05-30
tags:
  - 后端
  - Nestjs
  - 教程
description: 使用NestJS的API-06
author: xf
cover: src/images/cat-6.webp
coverAlt: Nestjs
category:
  - 后端
---
NestJS 致力于提高代码的可维护性和可测试性。为此，它实现了各种机制，如依赖注入。在本文中，我们将通过查看 TypeScript 编译器的输出来探究 NestJS 如何解决依赖问题。我们还努力理解 NestJS 所构建的模块化架构。

你可以在这个系列的代码仓库中找到代码。

<a name="7acea2ac"></a>
### 实现依赖注入的原因

你可能熟悉我的 TypeScript Express 系列。它采用了一种相对简单的方法来解析依赖。

```javascript
import { Router } from 'express';
import Controller from '../interfaces/controller.interface';
import AuthenticationService from './authentication.service';

class AuthenticationController implements Controller {
  public path = '/auth';
  public router = Router();
  public authenticationService = new AuthenticationService();

  // (...)
}
```

不幸的是，上述方法有几个缺点。每次我们创建一个 AuthenticationController 实例时，我们也会创建一个新的 AuthenticationService。如果我们在所有需要 AuthenticationService 的控制器中都采用上述方法，每个控制器都会收到一个单独的实例。这可能会随着时间的推移变得难以维护。

同时，测试性也受到影响。虽然在上述控制器初始化之前可以模拟 AuthenticationService，但这种解决方案并不被认为是理想的。

SOLID 原则之一称为控制反转（IoC）。它声明高级模块不应依赖于低级模块。一个直接的实现方法是先创建依赖实例，然后通过构造函数提供它们。

```javascript
import { Router } from 'express';
import Controller from '../interfaces/controller.interface';
import AuthenticationService from './authentication.service';

class AuthenticationController implements Controller {
  public path = '/auth';
  public router = Router();
  public authenticationService: AuthenticationService;

  constructor(authenticationService: AuthenticationService) {
    this.authenticationService = authenticationService;
  }

  // (...)
}

const authenticationService = new AuthenticationService();

const authenticationController = new AuthenticationController(authenticationService);
```

如果你想了解更多关于 SOLID 的信息，请查看将 SOLID 原则应用到你的 TypeScript 代码中。

虽然上述方法帮助我们克服了提到的问题，但这远非方便。这就是为什么 NestJS 实现了一个依赖注入机制，自动提供所有必要的依赖。

<a name="b0630ce3"></a>
### NestJS 中的依赖注入

让我们看一个我们在这个系列的第三部分中构建的类似控制器。

```javascript
import { Controller } from '@nestjs/common';
import { AuthenticationService } from './authentication.service';

@Controller('authentication')
export class AuthenticationController {
  constructor(private readonly authenticationService: AuthenticationService) {}

  // (...)
}
```

多亏了使用了 `private readonly`，我们不需要在构造函数的主体中分配 `authenticationService`。

`@Controller` 装饰器在其他方面确保了有关我们类的元数据被保存。`@Injectable` 装饰器也做到了这一点。

```javascript
import { Injectable } from '@nestjs/common';
import { UsersService } from '../users/users.service';
import { JwtService } from '@nestjs/jwt';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class AuthenticationService {
  constructor(
    private readonly usersService: UsersService,
    private readonly jwtService: JwtService,
    private readonly configService: ConfigService
  ) {}

  // (...)
}
```

TypeScript 编译器发出的元数据使 NestJS 可以稍后用来确定我们需要什么依赖。让我们检查一下 `AuthenticationService` 的输出。

```javascript
AuthenticationService = __decorate([
    common_1.Injectable(),
    __metadata("design:paramtypes", [users_service_1.UsersService, jwt_1.JwtService, config_1.ConfigService])
], AuthenticationService);
```

`design:paramtypes` 是一个描述参数类型元数据的键。借助它，我们可以获取到我们在 `AuthenticationService` 的构造函数中需要的类的引用数组。我们可以将其视为在编译时提取 `AuthenticationService` 的依赖。NestJS 在底层使用 `reflect-metadata` 包来处理上述元数据。

当一个 NestJS 应用启动时，它解析 `AuthenticationController` 所需的所有元数据。在底层，这可能会变得相当复杂，因为它可以处理循环依赖，例如。

如果你想深入了解 NestJS 如何提供所需的依赖，请查看 NestJS 创始人 Kamil Myśliwiec 的这次演讲。

<a name="fac54c34"></a>
### 模块

模块是我们应用程序的一部分，包含围绕特定功能的功能。

每个 NestJS 应用程序都有一个根模块。它为 NestJS 创建应用程序图提供了一个起点。

```javascript
import { Module } from '@nestjs/common';
import { PostsModule } from './posts/posts.module';
import { ConfigModule } from '@nestjs/config';
import { DatabaseModule } from './database/database.module';
import { AuthenticationModule } from './authentication/authentication.module';
import { UsersModule } from './users/users.module';

@Module({
  imports: [
    PostsModule,
    ConfigModule.forRoot({
      // ...
    }),
    DatabaseModule,
    AuthenticationModule,
    UsersModule,
  ],
  controllers: [],
  providers: [],
})
export class AppModule {}
```

NestJS 使用根模块来解析其他模块及其依赖。例如，我们有 `AuthenticationModule` 负责我们应用程序的认证。我们在这个系列的第三部分创建它，以处理注册和验证用户的过程。

正如你从 `authentication` 目录中看到的，一个模块可以包含许多东西。在上述情况下，它包装了控制器、服务和一些其他与认证过程相关的文件。

在认证过程

中进行用户的读取和创建是必要的。为了封装这一过程，我们创建了一个独立的 `UsersModule`。这表明模块在将我们的应用划分成可以一起工作的各个部分方面非常有用。

让我们检查一下我们如何在 `AuthenticationModule` 内使用 `UsersService`。为此，我们首先需要检查一下 `UsersModule`。

```javascript
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';
import { TypeOrmModule } from '@nestjs/typeorm';
import User from './user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  providers: [UsersService],
  exports: [UsersService]
})
export class UsersModule {}
```

这里关键的部分是 `providers` 和 `exports` 数组。

提供者是可以注入依赖的东西。一个例子就是服务。我们将 `UsersService` 放入 `UsersModule` 的 `providers` 数组中，表示它属于该模块。

如上所述，一个模块也可以导入其他模块。通过将 `UsersService` 放入 `exports` 数组，我们表明模块公开了它。我们可以将其视为模块的公共接口。

```javascript
import { Module } from '@nestjs/common';
import { AuthenticationService } from './authentication.service';
import { UsersModule } from '../users/users.module';
import { AuthenticationController } from './authentication.controller';
import { LocalStrategy } from './local.strategy';
import { JwtStrategy } from './jwt.strategy';

@Module({
  imports: [
    UsersModule
    // (...)
  ],
  providers: [AuthenticationService, LocalStrategy, JwtStrategy],
  controllers: [AuthenticationController]
})
export class AuthenticationModule {}
```

现在当我们导入 `UsersModule` 时，我们可以访问所有导出的提供者。因此，我们可以在 `AuthenticationService` 中使用 `UsersService`。

值得注意的是，在 NestJS 中，模块是单例的。这意味着 `UsersService` 的实例在所有模块中共享。在考虑如内存缓存等技术时，上述特性尤其关键。

<a name="169a31d8"></a>
### `@Global()` 装饰器

虽然创建全局模块可能被认为是一种不鼓励的设计决策，但它绝对是可能的。

当我们想要一组提供者在任何地方都可用时，我们可以使用 `@Global()` 装饰器。

```javascript
@Global()
@Module({
  imports: [TypeOrmModule.forFeature([User])],
  providers: [UsersService],
  exports: [UsersService]
})
export class UsersModule {}
```

现在我们不需要导入 `UsersModule` 就可以使用 `UsersService`。

我们应该只注册一次全局模块，最好的地方就是根模块。

<a name="25f9c7fa"></a>
### 总结

在本文中，我们更深入地探讨了 NestJS 如何与模块一起工作以及它是如何解析它们的依赖的。我们稍微检查了依赖注入在 Nest 中工作的一些原则。了解框架中的一些内部机制可以帮助我们更好地理解它，因此创建一个更复杂的应用结构。
