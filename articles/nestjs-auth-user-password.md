---
title: "NestJSでJWT認証を実装する"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nestjs"]
published: true
---

[Authentication](https://docs.nestjs.com/security/authentication) を参考に NestJS で認証機能を実装します。

## 必要なライブラリをインストールする

```shell
npm i bcrypt @nestjs/passport passport passport-local @nestjs/jwt passport-jwt
```

```shell
npm i -D @types/passport-local @types/passport-jwt
```

## Users リソースを作成する

NestJS CLI では `res` を引数に与えることで DTO, Controller, Entity, Module, Service などをまとめて作成できます。
ここでは `users` リソースを作成します。

```shell
nest g  --no-spec res users

# ? What transport layer do you use? REST API
# ? Would you like to generate CRUD entry points? Yes
```

## User エンティティ を修正する

自動で作成された `users/entities/user.entity.ts` を修正します。

```ts:src/users/entities/user.entity.ts
import {
  Column,
  CreateDateColumn,
  Entity,
  PrimaryGeneratedColumn,
  Timestamp,
  UpdateDateColumn,
} from 'typeorm';

@Entity('users')
export class User {
  @PrimaryGeneratedColumn({
    name: 'id',
    unsigned: true,
    type: 'smallint',
    comment: 'ID',
  })
  readonly id: number;

  @Column('varchar', { comment: 'ユーザー名' })
  username: string;

  @Column('varchar', { name: 'hashed_password', comment: 'パスワードハッシュ' })
  hashedPassword: string;

  @CreateDateColumn({
    name: 'create_at',
    comment: '作成時刻',
  })
  readonly createdAt?: Timestamp;

  @UpdateDateColumn({
    name: 'updated_at',
    comment: '更新時刻',
  })
  readonly updatedAt?: Timestamp;
}
```

## マイグレーションを実行して users テーブルを作成

エンティティを作成した後は、マイグレーションを実行します。

`src/database/database.module.ts` に User の定義を追加します。

```ts:src/database/database.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from '../users/entities/user.entity';

export const databaseEntities = [User];
export const migrationFilesDir = './migrations/*.ts';

@Module({
  imports: [
    TypeOrmModule.forRootAsync({
      imports: [
        ConfigModule.forRoot({
          envFilePath: ['.env'],
        }),
      ],
      inject: [ConfigService],
      useFactory: (configService: ConfigService) => ({
        type: 'mysql',
        host: configService.get('DATABASE_HOST'),
        port: configService.get('DATABASE_PORT'),
        database: configService.get('DATABASE_NAME'),
        username: configService.get('DATABASE_USER'),
        password: configService.get('DATABASE_PASSWORD'),
        entities: databaseEntities,
        synchronize: false,
      }),
    }),
  ],
})
export class DatabaseModule {}

```

ここで、`src/database/database.module.ts` ってなんぞ！？と感じた方は、1 つ前の記事を参照してください。
TypeORM の設定ファイルです。

[NestJS+TypeORM 0.3 系でマイグレーションを実行する](https://zenn.dev/fjsh/articles/nestjs-typeorm-migration)

Docker コンテナに入ります。

```shell
docker exec -it nest /bin/sh
```

マイグレーションファイルを作成します。

```shell
npx typeorm-ts-node-esm migration:generate ./migrations/CreateUser -d src/database/database.source.ts
```

マイグレーションを実行します。

```shell
npx typeorm-ts-node-esm migration:run -d src/database/database.source.ts
```

## User Repository を作成する

UsersService から Repository を通じてデータをテーブルに格納します。
そのための `users.repository.ts` を作成します。

```shell
touch src/users/users.repository.ts
```

```ts:src/users/users.repository.ts
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from './entities/user.entity';
import { Injectable } from '@nestjs/common';

@Injectable()
export class UsersRepository extends Repository<User> {
  constructor(@InjectRepository(User) repository: Repository<User>) {
    super(repository.target, repository.manager, repository.queryRunner);
  }
}

```

## `users.module.ts` を修正する

TypeORM の User エンティティを追加したので、`users.module.ts` にも User を定義して、 TypeORM に User を使うことを教えてあげます。

```ts:src/users/users.module.ts
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';
import { UsersController } from './users.controller';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './entities/user.entity';
import { UsersRepository } from './users.repository';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  controllers: [UsersController],
  providers: [UsersService, UsersRepository],
  exports: [UsersService],
})
export class UsersModule {}
```

`forFeature()` で TypeORM にエンティティを登録します。

## DTO の修正

あとでユーザーを作成する機能を作るので、DTO を作成しておきます。

```shell
npm i class-validator class-transformer
```

```ts:src/users/dto/create-user.dto.ts
import { IsNotEmpty } from 'class-validator';

export class CreateUserDto {
  @IsNotEmpty({ message: '名前は必須です。' })
  username: string;

  @IsNotEmpty({ message: 'パスワードは必須です。' })
  password: string;
}
```

ここでは必要最低限のバリデーションだけ設定していますが、本番運用では [class-validator](https://github.com/typestack/class-validator) を参考に、しっかりバリデーションをかけてください。

## UsersService の修正

ユーザーを作成する機能と、ユーザー名でユーザーを見つける機能を作成します。

```ts:src/users/users.service.ts
import { Injectable } from '@nestjs/common';
import { UsersRepository } from './users.repository';

@Injectable()
export class UsersService {
  constructor(private readonly usersRepository: UsersRepository) {}
  async create(username: string, hashedPassword: string) {
    return await this.usersRepository.save({ username, hashedPassword });
  }

  async findUserByName(username: string) {
    return this.usersRepository.findOneBy({ username });
  }
}
```

## UsersController の修正

```ts:src/users/users.controller.ts
import { Controller, Get, Post, Body, UseGuards } from '@nestjs/common';
import { UsersService } from './users.service';
import { CreateUserDto } from './dto/create-user.dto';
import * as bcrypt from 'bcrypt';
import { JwtAuthGuard } from '../auth/jwt-auth.guard';

@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Post('/signup')
  async create(@Body() createUserDto: CreateUserDto) {
    const { username, password } = createUserDto;
    const saltOrRounds = 10;
    const hashedPassword = await bcrypt.hash(password, saltOrRounds);

    return await this.usersService.create(username, hashedPassword);
  }

  @UseGuards(JwtAuthGuard)
  @Get('/profile')
  profile() {
    return 'Profile was called';
  }
}
```

`create`というメソッドはユーザーを作成するもので、`profile`は認証機構を確認するためのテストメソッドです。

パスワードはハッシュ化してデータベースに保存します。

`await bcrypt.hash(password, saltOrRounds)` で何をしているのかは、以下の記事がわかりやすかったです。

[Bcrypt を用いてパスワードをハッシュ化する (Node.js)](https://blog.kkty.jp/entry/2019/04/13/082214)

## auth モジュールの作成

次に認証機構を作っていきます。

```shell
nest g res auth

# ? What transport layer do you use? REST API
# ? Would you like to generate CRUD entry points? Yes
```

自動で生成される `auth/entities`, `dto/create-auth.dto.ts`, `dto/update-auth.dto.ts`は不要なので消してしまって OK です。

## AuthService でユーザー名とパスワードで認証をかけるメソッドを作る

`src/auth/auth.service.ts`を修正します。

```ts:src/auth/auth.service.ts
import { Injectable, NotAcceptableException } from "@nestjs/common";
import { UsersService } from "../users/users.service";
import { JwtService } from "@nestjs/jwt";
import * as bcrypt from "bcrypt";
import { User } from "../users/entities/user.entity";
import { LoginDto } from "./dto/login.dto";

@Injectable()
export class AuthService {
  constructor(
    private readonly usersService: UsersService,
    private jwtService: JwtService
  ) {}
  async validateUser(
    username: string,
    password: string
  ): Promise<Omit<User, "hashedPassword"> | undefined> {
    const user = await this.usersService.findUserByName(username);
    if (!user) {
      throw new NotAcceptableException("ユーザーが存在しません。");
    }
    const passwordValid = await bcrypt.compare(password, user.hashedPassword);

    if (passwordValid) {
      const { hashedPassword, ...result } = user;
      return result;
    }

    return undefined;
  }

  async login(loginDto: LoginDto) {
    const payload = { username: loginDto.username, sub: loginDto.id };
    return {
      access_token: this.jwtService.sign(payload),
    };
  }
}
```

ここで定義した `validateUser` が `LocalStrategy` から呼ばれます。

`LocalStrategy` は `AuthController` が `login` を呼ぶときの「ガード」として使われます。

ガードとは、リクエストをメソッドに通すか通さないかを決める役割を果たします。

`LocalStrategy` はユーザー名とパスワードが正しくないと、 Controller のメソッドを実行しませんよ、というコントローラのボディガードのような役割を果たします。

## LocalStrategy の作成

```ts:src/auth/local.strategy.ts
import { PassportStrategy } from '@nestjs/passport';
import { Strategy } from 'passport-local';
import { AuthService } from './auth.service';
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { User } from '../users/entities/user.entity';

@Injectable()
export class LocalStrategy extends PassportStrategy(Strategy) {
  constructor(private authService: AuthService) {
    super();
  }

  async validate(
    username: string,
    password: string,
  ): Promise<Omit<User, 'hashedPassword'>> {
    const user = await this.authService.validateUser(username, password);
    if (!user) {
      throw new UnauthorizedException();
    }
    return user;
  }
}
```

上の `LocalStrategy` をクラスとして呼び出せるように `LocalAuthGuard` を作成します。

```ts:src/auth/local-auth.guard.ts
import { AuthGuard } from "@nestjs/passport";
import { Injectable } from "@nestjs/common";

@Injectable()
export class LocalAuthGuard extends AuthGuard("local") {}
```

コントローラの `login` メソッドが呼ばれるときに渡される DTO を作成します。

```ts:src/auth/dto/login.dto.ts
export class LoginDto {
  id: number;
  username: string;
}
```

次に `login` を受け付けるためのコントローラを作成します。

```ts:src/auth/auth.controller.ts
import { AuthService } from './auth.service';
import { Controller, Post, UseGuards, Body } from '@nestjs/common';
import { LocalAuthGuard } from './local-auth.guard';
import { LoginDto } from './dto/login.dto';

@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @UseGuards(LocalAuthGuard)
  @Post('/login')
  async login(@Body() loginDto: LoginDto) {
    return this.authService.login(loginDto);
  }
}
```

## username と password で validate

少しハマったのですが、 `LocalStrategy` の `validate()` メソッドには `(username: string, password: string)` のように、 `username` と `password` が渡されなければいけません。

ここを `name: string` みたいにしてしまうと、メソッド自体が呼び出されず、 Unauthorized エラーが返されてしまいます。

[NestJS and passport-local : AuthGuard('local') validate() never get called](https://stackoverflow.com/questions/70896560/nestjs-and-passport-local-authguardlocal-validate-never-get-called)

以下のように `@UseGuards(LocalAuthGuard)` のデコレータを付与して `/login` を呼び出したときは、

```ts
@UseGuards(LocalAuthGuard)
@Post('/login')
async login(@Body() loginDto: LoginDto) {
    Logger.debug(`AuthController#login: ${loginDto}`);
    return this.authService.login(loginDto);
}
```

- `local.strategy` の `validate`
- `AuthService` の `validateUser`
- `AuthController` の `login` メソッド
- `AuthService` の `login` メソッド

の順番で呼ばれていました。

Controller の メソッドが呼ばれる"前"に、 Guard で `username` と `password` を検証してくれていますね。

## JWT 認証の実装

秘密鍵の情報を読み込むための定数を定義します。

```ts:src/auth/constants.ts
// 本番運用では process.env.SECRET_KEY からは直接読み込まず、configuration を使う。秘密鍵は十分な長さにする。
export const jwtConstants = {
  secret: process.env.SECRET_KEY ?? 'secretKey',
};
```

`JwtStrategy` を作成します。

```ts:src/auth/jwt.strategy.ts
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { Injectable } from '@nestjs/common';
import { jwtConstants } from './constants';

interface JWTPayload {
  userId: string;
  username: string;
}

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: jwtConstants.secret,
    });
  }

  async validate(payload: JWTPayload) {
    return { userId: payload.userId, username: payload.username };
  }
}
```

`JwtAuthGuard` を作成します。

```ts:src/auth/jwt-auth.guard.ts
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}
```

ここまでで必要なコードは揃ったので、動作確認してきます。

## ユーザーを登録する

まずはユーザーを作成します。

```shell
curl -X POST http://localhost/users/signup -d '{"username": "testuser", "password": "testpass"}' -H "Content-Type: application/json"

