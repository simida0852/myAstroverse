---
title: 使用NestJS的API-03
slug: 使用NestJS的API-03
pubDate: 2022-05-30
tags:
  - 后端
  - Nestjs
  - 教程
description: 使用NestJS的API-03
author: xf
cover: src/images/cat-2.webp
coverAlt: Nestjs
category:
  - 后端
---
认证是几乎所有网络应用程序的关键部分。有很多方法来处理认证问题，我们在TypeScript Express系列中手动处理过。这一次，我们将探讨最受欢迎的Node.js认证库——Passport。同时，我们还会注册用户并通过散列（hashing）来保护他们的密码。

你可以在这个系列的仓库中找到所有代码。随时给它点个星。

定义用户实体<br />考虑认证的第一件事是注册我们的用户。为此，我们需要为用户定义一个实体。

```typescript
// users/user.entity.ts

import { Column, Entity, PrimaryGeneratedColumn } from 'typeorm';

@Entity()
class User {
  @PrimaryGeneratedColumn()
  public id?: number;

  @Column({ unique: true })
  public email: string;

  @Column()
  public name: string;

  @Column()
  public password: string;
}

export default User;
```

上面的唯一新事物是`unique`标志。它表明不能有两个电子邮件相同的用户。这个功能内置于PostgreSQL中，帮助我们保持数据的一致性。稍后在认证时，我们依赖电子邮件的唯一性。

我们需要对用户执行一些操作。为此，让我们创建一个服务。

```typescript
// users/users.service.ts

import { HttpException, HttpStatus, Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import User from './user.entity';
import CreateUserDto from './dto/createUser.dto';

@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private usersRepository: Repository<User>
  ) {}

  async getByEmail(email: string) {
    const user = await this.usersRepository.findOne({ email });
    if (user) {
      return user;
    }
    throw new HttpException('该电子邮件的用户不存在', HttpStatus.NOT_FOUND);
  }

  async create(userData: CreateUserDto) {
    const newUser = await this.usersRepository.create(userData);
    await this.usersRepository.save(newUser);
    return newUser;
  }
}

// users/dto/createUser.dto.ts

export class CreateUserDto {
  email: string;
  name: string;
  password: string;
}

export default CreateUserDto;
```

以上全部内容都被一个模块包装起来。

```typescript
// users/users.module.ts

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

处理密码<br />关于注册的一个基本事情是我们不想以明文形式保存密码。如果我们的数据库被侵入，我们的密码就会直接暴露出来。

为了使密码更加安全，我们将它们散列。在这个过程中，散列算法将一个字符串转换成另一个字符串。如果我们改变字符串的一个字符，结果完全不同。

上述操作只能单向执行，不容易逆转。这意味着我们不知道用户的密码。当用户尝试登录时，我们需要再次执行这个操作。然后，我们将结果与数据库中保存的结果进行比较。

由于对同一个字符串进行两次散列会得到相同的结果，我们使用盐（salt）。它防止具有相同密码的用户拥有相同的散列值。盐是添加到原始密码中的随机字符串，以每次获得不同的结果。

使用bcrypt<br />我们使用由bcrypt npm包实现的bcrypt散列算法。它负责散列字符串、比较明文字符串与散列值以及添加盐。

使用bcrypt可能是一个对CPU密集的任务。幸运的是，我们的bcrypt实现使用了一个线程池，允许它在一个额外的线程中运行。因此，我们的应用程序在生成散列时可以执行其他任务。

```
npm install @types/bcrypt bcrypt
```

当我们使用bcrypt时，我们定义盐轮（salt rounds）。它归结为成本因素，控制接收结果所需的时间。每增加一次，时间就会翻倍。成本因素越大，使用暴力破解逆转散列就越困难。一般来说，10个盐轮就足够了。

用于散列的盐是结果的一部分，所以不需要单独保留。

```typescript
const passwordInPlaintext = '12345678';

const hash = await bcrypt.hash(passwordInPlaintext, 10);

