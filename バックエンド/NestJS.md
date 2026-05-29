# NestJS メモ

Node.js のサーバーサイドフレームワーク。Angular ライクな DI + デコレータ + モジュール構成で、構造化された API サーバーを書く。Express(または Fastify)の上に乗る。

---

## 1. NestJS の全体像

| 構成要素                | 役割                                                                       | デコレータ / インターフェース        |
| ----------------------- | -------------------------------------------------------------------------- | ------------------------------------ |
| **Module**              | アプリを機能単位に分割する箱。依存(imports)・コントローラ・プロバイダをまとめる | `@Module({...})`                     |
| **Controller**          | HTTP ルーティング担当。リクエストを受けてレスポンスを返す                   | `@Controller()` / `@Get()` 等        |
| **Provider**(Service 等) | ビジネスロジック・DB アクセス等。DI で注入される                            | `@Injectable()`                      |
| **Middleware**          | ルートハンドラの前に走る処理(Express の middleware そのもの)              | `NestMiddleware` 実装                |
| **Guard**               | 認証・認可。ハンドラ実行の可否を決める                                     | `CanActivate` 実装 + `@UseGuards()`  |
| **Interceptor**         | ハンドラ前後をラップ(ロギング・レスポンス変換等)。`Observable` を返す      | `NestInterceptor` 実装               |
| **Pipe**                | 引数のバリデーション・変換                                                  | `PipeTransform` 実装                 |

### リクエスト処理の順序

```
HTTP リクエスト
  → Middleware
  → Guard           ← 認証/認可。false や throw でここで止まる
  → Interceptor(前)
  → Pipe            ← 引数バリデーション・変換
  → Route Handler   ← コントローラのメソッド
  → Interceptor(後)
  → レスポンス
```

`AsyncLocalStorage` でコンテキストを引き回したい場合、**Middleware の `next()` を `runWithSomething(..., () => next())` で包む**のが確実(Interceptor は Observable の購読タイミングがズレるので注意)。

---

## 2. Module(`@Module`)

```ts
@Module({
  imports: [ConfigModule, PrismaModule],   // 他 Module を取り込む(その exports が使える)
  controllers: [UsersController],           // この Module が持つコントローラ
  providers: [UsersService],                // この Module が生成する Provider
  exports: [UsersService],                  // 他 Module に公開する Provider
})
export class UsersModule {}
```

- `@Global()` を付けると、その Module の exports が **全 Module で import 不要で注入可能**になる(Prisma / Config のように「どこでも使う」ものに適用)
- ルート Module(`AppModule`)が全ての起点。`NestFactory.create(AppModule)` でここから DI コンテナを構築
- Middleware の登録は `AppModule implements NestModule` の `configure(consumer)` で行う:
  ```ts
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(TenantMiddleware).forRoutes('*');   // 全ルート or .forRoutes('users') 等
  }
  ```

---

## 3. DI と Provider / Service

```ts
@Injectable()
export class UsersService {
  constructor(private readonly prisma: PrismaService) {}   // コンストラクタ注入
  findAll() { return this.prisma.user.findMany(); }
}

@Controller('users')
export class UsersController {
  constructor(private readonly users: UsersService) {}     // ここでも注入
  @Get() getAll() { return this.users.findAll(); }
}
```

- `@Injectable()` を付けたクラスは DI コンテナが管理(デフォルトはシングルトン)
- コンストラクタの引数の **型** を見て NestJS が自動で対応するインスタンスを渡す(`emitDecoratorMetadata` が必要、§10)
- 注入されるには「同じ Module の providers にある」or「import した Module の exports にある」or「`@Global()` Module の exports」のいずれか

---

## 4. Lifecycle hooks

| インターフェース           | メソッド                          | 呼ばれるタイミング                       |
| -------------------------- | --------------------------------- | ---------------------------------------- |
| `OnModuleInit`             | `onModuleInit()`                  | Module の依存解決後、リクエスト受付前    |
| `OnApplicationBootstrap`   | `onApplicationBootstrap()`        | 全 Module 初期化完了後                   |
| `OnModuleDestroy`          | `onModuleDestroy()`               | アプリ終了処理の開始時                   |
| `OnApplicationShutdown`    | `onApplicationShutdown(signal?)`  | プロセス終了シグナル受信時               |

典型例: DB 接続の管理

```ts
@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  async onModuleInit()    { await this.$connect(); }
  async onModuleDestroy() { await this.$disconnect(); }
}
```

---

## 5. Controller とルーティング

```ts
@Controller('workspaces')                    // パス prefix → /workspaces
@UseGuards(ClerkAuthGuard)                    // このコントローラ全体に Guard
export class WorkspacesController {
  @Get(':slug')                               // GET /workspaces/:slug
  async getWorkspace(
    @Param('slug') slug: string,              // URL パスパラメータ
    @Query('q') q?: string,                   // クエリ ?q=...
    @Body() body?: SomeDto,                   // リクエストボディ
    @Headers('authorization') auth?: string,  // ヘッダー
    @CurrentUser() user: AuthUser,            // 自作パラメータデコレータ(§8)
  ) {
    if (!found) throw new NotFoundException();  // → 自動で 404 JSON
    return { ... };                             // → 自動で JSON シリアライズ
  }
}
```

- 例外: `HttpException` のサブクラス(`NotFoundException` / `UnauthorizedException` / `BadRequestException` / `ForbiddenException` 等)を throw すると、NestJS が `{ message, error, statusCode }` の JSON を該当ステータスで返す
- `@HttpCode(204)` / `@Header('Cache-Control', '...')` でレスポンスを調整