# {"username":"testuser","hashedPassword":"$2b$10$EgbwLFaZs0V.0bfXbci5meVGSZaCXYvGWaLtlbCFxm7b7A2/bNkFm","id":2,"createdAt":"2022-11-06T23:38:23.080Z","updatedAt":"2022-11-06T23:38:23.080Z"
```

上のコマンドで、データベースの users テーブルに testuser が作られます。パスワードはハッシュ化されて保存されています。

## ログインしてアクセストークンを取得する

```shell
curl -X POST http://localhost/auth/login -d '{"username": "testuser", "password": "testpass"}' -H "Content-Type: application/json"

# {"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6InRlc3R1c2VyIiwiaWF0IjoxNjY3NzQ1NTg1LCJleHAiOjE2Njc3NDY3ODV9.6GTLE66v6L6uagU7vt9UsS_gs3f6G5z47maAkRF9cok"}
```

## JWT 認証を検証する

上で取得したトークンをつけずに、 `@UseGuards(JwtAuthGuard)` でガードされたルートを呼び出してみます。

具体的には、 `users/users.controller.ts` の以下の部分です。

```ts
  @UseGuards(JwtAuthGuard)
  @Get('/profile')
  profile() {
    return 'Profile was called';
  }
```

```shell
curl http://localhost/users/profile

