---
title: 使用NestJS的API-09
slug: 使用NestJS的API-09
pubDate: 2022-05-30
tags:
  - 后端
  - Nestjs
  - 教程
description: 使用NestJS的API-09
author: xf
cover: src/images/cat-9.webp
coverAlt: Nestjs
category:
  - 后端
---

在本系列的前一部分，我们聚焦于单元测试。这一次，我们将探讨集成测试。在这篇文章中，我们将解释集成测试的原理以及它们与单元测试的不同之处。我们将使用 Jest 编写一些集成测试来测试我们的服务，并研究 SuperTest 库来测试我们的控制器。

<a name="e3219359"></a>
### 使用集成测试测试 NestJS 服务

当我们的单元测试通过时，这表明我们系统的各部分能够很好地独立工作。然而，一个应用程序由许多需要协同工作的部分组成。集成测试的任务是验证所有部件是否能够整合在一起。我们可以通过集成系统的两个或更多部分来编写这样的测试。

让我们测试一下`AuthenticationService`如何与`UsersService`集成。

```
src/authentication/tests/authentication.service.spec.ts
```

```typescript
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
  let usersService: UsersService;
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
    usersService = await module.get(UsersService);
  });
  describe('when accessing the data of authenticating user', async () => {
    it('should attempt to get the user by email', async () => {
      const getByEmailSpy = jest.spyOn(usersService, 'getByEmail');
      await authenticationService.getAuthenticatedUser('user@email.com', 'strongPassword');
      expect(getByEmailSpy).toBeCalledTimes(1);
    });
  });
});
```

首先要注意的是，我们模拟了一些我们使用的服务。即使我们想编写一个集成测试，这并不意味着我们需要包括系统的每个部分。

<a name="9008e007"></a>
#### 模拟系统的一部分

我们需要决定包含系统的哪些部分。假设我们想测试`AuthenticationService`与`UsersService`的集成。进一步地，我们还模拟了bcrypt库。

由于我们的`AuthenticationService`直接导入它，不是直接明显的模拟它。为此，我们需要使用`jest.mock`。

```typescript
jest.mock('bcrypt');
```

现在我们明确声明模拟了bcrypt，我们可以提供它的实现。

```typescript
import * as bcrypt from 'bcrypt';

describe('The AuthenticationService', () => {
  let bcryptCompare: jest.Mock;
  beforeEach(async () => {
    bcryptCompare = jest.fn().mockReturnValue(true);
    (bcrypt.compare as jest.Mock) = bcryptCompare;
  });
});
```

通过在顶部声明`bcryptCompare`，我们现在可以为每个测试更改它的实现。

我们对存储库做了类似的事情。

```typescript
import User from '../../users/user.entity';

const mockedUser: User = {
  id: 1,
  email: 'user@email.com',
  name: 'John',
  password: 'hash',
  address: {
    id: 1,
    street: 'streetName',
    city: 'cityName',
    country: 'countryName',
  },
};
```

```typescript
import User from '../../users/user.entity';
import * as bcrypt from 'bcrypt';
import mockedUser from './user.mock';

jest.mock('bcrypt');

describe('The AuthenticationService', () => {
  let bcryptCompare: jest.Mock;
  let userData: User;
  let findUser: jest.Mock;

  beforeEach(async () => {
    bcryptCompare = jest.fn().mockReturnValue(true);
    (bcrypt.compare as jest.Mock) = bcryptCompare;

    userData = {
      ...mockedUser,
    };
    findUser = jest.fn().mockResolvedValue(userData);
    const usersRepository = {
      findOne: findUser,
    };
  });
});
```

<a name="24215504"></a>
#### 为每个测试提供不同的实现

完成以上所有工作后，我们可以为各种测试提供模拟服务的不同实现。

