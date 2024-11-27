---
title: 使用NestJS的API-08
slug: 使用NestJS的API-08
pubDate: 2022-05-30
tags:
  - 后端
  - Nestjs
  - 教程
description: 使用NestJS的API-08
author: xf
cover: src/images/cat-8.webp
coverAlt: Nestjs
category:
  - 后端
---
对我们的应用程序进行测试可以在创建功能齐全的API时增加我们的信心。在这篇文章中，我们将探讨如何通过编写单元测试来测试我们的应用程序。我们这样做是通过使用NestJS内置的一些实用工具以及Jest库。

如果你想先更好地了解Jest，请查看JavaScript测试教程的第一部分。

<a name="5f075566"></a>
### 使用单元测试测试NestJS

单元测试的工作是验证单独的代码片段。被测试的单元可以是一个模块、一个类或一个函数。我们的每个测试应该是独立的，彼此之间互不依赖。通过编写单元测试，我们可以确保应用程序的各个部分按预期工作。

让我们为`AuthenticationService`编写一些测试。

```typescript
src/authentication/tests/authentication.service.spec.ts

import { AuthenticationService } from '../authentication.service';
import { UsersService } from '../../users/users.service';
import { Repository } from 'typeorm';
import User from '../../users/user.entity';
import { JwtService } from '@nestjs/jwt';
import { ConfigService } from '@nestjs/config';
 
describe('The AuthenticationService', () => {
  const authenticationService = new AuthenticationService(
    new UsersService(
      new Repository<User>()
    ),
    new JwtService({
      secretOrPrivateKey: 'Secret key'
    }),
    new ConfigService()
  );
  describe('when creating a cookie', () => {
    it('should return a string', () => {
      const userId = 1;
      expect(
        typeof authenticationService.getCookieWithJwtToken(userId)
      ).toEqual('string');
    });
  });
});
```

执行`npm run test`时，Jest会查找以`.spec.ts`结尾的文件并执行它们。

我们可以改进上述代码。我们的每个测试需要是独立的，我们需要确保这一点。如果我们在上述文件中添加更多测试，所有测试都将使用相同的`AuthenticationService`实例。这违反了所有测试都是独立的规则。

为了解决这个问题，我们可以使用`beforeEach`，它会在每个测试之前运行。

```typescript
src/authentication/tests/authentication.service.spec.ts

import { AuthenticationService } from '../authentication.service';
import { UsersService } from '../../users/users.service';
import { Repository } from 'typeorm';
import User from '../../users/user.entity';
import { JwtService } from '@nestjs/jwt';
import { ConfigService } from '@nestjs/config';
 
describe('The AuthenticationService', () => {
  let authenticationService: AuthenticationService;
  beforeEach(() => {
    authenticationService = new AuthenticationService(
      new UsersService(
        new Repository<User>()
      ),
      new JwtService({
        secretOrPrivateKey: 'Secret key'
      }),
      new ConfigService()
    );
  });
  describe('when creating a cookie', () => {
    it('should return a string', () => {
      const userId = 1;
      expect(
        typeof authenticationService.getCookieWithJwtToken(userId)
      ).toEqual('string');
    });
  });
});
```

现在我们可以确保`authentication.service.spec.ts`文件中的每个测试都获得一个全新的`AuthenticationService`实例。

不幸的是，上述代码看起来不是很优雅。因为`AuthenticationService`的构造函数期望一些依赖，所以到目前为止我们手动提供了它们。

<a name="2fc742c2"></a>
### 创建测试模块

幸运的是，NestJS为我们提供了内置的实用工具来处理上述问题。

```bash
npm install @nestjs/testing
```

通过使用`Test.createTestingModule().compile()`，我们可以创建一个其依赖项已解决的模块。

