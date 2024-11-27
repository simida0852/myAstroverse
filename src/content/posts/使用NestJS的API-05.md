---
title: 使用NestJS的API-05
slug: 使用NestJS的API-05
pubDate: 2022-05-30
tags:
  - 后端
  - Nestjs
  - 教程
description: 使用NestJS的API-05
author: xf
cover: src/images/cat-4.webp
coverAlt: Nestjs
category:
  - 后端
---
有时候，我们需要对输出的数据执行额外操作。我们可能不想暴露特定的属性或以某种其他方式修改响应。在这篇文章中，我们将探讨NestJS为我们提供的各种解决方案，以改变我们在响应中发送的数据。

你可以在这个系列的这个仓库中找到代码。

<a name="c4d08df9"></a>
### 序列化

首先需要关注的是序列化。这是一个在返回给用户之前转换响应数据的过程。

在这个系列的前几部分中，我们在API的各个部分中移除了密码。更好的方法是使用`class-transformer`。

```typescript
// users/user.entity.ts

import { Column, Entity, PrimaryGeneratedColumn } from 'typeorm';
import { Exclude } from 'class-transformer';

@Entity()
class User {
  @PrimaryGeneratedColumn()
  public id?: number;

  @Column({ unique: true })
  public email: string;

  @Column()
  public name: string;

  @Column()
  @Exclude()
  public password: string;
}

export default User;
```

NestJS配备了`ClassSerializerInterceptor`，它在底层使用`class-transformer`。要应用上述转换，我们需要在我们的控制器中使用它：

```typescript
@Controller('authentication')
@UseInterceptors(ClassSerializerInterceptor)
class AuthenticationController
```

如果我们发现自己在很多控制器中添加`ClassSerializerInterceptor`，我们可以改为全局配置它。

```typescript
// main.ts

import { NestFactory, Reflector } from '@nestjs/core';
import { AppModule } from './app.module';
import * as cookieParser from 'cookie-parser';
import { ClassSerializerInterceptor, ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  app.useGlobalInterceptors(new ClassSerializerInterceptor(
    app.get(Reflector))
  );
  app.use(cookieParser());
  await app.listen(3000);
}
bootstrap();
```

`ClassSerializerInterceptor`初始化时需要`Reflector`。

默认情况下，我们实体的所有属性都是暴露的。我们可以通过提供额外的选项给`class-transformer`来改变这个策略。为此，我们需要使用`@SerializeOptions()`装饰器。

```typescript
@Controller('authentication')
@SerializeOptions({
  strategy: 'excludeAll'
})
export class AuthenticationController
```

```typescript
// users/user.entity.ts

import { Column, Entity, PrimaryGeneratedColumn } from 'typeorm';
import { Expose } from 'class-transformer';

@Entity()
class User {
  @PrimaryGeneratedColumn()
  public id?: number;

  @Column({ unique: true })
  @Expose()
  public email: string;

  @Column()
  @Expose()
  public name: string;

  @Column()
  public password: string;
}

export default User;
```

`@SerializeOptions()`装饰器有更多你可能会发现有用的选项。它匹配你可以为`class-transformer`中的`classToPlain`方法提供的选项。

`class-transformer`库有相当多有用的特性。另一个值得注意的特性是转换值的能力。为了演示它，让我们创建一个可为空的列：

```typescript
@Entity()
class Post {
  // ...

  @Column({ nullable: true })
  public category?: string;
}
```

由于`category`是一个可为空的列，它是可选的，它的值是`null`，直到我们设置它。这意味着在响应中发送`null`值：

上述行为可能被认为是不可取的，解决它最直接的方式是使用`@Transform`装饰器。如果值等于`null`，我们不想在响应中发送它。

```typescript
@Column({ nullable: true })
@Transform(value => {
  if (value !== null) {
    return value;
  }
})
public category?: string;
```

<a name="36623387"></a>
### 使用`@Res()`装饰器的问题

在这个系列的前一部分中，

我们使用了`@Res()`装饰器来访问Express响应对象。借助它，我们能够将cookies附加到响应中。

