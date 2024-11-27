---
title: 使用NestJS的API-04
slug: 使用NestJS的API-04
pubDate: 2022-05-30
tags:
  - 后端
  - Nestjs
  - 教程
description: 使用NestJS的API-04
author: xf
cover: src/images/cat-3.webp
coverAlt: Nestjs
category:
  - 后端
---
NestJS 在处理错误和验证数据方面表现出色，这很大程度上得益于使用了装饰器。本文将介绍 NestJS 提供的一些功能，如异常过滤器和验证管道。

本系列代码位于此仓库，旨在成为官方 Nest 框架 TypeScript 入门版本的扩展。

<a name="b4d2695a"></a>
### 异常过滤器

Nest 有一个异常过滤器，负责处理我们应用中的错误。当我们没有自己处理异常时，异常过滤器会为我们处理。它会处理异常，并以用户友好的格式在响应中发送。

默认的异常过滤器叫做 `BaseExceptionFilter`。我们可以查看 NestJS 的源码以了解其行为。

```typescript
export class BaseExceptionFilter<T = any> implements ExceptionFilter<T> {
  // ...
  catch(exception: T, host: ArgumentsHost) {
    // ...
    if (!(exception instanceof HttpException)) {
      return this.handleUnknownError(exception, host, applicationRef);
    }
    const res = exception.getResponse();
    const message = isObject(res) ? res : { statusCode: exception.getStatus(), message: res };
    // ...
  }
  public handleUnknownError(exception: T, host: ArgumentsHost, applicationRef: AbstractHttpAdapter | HttpServer) {
    const body = { statusCode: HttpStatus.INTERNAL_SERVER_ERROR, message: MESSAGES.UNKNOWN_EXCEPTION_MESSAGE };
    // ...
  }
}
```

每当应用中出现错误时，`catch` 方法就会运行。从上面的代码中，我们可以得到一些关键信息。

<a name="HttpException"></a>
#### HttpException

Nest 期望我们使用 `HttpException` 类。如果我们不使用，它会将错误解释为意外，并响应 500 内部服务器错误。

在本系列的前几部分中，我们已经使用过 `HttpException`：

```typescript
throw new HttpException('Post not found', HttpStatus.NOT_FOUND);
```

构造函数需要两个参数：响应体和状态码。对于后者，我们可以使用提供的 `HttpStatus` 枚举。

如果我们提供一个字符串作为响应的定义，NestJS 会将其序列化为包含两个属性的对象：

- `statusCode`：包含我们选择的 HTTP 状态码
- `message`：我们提供的描述

我们可以通过提供一个对象作为 `HttpException` 构造函数的第一个参数来覆盖上述行为。

我们经常会发现自己多次抛出类似的异常。为了避免代码重复，我们可以创建自定义异常。要做到这一点，我们需要扩展 `HttpException` 类。

```typescript
import { HttpException, HttpStatus } from '@nestjs/common';
 
class PostNotFoundException extends HttpException {
  constructor(postId: number) {
    super(`Post with id ${postId} not found`, HttpStatus.NOT_FOUND);
  }
}
```

我们的自定义 `PostNotFoundException` 调用了 `HttpException` 的构造函数。因此，我们可以通过不必每次想抛出错误时都定义消息来清理我们的代码。

NestJS 有一组扩展了 `HttpException` 的异常。其中之一是 `NotFoundException`。我们可以重构上面的代码并使用它。

我们可以在文档中找到内置 HTTP 异常的完整列表。

```typescript
import { NotFoundException } from '@nestjs/common';
 
class PostNotFoundException extends NotFoundException {
  constructor(postId: number) {
    super(`Post with id ${postId} not found`);
  }
}
```

`NotFoundException` 类的第一个参数是一个额外的 `error` 属性。这样，我们的 `message` 由 `NotFoundException` 定义，并基于状态。

<a name="7cd0c03a"></a>
#### 扩展 `BaseExceptionFilter`

默认的 `BaseExceptionFilter` 可以处理大多数常规情况。然而，我们可能想以某种方式修改它。最简单的方法是创建一个扩展它的过滤器。

```typescript
import { Catch, ArgumentsHost } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';
 
@Catch()
export class ExceptionsLoggerFilter extends BaseExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) { console.log('异常抛出', exception);
    super.catch(exception, host);
  }
}
```
`@Catch()` 装饰器意味着我们想要我们的过滤器捕获所有异常。我们可以为其提供单个异常类型或一个列表。

`ArgumentsHost` 为我们提供了访问应用执行上下文的方式。我们将在本系列的后续部分中探讨这一点。

我们可以通过三种方式使用我们的新过滤器。第一种方式是通过 `app.useGlobalFilters` 在所有路由中全局使用它。

```typescript

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  const { httpAdapter } = app.get(HttpAdapterHost);
  app.useGlobalFilters(new ExceptionsLoggerFilter(httpAdapter));
  app.use(cookieParser());
  await app.listen(3000);
}
bootstrap();

```

更好的全局注入我们过滤器的方式是将其添加到我们的 `AppModule` 中。这样，我们可以向我们的过滤器注入额外的依赖项。

```typescript
@Module({
  // ...
  providers: [
    {
      provide: APP_FILTER,
      useClass: ExceptionsLoggerFilter,
    },
  ],
})
export class AppModule {}
```

绑定过滤器的第三种方式是使用 `@UseFilters` 装饰器。我们可以为其提供单个过滤器或一个过滤器列表。

