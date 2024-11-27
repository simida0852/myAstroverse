---
title: 使用NestJS的API-10
slug: 使用NestJS的API-10
pubDate: 2022-05-30
tags:
  - 后端
  - Nestjs
  - 教程
description: 使用NestJS的API-10
author: xf
cover: src/images/cat-10.webp
coverAlt: Nestjs
category:
  - 后端
---

在数据库中直接存储文件虽然可行，但可能不是最佳方法。文件可能占用大量空间，这可能会影响应用程序的性能。此外，它会增加数据库的大小，因此使备份变得更大和更慢。一个好的替代方案是使用外部提供商（如Google Cloud Azure或Amazon AWS）单独存储文件。

在本文中，我们将探讨如何将文件上传到Amazon Simple Storage Service，也称为S3。你可以在这个仓库中找到本系列的所有代码。

<a name="04b905e9"></a>
### 连接到Amazon S3

Amazon S3提供了我们可以用于任何类型文件的存储空间。我们组织文件进入存储桶，并通过SDK在我们的API中管理它们。

一旦我们创建了AWS账户，我们可以作为根用户登录。尽管我们可能会授权根用户通过我们的API使用S3，但这并不是最佳方法。

<a name="b5eb59fc"></a>
#### 设置用户

让我们创建一个具有限制权限集的新用户。为此，我们需要打开Identity and Access Management（IAM）面板并创建一个用户：

由于我们希望这个用户能够管理与S3连接的一切，让我们设置适当的访问权限。

完成后，我们会得到Access key ID和Secret access key。我们需要这些通过我们的API连接到AWS。我们还需要选择一个可用的区域。

让我们将它们添加到我们的.env文件中：

```
.env

# ...
AWS_REGION=eu-central-1
AWS_ACCESS_KEY_ID=*******
AWS_SECRET_ACCESS_KEY=*******
```

也让我们将其添加到AppModule中的环境变量验证模式中：

```javascript
ConfigModule.forRoot({
  validationSchema: Joi.object({
    // ...
    AWS_REGION: Joi.string().required(),
    AWS_ACCESS_KEY_ID: Joi.string().required(),
    AWS_SECRET_ACCESS_KEY: Joi.string().required(),
    // ...
  })
})
```

<a name="c59b1a23"></a>
#### 通过SDK连接到AWS

一旦我们有了必要的变量，我们就可以使用官方的Node SDK连接到AWS了。首先安装它。

```bash
npm install aws-sdk @types/aws-sdk
```

既然我们已经拥有配置SDK所需的一切，让我们使用它。其中一种方法是在我们的main.ts文件中直接使用aws-sdk。

```javascript
main.ts

// 引入必要的库和模块
async function bootstrap() {
  // 应用的初始化和配置
  const configService = app.get(ConfigService);
  config.update({
    // 使用配置服务获取环境变量并配置AWS
  });
  // 应用监听端口
}
bootstrap();
```

<a name="79395f2a"></a>
#### 创建我们的第一个存储桶

在Amazon S3中，数据是按存储桶组织的。我们可以有多个设置不同的存储桶。

让我们打开Amazon S3面板并创建一个存储桶。请注意，存储桶的名称必须是唯一的。

我们可以设置我们的存储桶包含公共文件。我们上传到这个存储桶的所有文件都将是公开可用的。我们可能使用它来管理像头像这样的文件。

这里的最后一步是将存储桶的名称添加到我们的环境变量中。

```
.env

# ...
AWS_PUBLIC_BUCKET_NAME=nestjs-series-public-bucket
```

src/app.module.ts

```javascript
ConfigModule.forRoot({
  validationSchema: Joi.object({
    // ...
    AWS_PUBLIC_BUCKET_NAME: Joi.string().required(),
    // ...
  })
})
```

<a name="f6befa5f"></a>
#### 通过我们的API上传图片

由于我们已经设置好了AWS连接，我们可以继续上传我们的文件。作为开始，让我们创建一个PublicFile实体。

```javascript
src/files/publicFile.entity.ts

// 实体定义
```

通过直接在数据库中保存URL，我们可以非常快速地访问公共文件。`key`属性在存储桶中唯一标识文件。如果我们想要访问文件，例如如果我们想要删除它，我们需要它。

下一步是创建一个服务，该服务将文件上传到存储桶，并将有关文件的数据保存到我们的Postgres数据库中。由于我们希望密钥是唯一的，我们使用uuid库：

```bash
npm install uuid @types/uuid
```

src/files/files.service.ts

```javascript
import { Injectable } from '@nestjs/common';
// 导入必要的模块和服务

@Injectable()
export class FilesService {
  // 服务的构造函数和方法
  async uploadPublicFile(dataBuffer: Buffer, filename: string) {
    // 实现上传公共文件的逻辑
  }
}
```

<a name="0492b2aa"></a>
#### 创建上传文件的端点

现在我们需要为用户创建一个上传头像的端点。为了将文件与用户关联起来，我们需要通过添加`avatar`列来修改`UserEntity`。

```javascript
src/users/user.entity.ts

// 用户实体的更新
```

如果你想了解更多关于Postgres和TypeORM的关系，可以查看API with NestJS #7. Creating relationships with Postgres and TypeORM。

让我们向`UsersService`添加一个上传文件并将它们链接到用户的方法。

```javascript
src/users/users.service.ts

// 服务的实现
```

这可能是包含一些功能的合适位置，比如检查图像的大小或压缩它。

最后一部分是添加用户可以发送头像的端点。为此，我们遵循NestJS文档并使用在底层利用multer的`FileInterceptor`。

```javascript
src/users/users.controller.ts

// 控制器的实现
```

上面的文件有一些有用的属性，比如mimetype。如果你想进行一些额外的验证并禁止某些类型的文件，你可以使用它。

<a name="d57e4626"></a>
#### 删除现有文件

除了上传文件外，我们还需要一种删除文件的方法。为了保持我们的数据库与Amazon S3存储一致，我们从两个地方删除文件。首先

让我们在`FilesService`中添加删除文件的方法。为了保持我们的数据库与Amazon S3存储一致，我们需要从两个地方删除文件。

```javascript
src/files/files.service.ts

// Service的实现
```

现在，我们需要在`UsersService`中使用它。一个重要的补充是，当用户在已经拥有头像的情况下上传头像时，我们会删除旧的头像。

```javascript
src/users/users.service.ts

// Service的实现
```

<a name="25f9c7fa"></a>
### 总结

在这篇文章中，我们学习了Amazon S3的基础知识以及如何在我们的API中使用它。为此，我们提供了AWS SDK所需的必要凭据。多亏了这一点，我们能够上传和删除AWS上的文件。我们还保持了我们的数据库与Amazon S3同步，以跟踪我们的文件。为了通过API上传文件，我们使用了利用Multer的`FileInterceptor`。

由于Amazon S3的功能不仅限于处理公共文件，所以这里还有很多内容需要涵盖，你可能会在本系列中期待到它。