```typescript
src/authentication/tests/authentication.service.spec.ts

import { AuthenticationService } from '../authentication.service';
import { Test } from '@nestjs/testing';
import { UsersModule } from '../../users/users.module';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { JwtModule } from '@nestjs/jwt';
import { DatabaseModule } from '../../database/database.module';
import * as Joi from '@hapi/joi';
 
describe('The AuthenticationService', () => {
  let authenticationService: AuthenticationService;
  beforeEach(async () => {
    const module = await Test.createTestingModule({
      imports: [
        UsersModule,
        ConfigModule.forRoot({
          validationSchema: Joi.object({
            POSTGRES_HOST: Joi.string().required(),
            POSTGRES_PORT: Joi.number().required(),
            POSTGRES_USER: Joi.string().required(),
            POSTGRES_PASSWORD: Joi.string().required(),
            POSTGRES_DB: Joi.string().required(),
            JWT_SECRET: Joi.string().required(),
            JWT_EXPIRATION_TIME: Joi.string().required(),
            PORT: Joi.number()
          })
        }),
        DatabaseModule,
        JwtModule.registerAsync({
          imports: [ConfigModule],
          inject: [ConfigService],
          useFactory: async (configService: ConfigService) => ({
            secret: configService.get('JWT_SECRET'),
            signOptions: {
              expiresIn: `${configService.get('JWT_EXPIRATION_TIME')}s`
            }
          })
        })
      ],
      providers: [
        AuthenticationService
      ]
    }).compile();
    authenticationService = await module.get(AuthenticationService);
  });
  describe('when creating a cookie', () => {
    it('should return a string', () => {
      const userId = 1;
      expect(
        typeof authenticationService.getCookieWithJwtToken(userId)
      ).toEqual('string');
    });
  });
});
```

上述代码仍然存在相当多的问题。让我们一一处理。

<a name="eae5f5c2"></a>
### 模拟数据库连接

上述最大的问题是我们使用了`DatabaseModule`，这意味着连接到真实数据库。进行单元测试时，我们希望避免这种情况。

从我们的导入中删除`DatabaseModule`后，我们可以看到一个错误：

```
Error: Nest can’t resolve dependencies of the UserRepository (?). Please make sure that the argument Connection at index [0] is available in the TypeOrmModule context.
```

为了解决这个问题，我们需要提供一个模拟的用户仓库。为

对我们的应用程序进行测试可以在创建功能齐全的API时增加我们的信心。本文将探讨如何通过编写单元测试来测试我们的应用程序。我们将利用NestJS内建的一些工具以及Jest库来实现这一目标。

如果您想先更好地了解Jest，请查看JavaScript测试教程的第一部分。

<a name="5f075566-1"></a>
### 使用单元测试测试NestJS

单元测试的任务是验证单个代码片段。被测试的单元可以是一个模块、一个类或一个函数。我们的每个测试都应该是独立的，彼此之间没有依赖。通过编写单元测试，我们可以确保应用程序的各个部分按预期工作。

让我们为`AuthenticationService`编写一些测试。

```typescript
src/authentication/tests/authentication.service.spec.ts

import { AuthenticationService } from '../authentication.service';
import { UsersService } from '../../users/users.service';
import { Repository } from 'typeorm';
import User from '../../users/user.entity';
import { JwtService } from '@nestjs/jwt';
import { ConfigService } from '@nestjs/config';

describe('The AuthenticationService', () => {
  const authenticationService = new AuthenticationService(
    new UsersService(
      new Repository<User>()
    ),
    new JwtService({
      secretOrPrivateKey: 'Secret key'
    }),
    new ConfigService()
  );
  describe('when creating a cookie', () => {
    it('should return a string', () => {
      const userId = 1;
      expect(
        typeof authenticationService.getCookieWithJwtToken(userId)
      ).toEqual('string');
    });
  });
});
```

执行`npm run test`时，Jest会查找以`.spec.ts`结尾的文件并执行它们。

我们可以改进上述代码。我们的每个测试都需要是独立的，并且我们需要确保这一点。如果我们在上述文件中添加更多测试，所有的测试都将使用相同的`AuthenticationService`实例，这违反了所有测试都应该是独立的原则。

为了解决这个问题，我们可以使用`beforeEach`，它会在每个测试之前运行。

```typescript
src/authentication/tests/authentication.service.spec.ts

import { AuthenticationService } from '../authentication.service';
import { UsersService } from '../../users/users.service';
import { Repository } from 'typeorm';
import User from '../../users/user.entity';
import { JwtService } from '@nestjs/jwt';
import { ConfigService } from '@nestjs/config';

describe('The AuthenticationService', () => {
  let authenticationService: AuthenticationService;
  beforeEach(() => {
    authenticationService = new AuthenticationService(
      new UsersService(
        new Repository<User>()
      ),
      new JwtService({
        secretOrPrivateKey: 'Secret key'
      }),
      new ConfigService()
    );
  });
  describe('when creating a cookie', () => {
    it('should return a string', () => {
      const userId = 1;
      expect(
        typeof authenticationService.getCookieWithJwtToken(userId)
      ).toEqual('string');
    });
  });
});
```

