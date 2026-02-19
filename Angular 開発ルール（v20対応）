# Angular 開発ルール（v20対応）
> 対象読者：経験3年目以上のエンジニア  
> 「何をするか」だけでなく「なぜそうするか」を理解してルールを使いこなすことを目的とする。

---

## 目次

1. [アーキテクチャ概要](#1-アーキテクチャ概要)
2. [ディレクトリ構成](#2-ディレクトリ構成)
3. [モジュール管理](#3-モジュール管理)
4. [Component 設計](#4-component-設計)
5. [Template 設計](#5-template-設計)
6. [Pages 設計](#6-pages-設計)
7. [Service 設計（インフラ層）](#7-service-設計インフラ層)
8. [Store・型定義管理](#8-store型定義管理)
9. [非同期処理（RxJS）](#9-非同期処理rxjs)
10. [変更検知戦略](#10-変更検知戦略)
11. [DI（依存性注入）](#11-di依存性注入)
12. [フォーム管理](#12-フォーム管理)
13. [ルーティング](#13-ルーティング)
14. [エラーハンドリング](#14-エラーハンドリング)
15. [テスト（Jest）](#15-テストjest)
16. [命名規則](#16-命名規則)
17. [禁止事項一覧](#17-禁止事項一覧)

---

## 1. アーキテクチャ概要

```
HTTP / External API
      ↓
  [ Service ]   ← インフラ層（APIコール・レスポンス型変換）
      ↓
  [ Store ]     ← 状態管理・型定義の起点（NgRx）
      ↓
  [ Effects ]   ← 非同期の副作用（ServiceとStoreの橋渡し）
      ↓
  [ Pages ]     ← ビジネスロジックの集約・Storeとの接続
      ↓
  [ Template ]  ← コンポーネントの組み立て場所（レイアウト）
      ↓
  [ Component ] ← 最小表示単位・ロジックを持たない
```

### なぜこの層構造にするのか

**「どこに何を書けばいいか」を全員が迷わないようにするため**、そして**変更の影響範囲を予測可能にするため**です。

層構造がなく自由に書いていると、時間が経つにつれてどのファイルも似たような処理を持ち始めます。「このバグはどこを直せばいい？」という問いに5分以内に答えられないコードベースは、暗黙のルールがないか、あっても守られていない状態です。

各層の責務を厳格に分けると次のメリットが生まれます。

- **影響範囲の局所化**：APIレスポンスの型が変わっても変更は Service だけ、表示デザインが変わっても変更は Component だけ
- **テストのしやすさ**：1層が1責務しか持たないため、テスト対象が明確になる
- **レビューの効率化**：コードを見た瞬間にどの責務のファイルかわかる

### 責務の一覧

| 層 | 責務 | 持ってはいけないもの |
|----|------|---------------------|
| Component | 表示・イベント発火 | ビジネスロジック・API呼び出し |
| Template | コンポーネントの配置・レイアウト | ビジネスロジック |
| Pages | ビジネスロジック・Store 操作 | 直接的な HTTP 通信 |
| Effects | 非同期副作用 | 表示ロジック |
| Store | 状態保持・型定義 | 副作用 |
| Service | HTTP 通信・インフラ処理 | ビジネスロジック・Store 参照 |

---

## 2. ディレクトリ構成

```
src/
├── app/
│   ├── core/                        # アプリ全体で1インスタンスのもの
│   │   ├── guards/
│   │   ├── interceptors/
│   │   ├── handlers/
│   │   └── core.module.ts
│   │
│   ├── shared/                      # 複数機能で使い回す汎用部品
│   │   ├── components/
│   │   ├── directives/
│   │   ├── pipes/
│   │   └── shared.module.ts
│   │
│   ├── features/                    # 機能ごとに閉じた実装
│   │   └── [feature-name]/
│   │       ├── components/          # 最小コンポーネント単位
│   │       │   └── [name]/
│   │       │       ├── [name].component.ts
│   │       │       ├── [name].component.html
│   │       │       ├── [name].component.scss
│   │       │       └── [name].component.spec.ts
│   │       ├── templates/           # コンポーネント組み立て場所
│   │       │   └── [name]/
│   │       │       ├── [name].template.ts
│   │       │       ├── [name].template.html
│   │       │       └── [name].template.spec.ts
│   │       ├── pages/               # ビジネスロジック集約
│   │       │   └── [name]/
│   │       │       ├── [name].page.ts
│   │       │       ├── [name].page.html
│   │       │       ├── [name].page.spec.ts
│   │       │       └── [name].page.functions.ts   # 純粋関数の切り出し（任意）
│   │       ├── store/
│   │       │   ├── [name].actions.ts
│   │       │   ├── [name].reducer.ts
│   │       │   ├── [name].selectors.ts
│   │       │   ├── [name].effects.ts
│   │       │   └── [name].state.ts              # 型定義の起点
│   │       ├── services/
│   │       │   └── [name].service.ts
│   │       └── [feature-name].module.ts
│   │
│   ├── app-routing.module.ts
│   ├── app.component.ts
│   └── app.module.ts
```

### なぜ features/ で機能ごとに閉じるのか

機能をディレクトリで閉じることで「**コロケーション（関連するものを近くに置く）**」が実現されます。ユーザー管理機能を修正するとき、`features/users/` を開けば必要なものがすべて揃います。

逆に、全コンポーネントを `src/components/`、全サービスを `src/services/` のように種類別にフラットに並べると、1つの機能を追いかけるためにファイルツリーを何度も上下移動することになります。プロジェクトが大きくなるほどこのコストは増大します。

### なぜ core/ と shared/ を分けるのか

この2つは性質が異なります。

`core/` はアプリ全体で **インスタンスが1つしか存在してはいけない**ものを置く場所です（Interceptor・GlobalErrorHandler・認証サービス等）。複数インスタンスが生成されると認証トークンが二重管理されたり、ログが重複するバグが起きます。

`shared/` は **何度でも使い回していいUI部品や純粋なユーティリティ**を置く場所です（汎用ボタン・日付パイプ等）。インスタンスが複数存在しても問題ありません。

---

## 3. モジュール管理

### 基本方針

- **Standalone コンポーネントは使用禁止**（NgModule で管理）
- `AppModule`・`CoreModule`・`SharedModule`・各 `FeatureModule` の4層構造
- `CoreModule` は `AppModule` のみがインポートし、`forRoot()` パターンを使用

```typescript
// feature.module.ts の基本形
@NgModule({
  declarations: [
    FeaturePageComponent,
    FeatureTemplateComponent,
    FeatureItemComponent,
  ],
  imports: [
    CommonModule,
    SharedModule,
    ReactiveFormsModule,
    StoreModule.forFeature(featureFeatureKey, featureReducer),
    EffectsModule.forFeature([FeatureEffects]),
    FeatureRoutingModule,
  ],
})
export class FeatureModule {}
```

### なぜ Standalone を禁止するのか

Angular v14 から導入された Standalone は「NgModule を書かずにコンポーネントを使える」便利な仕組みですが、現在のプロジェクトでは **NgModule による一元管理のほうが得られるメリットが大きい**ため採用しません。

NgModule の利点は次の通りです。

- **依存関係の可視化**：`declarations` と `imports` を見れば、そのモジュールが何を使い何に依存しているかが1ファイルで把握できる
- **スコープの明示的な制御**：Standalone は各コンポーネントが個別にインポートを管理するため、大規模になると「このコンポーネントがどこで使われているか」が追いにくくなる
- **チームの学習コスト統一**：NgModule ベースの書き方はAngularの伝統的なパターンであり、参考資料が豊富

Standalone への移行は、プロジェクト全体で方針を揃えて一括移行するタイミングで検討します。

### CoreModule のガード

```typescript
// core.module.ts
// なぜこのガードが必要か：
// CoreModule が複数回インポートされると Interceptor 等が
// 二重に登録されてリクエストが2回送られるバグが起きる。
// コンストラクタで親モジュールの存在を確認して防ぐ。
@NgModule({ ... })
export class CoreModule {
  constructor(@Optional() @SkipSelf() parentModule: CoreModule) {
    if (parentModule) {
      throw new Error('CoreModule は AppModule のみがインポートできます');
    }
  }
}
```

---

## 4. Component 設計

### 基本方針

- **最小コンポーネント単位**：1つのUIパーツのみを担当
- **ロジックを持たない**：表示値の最終変換のみ（`pure function` 的な扱い）
- `@Input()` でデータを受け取り、`@Output()` でイベントを発火するのみ
- `ChangeDetectionStrategy.OnPush` を**必須**とする

```typescript
@Component({
  selector: 'app-user-card',
  templateUrl: './user-card.component.html',
  styleUrls: ['./user-card.component.scss'],
  changeDetection: ChangeDetectionStrategy.OnPush,  // 必須（理由は §10 参照）
})
export class UserCardComponent {
  @Input({ required: true }) user!: UserCardViewModel;
  @Output() selected = new EventEmitter<string>();

  onSelect(): void {
    this.selected.emit(this.user.id);
  }
}
```

### なぜ Component はロジックを持たないのか

Component をロジックのない「**純粋な表示装置**」にすると、次のことが保証されます。

- **同じ Input を渡せば必ず同じ表示になる**（べき等性）
- **テストが超シンプルになる**：渡したデータが表示されているか確認するだけでよい
- **デザイン変更がロジックに影響しない**：HTMLを変えてもビジネスロジックは壊れない

ロジックが混入しはじめる典型パターンは「表示用の加工をコンポーネント内でやり始める」です。例えば日付フォーマット変換・状態に応じたラベル文字列の生成などは Selector に追い出します。

### なぜ ViewModel を経由するのか

Store の型（`User` エンティティ）をそのまま Component に渡すと、Store の構造変更が Component まで影響します。ViewModel（表示専用の型）を Selector でマッピングすることで **Store 層と表示層の結合を切り離し**ます。

```typescript
// ❌ Store の型をそのまま渡している
@Input() user!: User;  // User の構造が変わると Component まで修正が必要

// ✅ 表示用にマッピングした ViewModel を渡す
@Input() user!: UserCardViewModel;  // Store 構造が変わっても Selector だけ修正すればよい

// store/user.selectors.ts で変換
export const selectUserCardViewModels = createSelector(
  selectAllUsers,
  (users): UserCardViewModel[] => users.map(toUserCardViewModel)
);
```

---

## 5. Template 設計

### 基本方針

- **コンポーネントの組み立て場所**：レイアウトとコンポーネント配置のみ
- Pages から渡されたデータをコンポーネントへ中継する
- 独自のビジネスロジックは持たない

```typescript
@Component({
  selector: 'app-user-list-template',
  templateUrl: './user-list.template.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserListTemplateComponent {
  @Input({ required: true }) users!: UserCardViewModel[];
  @Input({ required: true }) isLoading!: boolean;
  @Output() userSelected = new EventEmitter<string>();
}
```

```html
<!-- user-list.template.html -->
<app-loading-spinner *ngIf="isLoading" />

<div class="user-list">
  <app-user-card
    *ngFor="let user of users; trackBy: trackById"
    [user]="user"
    (selected)="userSelected.emit($event)"
  />
</div>
```

### なぜ Pages と Template を分けるのか

**Pages にビジネスロジック、Template にレイアウト** と分けることで、2つの独立した変更軸を分離できます。

- ページのビジネスロジックが変わる（表示条件・取得タイミング等）→ Pages だけ変更
- UIのレイアウトが変わる（グリッドからリストへの切り替え等）→ Template だけ変更

また、Template は Pages から切り離されているため「同じレイアウトを別のページでも使い回す」リファクタリングが容易になります。

### `trackBy` の必須化

```typescript
// なぜ trackBy が必要か：
// trackBy がないと、*ngFor は配列が更新されるたびに
// 全DOMを破棄して再生成する。
// trackBy を指定すると、id が同じ要素は DOM を再利用するため
// 特にリストが長い場合のパフォーマンスが大幅に改善する。
trackById(_index: number, item: { id: string }): string {
  return item.id;
}
```

---

## 6. Pages 設計

### 基本方針

- **ビジネスロジックの集約場所**：Store の Dispatch・Select はここのみ
- `Observable` を `async` パイプ経由で Template に渡す
- 直接 DOM 操作・`subscribe()` 内での State 変更は禁止

```typescript
@Component({
  selector: 'app-user-list-page',
  templateUrl: './user-list.page.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserListPageComponent implements OnInit, OnDestroy {
  private readonly store = inject(Store);
  private readonly destroy$ = new Subject<void>();

  readonly users$: Observable<UserCardViewModel[]> = this.store.select(selectUserCardViewModels);
  readonly isLoading$: Observable<boolean> = this.store.select(selectUsersLoading);

  ngOnInit(): void {
    this.store.dispatch(UserActions.loadUsers());
  }

  onUserSelected(userId: string): void {
    this.store.dispatch(UserActions.selectUser({ userId }));
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

```html
<!-- user-list.page.html -->
<app-user-list-template
  [users]="users$ | async"
  [isLoading]="isLoading$ | async"
  (userSelected)="onUserSelected($event)"
/>
```

### なぜ Store の操作を Pages に集約するのか

Store へのアクセスが Component・Template に散らばると「どこで State が変わったか」を追うのが困難になります。Pages を **Store との唯一の接続点** にすることで、デバッグ時に「State の変更を調べるなら Pages を見る」という一貫したルールが成立します。

NgRx DevTools と組み合わせると、Action の発火タイミングと State の変化が時系列で追えるため、Pages に集約するメリットがさらに大きくなります。

### なぜ `async` パイプを使うのか

`async` パイプは Angular が管理する仕組みのため、コンポーネントが破棄されると**自動的に購読を解除**します。手動で `subscribe()` して `ngOnDestroy` で `unsubscribe()` するコードを書き忘れると、コンポーネントが破棄されても Observable が生き続けるメモリリークが発生します。`async` パイプを使うとこの問題を根本的に防げます。

### 関数ファイルの分離（任意）

Pages 内のビジネスロジックが複雑になったとき、**純粋関数**として切り出せるものは `*.page.functions.ts` に移動します。

```typescript
// user-list.page.functions.ts
// なぜ切り出すか：純粋関数にすることで TestBed 不要の
// シンプルな単体テストが書けるようになる。
export function toUserCardViewModel(user: User): UserCardViewModel {
  return {
    id: user.id,
    name: `${user.lastName} ${user.firstName}`,
    avatarUrl: user.profile?.avatarUrl ?? '/assets/default-avatar.png',
  };
}
```

---

## 7. Service 設計（インフラ層）

### 基本方針

- **HTTP通信・外部リソースへのアクセスのみ**を担当
- ビジネスロジックは持たない（Store への接続も禁止）
- 戻り値は常に `Observable<T>`
- `providedIn: 'root'` を原則とする

```typescript
@Injectable({ providedIn: 'root' })
export class UserApiService {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = '/api/users';

  getUsers(): Observable<UserResponse[]> {
    return this.http.get<UserResponse[]>(this.baseUrl);
  }

  getUserById(id: string): Observable<UserResponse> {
    return this.http.get<UserResponse>(`${this.baseUrl}/${id}`);
  }

  createUser(payload: CreateUserRequest): Observable<UserResponse> {
    return this.http.post<UserResponse>(this.baseUrl, payload);
  }
}
```

### なぜ Service をインフラ層に限定するのか

Service にビジネスロジックが入り始めると「Store からデータを取得して加工して別のエンドポイントに送る」ような処理が Service 内に増え、**どこで何が起きているか追いにくい**コードになります。

Service をインフラ層（「外と話す窓口」）に徹することで、テスト時に `HttpTestingController` だけモックすればよい状態になります。ビジネスロジックのテストは Pages・Effects・Store に集中できます。

### なぜ戻り値を `Observable<T>` に統一するのか

`Observable<T>` は「データが流れてくるパイプ」です。一度定義すれば、購読するまで実行されない（遅延評価）ため、Effects の中で `switchMap` で簡単にキャンセル・切り替えができます。`Promise` に変換してしまうと、この柔軟性が失われます。

### Interceptor の活用

```typescript
// core/interceptors/auth.interceptor.ts
// なぜ Interceptor を使うか：
// 全HTTPリクエストに共通する処理（認証ヘッダー付与・エラー変換）を
// 各 Service に書かずに一元管理できる。
// Service は「このエンドポイントを叩く」という関心だけに集中できる。
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthTokenService).getToken();
  const authReq = token
    ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } })
    : req;
  return next(authReq);
};
```

---

## 8. Store・型定義管理

### 基本方針

- **型定義の起点は Store**（`*.state.ts`）
- 各レイヤーは Store の型を `Pick` / `Partial` / `Omit` で継承して使う
- `EntityState` を積極活用してリスト管理を標準化する

### なぜ Store を型定義の起点にするのか

型定義がバラバラのファイルに分散していると、同じエンティティの型が `User`・`UserModel`・`UserData` のように微妙に違う形で複数存在し始めます。「どれが正しい型か」を判断するコストが発生し、型の不一致バグも起きやすくなります。

Store はアプリ全体の「**真実の単一源泉（Single Source of Truth）**」です。型定義もここに集約することで「正しい型はStoreを見ればわかる」という一貫性が生まれます。

### State 定義（型の起点）

```typescript
// store/user.state.ts
import { EntityState } from '@ngrx/entity';

// アプリ全体で使うエンティティの型
export interface User {
  id: string;
  firstName: string;
  lastName: string;
  email: string;
  profile: UserProfile | null;
}

export interface UserProfile {
  avatarUrl: string;
  bio: string;
}

// Store の状態型：ここがすべての型の起点
export interface UserState extends EntityState<User> {
  selectedUserId: string | null;
  loading: boolean;
  error: string | null;
}
```

### なぜ `EntityState` を使うのか

`EntityState` は NgRx が提供するリスト管理の標準パターンです。内部では `{ ids: string[], entities: { [id]: Entity } }` という正規化された構造で保持します。

配列（`User[]`）で持つと特定IDの要素を探すのに `O(n)` かかりますが、正規化構造では `O(1)` でアクセスできます。また `EntityAdapter` が `addOne`・`upsertMany`・`removeOne` 等の操作関数を提供するため、Reducer の実装量が大幅に減ります。

### Actions

```typescript
// store/user.actions.ts
// なぜ createActionGroup を使うか：
// 関連するアクションを1箇所に集め、
// source プレフィックス（"User"）が自動付与されるため
// DevTools でどの機能のアクションか一目でわかる。
export const UserActions = createActionGroup({
  source: 'User',
  events: {
    'Load Users': emptyProps(),
    'Load Users Success': props<{ users: User[] }>(),
    'Load Users Failure': props<{ error: string }>(),
    'Select User': props<{ userId: string }>(),
  },
});
```

### Selectors（型変換の橋渡し役）

```typescript
// store/user.selectors.ts
// なぜ Selector でViewModel変換をするのか：
// コンポーネントは表示に必要な型だけを知っていればいい。
// Store の構造変更の影響を Selector でとめることができる。
const { selectAll } = userAdapter.getSelectors();

export const selectUserState = createFeatureSelector<UserState>(userFeatureKey);
export const selectAllUsers = createSelector(selectUserState, selectAll);
export const selectUsersLoading = createSelector(selectUserState, (s) => s.loading);

export const selectUserCardViewModels = createSelector(
  selectAllUsers,
  (users): UserCardViewModel[] => users.map(toUserCardViewModel)
);
```

### Effects（非同期の副作用を担う）

```typescript
// store/user.effects.ts
// なぜ Effects で非同期処理をするのか：
// Reducer は純粋関数（同じ入力→同じ出力）でなければならない。
// 副作用（HTTP通信・タイマー等）は Reducer に書けないため、
// Effects が「アクションを受け取りServiceを呼び、結果をアクションに変換して返す」
// 役割を担う。
@Injectable()
export class UserEffects {
  private readonly actions$ = inject(Actions);
  private readonly userApi = inject(UserApiService);

  loadUsers$ = createEffect(() =>
    this.actions$.pipe(
      ofType(UserActions.loadUsers),
      switchMap(() =>
        this.userApi.getUsers().pipe(
          map((users) => UserActions.loadUsersSuccess({ users })),
          catchError((error: HttpErrorResponse) =>
            of(UserActions.loadUsersFailure({ error: error.message }))
          )
        )
      )
    )
  );
}
```

---

## 9. 非同期処理（RxJS）

### 基本方針

- **非同期処理は RxJS のみ**（`Promise` / `async-await` 禁止）
- `subscribe()` は Pages・Effects・CoreModule の限定された場所のみ
- Template では `async` パイプを使用し、購読管理を Angular に委ねる

### なぜ Promise ではなく Observable なのか

`Promise` は「一度だけ解決される非同期処理」です。対して `Observable` は「時間の流れに沿った複数の値の列」を扱えます。

Angularのアプリでは「ユーザー操作・HTTPレスポンス・Storeの状態変化」という複数の非同期イベントを組み合わせて処理することが多く、これらを **宣言的に合成できる** RxJS の方が適しています。例えばユーザーの入力をデバウンスしてAPIを叩き、最新のリクエストだけを有効にする処理は RxJS では数行で書けますが、Promiseでは複雑な状態管理が必要です。

### Operator の使い分け

| シーン | Operator | なぜこれを使うか |
|--------|----------|----------------|
| HTTP通信（最新リクエストのみ有効） | `switchMap` | 前のリクエストを自動キャンセルする。連打対策になる |
| 全リクエストを順番に処理 | `concatMap` | 先入れ先出しで順序を保証する |
| 並列処理 | `mergeMap` | 順序を気にせず同時に実行する |
| エラー時に別のObservableへ | `catchError` + `of()` | エラーをStreamの外に出さず、Storeに渡す形に変換する |
| 値の変換 | `map` | 純粋な変換。副作用を書いてはいけない |
| 副作用（ログ・デバッグ） | `tap` | Streamの値を変えずに副作用だけ実行する |
| 重複排除 | `distinctUntilChanged` | 同じ値を連続して流さない。無駄な再レンダリングを防ぐ |
| デバウンス | `debounceTime` | 最後のイベントからn ms後に値を流す。入力フォームの検索等 |
| 複数Observableを常に最新で合成 | `combineLatest` | どちらかが更新されたら両方の最新値でStreamを流す |
| 複数Observableを全部完了後に合成 | `forkJoin` | 複数のHTTPリクエストを並列に投げて全完了を待つ |

### なぜ `switchMap` が連打対策になるのか

```typescript
// switchMap の動作を理解する
searchInput$.pipe(
  debounceTime(300),
  switchMap((query) => this.searchApi.search(query))
  // ↑ 新しい query が来たとき、前の HTTP リクエストを
  //   自動でキャンセルして新しいリクエストを開始する。
  //   mergeMap だとキャンセルされず古いレスポンスが後から
  //   届いて表示が上書きされるバグが起きる。
)
```

### メモリリーク防止（`subscribe()` を使う場合）

```typescript
// なぜ takeUntil が必要か：
// Angular がコンポーネントを破棄しても、
// subscribe() した Observable は購読を続ける。
// 破棄済みコンポーネントへの参照が残ることでメモリリークが起き、
// 最悪の場合、破棄済みコンポーネントに値が届いてエラーになる。
private readonly destroy$ = new Subject<void>();

someObservable$
  .pipe(takeUntil(this.destroy$))
  .subscribe((value) => { /* 副作用処理 */ });

ngOnDestroy(): void {
  this.destroy$.next();    // destroy$ に値を流してすべての takeUntil を完了させる
  this.destroy$.complete(); // destroy$ 自体を完了させてメモリを解放する
}
```

### 禁止パターン

```typescript
// ❌ Promise 使用
async loadData(): Promise<void> {
  const data = await this.service.getData().toPromise();
  // → toPromise() は非推奨。RxJS の合成も使えなくなる
}

// ❌ subscribe 内での State 変更
this.users$.subscribe(users => {
  this.store.dispatch(SomeAction({ users }));
  // → Effects で行うべき。subscribe 内で dispatch すると
  //   Action の発火タイミングが追いにくくなる
});

// ✅ 正しいパターン（async パイプで自動購読管理）
readonly users$ = this.store.select(selectAllUsers);
// template 側: [users]="users$ | async"
```

---

## 10. 変更検知戦略

### 基本方針

- **全コンポーネントで `ChangeDetectionStrategy.OnPush` を必須**
- Immutable なデータを扱い、参照変更で変更検知をトリガーする
- `async` パイプは自動で `markForCheck()` を呼ぶため積極活用する

### なぜ `OnPush` にするのか（Angular の変更検知の仕組みから理解する）

Angular はデフォルト（`Default` 戦略）だと、あらゆるブラウザイベント（クリック・タイマー・XHRレスポンス等）が発生するたびに **アプリ全コンポーネントを上から下へ走査**して変更がないか確認します。これは `zone.js` が非同期イベントに割り込む仕組みで実現されています。

`OnPush` にすると、そのコンポーネントの検知は **次の条件が満たされたときのみ**実行されます。

1. `@Input()` の参照が変わった
2. コンポーネント内または子コンポーネントで DOM イベントが発火した
3. `async` パイプが新しい値を受け取った
4. 手動で `markForCheck()` が呼ばれた

全コンポーネントを `OnPush` にすると、Store → Selector → `async` パイプという流れでのみ変更検知が起きるようになり、**無駄な検知サイクルを排除**できます。コンポーネントが増えるほどこのパフォーマンス差は大きくなります。

```typescript
// ❌ OnPush が機能しない典型的なアンチパターン
this.users.push(newUser);
// → 配列の参照は変わっていないので OnPush は検知しない

// ✅ 新しい参照で置き換える（Immutable な操作）
this.users = [...this.users, newUser];

// ✅ Store + async パイプを使えばこの問題は自動解決
// Store は state を必ず新しいオブジェクトとして返すため、
// 参照変更が保証される
```

### 手動での変更検知（最小限に留める）

`async` パイプが使えない場面（サードパーティライブラリのコールバック等 Zone の外から変更が起きる場合）のみ使用します。

```typescript
private readonly cdr = inject(ChangeDetectorRef);

externalCallback(data: SomeData): void {
  this.data = data;
  this.cdr.markForCheck(); // OnPush のコンポーネントに「次の検知サイクルで自分を確認して」と伝える
}
```

---

## 11. DI（依存性注入）

### 基本方針

- コンストラクタインジェクションより **`inject()` 関数を推奨**
- `providedIn: 'root'` を基本とする

### なぜ `inject()` 関数を推奨するのか

Angular v14 以降で使える `inject()` 関数はコンストラクタの外でも呼べるため、次のメリットがあります。

- コンストラクタの引数リストが肥大化しない（依存が増えるほど差が出る）
- ミックスイン・関数ベースのユーティリティ内でも DI が使える
- 型推論が効きやすくコードが簡潔になる

```typescript
// ✅ inject() 関数（推奨）
@Component({ ... })
export class SomeComponent {
  private readonly store = inject(Store);
  private readonly router = inject(Router);
  private readonly userApi = inject(UserApiService);
}

// コンストラクタインジェクション（旧来の書き方として許容）
constructor(
  private readonly store: Store,
  private readonly router: Router,
  private readonly userApi: UserApiService,
) {}
```

### InjectionToken の活用

```typescript
// なぜ InjectionToken を使うか：
// 文字列や数値などのプリミティブな値（APIのURL等）をDIする場合、
// 型だけではどの値を注入するか特定できない。
// InjectionToken はその値に一意なキーを与えてDIコンテナに登録できる。
export const API_URL = new InjectionToken<string>('API_URL');

// app.module.ts
providers: [
  { provide: API_URL, useValue: environment.apiUrl }
]

// service内での使用（環境変数を直接 import せず DI で受け取る）
private readonly apiUrl = inject(API_URL);
```

---

## 12. フォーム管理

### 基本方針

- **`ReactiveFormsModule` を使用**（`FormsModule` / `ngModel` は禁止）
- フォームの定義は Pages で行い、Component に `FormGroup` を `@Input` で渡す
- バリデーションは `Validators` / カスタムバリデータ関数で統一

### なぜ Reactive Forms（リアクティブフォーム）なのか

`ngModel`（テンプレート駆動フォーム）はHTMLに状態が分散するため、複雑なバリデーション・動的フォームの管理が難しくなります。

Reactive Forms では **TypeScript 側でフォームの構造・バリデーション・状態がすべて管理**されるため、次のことが容易になります。

- フォームの状態が Observable として取得できる（RxJS と統合しやすい）
- バリデーション関数が純粋関数として書けるためテストが簡単
- 型安全（v14以降の Typed Forms）

```typescript
// pages/user-create.page.ts
export class UserCreatePageComponent {
  private readonly fb = inject(FormBuilder);
  private readonly store = inject(Store);

  // TypedFormGroupを使った型安全なフォーム定義
  readonly userForm = this.fb.group({
    firstName: this.fb.control('', { validators: [Validators.required, Validators.maxLength(50)], nonNullable: true }),
    lastName: this.fb.control('', { validators: [Validators.required, Validators.maxLength(50)], nonNullable: true }),
    email: this.fb.control('', { validators: [Validators.required, Validators.email], nonNullable: true }),
  });

  onSubmit(): void {
    if (this.userForm.invalid) return;
    // getRawValue() は disabled なコントロールの値も含めて取得する
    // value だと disabled コントロールが undefined になり型が崩れる
    this.store.dispatch(UserActions.createUser({ payload: this.userForm.getRawValue() }));
  }
}
```

---

## 13. ルーティング

### 基本方針

- **遅延ロード（`loadChildren`）を機能モジュール単位で必須**
- ルート定義は `*-routing.module.ts` に集約
- Guards・Resolvers は `core/guards/` に配置

### なぜ遅延ロードを必須にするのか

Angular はデフォルトだと起動時にすべてのモジュールを読み込みます。`loadChildren` で遅延ロードにすると「そのページを初めて表示するとき」に初めてモジュールを取得します。

ユーザーが最初に開くページのバンドルサイズを小さくできるため、**初期表示速度（LCP）の改善**に直結します。機能が増えるほど差が開きます。

```typescript
// app-routing.module.ts
const routes: Routes = [
  {
    path: 'users',
    // このモジュールは /users に初めてアクセスしたときに動的ロードされる
    loadChildren: () =>
      import('./features/users/users.module').then((m) => m.UsersModule),
  },
  { path: '', redirectTo: 'users', pathMatch: 'full' },
  { path: '**', loadChildren: () => import('./features/not-found/not-found.module').then(m => m.NotFoundModule) },
];
```

### Guard の実装（関数型 Guard）

```typescript
// core/guards/auth.guard.ts
// なぜ関数型 Guard を使うか：
// クラス型 Guard（implements CanActivate）は Angular v15 以降で非推奨。
// 関数型にすると TestBed が不要な単純な関数テストができる。
export const authGuard: CanActivateFn = () => {
  const authService = inject(AuthService);
  const router = inject(Router);

  return authService.isAuthenticated$.pipe(
    map((isAuth) => isAuth || router.createUrlTree(['/login']))
    // true ならナビゲーション許可、false なら /login へリダイレクト
  );
};
```

---

## 14. エラーハンドリング

### 基本方針

- HTTP エラーは `HttpInterceptor` で一元ハンドリング
- ビジネスロジックのエラーは `Effects` の `catchError` で Store へ反映
- ユーザー向けエラー表示は Store の `error` state を通じて行う

### なぜエラーを Store に流すのか

エラー状態を Store で管理することで「エラーが起きているかどうか」が Observable として取得でき、画面のどこからでも参照できます。`catch` したエラーをコンポーネント内のローカル変数に持つと、他のコンポーネントからは参照できず、ローカル通知しか実装できません。

```typescript
// effects でのエラーハンドリング
loadUsers$ = createEffect(() =>
  this.actions$.pipe(
    ofType(UserActions.loadUsers),
    switchMap(() =>
      this.userApi.getUsers().pipe(
        map((users) => UserActions.loadUsersSuccess({ users })),
        catchError((error: HttpErrorResponse) =>
          // エラーを Observable の外（例外）に出さず、
          // Failure アクションというイベントとして Store に流す。
          // これにより Effects の Stream が死なずに次のアクションを受け取れる。
          of(UserActions.loadUsersFailure({ error: error.message }))
        )
      )
    )
  )
);
```

### なぜ `catchError` で `of()` を使うのか

Effects の `createEffect` が受け取る Observable はアプリが動いている間ずっと生き続けます。`catchError` でエラーをそのまま throw してしまうと **Effects の Stream 自体が終了**し、以降そのアクションを受け取れなくなります。`of(FailureAction)` で1件のアクションに変換することで、Stream を終わらせずにエラーを安全に処理できます。

### グローバルエラーハンドラー

```typescript
// core/handlers/global-error.handler.ts
// なぜグローバルエラーハンドラーを用意するか：
// try-catch や catchError で捕捉し損ねた予期しないエラーの
// 最後の受け皿として機能する。これがないとエラーがコンソールに
// 出るだけでユーザーに何も通知されない。
@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  private readonly store = inject(Store);

  handleError(error: unknown): void {
    console.error('Unhandled error:', error);
    this.store.dispatch(AppActions.globalError({ message: 'システムエラーが発生しました' }));
  }
}

// core.module.ts
providers: [
  { provide: ErrorHandler, useClass: GlobalErrorHandler }
]
```

---

## 15. テスト（Jest）

### 基本方針

- **テストフレームワークは Jest のみ**
- 単体テスト優先、インテグレーションテストは Pages レベルで実施
- `TestBed` を活用し、Angular DI をそのままテスト環境に持ち込む

### なぜ単体テスト優先なのか

層を責務で明確に分けているため、各層は独立してテストできます。Component のテストは `@Input` に値を渡して表示を確認するだけ、Reducer のテストは関数呼び出しだけです。結合度が低いので**モックが少なくて済み、テストが速く安定します**。

### Component テスト

```typescript
// なぜ data-testid を使うか：
// class や element セレクタはスタイル変更でテストが壊れる。
// data-testid は「テスト用の識別子」と明示的に分離している。
describe('UserCardComponent', () => {
  let fixture: ComponentFixture<UserCardComponent>;
  let component: UserCardComponent;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [UserCardComponent],
    }).compileComponents();

    fixture = TestBed.createComponent(UserCardComponent);
    component = fixture.componentInstance;
    component.user = mockUserCardViewModel();
    fixture.detectChanges();
  });

  it('ユーザー名が表示される', () => {
    const el = fixture.nativeElement.querySelector('[data-testid="user-name"]');
    expect(el.textContent).toContain('山田 太郎');
  });

  it('クリック時にselectedイベントが発火する', () => {
    const spy = jest.fn();
    component.selected.subscribe(spy);
    component.onSelect();
    expect(spy).toHaveBeenCalledWith(mockUserCardViewModel().id);
  });
});
```

### Store テスト（Reducer・Selector は純粋関数なのでシンプル）

```typescript
// store/user.reducer.spec.ts
// Reducer は「(state, action) => newState」の純粋関数なので
// TestBed が不要。関数を呼ぶだけでテストできる。
describe('UserReducer', () => {
  it('loadUsersSuccess でエンティティが更新されローディングが解除される', () => {
    const initialState = userAdapter.getInitialState({
      loading: true,
      error: null,
      selectedUserId: null,
    });
    const users = [mockUser()];

    const nextState = userReducer(initialState, UserActions.loadUsersSuccess({ users }));

    expect(nextState.loading).toBe(false);
    expect(selectAll(nextState)).toHaveLength(1);
    expect(nextState.error).toBeNull();
  });
});
```

### Service テスト

```typescript
describe('UserApiService', () => {
  let service: UserApiService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
    });
    service = TestBed.inject(UserApiService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  // なぜ afterEach で verify するか：
  // 期待したリクエストが発行されなかった場合や
  // 未処理のリクエストが残っている場合にテストを失敗させるため。
  afterEach(() => httpMock.verify());

  it('getUsers が GET /api/users を呼ぶ', (done) => {
    service.getUsers().subscribe((users) => {
      expect(users).toHaveLength(1);
      done();
    });

    const req = httpMock.expectOne('/api/users');
    expect(req.request.method).toBe('GET');
    req.flush([mockUserResponse()]);
  });
});
```

### テスト用モックファクトリー

```typescript
// testing/mock-factories.ts
// なぜファクトリー関数にするか：
// テストごとにオブジェクトリテラルを書くと
// 型が変わったとき修正箇所が爆発する。
// ファクトリーを1箇所に集めておくと変更が1点で済む。
export const mockUser = (override?: Partial<User>): User => ({
  id: 'user-1',
  firstName: '太郎',
  lastName: '山田',
  email: 'taro@example.com',
  profile: null,
  ...override,  // テストごとに必要な部分だけ上書きできる
});
```

---

## 16. 命名規則

### なぜ命名規則を統一するのか

ファイル名・クラス名が予測可能になることで「ユーザー管理の Service はどこ？」という問いに **Angular CLI の補完なしでも即答できる**状態になります。

| 対象 | 命名例 | ルール |
|------|--------|--------|
| Component | `UserCardComponent` | PascalCase + `Component` |
| Template Component | `UserListTemplateComponent` | PascalCase + `TemplateComponent` |
| Page Component | `UserListPageComponent` | PascalCase + `PageComponent` |
| Service | `UserApiService` | PascalCase + `Service` |
| Store State | `UserState` | PascalCase + `State` |
| Action Group | `UserActions` | PascalCase + `Actions` |
| Selector | `selectAllUsers` | `select` + camelCase |
| Effects Class | `UserEffects` | PascalCase + `Effects` |
| ファイル | `user-card.component.ts` | kebab-case |
| Observable 変数 | `users$` | `$` サフィックス |
| Subject 変数 | `destroy$` | `$` サフィックス |

`$` サフィックスはRxJSコミュニティの慣習です。変数を見た瞬間に「これはObservable、購読が必要」と判断でき、通常の変数との混在バグを防ぎます。

---

## 17. 禁止事項一覧

| 禁止 | 代替手段 | なぜ禁止か |
|------|----------|-----------|
| Standalone Component の使用 | NgModule で管理 | 現プロジェクトは NgModule ベースで統一するため。混在するとモジュール解決が複雑になる |
| `Promise` / `async-await` | RxJS `Observable` | RxJS の合成演算子（switchMap 等）が使えなくなり非同期処理の一貫性が崩れる |
| Template-driven Form (`ngModel`) | Reactive Forms | 型安全性・バリデーション管理・テスト容易性がすべて劣る |
| `*ngFor` での `trackBy` 省略 | 必ず `trackBy` 指定 | 全 DOM の再生成が起きパフォーマンスが劣化する |
| `ChangeDetectionStrategy.Default` | `OnPush` を使用 | 無駄な変更検知サイクルが全体のパフォーマンスを悪化させる |
| Component / Template での Store 直接操作 | Pages 経由で操作 | State 変更の追跡が困難になる |
| Service でのビジネスロジック | Service はインフラ層のみ | テストの複雑化・責務の分散につながる |
| `subscribe()` 内での State 変更 | Effects で処理 | Action の発火タイミングが DevTools で追えなくなる |
| 型定義の重複定義 | Store の型を `Pick` / `Partial` で継承 | 型の乖離バグが発生し、どれが正しい型かわからなくなる |
| `any` 型の使用 | 明示的な型定義 | TypeScript の恩恵（型チェック・補完）をすべて失う |