const isPasswordMatching = await bcrypt.compare(passwordInPlaintext, hashedPassword);
console.log(isPasswordMatching); // true
```

创建认证服务<br />有了上述知识，我们可以开始实现基本的注册和登录功能。为此，我们首先需要定义一个认证服务。

认证意味着检查用户的身份。它提供了一个问题的答案：用户是谁？

授权是关于访问资源的。它回答的问题是：用户有权执行这个操作吗？

```typescript
// authentication/authentication.service.ts

export class AuthenticationService {
  constructor(
    private readonly usersService: UsersService
  ) {}

  public async register(registrationData: RegisterDto) {
    const hashedPassword = await bcrypt.hash(registrationData.password, 10);
    try {
      const createdUser = await this.usersService.create({
        ...registrationData,
        password: hashedPassword,
      });
      createdUser.password = undefined;
      return createdUser;
    } catch (error) {
      if (error?.code === PostgresErrorCode.UniqueViolation) {
        throw new HttpException('该电子邮件的用户已存在', HttpStatus.BAD_REQUEST);
      }
      throw new HttpException('出了点问题', HttpStatus.INTERNAL_SERVER_ERROR);
    }
  }
  // (...)
}
```

createdUser.password = undefined并不是不发送响应中的密码的最干净的方式。在这个系列的后续部分，我们会探索帮助我们处理这一问题的机制。

上述过程中发生了几件值得注意的事情。我们创建了一个哈希并将其连同其他数据一起传递给`usersService.create`方法。这里我们使用了`try...catch`语句，因为当它可能失败时，有一个重要的情况需要处理。

如果已经存在具有该电子邮件的用户，则`usersService.create`方法会抛出一个错误。由于我们的唯一列导致了这个错误，错误来自Postgres。

为了理解这个错误，我们需要查阅PostgreSQL错误代码文档页面。由于`unique_violation`的代码是`23505`，我们可以创建一个枚举来干净地处理它。

```typescript
// database/postgresErrorCodes.enum.ts

enum PostgresErrorCode {
  UniqueViolation = '23505'
}
```

由于在上述服务中我们明确指出这个电子邮件的用户已经存在，实现一个机制防止攻击者对我们的API进行暴力破解以获取注册电子邮件列表可能是个好主意。

我们接下来需要实现的是登录。

```typescript
// authentication/authentication.service.ts

export class AuthenticationService {
  constructor(
    private readonly usersService: UsersService
  ) {}

  // (...)

  public async getAuthenticatedUser(email: string, hashedPassword: string) {
    try {
      const user = await this.usersService.getByEmail(email);
      const isPasswordMatching = await bcrypt.compare(
        hashedPassword,
        user.password
      );
      if (!isPasswordMatching) {
        throw new HttpException('提供的凭据错误', HttpStatus.BAD_REQUEST);
      }
      user.password = undefined;
      return user;
    } catch (error) {
      throw new HttpException('提供的凭据错误', HttpStatus.BAD_REQUEST);
    }
  }
}
```

上述代码中重要的一点是，无论电子邮件还是密码错误，我们返回相同的错误。这样做可以防止一些旨在获取我们数据库中注册电子邮件列表的攻击。

关于上述代码，我们可能想要改进的一个小事情是，在我们的`logIn`方法中，我们抛出了一个我们随后在本地捕获的异常。这可能被认为是令人困惑的。让我们创建一个单独的方法来验证密码：

```typescript
public async getAuthenticatedUser(email: string, plainTextPassword: string) {
  try {
    const user = await this.usersService.getByEmail(email);
    await this.verifyPassword(plainTextPassword, user.password);
    user.password = undefined;
    return user;
  } catch (error) {
    throw new HttpException('提供的凭据错误', HttpStatus.BAD_REQUEST);
  }
}

