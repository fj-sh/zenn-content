---
title: "NestJSでサービスとコントローラーのユニットテストを作る"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["NestJS"]
published: true
---

[NestJS+TypeORM 0.3 で CRUD 操作を行う](https://zenn.dev/fjsh/articles/nestjs-with-typeorm) の続きです。
前の記事では、NestJS で超簡単な TODO アプリの CRUD 操作を実装してみました。

この記事では、作成したアプリケーションにテストを追加していきます。

- サービスのテスト
- コントローラーのテスト

の順で作成します。

## サービスのテストを作ってみる

```ts
import { Test, TestingModule } from "@nestjs/testing";
import { TasksService } from "./tasks.service";
import { TaskRepository } from "./task.repository";
import { UpdateTaskDto } from "./dto/update-task.dto";
import { DeleteResult, UpdateResult } from "typeorm";

describe("TasksServiceの正常系のテスト", () => {
  const mockFind = () => {
    return [
      {
        id: 1,
        name: "勉強",
      },
      {
        id: 2,
        name: "洗濯",
      },
    ];
  };

  const mockFindOneBy = (id: number) => {
    return {
      id: 1,
      name: "勉強",
    };
  };

  const mockUpdate = (id: number, name: string): UpdateResult => {
    return {
      raw: 1,
      affected: 2,
      generatedMaps: [],
    };
  };

  const mockDelete = (): DeleteResult => {
    return {
      raw: 1,
      affected: 2,
    };
  };

  let tasksService: TasksService;
  let taskRepository: TaskRepository;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        TasksService,
        {
          provide: TaskRepository,
          useFactory: () => ({
            find: jest.fn(mockFind),
            findOneBy: jest.fn(mockFindOneBy),
            update: jest.fn(mockUpdate),
            delete: jest.fn(mockDelete),
          }),
        },
      ],
    }).compile();

    tasksService = module.get<TasksService>(TasksService);
    taskRepository = module.get<TaskRepository>(TaskRepository);
    jest.clearAllMocks();
  });

  it("should be defined", () => {
    expect(tasksService).toBeDefined();
    expect(taskRepository).toBeDefined();
  });

  describe("findAll", () => {
    it("タスクの配列を返すこと", async () => {
      await tasksService.findAll();
      const expected = [
        { id: 1, name: "勉強" },
        { id: 2, name: "洗濯" },
      ];
      const actual = await tasksService.findAll();
      expect(actual).toEqual(expected);
    });
  });

  describe("findOne", () => {
    it("タスクの配列を返すこと", async () => {
      const id = 1;
      const expected = { id: 1, name: "勉強" };
      const actual = await tasksService.findOne(id);
      expect(actual).toEqual(expected);
    });
  });

  describe("update", () => {
    it("更新されたタスクを返すこと", async () => {
      const updateTargetId = 1;
      const updateDto: UpdateTaskDto = {
        name: "更新された勉強",
      };

      // update 内では findOneBy が呼び出されている。 findOneBy のモックからは { id: 1, name: '勉強' } が返される。
      const expected = { id: 1, name: "勉強" };

      const actual = await tasksService.update(updateTargetId, updateDto);
      expect(actual).toEqual(expected);
    });
  });

  describe("remove", () => {
    it("削除された結果を返すこと", async () => {
      const targetId = 1;
      const expected = {
        raw: 1,
        affected: 2,
      };
      const actual = await tasksService.remove(targetId);
      expect(actual).toEqual(expected);
    });
  });
});
```

### サービスのテストを作る上で参考にした記事

- [Unit testing NestJS applications with Jest](https://blog.logrocket.com/unit-testing-nestjs-applications-with-jest/)
- [Testing NestJS services with integration tests](https://wanago.io/2020/07/13/api-nestjs-testing-services-controllers-integration-tests/)
- [Unit testing a NestJS App — In shortest steps.](https://nishabe.medium.com/unit-testing-a-nestjs-app-in-shortest-steps-bbe83da6408)
  - これがうまく動いた記事
- [Using test database when e2e-testing NestJS](https://www.anycodings.com/1questions/1326348/using-test-database-when-e2e-testing-nestjs)
  - インメモリ DB を使うパターン

## コントローラーのテストを作ってみる

```ts:src/tasks/tasks.controller.spec.ts
import { Test, TestingModule } from "@nestjs/testing";
import { TasksController } from "./tasks.controller";
import { TasksService } from "./tasks.service";
import { Task } from "./entities/task.entity";
import { CreateTaskDto } from "./dto/create-task.dto";
import { UpdateTaskDto } from "./dto/update-task.dto";
import { DeleteResult } from "typeorm";

describe("TasksController - 正常系", () => {
  let controller: TasksController;
  let spyService: TasksService;

  const TasksServiceProvider = {
    provide: TasksService,
    useFactory: () => ({
      create: jest.fn((createTaskDto: CreateTaskDto): Promise<Task> => {
        return Promise.resolve({ id: 1, name: "勉強" });
      }),
      findAll: jest.fn((): Promise<Task[]> => {
        return Promise.resolve([
          { id: 1, name: "勉強" },
          { id: 2, name: "洗濯" },
        ]);
      }),
      update: jest.fn((): Promise<Task> => {
        return Promise.resolve({
          id: 1,
          name: "勉強",
        });
      }),
      remove: jest.fn((): Promise<DeleteResult> => {
        return Promise.resolve({
          raw: 1,
        });
      }),
    }),
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [TasksController],
      providers: [TasksServiceProvider],
    }).compile();

    controller = module.get<TasksController>(TasksController);
    spyService = module.get<TasksService>(TasksService);
  });

  it("should be defined", () => {
    expect(controller).toBeDefined();
  });

  describe("create メソッドが呼ばれたとき", () => {
    const createTask: CreateTaskDto = {
      name: "勉強",
    };
    it("TasksService の create メソッドが呼ばれること", () => {
      controller.create(createTask);
      expect(spyService.create).toHaveBeenCalled();
    });
    it("作成されたタスクが返されること", async () => {
      const expected: Task = {
        id: 1,
        name: "勉強",
      };

      const actual = await controller.create(createTask);
      expect(actual).toEqual(expected);
    });
  });

  describe("findAll メソッドが呼ばれたとき", () => {
    it("TasksService の findAll メソッドが呼ばれること", () => {
      controller.findAll();
      expect(spyService.findAll).toHaveBeenCalled();
    });
    it("タスクのリストが返されること", async () => {
      const expected: Task[] = [
        {
          id: 1,
          name: "勉強",
        },
        {
          id: 2,
          name: "洗濯",
        },
      ];

      const actual = await controller.findAll();
      expect(actual).toEqual(expected);
    });
  });

  describe("update メソッドが呼ばれたとき", () => {
    const updateId = "1";
    const updateTask: UpdateTaskDto = {
      name: "勉強",
    };
    it("TasksService の update メソッドが呼ばれること", () => {
      controller.update(updateId, updateTask);
      expect(spyService.update).toHaveBeenCalled();
    });
    it("更新されたタスクが返されること", async () => {
      const expected: Task = {
        id: 1,
        name: "勉強",
      };

      const actual = await controller.update(updateId, updateTask);
      expect(actual).toEqual(expected);
    });
  });

  describe("remove メソッドが呼ばれたとき", () => {
    const removeId = "1";
    it("TasksService の remove メソッドが呼ばれること", () => {
      controller.remove(removeId);
      expect(spyService.remove).toHaveBeenCalled();
    });
  });
});
```

### コントローラーのテストを作る上で参考にした記事

- [Unit testing a NestJS App — In shortest steps.](https://nishabe.medium.com/unit-testing-a-nestjs-app-in-shortest-steps-bbe83da6408)

以下、基礎知識的な内容をまとめておきます。

## providers とは？

> プロバイダは Nest の基本概念です。基本的な Nest クラスの多くはサービス、リポジトリ、ファクトリー、ヘルパーなどのプロバイダとして扱われます。プロバイダのアイデアは依存関係を注入できることです。これはオブジェクトが互いにさまざまな関係を作成できることを意味し、オブジェクトのインスタンスを「繋げる」機能は Nest ランタイムシステムに委任できます。プロバイダは単に@Injectable()デコレータが付けられたクラスです。
> [Providers | NestJS 【翻訳】](https://qiita.com/mana-vv/items/745ff502837cdba55011) > [Standard providers](https://docs.nestjs.com/fundamentals/custom-providers#standard-providers)

NestJS がインスパイアされているという Angular の説明は以下のとおりです。

> プロバイダーは依存性がある値を取り出す方法を Dependency Injection システムへ指示します。ほとんどの場合、これらの依存性はあなたが作成して提供するサービスによるものです。
> [モジュールに依存オブジェクトを提供する](https://angular.jp/guide/providers)

Provider とは...

- 形式的には、@Injectable() デコレータを適用したクラスのこと
- 依存対象 (Dependency) として注入 (inject) される
- Controller から、複雑なタスクを依頼される

[Providers](https://zenn.dev/morinokami/articles/nestjs-overview)

## useFactory とは？

以下は公式の定義です。

useFactory 構文により、プロバイダを動的に作成することができます。

実際のプロバイダは、ファクトリー関数から返される値によって提供されます。

ファクトリー関数は、必要なだけ単純でも複雑でもかまいません。単純なファクトリーでは、他のプロバイダに依存することはありません。

より複雑なファクトリーは、その結果を計算するために必要な他のプロバイダを自ら注入することができます。

後者の場合、ファクトリープロバイダの構文には、関連するメカニズムが 2 つあります。

1. ファクトリー関数は、（オプションの）引数を受け取ることができます。
2. オプションの inject プロパティは、Nest が解決するプロバイダの配列を受け入れ、インスタンス化処理中にファクトリ関数への引数として渡します。また、これらのプロバイダはオプションとしてマークすることができます。この 2 つのリストには相関関係があるはずです。Nest は、inject リストのインスタンスを、同じ順番でファクトリー関数の引数として渡します。以下の例では、このことを実証しています。

```ts
const connectionProvider = {
  provide: "CONNECTION",
  useFactory: (optionsProvider: OptionsProvider, optionalProvider?: string) => {
    const options = optionsProvider.get();
    return new DatabaseConnection(options);
  },
  inject: [OptionsProvider, { token: "SomeOptionalProvider", optional: true }],
  //       \_____________/            \__________________/
  //        This provider              The provider with this
  //        is mandatory.              token can resolve to `undefined`.
};

@Module({
  providers: [
    connectionProvider,
    OptionsProvider,
    // { provide: 'SomeOptionalProvider', useValue: 'anything' },
  ],
})
export class AppModule {}
```

[Factory providers:](https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory)