现在我们可以确保`authentication.service.spec.ts`文件中的每个测试都获得一个全新的`AuthenticationService`实例。

不幸的是，上述代码看起来不是很优雅。因为`AuthenticationService`的构造函数期望一些依赖，所以到目前为止我们手动提供了它们。

<a name="2fc742c2-1"></a>
### 创建测试模块

幸运的是，NestJS为我们提供了内置的实用工具来解决上述问题。

```bash
npm install @nestjs/testing
```

通过使用`Test.createTestingModule().compile()`，我们可以创建一个其依赖项已解决的模块。

```typescript
src/authentication/tests/authentication.service.spec.ts

import { AuthenticationService } from '../authentication.service';
import { Test } from '@nestjs/testing';
import { UsersModule } from '../../users/users.module';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { JwtModule } from '@nestjs/jwt';
import { DatabaseModule } from '../../database/database.module';
import * as Joi from '@hapi/joi';

describe('The AuthenticationService', () => {
  let authenticationService: AuthenticationService;
  beforeEach(async () => {
    const module = await Test.createTestingModule({
      imports: [
        UsersModule,
        ConfigModule.forRoot({
          validationSchema: Joi.object({
            POSTGRES_HOST: Joi.string().required(),
            POSTGRES_PORT: Joi.number().required(),
            POSTGRES_USER: Joi.string().required(),
            POSTGRES_PASSWORD: Joi.string().required(),
            POSTGRES_DB: Joi.string().required(),
            JWT_SECRET: Joi.string().required(),
            JWT_EXPIRATION_TIME: Joi.string().required(),
            PORT: Joi.number()
          })
        }),
        DatabaseModule,
        JwtModule.registerAsync({
          imports: [ConfigModule],
          inject: [ConfigService],
          useFactory: async (configService: ConfigService) => ({
            secret: configService.get('JWT_SECRET'),
            signOptions: {
              expiresIn: `${configService.get('JWT_EXPIRATION_TIME')}s`
            }
          })
        })
      ],
      providers: [
        AuthenticationService
      ]
    }).compile();
    authenticationService = await module.get<AuthenticationService>(AuthenticationService);
  });
  describe('when creating a cookie', () => {
    it('should return a string', () => {
      const userId = 1;
      expect(
        typeof authenticationService.getCookieWithJwtToken(userId)
      ).toEqual('string');
    });
  });
});
```

上述代码仍然存在一些问题，让我们一一解决它们。

<a name="eae5f5c2-1"></a>
### 模拟数据库连接

上述代码最大的问题是我们使用了`DatabaseModule`，这意味着连接到了真实数据库。在进行单元测试时，我们希望避免这种情况。

从我们的导入中移除`DatabaseModule`后，我们会看到一个错误：

```
Error: Nest can’t resolve dependencies of the UserRepository (?). Please make sure that the argument Connection at index [0] is available in the TypeOrmModule context.
```

为了解决这个问题，我们需要提供一个模拟的用户

库。要解决这个问题，我们需要使用`@nestjs/typeorm`中的`getRepositoryToken`提供一个模拟的用户仓库。

```typescript
import User from '../../users/user.entity';

providers: [
  AuthenticationService,
  {
    provide: getRepositoryToken(User),
    useValue: {}
  }
]
```

遗憾的是，上述错误仍然存在。这是因为我们导入了包含`TypeOrmModule.forFeature([User])`的`UsersModule`。在编写单元测试时，我们应该避免导入我们的模块，因为我们还不想测试类之间的集成。我们需要改为在我们的providers中添加`UsersService`。<br />src/authentication/tests
```typescript

authentication.service.spec.ts

import { AuthenticationService } from '../authentication.service';
import { Test } from '@nestjs/testing';
import { ConfigModule } from '@nestjs/config';
import { JwtModule } from '@nestjs/jwt';
import { getRepositoryToken } from '@nestjs/typeorm';
import User from '../../users/user.entity';
import { UsersService } from '../../users/users.service';

describe('The AuthenticationService', () => {
  let authenticationService: AuthenticationService;
  beforeEach(async () => {
    const module = await Test.createTestingModule({
      imports: [
        ConfigModule.forRoot({
          // ...
        }),
        JwtModule.registerAsync({
          // ...
        }),
      ],
      providers: [
        UsersService,
        AuthenticationService,
        {
          provide: getRepositoryToken(User),
          useValue: {},
        },
      ],
    }).compile();
    authenticationService = await module.get<AuthenticationService>(AuthenticationService);
  });
  describe('when creating a cookie', () => {
    it('should return a string', () => {
      const userId = 1;
      expect(
        typeof authenticationService.getCookieWithJwtToken(userId)
      ).toEqual('string');
    });
  });
});
```