```typescript
@Get(':id')
@UseFilters(ExceptionsLoggerFilter)
getPostById(@Param('id') id: string) {
  return this.postsService.getPostById(Number(id));
}
```

以上并不是记录异常的最佳方法。NestJS 有一个内置的日志记录器，我们将在本系列的后续部分中介绍。

<a name="b5ed624a"></a>
#### 实现 `ExceptionFilter` 接口

如果我们需要为错误定制完全自定义的行为，我们可以从头开始构建我们的过滤器。它需要实现 `ExceptionFilter` 接口。让我们来看一个例子：

```typescript
@Catch(NotFoundException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: NotFoundException, host: ArgumentsHost) {
    const context = host.switchToHttp();
    const response = context.getResponse<Response>();
    const request = context.getRequest<Request>();
    const status = exception.getStatus();

    const message = exception.getMessage();
 
    response.status(status).json({
      message,
      statusCode: status,
      time: new Date().toISOString(),
    });
  }
}
```

上述有几个值得注意的地方。由于我们使用 `@Catch(NotFoundException)`，这个过滤器只针对 `NotFoundException` 运行。

`host.switchToHttp` 方法返回 `HttpArgumentsHost` 对象，包含有关 HTTP 上下文的信息。我们将在本系列的后续部分中，讨论执行上下文时，更多地探索这一点。

<a name="cd8992b6"></a>
### 验证

我们绝对应该验证传入的数据。在 TypeScript Express 系列中，我们使用 `class-validator` 库。NestJS 也集成了它。

NestJS 带有一套内置的管道。管道通常用于转换输入数据或验证它。今天我们只使用预定义的管道，但在本系列的后续部分中，我们可能会看看如何创建自定义的管道。

要开始验证数据，我们需要 `ValidationPipe`。

```typescript
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  app.use(cookieParser());
  await app.listen(3000);
}
bootstrap();
```

在本系列的第一部分中，我们创建了数据传输对象（DTO）。它们定义了请求中发送的数据格式。它们是附加验证的完美地方。

```bash
npm install class-validator class-transformer
```

为了使 `ValidationPipe` 工作，我们还需要 `class-transformer` 库。

```typescript
import { IsEmail, IsString, IsNotEmpty, MinLength } from 'class-validator';
 
export class RegisterDto {
  @Is

Email()
  email: string;
 
  @IsString()
  @IsNotEmpty()
  name: string;
 
  @IsString()
  @IsNotEmpty()
  @MinLength(7)
  password: string;
}
 
export default RegisterDto;
```

多亏了我们将上述 `RegisterDto` 与 `@Body()` 装饰器一起使用，`ValidationPipe` 现在检查数据。

```typescript
@Post('register')
async register(@Body() registrationData: RegisterDto) {
  return this.authenticationService.register(registrationData);
}
```

我们可以使用更多的装饰器。查看 `class-validator` 文档以获取完整列表。您还可以创建自定义验证装饰器。

<a name="e67af0ac"></a>
#### 验证参数

我们也可以使用 `class-validator` 库来验证参数。

```typescript
import { IsNumberString } from 'class-validator';
 
class FindOneParams {
  @IsNumberString()
  id: string;
}
```

```typescript
@Get(':id')
getPostById(@Param() { id }: FindOneParams) {
  return this.postsService.getPostById(Number(id));
}
```

请注意，我们不再在这里使用 `@Param('id')`。相反，我们解构了整个 params 对象。

如果您使用 MongoDB 而不是 Postgres，`@IsMongoId()` 装饰器可能会在这里对您有用。

<a name="8e33794d"></a>
#### 处理 PATCH

在 TypeScript Express 系列中，我们讨论了 PUT 和 PATCH 方法之间的区别。总结起来，PUT 替换一个实体，而 PATCH 应用部分修改。执行部分更改时，我们需要跳过缺失的属性。

处理 PATCH 最直接的方式是向我们的 `ValidationPipe` 传递 `skipMissingProperties`。

```typescript
app.useGlobalPipes(new ValidationPipe({ skipMissingProperties: true }));
```

不幸的是，这将跳过我们所有 DTO 中的缺失属性。当发布数据时，我们不想这样做。相反，我们可以在更新数据时将 `IsOptional` 添加到所有属性中。

```typescript
import { IsString, IsNotEmpty, IsNumber, IsOptional } from 'class-validator';
 
export class UpdatePostDto {
  @IsNumber()
  @IsOptional()
  id: number;
 
  @IsString()
  @IsNotEmpty()
  @IsOptional()
  content: string;
 
  @IsString()
  @IsNotEmpty()
  @IsOptional()
  title: string;
}
```

不幸的是，上述解决方案并不十分干净。这里提供了一些解决方案来覆盖 `ValidationPipe` 的默认行为。

在本系列的后续部分中，我们将探讨如何实现 PUT 而不是 PATCH。

<a name="25f9c7fa"></a>
### 总结

在这篇文章中，我们探讨了 NestJS 中的错误处理和验证是如何工作的。通过了解默认 `BaseExceptionFilter` 的内部工作原理，我们现在知道如何正确处理各种异常。我们也知道了如果有这样的需求，如何改变默认行为。我们还了解了如何使用 `ValidationPipe` 和 `class-validator` 库来验证传入的数据。

NestJS 框架还有很多内容需要覆盖，敬请期待！
