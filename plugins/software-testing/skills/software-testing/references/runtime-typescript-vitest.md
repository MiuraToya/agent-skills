# 実行環境プロファイル: TypeScript + Vitest

このファイルは TypeScript + Vitest 向けの具体パターンをまとめる。
Jest 互換 API を採用しているため、本ファイルの考え方は Jest にも概ね流用できる（`vi` を `jest` に読み替える）。

## 目次

1. [一次情報](#runtime-ts-vitest-source)
2. [セットアップとレイアウト](#runtime-ts-vitest-layout)
3. [テスト名とドキュメンテーション](#runtime-ts-vitest-doc)
4. [test.each（パラメタライズド）](#runtime-ts-vitest-each)
5. [アサーション / 例外（同期・非同期）](#runtime-ts-vitest-assertion)
6. [セットアップ・ティアダウン](#runtime-ts-vitest-hooks)
7. [Fixture（test.extend）](#runtime-ts-vitest-fixture)
8. [モック（vi.fn / vi.spyOn / vi.mock）](#runtime-ts-vitest-mock)
9. [時刻・乱数の固定](#runtime-ts-vitest-timers)
10. [一時ディレクトリ・ファイルI/O](#runtime-ts-vitest-tmpdir)
11. [Integrationでの実依存/フェイク運用](#runtime-ts-vitest-integration)
12. [skip / todo / fails](#runtime-ts-vitest-skip)
13. [決定論チェック](#runtime-ts-vitest-determinism)

<a id="runtime-ts-vitest-source"></a>
## 一次情報

- Getting Started: https://vitest.dev/guide/
- Config Reference: https://vitest.dev/config/
- Test API（test.each / hooks / skip 等）: https://vitest.dev/api/
- Expect（マッチャ）: https://vitest.dev/api/expect.html
- Mocking ガイド: https://vitest.dev/guide/mocking.html
- Vi ユーティリティ（vi.fn / vi.mock / timers）: https://vitest.dev/api/vi.html
- Test Context / Fixtures: https://vitest.dev/guide/test-context.html

<a id="runtime-ts-vitest-layout"></a>
## 1. セットアップとレイアウト

- `vitest.config.ts` で `defineConfig` を使い、設定を1か所に集約する。
- `globals` は既定で無効。`describe / it / expect / vi` は `vitest` から明示的に import する（依存が読み取れ、型補完も効く）。
- バックエンド/ライブラリは `environment: 'node'`、DOM 依存は `jsdom` または `happy-dom` を選ぶ。
- モック状態のテスト間汚染を防ぐため、`restoreMocks: true` を既定にする（各テスト前に spy を復元）。
- 配置は `src/` 実装と対応させ、`*.test.ts` を近接配置（co-locate）または `tests/` に集約する。プロジェクトの既存方針に合わせる。

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    environment: 'node',
    globals: false,
    restoreMocks: true,
    include: ['src/**/*.test.ts', 'tests/**/*.test.ts'],
    setupFiles: ['tests/setup.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html'],
    },
  },
})
```

```jsonc
// tsconfig.json（テスト型を解決する最小設定）
{
  "compilerOptions": {
    "strict": true,
    "types": ["vitest/globals"] // globals を有効にする場合のみ必要。明示 import 運用なら不要。
  }
}
```

<a id="runtime-ts-vitest-doc"></a>
## 2. テスト名とドキュメンテーション

- `describe` に対象（公開関数・クラス）、`it` に `<条件>のとき<期待結果>となる` を自然言語で書く。テスト名自体が docstring の役割を担う。
- テスト名だけで意図が読み取れないケース（境界条件・防御的分岐・業務ルール由来）には、先頭コメントで「なぜこの検証が必要か」を補う。
- 3A（Arrange / Act / Assert）をコメントで区切る。空セクションは省略可、Act と Assert は必須。
- 変数名・コメントはプロダクションコードや設計書のユビキタス言語をそのまま使う。

```typescript
import { describe, it, expect } from 'vitest'
import { normalize } from '../src/normalize'

describe('normalize', () => {
  it('前後に空白を含む文字列のとき、小文字化かつトリム済みの値を返す', () => {
    // Act
    const result = normalize(' B ')

    // Assert
    expect(result).toBe('b')
  })
})
```

<a id="runtime-ts-vitest-each"></a>
## 3. test.each（パラメタライズド）

- 条件行列には `it.each` / `test.each` を使う。
- 可読性のため、ケースが多い・意味づけが要るときはオブジェクト形式（`$named` プレースホルダ）を優先する。
- 配列形式の書式トークン: `%s` 文字列 / `%i` 整数 / `%f` 浮動小数 / `%j` JSON / `%o` オブジェクト / `%#` 0始まり index。

```typescript
import { it, expect } from 'vitest'
import { normalize } from '../src/normalize'

it.each([
  { raw: 'A', expected: 'a' },
  { raw: ' B ', expected: 'b' },
])('raw=$raw のとき normalize は $expected を返す', ({ raw, expected }) => {
  // Act
  const result = normalize(raw)

  // Assert
  expect(result).toBe(expected)
})
```

<a id="runtime-ts-vitest-assertion"></a>
## 4. アサーション / 例外（同期・非同期）

- 値の比較は意図に合わせて選ぶ: 参照同一性/プリミティブは `toBe`、構造の深い比較は `toEqual`、型まで厳密に見るなら `toStrictEqual`、部分一致は `toMatchObject`。
- 同期例外は `expect(() => ...).toThrow(...)` で型・メッセージ（文字列部分一致または正規表現）を検証する。
- 非同期は必ず `await` を付け、`rejects.toThrow(...)` / `resolves.toEqual(...)` で検証する（`await` 漏れは検証が素通りする主因）。

```typescript
import { it, expect } from 'vitest'
import { divide, fetchUser } from '../src/service'

it('0 で除算したとき、"division" を含むメッセージの Error を送出する', () => {
  // Act / Assert
  expect(() => divide(1, 0)).toThrow(/division/)
})

it('存在しない userId のとき、Promise が NotFoundError で reject される', async () => {
  // Act / Assert
  await expect(fetchUser('missing')).rejects.toThrow('not found')
})

it('存在する userId のとき、ユーザーオブジェクトに解決される', async () => {
  // Act / Assert
  await expect(fetchUser('u1')).resolves.toMatchObject({ id: 'u1' })
})
```

<a id="runtime-ts-vitest-hooks"></a>
## 5. セットアップ・ティアダウン

- 共有 setup は `beforeEach` / `afterEach`（毎テスト）、`beforeAll` / `afterAll`（スイートで1回）で明示する。
- フックは Promise を返せば解決まで待機する。後始末は `afterEach` で確実に行い、テスト間で状態を共有しない。
- 単純な値共有よりも、関心が複数あるなら Fixture（次節）でスコープと後始末を一体化させる方が漏れにくい。

```typescript
import { describe, beforeEach, afterEach, it, expect } from 'vitest'
import { InMemoryUserRepo } from '../src/repo'

describe('UserRepo', () => {
  let repo: InMemoryUserRepo

  beforeEach(() => {
    // Arrange（各テスト前に新しいインスタンスを用意し、状態を分離する）
    repo = new InMemoryUserRepo()
  })

  afterEach(() => {
    repo.clear()
  })

  it('save した user を、同じ id で findById すると取り出せる', () => {
    // Act
    repo.save({ id: 'u1', name: 'alice' })

    // Assert
    expect(repo.findById('u1')).toEqual({ id: 'u1', name: 'alice' })
  })
})
```

<a id="runtime-ts-vitest-fixture"></a>
## 6. Fixture（test.extend）

- セットアップとティアダウンを1つの単位にまとめたいときは `test.extend` を使う。pytest の fixture 相当。
- オブジェクト形式では `use(value)` の前が Arrange、後がティアダウン。`await use(...)` の後に書いた処理がテスト終了後に必ず実行される。
- 型は generics で宣言する。fixture は他の fixture を引数で受け取って合成できる。

```typescript
import { test as base, expect } from 'vitest'
import { Database } from '../src/db'

interface Fixtures {
  db: Database
}

// db fixture を提供する test を定義する
const test = base.extend<Fixtures>({
  db: async ({}, use) => {
    // Arrange
    const db = new Database(':memory:')
    await db.migrate()

    await use(db) // ← ここでテスト本体が実行される

    // Teardown（テスト後に必ず呼ばれる）
    await db.close()
  },
})

test('insertUser した後に countUsers すると 1 を返す', async ({ db }) => {
  // Act
  await db.insertUser({ id: 'u1' })

  // Assert
  expect(await db.countUsers()).toBe(1)
})
```

<a id="runtime-ts-vitest-mock"></a>
## 7. モック（vi.fn / vi.spyOn / vi.mock）

- 原則は **spy 優先**。既存オブジェクト/モジュールのメソッドは `vi.spyOn` で監視し、テスト後に復元する（`restoreMocks: true` 推奨）。
- モジュール全体の差し替えが必要なときだけ `vi.mock` を使う。`vi.mock` はファイル先頭にホイストされるため、ファクトリ内で外部変数を参照する場合は `vi.hoisted` で巻き上げる。
- 元実装を一部だけ差し替えるときは `importOriginal` でスプレッドする（過剰モックを避ける）。
- 型付きで呼び出し検証するには `vi.mocked(...)` を使う。
- 呼び出し検証は `toHaveBeenCalledWith` / `toHaveBeenCalledOnce` / `toHaveBeenCalledTimes` を使い、観測可能な戻り値・副作用と併せて確認する（呼び出し回数だけを主目的にしない）。

```typescript
import { describe, it, expect, vi, afterEach } from 'vitest'
import * as mailer from '../src/mailer'
import { registerUser } from '../src/service'

afterEach(() => {
  vi.restoreAllMocks()
})

describe('registerUser', () => {
  it('登録に成功したとき、確認メールが当該アドレス宛に1回送信される', async () => {
    // Arrange（mailer.send を spy で監視し、実送信を抑止する）
    const sendSpy = vi.spyOn(mailer, 'send').mockResolvedValue(undefined)

    // Act
    const user = await registerUser({ email: 'a@example.com' })

    // Assert
    expect(user.email).toBe('a@example.com')
    expect(sendSpy).toHaveBeenCalledOnce()
    expect(sendSpy).toHaveBeenCalledWith('a@example.com')
  })
})
```

```typescript
// モジュール全体を差し替える例（外部 SDK 等）
import { describe, it, expect, vi } from 'vitest'
import { PaymentGateway } from '../src/gateway'
import { pay } from '../src/checkout'

vi.mock('../src/gateway')

describe('pay', () => {
  it('userId と amount を渡したとき、Gateway.charge が同じ引数で呼ばれ true を返す', async () => {
    // Arrange
    const charge = vi.fn().mockResolvedValue(true)
    vi.mocked(PaymentGateway).mockImplementation(
      () => ({ charge }) as unknown as PaymentGateway,
    )

    // Act
    const result = await pay('u1', 100)

    // Assert
    expect(result).toBe(true)
    expect(charge).toHaveBeenCalledWith('u1', 100)
  })
})
```

<a id="runtime-ts-vitest-timers"></a>
## 8. 時刻・乱数の固定

- 時刻依存は `vi.useFakeTimers()` + `vi.setSystemTime(new Date(...))` で固定し、`vi.advanceTimersByTime(ms)` でタイマーを進める。
- テスト後は `vi.useRealTimers()` で必ず戻す。
- 乱数は seed 可能な生成器を注入するか、`vi.spyOn(Math, 'random')` 等で固定する。本体実装に時刻/乱数を直書きせず、注入可能にしておくとテストが安定する。

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest'
import { createTimestampedId } from '../src/id'

describe('createTimestampedId', () => {
  beforeEach(() => {
    // Arrange（システム時刻を固定する）
    vi.useFakeTimers()
    vi.setSystemTime(new Date('2026-01-01T00:00:00Z'))
  })

  afterEach(() => {
    vi.useRealTimers()
  })

  it('固定時刻のとき、その時刻のエポックミリ秒を含む id を返す', () => {
    // Act
    const id = createTimestampedId()

    // Assert
    expect(id).toBe(`id-${Date.parse('2026-01-01T00:00:00Z')}`)
  })
})
```

<a id="runtime-ts-vitest-tmpdir"></a>
## 9. 一時ディレクトリ・ファイルI/O

- ファイルI/Oテストは OS の一時ディレクトリ配下に作業領域を作り、テスト後に削除する。Vitest に組み込みの tmp fixture は無いため、`fs.mkdtemp` と Fixture を組み合わせる（pytest の `tmp_path` 相当）。
- パスは `node:path` の `join` で組み立て、`/` 区切りに依存した文字列連結をしない。

```typescript
import { test as base, expect } from 'vitest'
import { mkdtemp, rm, readFile } from 'node:fs/promises'
import { tmpdir } from 'node:os'
import { join } from 'node:path'
import { writeReport } from '../src/report'

interface Fixtures {
  tmpDir: string
}

const test = base.extend<Fixtures>({
  tmpDir: async ({}, use) => {
    const dir = await mkdtemp(join(tmpdir(), 'report-'))
    await use(dir)
    await rm(dir, { recursive: true, force: true })
  },
})

test('レポートを書き込んだファイルから、同じテキストを読み戻せる', async ({ tmpDir }) => {
  // Arrange
  const reportPath = join(tmpDir, 'report.txt')

  // Act
  await writeReport(reportPath, 'ok')

  // Assert
  expect(await readFile(reportPath, 'utf-8')).toBe('ok')
})
```

<a id="runtime-ts-vitest-integration"></a>
## 10. Integrationでの実依存/フェイク運用

- DB境界の検証では、実DBを使うテストを最低1系統は持つ。TypeScript では **Testcontainers**（`@testcontainers/*`）でコンテナを起動・破棄するのが定番。
- 外部HTTP APIは **MSW（Mock Service Worker）** でネットワーク層をフェイクする。ハンドラを1か所に集約して再利用する。
- AWS SDK v3 は `aws-sdk-client-mock`（Vitest 互換なら `aws-sdk-client-mock-vitest`）でコマンド呼び出しを検証する。
- 使い分けの基本方針:
  - SQL/トランザクション/制約を確認したい → 実DBコンテナ（Testcontainers）
  - HTTP クライアントの組み立て・エラー処理・リトライを確認したい → MSW
  - AWS SDK 呼び出し（バケット操作・キー設計など）を確認したい → aws-sdk-client-mock
- 統合テストは `*.integration.test.ts` のように suffix で分離し、CIで実行ジョブを分けると管理しやすい。

```typescript
// MSW: HTTP 境界のフェイク。tests/setup.ts などに置く。
import { beforeAll, afterEach, afterAll } from 'vitest'
import { setupServer } from 'msw/node'
import { http, HttpResponse } from 'msw'

export const server = setupServer(
  http.get('https://api.example.com/users/:id', ({ params }) =>
    HttpResponse.json({ id: params.id, name: 'alice' }),
  ),
)

beforeAll(() => server.listen({ onUnhandledRequest: 'error' }))
afterEach(() => server.resetHandlers()) // テスト毎に既定ハンドラへ戻す
afterAll(() => server.close())
```

```typescript
// Testcontainers: 実 PostgreSQL に対する検証
import { describe, it, expect, beforeAll, afterAll } from 'vitest'
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from '@testcontainers/postgresql'
import { UserRepository } from '../src/user-repository'

describe('UserRepository (integration)', () => {
  let container: StartedPostgreSqlContainer
  let repo: UserRepository

  beforeAll(async () => {
    container = await new PostgreSqlContainer('postgres:16').start()
    repo = await UserRepository.connect(container.getConnectionUri())
  }, 60_000)

  afterAll(async () => {
    await repo.close()
    await container.stop()
  })

  it('実 PostgreSQL に insert した user を、同じ id で取得すると同じ name が返る', async () => {
    // Act
    await repo.insert({ id: 1, name: 'alice' })

    // Assert
    expect(await repo.findById(1)).toMatchObject({ name: 'alice' })
  })
})
```

<a id="runtime-ts-vitest-skip"></a>
## 11. skip / todo / fails

- 実行前提を満たせないときは `it.skip`、実行時条件で飛ばすなら `ctx.skip(condition, reason)`。
- 未実装の振る舞いは `it.todo('...')` でレポートに残す。
- 既知不具合の再現は `it.fails('repro #1234', ...)` を使い、修正されたら通常テストへ戻す（XPASS の放置を避ける）。

```typescript
import { it, expect } from 'vitest'

it.todo('上限超過のとき QuotaExceededError を送出する')

it.fails('再現 #1234: add の桁上がりが誤る', () => {
  expect(add(1, 2)).toBe(4)
})
```

<a id="runtime-ts-vitest-determinism"></a>
## 12. 決定論チェック

- 乱数を固定する（注入した seed か固定データ）。
- 時刻依存を固定する（`vi.useFakeTimers` + `vi.setSystemTime`）。
- 実ネットワークを原則使わない（MSW で `onUnhandledRequest: 'error'`、未モック通信を即失敗させる）。
- テスト間共有状態を排除する（`restoreMocks` / `afterEach` のクリーンアップ / Fixture のティアダウン）。
- 並行実行（`test.concurrent`・並列ワーカー）でも同結果になることを確認する。共有リソースに依存するテストは分離する。
</content>
</invoke>