---

## 6. Middleware

```ts
@Injectable()
export class TenantMiddleware implements NestMiddleware {
  constructor(private readonly prisma: PrismaService) {}     // DI も効く
  async use(req: Request, res: Response, next: NextFunction) {
    // 前処理...
    next();                                                   // ← 次のハンドラへ進む
  }
}
// AppModule の configure() で apply
```

- Express の middleware と同じ。`next()` を呼ぶと**その呼び出しスタック内で**下流のハンドラが実行される
- そのため `runWithContext(value, () => next())` のように包むと、`AsyncLocalStorage` のコンテキストがハンドラまで伝搬する
- `next()` を呼ばずに `res.send(...)` するとそこで打ち切り
- middleware から例外を throw すると NestJS の例外フィルタが拾って HTTP エラーにする

---

## 7. Guard

```ts
@Injectable()
export class ClerkAuthGuard implements CanActivate {
  constructor(private readonly config: ConfigService) {}
  async canActivate(ctx: ExecutionContext): Promise<boolean> {
    const req = ctx.switchToHttp().getRequest<Request>();    // HTTP の Request を取り出す
    const token = req.headers.authorization?.replace('Bearer ', '');
    if (!token) throw new UnauthorizedException();            // ← throw で 401
    const payload = await verifyToken(token, { secretKey: ... });
    req.user = { clerkUserId: payload.sub };                  // 後続に渡す情報をリクエストに載せる
    return true;                                              // true でハンドラへ、false だと 403
  }
}
```

- `@UseGuards(ClerkAuthGuard)` をコントローラ or ハンドラ or グローバル(`APP_GUARD` provider)に付ける
- `ExecutionContext` は HTTP / WebSocket / RPC 共通。`switchToHttp()` で HTTP に絞る
- `canActivate` が `false` → `403 Forbidden`。明示的に `401` にしたいなら `throw new UnauthorizedException()`
- 認証結果(ユーザー情報等)は `req.user` 等に載せ、`@CurrentUser()` デコレータで取り出す(§8)

---

## 8. パラメータデコレータの自作(`createParamDecorator`)

`@Param` / `@Body` / `@Query` の仲間を自作できる。「ハンドラ引数 = リクエストから抽出した値」の対応を定義する。

```ts
export const CurrentUser = createParamDecorator(
  (data: unknown, ctx: ExecutionContext): AuthUser => {
    const req = ctx.switchToHttp().getRequest<Request>();
    if (!req.user) throw new Error('Guard を付け忘れている');
    return req.user;                                          // ← この戻り値が引数の値になる
  },
);
// 使う側
@Get() handler(@CurrentUser() user: AuthUser) { ... }
```

- `createParamDecorator(fn)` の `fn(data, ctx)` の **戻り値が引数の中身**になる
- `data` は `@CurrentUser('clerkUserId')` のように渡せる引数(不要なら `_data`)
- メリット: ハンドラが Express の `Request` に依存しない / null チェックをデコレータ内に集約 / 全ハンドラで書き方が統一

---

## 9. 設定管理(`@nestjs/config`)

NestJS は `.env` を自動で読まない。`@nestjs/config` を入れて明示的に読む。

```ts
@Module({
  imports: [ConfigModule.forRoot({ isGlobal: true, envFilePath: '.env.local' })],
})
export class AppModule {}

// 使う側
@Injectable()
export class FooService {
  constructor(private config: ConfigService) {}
  get key() { return this.config.getOrThrow<string>('SOME_KEY'); }  // 無ければ throw
  get opt() { return this.config.get<string>('OPTIONAL'); }          // 無ければ undefined
}
```

- `isGlobal: true` で `ConfigService` を全 Module で注入可能に
- `envFilePath` は cwd 相対(`nest start` 実行ディレクトリ基準)。配列で複数指定可
- `.env` の値は `process.env` にも展開される(Prisma 等が `process.env.DATABASE_URL` で読める)
- バリデーション(`Joi` / `class-validator`)を `validationSchema` で噛ませられる

---

## 10. プロジェクトセットアップと TS 要件

| 項目              | 内容                                                                                          |
| ----------------- | --------------------------------------------------------------------------------------------- |
| CLI               | `@nestjs/cli`。`nest new` / `nest g controller foo` / `nest g service foo` で雛形生成         |
| dev               | `nest start --watch`(ファイル変更で再コンパイル&再起動)                                      |
| build             | `nest build`(内部は tsc。`dist/` に出力)                                                     |
| 設定              | `nest-cli.json`(`sourceRoot`, `compilerOptions.deleteOutDir` 等)                            |
| ランタイム        | デフォルト CommonJS(decorator + DI が最も安定)。ESM も可だが foot-gun が増える              |
| 必須 polyfill     | `import 'reflect-metadata'` を `main.ts` の最上段に。decorator のメタデータ反射に必要          |
| tsconfig 必須設定 | `experimentalDecorators: true`(`@Module` 等の旧式 decorator 構文)、`emitDecoratorMetadata: true`(コンストラクタ引数の型を実行時に保持 → DI が読む) |

```ts
// main.ts
import 'reflect-metadata';
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(process.env.PORT ?? 4000);
}
void bootstrap();
```

---

## 11. Prisma との統合

`PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy` で、DI 経由で型付き Client を提供 + lifecycle で接続管理(詳細は `データベース/Prisma.md` の §8 参照)。`$extends`(Client Extension)を適用する場合は「コンストラクタが拡張クライアントを返すクラス式」を `extends` 句に渡すパターンを使う。