```typescript
@HttpCode(200)
@UseGuards(LocalAuthenticationGuard)
@Post('log-in')
async logIn(@Req() request: RequestWithUser, @Res() response: Response) {
  const {user} = request;
  const cookie = this.authenticationService.getCookieWithJwtToken(user.id);
  response.setHeader('Set-Cookie', cookie);
  user.password = undefined;
  return response.send(user);
}
```

使用`@Res()`装饰器让我们失去了使用NestJS的一些优势。不幸的是，它干扰了`ClassSerializerInterceptor`。为了避免这种情况，我们可以遵循NestJS创造者的一些建议。如果我们使用`request.res`对象而不是`@Res()`装饰器，我们就不会将NestJS置于特定于express的模式中。

```typescript
@HttpCode(200)
@UseGuards(LocalAuthenticationGuard)
@Post('log-in')
async logIn(@Req() request: RequestWithUser) {
  const {user} = request;
  const cookie = this.authenticationService.getCookieWithJwtToken(user.id);
  request.res.setHeader('Set-Cookie', cookie);
  return user;
}
```

上述是一个巧妙的小技巧，我们利用它来同时利用NestJS内置的机制和直接访问响应对象。

<a name="991d91ca"></a>
### 自定义拦截器

上面我们使用`@Transform`装饰器来跳过单个属性，如果它等于`null`。对每个可为空的属性这样做看起来不是一个干净的方法。

幸运的是，除了使用`ClassSerializerInterceptor`，我们还可以创建我们自己的拦截器。拦截器可以服务于多种目的，其中之一是操作请求/响应流。

```typescript
// utils/excludeNull.interceptor.ts

import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';
import recursivelyStripNullValues from './recursivelyStripNullValues';

@Injectable()
export class ExcludeNullInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next
      .handle()
      .pipe(map(value => recursivelyStripNullValues(value)));
  }
}
```

每个拦截器都需要实现`NestInterceptor`，因此也就是`intercept`方法。它接受两个参数：

1. `ExecutionContext` - 它提供有关当前上下文的信息
2. `CallHandler` - 它包含调用路由处理程序并返回一个RxJS Observable的`handle`方法

`intercept`方法包装了请求/响应流，我们可以在路由处理程序执行前后添加逻辑。在上述代码中，我们调用了路由处理程序并修改了响应。

由于NestJS框架中有不少地方使用了RxJS，官方的TypeScript启动器已经包含了它。

```typescript
// utils/recursivelyStripNullValues.ts

function recursivelyStripNullValues(value: unknown): unknown {
  if (Array.isArray(value)) {
    return value.map(recursivelyStripNullValues);
  }
  if (value !== null && typeof value === 'object') {
    return Object.fromEntries(
      Object.entries(value).map(([key, value]) => [key, recursivelyStripNullValues(value)])
    );
  }
  if (value !== null) {
    return value;
  }
}
```

在上述函数中，我们递归遍历数据结构，并只保留不等于`null`的值。它对数组和普通对象都适用。

如果你想了解更多关于JavaScript中递归的信息，可以查看《使用递归遍历数据结构》。执行上下文和调用栈，同样每个递归函数都可以转换为迭代函数。

<a name="25f9c7fa"></a>
### 总结

在这篇中我们探讨了如何修改我们发送给用户的响应。虽然最直接的方法是使用`ClassSerializerInterceptor`来序列化响应，但我们也可以创建自己的拦截器。我们还探讨了如何绕过使用`@Res()`装饰器的问题。

图像格式：便携式网络图形（PNG）<br />每像素位数：24<br />颜色：真彩色<br />尺寸：1200 x 450<br />隔行扫描：是

图像格式：便携式网络图形（PNG）<br />每像素位数：24<br />颜色：真彩色<br />尺寸：570 x 423<br />隔行扫描：是

图像格式：便携式网络图形（PNG）<br />每像素位数：24<br />颜色：真彩色<br />尺寸：580 x 362<br />隔行扫描：是