private async verifyPassword(plainTextPassword: string, hashedPassword: string) {
  const isPasswordMatching = await bcrypt.compare(
    plainTextPassword,
    hashedPassword
  );
  if (!isPasswordMatching) {
    throw new HttpException('提供的凭据错误', HttpStatus.BAD_REQUEST);
  }
}
```

将我们的认证与Passport集成<br />在TypeScript Express系列中，我们手动处理了整个认证过程。NestJS文档建议使用Passport库，并为我们提供了这样做的手段。Passport为我们提供了认证的抽象，从而减轻了我们的一些重担。此外，它在许多开发者的生产环境中经过了充分的测试。

深入研究如何在没有Passport的情况下手动实现认证仍然是一个好主意。通过这样做，我们可以更好地理解这个过程。

应用程序对认证有不同的方法。Passport称这些机制为策略。我们想要实现的第一种策略是passport-local策略。这是一种使用用户名和密码进行认证的策略。

```
npm install @nestjs/passport passport @types/passport-local passport-local @types/express
```

要配置策略，我们需要提供一组特定于特定策略的选项。在NestJS中，我们通过扩展`PassportStrategy`类来做到这一点。

```typescript
// authentication/local.strategy.ts

import { Strategy } from 'passport-local';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable } from '@nestjs/common';
import { AuthenticationService } from './authentication.service';
import User from '../users/user.entity';

@Injectable()
export class LocalStrategy extends PassportStrategy(Strategy) {
  constructor(private authenticationService: AuthenticationService) {
    super({
      usernameField: 'email'
    });
  }
  async validate(email: string, password: string): Promise<User> {
    return this.authenticationService.getAuthenticatedUser(email, password);
  }
}
```

对于每种策略，Passport使用一组特定于特定策略的参数调用`validate`函数。对于本地策略，Passport需要一个具有用户名和密码的方法。在我们的案例中，电子邮件充当用户名。

我们还需要配置我们的AuthenticationModule以使用Passport。

```typescript
// authentication/authentication.module.ts

import { Module } from '@nestjs/common';
import { AuthenticationService } from './authentication.service';
import { UsersModule } from '../users/users.module';
import { AuthenticationController } from './authentication.controller';
import { PassportModule } from '@nestjs/passport';
import { LocalStrategy } from './local.strategy';

@Module({
  imports: [UsersModule, PassportModule],
  providers: [AuthenticationService, LocalStrategy],
  controllers: [AuthenticationController]
})
export class AuthenticationModule {}
```

使用内置的Passport Guards<br />上述模块使用了`AuthenticationController`。现在让我们创建它的基础。

下面我们使用Guards。Guard负责确定路由处理程序是否处理请求。本质上，它类似于Express.js中间件，但功能更强大。

我们将在这个系列的后续部分中详细关注守卫，并创建自定义守卫。今天，我们只使用现有的守卫。

```typescript
// authentication/authentication.controller.ts

import { Body, Controller, HttpCode, Post, UseGuards, Req } from '@nestjs/common';
import { AuthenticationService } from './authentication.service';
import RegisterDto from './dto/register.dto';
import RequestWithUser from './requestWithUser.interface';
import { LocalAuthenticationGuard } from './localAuthentication.guard';

@Controller('authentication')
export class AuthenticationController {
  constructor(
    private readonly authenticationService: AuthenticationService
  ) {}

  @Post('register')
  async register(@Body() registrationData: RegisterDto) {
    return this.authenticationService.register(registrationData);
  }

  @HttpCode(200)
  @UseGuards(LocalAuthenticationGuard)
  @Post('log-in')
  async logIn(@Req() request: RequestWithUser) {
    const user = request.user;
    user.password = undefined;
    return user;
  }
}
```

上面我们使用`@HttpCode(200)`，因为NestJS默认对POST请求响应`201 Created`。

```typescript
// authentication/localAuthentication.guard.ts

import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class LocalAuthenticationGuard extends AuthGuard('local') {}
```

直接将策略名称传递给`AuthGuard()`在控制器中可能不被认为是一种干净的方式。相反，我们创建我们自己的类。

```typescript
// authentication/requestWithUser.interface.ts

import { Request } from 'express';
import User from '../users/user.entity';

interface RequestWithUser extends Request {
  user: User;
}

