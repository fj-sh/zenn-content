---
title: "NestJS+TypeORM 0.3 でユニットテストしてみる"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["NestJS"]
published: true
---

[NestJS+TypeORM 0.3 で CRUD 操作を行う](https://zenn.dev/fjsh/articles/nestjs-with-typeorm) の続きです。

## サービスのテストを作ってみる（モックから値を返すパターン）

```ts
import { Test, TestingModule } from "@nestjs/testing";
import { TasksService } from "./tasks.service";
import { TaskRepository } from "./task.repository";

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

describe("TasksService", () => {
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
});
```

### TODO:

- その他の CRUD 操作のテストを追加する
- コントローラーとリポジトリのテストを追加する
- インメモリ DB を使うパターン（SQLite を使ってテストするパターン）のテストを追加する

## 参考

- [Unit testing NestJS applications with Jest](https://blog.logrocket.com/unit-testing-nestjs-applications-with-jest/)
- [Testing NestJS services with integration tests](https://wanago.io/2020/07/13/api-nestjs-testing-services-controllers-integration-tests/)
- [Unit testing a NestJS App — In shortest steps.](https://nishabe.medium.com/unit-testing-a-nestjs-app-in-shortest-steps-bbe83da6408)
  - これがうまく動いた記事
- [Using test database when e2e-testing NestJS](https://www.anycodings.com/1questions/1326348/using-test-database-when-e2e-testing-nestjs)
  - インメモリ DB を使うパターン