我们在`useValue`中放置的对象是我们的模拟仓库。稍后我们会在下面添加一些方法到它。

<a name="b6ebbe75"></a>
### 模拟ConfigService和JwtService

由于我们希望避免使用模块，我们也可以用模拟对象替换`ConfigModule`和`JwtModule`。更确切地说，我们需要提供模拟的`ConfigService`和`JwtService`。

一个干净的方法是为上述模拟创建单独的文件。

```typescript
src/utils/mocks/config.service.ts

const mockedConfigService = {
  get(key: string) {
    switch (key) {
      case 'JWT_EXPIRATION_TIME':
        return '3600';
    }
  },
};

src/utils/mocks/jwt.service.ts

const mockedJwtService = {
  sign: () => '',
};
```

当我们使用上述模拟时，我们的测试现在看起来像这样：

```typescript
src/utils/mocks/config.service.ts

import { AuthenticationService } from '../authentication.service';
import { Test } from '@nestjs/testing';
import { ConfigService } from '@nestjs/config';
import { JwtService } from '@nestjs/jwt';
import { getRepositoryToken } from '@nestjs/typeorm';
import User from '../../users/user.entity';
import { UsersService } from '../../users/users.service';
import mockedJwtService from '../../utils/mocks/jwt.service';
import mockedConfigService from '../../utils/mocks/config.service';

describe('The AuthenticationService', () => {
  let authenticationService: AuthenticationService;
  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        UsersService,
        AuthenticationService,
        {
          provide: ConfigService,
          useValue: mockedConfigService,
        },
        {
          provide: JwtService,
          useValue: mockedJwtService,
        },
        {
          provide: getRepositoryToken(User),
          useValue: {},
        },
      ],
    }).compile();
    authenticationService = await module.get(AuthenticationService);
  });
  describe('when creating a cookie', () => {
    it('should return a string', () => {
      const userId = 1;
      expect(
        typeof authenticationService.getCookieWithJwtToken(userId)
      ).toEqual('string');
    });
  });
});
```

<a name="e9465ae1"></a>
### 根据测试更改模拟

我们并不总是想在每个测试中以相同的方式模拟某些东西。要在测试之间更改我们的实现，我们可以使用`jest.Mock`。

```typescript
src/users/tests/users.service.spec.ts

import { Test } from '@nestjs/testing';
import { getRepositoryToken } from '@nestjs/typeorm';
import User from '../../users/user.entity';
import { UsersService } from '../../users/users.service';

describe('The UsersService', () => {
  let usersService: UsersService;
  let findOne: jest.Mock;
  beforeEach(async () => {
    findOne = jest.fn();
    const module = await Test.createTestingModule({
      providers: [
        UsersService,
        {
          provide: getRepositoryToken(User),
          useValue: {
            findOne,
          },
        },
      ],
    }).compile();
    usersService = await module.get(UsersService);
  });
  describe('when getting a user by email', () => {
    describe('and the user is matched', () => {
      let user: User;
      beforeEach(() => {
        user = new User();
        findOne.mockReturnValue(Promise.resolve(user));
      });
      it('should return the user', async () => {
        const fetchedUser = await usersService.getByEmail('test@test.com');
        expect(fetchedUser).toEqual(user);
      });
    });
    describe('and the user is not matched', () => {
      beforeEach(() => {
        findOne.mockReturnValue(undefined);
      });
      it('should throw an error', async () => {
        await expect(usersService.getByEmail('test@test.com')).rejects.toThrow();
      });
    });
  });
});
```
<a name="25f9c7fa"></a>
### 总结
本文探讨了如何在NestJS中编写单元测试。为此，我们使用了NestJS附带的Jest库。我们还使用了一些内建的实用工具来适当地模拟各种服务和模块。其中最重要的是模拟数据库连接，以便我们可以保持我们的测试是独立的。