export default RequestWithUser;
```

通过做上述所有事情，我们的/log-in路由由Passport处理。用户的数据附加到request对象上，这就是为什么我们扩展了Request接口。

如果用户成功认证，我们返回他的数据。否则，我们抛出一个错误。

使用JSON Web Tokens<br />我们的目标是限制应用程序的某些部分。这样，只有经过认证的用户才能访问它们。我们不希望他们每次请求时都需要认证。相反，我们需要一种方式让用户表明他们已经成功登录。

一种简单的方式是使用JSON Web Tokens。JWT是一个在我们的服务器上使用秘密密钥创建的字符串，只有我们可以解码它。我们希望在用户登录时给他们，以便它可以在每次请求时被发送回来。如果令牌有效，我们可以信任用户的身份。

```
npm install @nestjs/jwt passport-jwt @types/passport-jwt cookie-parser @types/cookie-parser
```

首先要做的是添加两个新的环境变量：JWT_SECRET和JWT_EXPIRATION_TIME。

我们可以使用任何字符串作为JWT秘密密钥。重要的是保守这个秘密，不要分享它。我们使用它在我们的应用程序中编码和解码令牌。

我们以秒为单位描述我们的过期时间，以增加安全性。如果某人的令牌被盗，攻击者可以以类似拥有密码的方式访问应用程序。由于过期时间的存在，这个问题部分得到解决，因为令牌会过期。

```typescript
// app.module.ts

ConfigModule.forRoot({
  validationSchema: Joi.object({
    //...
    JWT_SECRET: Joi.string().required(),
    JWT_EXPIRATION_TIME: Joi.string().required(),
  })
})
```

生成令牌<br />在这篇文章中，我们希望用户将JWT存储在cookies中。这比在Web存储中存储令牌有一定的优势，感谢HttpOnly指令。它不能直接通过浏览器中的JavaScript访问，使其更安全，更抵御如跨站脚本攻击等攻击。

如果您想了解更多关于cookies的信息，请查看Cookies: explaining document.cookie and the Set-Cookie header。

现在让我们配置JwtModule。

```typescript
// authentication/authentication.module.ts

import { Module } from '@nestjs/common';
import { AuthenticationService } from './authentication.service';
import { UsersModule } from '../users/users.module';
import { AuthenticationController } from './authentication.controller';
import { PassportModule } from '@nestjs/passport';
import { JwtModule } from '@nestjs/jwt';
import { ConfigModule, ConfigService } from '@nestjs/config';

@Module({
  imports: [
    UsersModule,
    PassportModule,
    ConfigModule,
    JwtModule.registerAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: async (configService: ConfigService) => ({
        secret: configService.get('JWT_SECRET'),
        signOptions: {
          expiresIn: `${configService.get('JWT_EXPIRATION_TIME')}s`,
        },
      }),
    }),
  ],
  providers: [AuthenticationService, LocalStrategy],
  controllers: [AuthenticationController],
})
export class AuthenticationModule {}
```

感谢上述内容，我们现在可以在我们的AuthenticationService中使用JwtService。

```typescript
// authentication/authentication.service.ts

@Injectable()
export class AuthenticationService {
  constructor(
    private readonly usersService: UsersService,
    private readonly jwtService: JwtService,
    private readonly configService: ConfigService,
  ) {}

  // ...

  public getCookieWithJwtToken(userId: number) {
    const payload: TokenPayload = { userId };
    const token = this.jwtService.sign(payload);
    return `Authentication=${token}; HttpOnly; Path=/; Max-Age=${this.configService.get('JWT_EXPIRATION_TIME')}`;
  }
}

// authentication/tokenPayload.interface.ts

interface TokenPayload {
  userId: number;
}
```

当用户成功登录时，我们需要通过发送`Set-Cookie`头部来发送`getCookieWithJwtToken`方法创建的令牌。为此，我们需要直接使用Response对象。

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
当浏览器接收到这个响应时，它设置了cookie，以便稍后使用。

接收令牌<br />为了能够轻松读取cookies，我们需要`cookie-parser`。

```typescript

// main.ts

import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import * as cookieParser from 'cookie-parser';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.use(cookieParser());
  await app.listen(3000);
}
bootstrap();
```

现在我们需要在用户请求数据时从Cookie头中读取令牌。为此，我们需要第二个passport策略。

```typescript
// authentication/jwt.strategy.ts

