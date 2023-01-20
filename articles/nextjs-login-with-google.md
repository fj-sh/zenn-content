---
title: "Next.js + Firebase で「Googleでログイン」して、NestJSでトークンを検証・認可を行う"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nextjs", "nestjs"]
published: true
---

## この記事で紹介する内容

- Next.js から Firebase 経由で Google のログイン画面を開き、ユーザー情報を取得する
- Google でログインすると、idToken を取得できる。その idToken を使って、NestJS の GraphQL API にリクエストを投げる
- NestJS 側で JWT トークンの検証を行い、正しい idToken を持っているリクエストにだけレスポンスを返す

## Firebase Authentication の Google 認証を有効化する

Firebase の認証設定は [NestJS+Firebase で「メールアドレス＋パスワード」でユーザー登録する](https://zenn.dev/fjsh/articles/nestjs-firebase-auth) に画面スクショ付きで記事を書いたので、ここではサクサクと書きます。

Firebase のコンソールを開きます。

左ペインの 構築 > Authentication を開きます。

真ん中のタブから 「Sign-in method」 を選びます。

新しいプロバイダを追加 > Google を選択すれば OK です。

## Next.js で「Login with Google」を実装する

以下のようなイメージのログインモーダルを実装します。
![](https://storage.googleapis.com/zenn-user-upload/b416201a59cf-20230120.jpg)

CSS や状態管理は今回の本筋ではないので、流します。

```ts:src/components/organisms/login-modal.tsx
import { firebase } from '../../lib/firebase';
import { useEffect } from 'react';
import loginModalStyle from '@/styles/components/organisms/login-modal.module.scss';
import { idTokenAtom } from '@/stores/atoms';
import { useAtom } from 'jotai';
import { RESET } from 'jotai/utils';
import { createUser } from '@/features/users/api/create-user';
import { LOGIN_MODAL_MESSAGE } from '@/lib/constants';

type Props = {
  onCloseButtonClick: () => void;
};

const LoginModal = ({ onCloseButtonClick }: Props) => {
  const [_idToken, setIdToken] = useAtom(idTokenAtom);

  const onClickSignIn = async () => {
    const provider = new firebase.auth.GoogleAuthProvider().setCustomParameters({
      prompt: 'select_account',
    });
    try {
      await firebase.auth().signInWithPopup(provider);
      const currentUser = await firebase.auth().currentUser;
      if (currentUser) {
        await createUser({
          uid: currentUser.uid,
          displayName: currentUser.displayName ?? '',
          email: currentUser.email ?? '',
          emailVerified: currentUser.emailVerified,
          providerId: currentUser.providerId ?? '',
          isAnonymous: currentUser.isAnonymous,
        });
      }
    } catch (error) {
      console.error(error);
    }
  };

  useEffect(() => {
    // 注意：ログアウト機能は今回のサンプルでは実装していません。
    const unsubscribe = firebase.auth().onAuthStateChanged(async (user) => {
      if (user) {
        const token = await user.getIdToken();
        setIdToken(token);
      } else {
        setIdToken(RESET);
      }
    });
  });

  return (
    <div className={loginModalStyle.popup}>
      <div className={loginModalStyle.modal}>
        <button
          className={loginModalStyle.close}
          aria-label={'モーダルを閉じる'}
          onClick={onCloseButtonClick}
        >
          <svg x='0px' y='0px' viewBox='0 0 27 27' xmlSpace='preserve' width='15' height='15'>
            <path
              fill='currentColor'
              className='st0'
              d='M16.3,13.5l9.1-9.1c0.3-0.3,0.3-0.9,0-1.2l-1.6-1.6c-0.3-0.3-0.9-0.3-1.2,0l-9.1,9.1L4.4,1.6 C4,1.2,3.5,1.2,3.2,1.6L1.6,3.2C1.2,3.5,1.2,4,1.6,4.4l9.1,9.1l-9.1,9.1c-0.3,0.3-0.3,0.9,0,1.2l1.6,1.6c0.3,0.3,0.9,0.3,1.2,0 l9.1-9.1l9.1,9.1c0.3,0.3,0.9,0.3,1.2,0l1.6-1.6c0.3-0.3,0.3-0.9,0-1.2L16.3,13.5z'
            ></path>
          </svg>
        </button>
        <div className={loginModalStyle.description}>{LOGIN_MODAL_MESSAGE}</div>
        <div className={loginModalStyle.button}>
          <button className={loginModalStyle.loginButton} onClick={onClickSignIn}>
            Login with Google
          </button>
        </div>
      </div>
    </div>
  );
};

export default LoginModal;
```

「Google でログイン」のポップアップを表示させているのは以下の部分です。

```ts
const onClickSignIn = async () => {
  const provider = new firebase.auth.GoogleAuthProvider().setCustomParameters({
    prompt: "select_account",
  });
  try {
    await firebase.auth().signInWithPopup(provider);
    const currentUser = await firebase.auth().currentUser;
    if (currentUser) {
      await createUser({
        uid: currentUser.uid,
        displayName: currentUser.displayName ?? "",
        email: currentUser.email ?? "",
        emailVerified: currentUser.emailVerified,
        providerId: currentUser.providerId ?? "",
        isAnonymous: currentUser.isAnonymous,
      });
    }
  } catch (error) {
    console.error(error);
  }
};
```

Google のログイン画面がポップアップで出てきて、ログインすると閉じます。

ログインすると `await firebase.auth().currentUser;` で Google のユーザー情報が取得できるようになります。

`currentUser` が取得できるようになると、そこから `idToken` も取得できます。

```ts
const idToken = await auth.currentUser?.getIdToken();
```

以下の部分では、ログインした後に NestJS にユーザー情報を登録しようとしています。

```ts
if (currentUser) {
  await createUser({
    uid: currentUser.uid,
    displayName: currentUser.displayName ?? "",
    email: currentUser.email ?? "",
    emailVerified: currentUser.emailVerified,
    providerId: currentUser.providerId ?? "",
    isAnonymous: currentUser.isAnonymous,
  });
}
```

### Firebase の初期設定はどこでやっている？

以下のようなファイルを作っています。
上のコンポーネントでも、このファイルで作った `firebase` を使っています。

```ts:src/lib/firebase.ts
import firebase from "firebase/compat/app";
import "firebase/compat/auth";

const firebaseConfig = {
  apiKey: process.env.NEXT_PUBLIC_FIREBASE_API_KEY,
  authDomain: process.env.NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN,
  projectId: process.env.NEXT_PUBLIC_FIREBASE_PROJECT_ID,
  storageBucket: process.env.NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET,
  messagingSenderId: process.env.NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID,
  appId: process.env.NEXT_PUBLIC_FIREBASE_APP_ID,
};

if (!firebase.apps.length) {
  firebase.initializeApp(firebaseConfig);
}
const auth = firebase.auth();

export { firebase, auth };
```

## idToken を Apollo Client に設定する

以下は mutation を投げている例です。

```ts:src/features/users/api/create-user.ts
import { auth } from '@/lib/firebase';
import { createApolloClient } from '@/lib/apis/apollo-client';
import { gql } from '@apollo/client';
import { User } from '@/features/users/types/user';

export const createUser = async (user: User) => {
  const idToken = await auth.currentUser?.getIdToken();
  const client = createApolloClient(idToken);
  try {
    const { data } = await client.mutate({
      variables: {
        createUserInput: {
          displayName: user.displayName,
          email: user.email,
          emailVerified: user.emailVerified,
          isAnonymous: user.isAnonymous,
          providerId: user.providerId,
          uid: user.uid,
        },
      },
      mutation: gql`
        mutation createUser($createUserInput: CreateUserInput!) {
          createUser(createUserInput: $createUserInput) {
            displayName
            email
            emailVerified
            isAnonymous
            providerId
            uid
          }
        }
      `,
    });
    return data;
  } catch (error) {
    console.error(error);
  }
};
```

`ApolloClient` を作って返す関数は以下のとおりです。

```ts:src/lib/apis/apollo-client.ts
import { ApolloClient, createHttpLink, InMemoryCache } from "@apollo/client";
import { setContext } from "@apollo/client/link/context";

export function createApolloClient(idToken: string | undefined) {
  const httpLink = createHttpLink({
    uri: "http://localhost:3000/graphql",
    // fetchOptions: {
    //   mode: 'cors',
    // },
    // credentials: 'include',
  });

  const authLink = setContext((_, { headers }) => {
    return {
      headers: {
        ...headers,
        authorization: idToken ? `Bearer ${idToken}` : "",
      },
    };
  });

  return new ApolloClient({
    ssrMode: typeof window === "undefined",
    link: authLink.concat(httpLink),
    cache: new InMemoryCache(),
  });
}
```

引数で `idToken` が渡された場合は、ヘッダに `Bearer ${idToken}` を設定しています。

- [Apollo Docs Authentication](https://www.apollographql.com/docs/react/networking/authentication/)
- [The best way to pass authorization header in nextJs using Apollo client? ReferenceError: localStorage is not defined](https://stackoverflow.com/questions/63054130/the-best-way-to-pass-authorization-header-in-nextjs-using-apollo-client-referen)

## idToken 付きで Query を投げてみる

上のサンプルは mutation を投げる例ですが、 query を投げる例も貼っておきます。

mutation のときと同じく、 `const client = createApolloClient(idToken);` で `idToken` をヘッダにつけています。

```ts
import { gql } from "@apollo/client";
import { createApolloClient } from "@/lib/apis/apollo-client";
import { auth } from "@/lib/firebase";
import { User } from "@/features/users/types/user";

export const getUsers = async () => {
  const idToken = await auth.currentUser?.getIdToken();
  if (!idToken) {
    return [];
  }
  const client = createApolloClient(idToken);
  try {
    const { data } = await client.query({
      query: gql`
        query users {
          users {
            displayName
            email
            emailVerified
            isAnonymous
            photoURL
            providerId
            uid
          }
        }
      `,
    });

    if (data == null) {
      return [];
    }
    const users: User[] = data.users.map((item: any) => {
      return {
        displayName: item.displayName,
      };
    });

    return users;
  } catch (error) {
    console.error(error);
  }
};
```

## NestJS で JWT 認証を行ってみる

サーバー側では、`firebase-admin` をインポートして、 `admin.auth().verifyIdToken(idToken);` にクライアントから投げた `idToken` を渡します。
（この `idToken` には `Bearer ` は含まれません）

```ts:src/auth/auth.service.ts
import * as admin from 'firebase-admin';
import {
  HttpException,
  Injectable,
  UnauthorizedException,
} from '@nestjs/common';
import { DecodedIdToken } from 'firebase-admin/lib/auth';

@Injectable()
export class AuthService {
  async validateUser(idToken: string): Promise<DecodedIdToken> {
    if (!idToken) {
      throw new UnauthorizedException('認証が必要です。');
    }

    try {
      return await admin.auth().verifyIdToken(idToken);
    } catch (error) {
      throw new HttpException('Forbidden', error);
    }
  }
}
```

トークンが正しくない場合は例外が投げられ、正しいときはユーザー情報が返ってきます。

サーバー側の Firebase 　の初期設定は main.ts で行います。

```ts:src/main.ts
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";
import { ConfigService } from "@nestjs/config";
import { initializeApp } from "firebase/app";
import { FirebaseApp } from "@firebase/app";
import * as admin from "firebase-admin";
import { ServiceAccount } from "firebase-admin";

export let firebaseApp: FirebaseApp = undefined;

async function bootstrap() {
  const app = await NestFactory.create(AppModule, { cors: true });
  app.enableCors();
  const configService: ConfigService = app.get(ConfigService);
  const firebaseConfig = {
    apiKey: configService.get<string>("API_KEY"),
    authDomain: configService.get<string>("AUTH_DOMAIN"),
    projectId: configService.get<string>("PROJECT_ID"),
    storageBucket: configService.get<string>("STORAGE_BUCKET"),
    messagingSenderId: configService.get<string>("MESSAGING_SENDER_ID"),
    appId: configService.get<string>("APP_ID"),
  };
  firebaseApp = initializeApp(firebaseConfig);

  const adminConfig: ServiceAccount = {
    projectId: configService.get<string>("FIREBASE_PROJECT_ID"),
    privateKey: configService
      .get<string>("FIREBASE_PRIVATE_KEY")
      .replace(/\\n/g, "\n"),
    clientEmail: configService.get<string>("FIREBASE_CLIENT_EMAIL"),
  };
  admin.initializeApp({
    credential: admin.credential.cert(adminConfig),
  });
  await app.listen(3000);
}
bootstrap();
```

キーとなる値は `.env` に設定してください。

## Guard を作成する

以下のように Guard を作って...

```ts:src/auth/guard/auth.guard.ts
import {
  CanActivate,
  ExecutionContext,
  Injectable,
  UnauthorizedException,
} from "@nestjs/common";
import { AuthService } from "../auth.service";
import { GqlExecutionContext } from "@nestjs/graphql";

@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private readonly authService: AuthService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const ctx = GqlExecutionContext.create(context);
    const requestHeaders = ctx.getContext().req.headers;
    if (!requestHeaders) {
      throw new Error("ヘッダが正しく設定されていません。");
    }
    const idToken: string = requestHeaders.authorization.replace("Bearer ", "");
    try {
      const user = await this.authService.validateUser(idToken);
      ctx.getContext().req["user"] = user;
      return user !== undefined;
    } catch (error) {
      throw new UnauthorizedException("認証情報が正しくありません。");
    }
  }
}
```

Resolver でこんな感じで使います。

```ts
@UseGuards(AuthGuard)
@Query(() => [User], { name: 'users' })
findAll(@UserParam() user: DecodedIdToken) {
    console.log('findAllが呼ばれました:', user.email);
    return this.usersService.findAll();
}
```

Guard を使っている側のコードの全体像。

```ts:src/users/users.resolver.ts
import { Resolver, Query, Mutation, Args, Int } from '@nestjs/graphql';
import { UsersService } from './users.service';
import { User } from './entities/user.entity';
import { CreateUserInput } from './dto/create-user.input';
import { UpdateUserInput } from './dto/update-user.input';
import { UseGuards } from '@nestjs/common';
import { AuthGuard } from '../auth/guard/auth.guard';
import { UserParam } from './decorators/user.decorator';
import { DecodedIdToken } from 'firebase-admin/lib/auth';

@Resolver(() => User)
export class UsersResolver {
  constructor(private readonly usersService: UsersService) {}

  @Mutation(() => User)
  async createUser(@Args('createUserInput') createUserInput: CreateUserInput) {
    const user = await this.usersService.findUserByUid(createUserInput.uid);
    if (user) {
      return user;
    }
    return this.usersService.create(createUserInput);
  }

  @UseGuards(AuthGuard)
  @Query(() => [User], { name: 'users' })
  findAll(@UserParam() user: DecodedIdToken) {
    return this.usersService.findAll();
  }

  @Query(() => User, { name: 'user' })
  findOne(@Args('userId', { type: () => Int }) userId: number) {
    return this.usersService.findOne(userId);
  }

  @Mutation(() => User)
  updateUser(@Args('updateUserInput') updateUserInput: UpdateUserInput) {
    return this.usersService.update(updateUserInput.userId, updateUserInput);
  }

  @Mutation(() => User)
  removeUser(@Args('userId', { type: () => Int }) userId: number) {
    return this.usersService.remove(userId);
  }
}
```

`findAll(@UserParam() user: DecodedIdToken)` となっている部分では、認証時に取得したユーザー情報をカスタムデコレータから取得する、ということをやっています。

## Request Context にユーザー情報を設定する

`AuthGuard` の

```ts
const ctx = GqlExecutionContext.create(context);
// 略
ctx.getContext().req["user"] = user;
```

の部分でリクエストコンテキストにユーザー情報を入れています。

ここで入れたユーザー情報をカスタムデコレータ経由で渡せるようにします。

## カスタムデコレータでユーザー情報を渡す

```ts:src/users/decorators/user.decorator.ts
import { createParamDecorator, ExecutionContext } from "@nestjs/common";
import { GqlExecutionContext } from "@nestjs/graphql";
import { DecodedIdToken } from "firebase-admin/lib/auth";

export const UserParam = createParamDecorator(
  (data: unknown, context: ExecutionContext): DecodedIdToken => {
    const gqlContext = GqlExecutionContext.create(context);
    return gqlContext.getContext().req.user;
  }
);
```

このようなカスタムデコレータを作っておけば、以下の `@UserParam() user` のようにして、ユーザー情報を取り出すことができます。

```ts
@UseGuards(AuthGuard)
@Query(() => [User], { name: 'users' })
findAll(@UserParam() user: DecodedIdToken) {
    return this.usersService.findAll();
}
```

カスタムデコレータにユーザーを入れる部分は、 GraphQL での実装です。
普通の REST API に使いたいときは、公式のサンプルの通りに実装してください。

（GraphQL は `const gqlContext = GqlExecutionContext.create(context);` みたいなことをやらなければいけない）

- [Custom decorators](https://docs.nestjs.com/graphql/other-features#custom-decorators)
- [Custom route decorators](https://docs.nestjs.com/custom-decorators)

## 参考

- [next.js + vercel + firebase authentication で JWT の検証を行う + Graphql](https://mizchi.dev/202007271454-next-arch)
- [Nest.js で認証を導入する](https://note.com/ryoppei/n/n7409f0466403)