```typescript
describe('when accessing the data of authenticating user', () => {
  describe('and the provided password is not valid', () => {
    beforeEach(() => {
      bcryptCompare.mockReturnValue(false);
    });
    it('should throw an error', async () => {
      await expect(
        authenticationService.getAuthenticatedUser('user@email.com', 'strongPassword')
      ).rejects.toThrow();
    });
  });
  describe('and the provided password is valid', () => {
    beforeEach(() => {
      bcryptCompare.mockReturnValue(true);
    });
    describe('and the user is found in the database', () => {
      beforeEach(() => {
        findUser.mockResolvedValue(userData);
      });
      it('should return the user data', async () => {
        const user = await authenticationService.getAuthenticatedUser('user@email.com', 'strongPassword');
        expect(user).toBe(userData);
      });
    });
    describe('and the user is not found in the database', () => {
      beforeEach(() => {
        findUser.mockResolvedValue(undefined);
      });
      it('should throw an error', async () => {
        await expect(
          authenticationService.getAuthenticatedUser('user@email.com', 'strongPassword')
        ).rejects.toThrow();
      });
    });
  });
});
```

上述内容中，我们在`beforeEach`函数中指定了我们的模拟如何工作。这样做，它会在特定`describe()`块中

的所有测试之前运行。

如果你想彻底检查上述测试套件，请查看仓库中的这个文件。

<a name="bbb477b9"></a>
#### 测试控制器

我们通过执行真实请求来进行另一种类型的集成测试。通过这样做，我们可以测试我们的控制器。这更接近于我们的应用程序被使用的方式。为此，我们使用SuperTest库。

```
npm install supertest
```

现在让我们测试`AuthenticationController`如何与`AuthenticationService`和`UsersService`集成。

我们从模拟应用程序的一些部分开始。

```typescript
let app: INestApplication;
let userData: User;
beforeEach(async () => {
  userData = {
    ...mockedUser,
  };
  const usersRepository = {
    create: jest.fn().mockResolvedValue(userData),
    save: jest.fn().mockReturnValue(Promise.resolve()),
  };

  const module = await Test.createTestingModule({
    controllers: [AuthenticationController],
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
        useValue: usersRepository,
      },
    ],
  }).compile();
  app = module.createNestApplication();
  app.useGlobalPipes(new ValidationPipe());
  await app.init();
});
```

请注意，如果我们想验证我们的验证，上述我们还需要应用`ValidationPipe`。

一旦我们的模块准备就绪，我们就可以对其进行一些测试。让我们从注册流程开始。

```typescript
describe('when registering', () => {
  describe('and using valid data', () => {
    it('should respond with the data of the user without the password', () => {
      const expectedData = {
        ...userData,
      };
      delete expectedData.password;
      return request(app.getHttpServer())
        .post('/authentication/register')
        .send({
          email: mockedUser.email,
          name: mockedUser.name,
          password: 'strongPassword',
        })
        .expect(201)
        .expect(expectedData);
    });
  });
  describe('and using invalid data', () => {
    it('should throw an error', () => {
      return request(app.getHttpServer())
        .post('/authentication/register')
        .send({
          name: mockedUser.name,
        })
        .expect(400);
    });
  });
});
```

上述我们执行了真实的HTTP请求并测试了`authentication/register`端点。如果我们提供有效的数据，我们期望它能够正确工作。否则，我们期望它会抛出错误。

除了上述简单的测试之外，我们还可以进行更彻底的测试。例如，我们可以验证响应头。要查看SuperTest功能的完整列表，请查看文档。

要查看整个控制器测试套件，请在仓库中查看。

<a name="25f9c7fa"></a>
#### 总结

在这篇文章中，我们探讨了为我们的NestJS API编写集成测试的方法。除了测试我们的服务如何集成之外，我们还使用了SuperTest库并测试了一个控制器。通过编写集成测试，我们可以彻底验证我们的应用是否按预期工作。因此，这是一个值得深入探讨的话题。

© Marcin Wanago 2023 | 隐私政策<br />图像格式：便携式网络图形（PNG）<br />每像素位数：24<br />颜色：真彩色<br />尺寸：1200 x 450<br />交错：是