import { ExtractJwt, Strategy } from 'passport-jwt';
import { PassportStrategy } from '@nestjs/passport';
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { Request } from 'express';
import { UsersService } from '../users/users.service';
import TokenPayload from './tokenPayload.interface';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(
    private readonly configService: ConfigService,
    private readonly userService: UsersService,
  ) {
    super({
      jwtFromRequest: ExtractJwt.fromExtractors([(request: Request) => {
        return request?.cookies?.Authentication;
      }]),
      secretOrKey: configService.get('JWT_SECRET'),
    });
  }

  async validate(payload: TokenPayload) {
    return this.userService.getById(payload.userId);
  }
}
```

上面有几个值得注意的地方。我们通过从cookie中读取令牌来扩展默认的JWT策略。

当我们成功访问令牌时，我们使用里面编码的用户id。有了它，我们可以通过`userService.getById`方法获取整个用户数据。我们还需要将其添加到我们的`UsersService`中。

```typescript
// users/users.service.ts

@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private usersRepository: Repository<User>,
  ) {}

  async getById(id: number) {
    const user = await this.usersRepository.findOne({ id });
    if (user) {
      return user;
    }
    throw new HttpException('这个id的用户不存在', HttpStatus.NOT_FOUND);
  }

  // (...)
}
```

得益于在令牌编码过程中运行的`validate`方法，我们可以访问所有用户数据。

我们现在需要将我们的新JwtStrategy添加到`AuthenticationModule`中。

```typescript
// authentication/authentication.module.ts

@Module({
  // (...)
  providers: [AuthenticationService, LocalStrategy, JwtStrategy],
})
export class AuthenticationModule {}
```

要求我们的用户认证<br />现在我们可以要求我们的用户在向我们的API发送请求时进行认证。首先，我们需要创建我们的`JwtAuthenticationGuard`。

```typescript
// authentication/jwt-authentication.guard.ts

import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export default class JwtAuthenticationGuard extends AuthGuard('jwt') {}
```

现在，每当我们希望我们的用户在发起请求之前进行认证时，我们都可以使用它。例如，我们可能希望在通过我们的API创建帖子时这样做。

```typescript
// posts/posts.controller.ts

import { Body, Controller, Post, UseGuards } from '@nestjs/common';
import PostsService from './posts.service';
import CreatePostDto from './dto/createPost.dto';
import JwtAuthenticationGuard from '../authentication/jwt-authentication.guard';

@Controller('posts')
export default class PostsController {
  constructor(
    private readonly postsService: PostsService,
  ) {}

  @Post()
  @UseGuards(JwtAuthenticationGuard)
  async createPost(@Body() post: CreatePostDto) {
    return this.postsService.createPost(post);
  }

  // (...)
}
```

登出

JSON Web Tokens是无状态的。我们不能以直接的方式改变一个令牌使其无效。实现登出的最简单方式就是从浏览器中移除令牌。由于我们设计的cookies是`HttpOnly`的，我们需要创建一个清除它的端点。

```typescript
// authentication/authentication.service.ts

export class AuthenticationService {
  // (...)
 
  public getCookieForLogOut() {
    return `Authentication=; HttpOnly; Path=/; Max-Age=0`;
  }
}

// authentication/authentication.controller.ts

@Controller('authentication')
export class AuthenticationController {
  // (...)
  @UseGuards(JwtAuthenticationGuard)
  @Post('log-out')
  async logOut(@Req() request: RequestWithUser, @Res() response: Response) {
    response.setHeader('Set-Cookie', this.authenticationService.getCookieForLogOut());
    return response.sendStatus(200);
  }
}
```

验证令牌<br />我们需要的一个重要额外功能是验证JSON Web Tokens并返回用户数据。这样，浏览器可以检查当前令牌是否有效，并获取当前登录用户的数据。

```typescript
@Controller('authentication')
export class AuthenticationController {
  // (...)
  @UseGuards(JwtAuthenticationGuard)
  @Get()
  authenticate(@Req() request: RequestWithUser) {
    const user = request.user;
    user.password = undefined;
    return user;
  }
}
```

总结<br />在这篇文章中，我们覆盖了在NestJS中注册和登录用户。为了实现它，我们使用了bcrypt来散列密码以确保它们的安全。为了认证用户，我们使用了JSON Web Tokens。上述功能仍有改进的空间。例如，我们应该更干净地排除密码。此外，我们可能想要实现令牌刷新功能。请继续关注更多关于NestJS的文章！