# {"statusCode":401,"message":"Unauthorized"}%
```

`Unauthorized` が返ってきますね。

次に Bearer Token をセットしてリクエストを投げてみます。

```shell
curl http://localhost/users/profile -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6InRlc3R1c2VyIiwiaWF0IjoxNjY3NzQ1NTg1LCJleHAiOjE2Njc3NDY3ODV9.6GTLE66v6L6uagU7vt9UsS_gs3f6G5z47maAkRF9cok"

# Profile was called
```

レスポンスが返ってきます。

では、適当な Bearer Token をセットするとどうなるでしょうか？

```shell
curl http://localhost/users/profile -H "Authorization: Bearer hogehogehoge"

# {"statusCode":401,"message":"Unauthorized"}
```

やはり `Unauthorized` が返ってきます。

JWT 認証のガードも実装できました。

## 参考

- [How to implement JWT authentication in NestJS](https://blog.logrocket.com/how-to-implement-jwt-authentication-nestjs/)
- [NestJS で JWT を使った認証を実装する](https://zenn.dev/uttk/articles/9095a28be1bf5d)
- [JWT 認証の仕組み・脆弱性について軽くまとめ](https://deecode.net/?p=1544)
- [JWT を用いた認証方法（JWT 発行と検証方法）](https://qiita.com/katamotokosuke/items/0c97f42ecb8207767db2)

## NestJS シリーズ

1. [NestJS+Nginx+MySQL の開発環境を Docker 化する](https://zenn.dev/fjsh/articles/nestjs-with-docker)
2. [NestJS+TypeORM 0.3 系で MySQL に接続してエラーなく起動する](https://zenn.dev/fjsh/articles/nestjs-typeorm-connect-mysql)
3. [NestJS+TypeORM 0.3 系でマイグレーションを実行する](https://zenn.dev/fjsh/articles/nestjs-typeorm-migration)
4. [NestJS で JWT 認証を実装する](https://zenn.dev/fjsh/articles/nestjs-auth-user-password)
