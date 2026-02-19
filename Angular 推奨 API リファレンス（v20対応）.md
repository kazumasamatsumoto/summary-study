# Angular 推奨 API リファレンス（v20対応）
> 開発ルールと合わせて使う「どのAPIをいつ使うか」のリファレンス。  
> APIの概要・使いどころ・使ってはいけない場面をセットで記載する。

---

## 目次

### コンポーネント・テンプレート系
1. [@Input / @Output](#1-input--output)
2. [@ViewChild / @ViewChildren](#2-viewchild--viewchildren)
3. [@ContentChild / @ContentChildren](#3-contentchild--contentchildren)
4. [ChangeDetectorRef](#4-changedetectorref)
5. [ng-content / ng-container / ng-template](#5-ng-content--ng-container--ng-template)
6. [組み込みディレクティブ（*ngIf / *ngFor / *ngSwitch）](#6-組み込みディレクティブngif--ngfor--ngswitch)
7. [AsyncPipe（async パイプ）](#7-asyncpipeasync-パイプ)

### DI・サービス系
8. [inject()](#8-inject)
9. [InjectionToken](#9-injectiontoken)
10. [APP_INITIALIZER](#10-app_initializer)

### ルーティング系
11. [Router / ActivatedRoute](#11-router--activatedroute)
12. [CanActivateFn（Guard）](#12-canactivatefnguard)
13. [ResolveFn（Resolver）](#13-resolvefnresolver)

### HTTP・インターセプター系
14. [HttpClient](#14-httpclient)
15. [HttpInterceptorFn（Interceptor）](#15-httpinterceptorfninterceptor)
16. [HttpErrorResponse](#16-httperrorresponse)

### フォーム系
17. [FormBuilder / FormGroup / FormControl](#17-formbuilder--formgroup--formcontrol)
18. [Validators / カスタムバリデータ](#18-validators--カスタムバリデータ)
19. [AbstractControl](#19-abstractcontrol)

### ライフサイクル系
20. [OnInit / OnDestroy](#20-oninit--ondestroy)
21. [AfterViewInit / AfterContentInit](#21-afterviewinit--aftercontentinit)
22. [OnChanges / SimpleChanges](#22-onchanges--simplechanges)

### NgRx 系
23. [Store / inject(Store)](#23-store--injectstore)
24. [createActionGroup / createAction](#24-createactiongroup--createaction)
25. [createReducer / on](#25-createreducer--on)
26. [createSelector / createFeatureSelector](#26-createselector--createfeatureselector)
27. [createEffect / Actions / ofType](#27-createeffect--actions--oftype)
28. [EntityAdapter / EntityState](#28-entityadapter--entitystate)

### ユーティリティ系
29. [Pipe（カスタムパイプ）](#29-pipeカスタムパイプ)
30. [Directive（カスタムディレクティブ）](#30-directiveカスタムディレクティブ)
31. [ErrorHandler](#31-errorhandler)
32. [DestroyRef](#32-destroyref)

---

## コンポーネント・テンプレート系

---

### 1. @Input / @Output

**概要**

コンポーネント間のデータフローを定義するデコレータ。`@Input` は親→子へのデータ受け渡し、`@Output` は子→親へのイベント通知。

**使いどころ**

- Component・Template 間のすべてのデータ受け渡しに使う
- Store を持たない「表示専用コンポーネント」のデータ口として使う

```typescript
export class UserCardComponent {
  // v15以降: required: true で必須バインディングを明示できる
  // → 渡し忘れをコンパイル時に検出できる
  @Input({ required: true }) user!: UserCardViewModel;

  // aliasを使うと外部の属性名と内部の変数名を分けられる
  // <app-card [isActive]="flag"> → 内部では active で扱う
  @Input({ alias: 'isActive' }) active = false;

  @Output() selected = new EventEmitter<string>();
  @Output() deleted = new EventEmitter<void>();
}
```

**使ってはいけない場面**

- 親子関係にないコンポーネント間の通信には使わない → Store を使う
- 3階層以上の深いバケツリレー（prop drilling）には使わない → Store か DI を検討する

---

### 2. @ViewChild / @ViewChildren

**概要**

コンポーネントの**テンプレート内**にある子コンポーネント・DOM要素への参照を取得するデコレータ。

**使いどころ**

- サードパーティライブラリのコンポーネントに対してメソッドを直接呼び出す必要があるとき
- フォームのフォーカス制御など、DOM 操作が避けられない場合
- `ng-template` の `TemplateRef` を取得して動的に描画するとき

```typescript
export class UserFormPageComponent implements AfterViewInit {
  // 単一要素の取得
  @ViewChild('nameInput') nameInputRef!: ElementRef<HTMLInputElement>;

  // 子コンポーネントへの参照
  @ViewChild(UserListTemplateComponent) listTemplate!: UserListTemplateComponent;

  // static: true → ngOnInit で使える（テンプレートが条件分岐の外にある場合）
  // static: false（デフォルト） → AfterViewInit 以降で使える
  @ViewChild('dialog', { static: false }) dialogRef!: ElementRef;

  // 複数要素の取得（QueryList として返ってくる）
  @ViewChildren(UserCardComponent) cards!: QueryList<UserCardComponent>;

  ngAfterViewInit(): void {
    // AfterViewInit 以降でアクセスする（それ以前は undefined）
    this.nameInputRef.nativeElement.focus();
  }
}
```

**使ってはいけない場面**

- データのやり取りに使わない → `@Input` / `@Output` を使う
- ビジネスロジックの実行に使わない → Store の dispatch を使う
- `nativeElement` への直接アクセスは最小限に（SSR 環境で動かなくなる）

---

### 3. @ContentChild / @ContentChildren

**概要**

コンポーネントの `<ng-content>` でスロットに投影された子要素への参照を取得するデコレータ。`@ViewChild` との違いは「**自分のテンプレートの外から投影されたもの**を取得する」点。

**使いどころ**

- 汎用コンテナコンポーネント（カード・モーダル・タブ等）を作るとき
- `ng-content` で外から受け取った子コンポーネントのプロパティ・メソッドを参照するとき

```typescript
// 汎用タブコンポーネントの例
@Component({
  selector: 'app-tabs',
  template: `
    <ng-content select="app-tab-panel"></ng-content>
  `
})
export class TabsComponent implements AfterContentInit {
  @ContentChildren(TabPanelComponent) panels!: QueryList<TabPanelComponent>;

  ngAfterContentInit(): void {
    // 投影されたパネルが揃ってから初期化
    this.panels.first?.activate();
  }
}
```

**使ってはいけない場面**

- 通常の親子コンポーネントのデータ受け渡しには使わない → `@Input` を使う
- 機能コンポーネントに使う必要はほぼない。共有の汎用UIコンポーネントでのみ検討する

---

### 4. ChangeDetectorRef

**概要**

Angular の変更検知を手動でコントロールするサービス。`OnPush` を使用している場合に、Angular の自動検知の外側から変更を通知する手段。

**使いどころ**

- Zone の外からデータが変わる場合（サードパーティライブラリのコールバック・WebSocket 等）
- `OnPush` コンポーネントで `async` パイプを使えない場面

```typescript
export class RealtimeChartComponent implements OnInit {
  private readonly cdr = inject(ChangeDetectorRef);
  chartData: number[] = [];

  ngOnInit(): void {
    // Zone の外のコールバック（例: WebSocket）
    externalWebSocket.on('data', (data: number[]) => {
      this.chartData = data;
      // Angular に「このコンポーネントは次の検知サイクルで確認して」と伝える
      this.cdr.markForCheck();
    });
  }
}
```

| メソッド | 説明 |
|---------|------|
| `markForCheck()` | 次の検知サイクルでこのコンポーネントとその親を検知対象にする |
| `detectChanges()` | 今すぐこのコンポーネントの変更検知を実行する（使用は極力避ける） |
| `detach()` | このコンポーネントを変更検知の対象から外す（高度な最適化） |

**使ってはいけない場面**

- Store + `async` パイプで解決できる場合は使わない
- `detectChanges()` を多用しない（パフォーマンス悪化・設計の見直しサインと考える）

---

### 5. ng-content / ng-container / ng-template

**概要**

テンプレートのレイアウトと動的描画を制御する3つの組み込み要素。

**使いどころと使い分け**

```html
<!-- ① ng-content: 外から渡されたテンプレートを投影するスロット -->
<!-- 汎用カードコンポーネントの内側で使う -->
<div class="card">
  <ng-content select="[card-header]"></ng-content>  <!-- named slot -->
  <ng-content></ng-content>                          <!-- default slot -->
</div>

<!-- 使う側 -->
<app-card>
  <h2 card-header>タイトル</h2>
  <p>本文テキスト</p>
</app-card>
```

```html
<!-- ② ng-container: DOMに余分な要素を追加せずに構造ディレクティブを使うラッパー -->
<!-- div を増やすとスタイルが崩れる場面で活躍 -->
<ng-container *ngIf="isLoggedIn">
  <app-user-menu />
  <app-notification-badge />
</ng-container>

<!-- *ngFor と *ngIf を同じ要素に使いたい場合も -->
<ng-container *ngFor="let item of items">
  <li *ngIf="item.isVisible">{{ item.name }}</li>
</ng-container>
```

```html
<!-- ③ ng-template: 即時描画されないテンプレートの定義 -->
<!-- *ngIf の else 節や、動的コンポーネントのテンプレートとして使う -->
<div *ngIf="users.length > 0; else emptyState">
  <app-user-card *ngFor="let user of users" [user]="user" />
</div>
<ng-template #emptyState>
  <p>ユーザーが存在しません</p>
</ng-template>
```

**使い分けの判断基準**

| 要素 | DOMに出力 | 使い場面 |
|------|----------|---------|
| `ng-content` | 投影内容のみ | 汎用コンポーネントのスロット定義 |
| `ng-container` | なし | 余分なDOMを増やさず構造ディレクティブを使いたい |
| `ng-template` | なし（明示的に描画するまで） | 遅延描画・else節・テンプレート参照 |

---

### 6. 組み込みディレクティブ（*ngIf / *ngFor / *ngSwitch）

**概要**

テンプレートの表示制御に使うAngular標準のディレクティブ。

**使いどころ**

```html
<!-- *ngIf: 条件付き表示。else 節を使うと状態の網羅性が見えやすい -->
<app-user-detail *ngIf="user$ | async as user; else loading" [user]="user" />
<ng-template #loading><app-skeleton /></ng-template>

<!-- *ngFor: リスト描画。trackBy は必須（開発ルール参照） -->
<app-user-card
  *ngFor="let user of users; trackBy: trackById; let i = index; let last = last"
  [user]="user"
  [isLast]="last"
/>

<!-- *ngSwitch: 3つ以上の条件分岐（*ngIf の else-if 連鎖より読みやすい） -->
<ng-container [ngSwitch]="status">
  <app-loading    *ngSwitchCase="'loading'" />
  <app-error      *ngSwitchCase="'error'" />
  <app-user-list  *ngSwitchCase="'success'" [users]="users" />
  <app-empty      *ngSwitchDefault />
</ng-container>
```

**使い分け**

| ディレクティブ | 使い場面 |
|-------------|---------|
| `*ngIf` | 2択の表示切り替え |
| `*ngFor` | 配列のリスト描画（`trackBy` 必須） |
| `*ngSwitch` | 3つ以上の状態による切り替え |
| `[ngClass]` | 条件に応じた CSS クラスの付け外し |
| `[ngStyle]` | 動的なインラインスタイル（極力 CSS で解決する） |

---

### 7. AsyncPipe（async パイプ）

**概要**

`Observable` または `Promise` を購読し、最新の値をテンプレートに渡す組み込みパイプ。コンポーネントが破棄されると自動的に購読を解除する。

**使いどころ**

開発ルールで `subscribe()` の代わりに**必ず使う**こととしているもの。

```html
<!-- 基本パターン -->
<div *ngIf="users$ | async as users">
  <p>{{ users.length }} 件</p>
</div>

<!-- 複数のObservableを合成する場合は combineLatest + async -->
<!-- page.ts -->
readonly vm$ = combineLatest({
  users: this.store.select(selectAllUsers),
  isLoading: this.store.select(selectUsersLoading),
  error: this.store.select(selectUsersError),
});

<!-- page.html -->
<ng-container *ngIf="vm$ | async as vm">
  <app-loading-spinner *ngIf="vm.isLoading" />
  <app-error-message *ngIf="vm.error" [message]="vm.error" />
  <app-user-list-template
    *ngIf="!vm.isLoading && !vm.error"
    [users]="vm.users"
  />
</ng-container>
```

**`as` 構文を使うと良い理由**

`users$ | async` を `*ngIf="users$ | async as users"` と書くと、同じ `Observable` を複数箇所で使っても購読は1回で済みます。`users$ | async` を2箇所に書くと2回購読されてHTTPリクエストが2回走る場合があります（`shareReplay` で防ぐこともできますが、`as` 構文が最もシンプル）。

**使ってはいけない場面**

- `async` パイプを使うために不自然な `Observable` を作ることはしない
- 購読結果を別の処理に渡したいだけなら Effects を検討する

---

## DI・サービス系

---

### 8. inject()

**概要**

DI コンテキスト内でインスタンスを取得する関数。コンストラクタの外で使えるため、コンストラクタの引数リストを増やさずに依存を注入できる。

**使いどころ**

- コンポーネント・サービス・ガード・インターセプターのすべてのクラスで使う
- `injection context`（クラスフィールドの初期化タイミング）で呼ぶ

```typescript
@Component({ ... })
export class UserListPageComponent {
  // クラスフィールドの初期化タイミングは injection context なので inject() が使える
  private readonly store = inject(Store);
  private readonly router = inject(Router);
  private readonly activatedRoute = inject(ActivatedRoute);

  // Optional: 存在しない場合は null を返す
  private readonly optionalService = inject(OptionalService, { optional: true });

  // Self: 自分のインジェクターのみを探す（親のインジェクターを見ない）
  private readonly localService = inject(LocalService, { self: true });
}

// クラス外でも injection context 内（createEffect等）では使える
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authToken = inject(AuthTokenService).getToken();  // ここでも使える
  ...
};
```

**使ってはいけない場面**

- `ngOnInit` 等ライフサイクルメソッドの内部では使えない（injection context 外のため）
- コンストラクタの内部では使えるが、フィールド宣言で使う方が読みやすい

---

### 9. InjectionToken

**概要**

文字列・数値・設定オブジェクトなど、クラス以外の値を DI コンテナに登録するためのトークン。

**使いどころ**

- `environment.ts` の値（API URL・フィーチャーフラグ等）を DI で注入するとき
- テスト時に値を差し替えたい設定値があるとき

```typescript
// tokens/app-config.token.ts
export interface AppConfig {
  apiUrl: string;
  featureFlags: { darkMode: boolean };
}

export const APP_CONFIG = new InjectionToken<AppConfig>('APP_CONFIG');

// app.module.ts
providers: [
  {
    provide: APP_CONFIG,
    useValue: {
      apiUrl: environment.apiUrl,
      featureFlags: environment.featureFlags,
    },
  },
]

// service.ts
export class UserApiService {
  private readonly config = inject(APP_CONFIG);
  private readonly baseUrl = this.config.apiUrl + '/users';
}

// spec.ts でのテスト時の差し替え
TestBed.configureTestingModule({
  providers: [
    { provide: APP_CONFIG, useValue: { apiUrl: 'http://test-api', featureFlags: { darkMode: false } } }
  ]
});
```

---

### 10. APP_INITIALIZER

**概要**

アプリの起動時（`AppModule` のブートストラップ前）に非同期処理を実行するためのトークン。

**使いどころ**

- アプリ起動前に認証状態の確認（ローカルストレージのトークン検証等）
- サーバーから初期設定・フィーチャーフラグを取得してから画面を表示する
- 翻訳ファイルの事前ロード

```typescript
// core.module.ts
function initializeAuth(authService: AuthService): () => Observable<void> {
  // 関数を返す関数にする（遅延実行のため）
  return () => authService.checkAndRestoreSession();
}

@NgModule({
  providers: [
    {
      provide: APP_INITIALIZER,
      useFactory: initializeAuth,
      deps: [AuthService],
      multi: true,  // 複数の初期化処理を登録できる（全部完了してからアプリ起動）
    },
  ],
})
export class CoreModule {}
```

**使ってはいけない場面**

- 毎回実行するような処理には使わない（起動時のみ）
- 重い処理を入れると初期ロード時間が伸びるため最小限にする

---

## ルーティング系

---

### 11. Router / ActivatedRoute

**概要**

`Router` はプログラムによるナビゲーション、`ActivatedRoute` は現在のルート情報（パラメータ・クエリ・データ等）の取得に使う。

**使いどころ**

```typescript
// Pages のみで使用する（Component・Template では使わない）
export class UserDetailPageComponent implements OnInit {
  private readonly router = inject(Router);
  private readonly activatedRoute = inject(ActivatedRoute);
  private readonly store = inject(Store);

  ngOnInit(): void {
    // パスパラメータの取得（Observable で流れてくる）
    this.activatedRoute.paramMap.pipe(
      map((params) => params.get('id')!),
      takeUntil(this.destroy$),
    ).subscribe((id) => {
      this.store.dispatch(UserActions.loadUserById({ id }));
    });

    // クエリパラメータの取得
    this.activatedRoute.queryParamMap.pipe(
      map((params) => params.get('tab') ?? 'profile'),
    );

    // ルートに紐づいた静的データ（route の data プロパティ）
    const { title } = this.activatedRoute.snapshot.data;
  }

  // Pages のイベントハンドラーでナビゲーション
  onEdit(userId: string): void {
    this.router.navigate(['/users', userId, 'edit']);
  }

  onBack(): void {
    this.router.navigate(['..'], { relativeTo: this.activatedRoute });
  }
}
```

**使い分け: `snapshot` vs `Observable`**

| 方法 | 使い場面 |
|------|---------|
| `activatedRoute.snapshot.paramMap` | ページロード時に1回だけ取得すれば十分な場合 |
| `activatedRoute.paramMap`（Observable）| 同じコンポーネントが使い回される場合（リスト→詳細→戻る→別の詳細等）|

同じコンポーネントが再利用されるルートでは `snapshot` だとパラメータが更新されないため、Observable を使う。

---

### 12. CanActivateFn（Guard）

**概要**

ルートへのアクセス可否を制御する関数型ガード。

**使いどころ**

- 認証チェック（未ログインならログインページへ）
- 権限チェック（管理者以外はアクセス不可）
- フィーチャーフラグによる機能の出し分け

```typescript
// core/guards/auth.guard.ts
export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  return authService.isAuthenticated$.pipe(
    take(1),  // Guardは1回だけ値を評価すればよい
    map((isAuth) => {
      if (isAuth) return true;
      // UrlTree を返すと Angular がリダイレクトを処理する
      return router.createUrlTree(['/login'], {
        queryParams: { returnUrl: state.url }  // ログイン後に元のURLへ戻せる
      });
    })
  );
};

// core/guards/admin.guard.ts
export const adminGuard: CanActivateFn = () => {
  const authService = inject(AuthService);
  const router = inject(Router);

  return authService.currentUser$.pipe(
    take(1),
    map((user) => user?.role === 'admin' || router.createUrlTree(['/forbidden']))
  );
};

// routing.module.ts で使う
{
  path: 'admin',
  canActivate: [authGuard, adminGuard],  // 複数のガードを直列実行できる
  loadChildren: () => import('./features/admin/admin.module').then(m => m.AdminModule),
}
```

---

### 13. ResolveFn（Resolver）

**概要**

ルートが有効化される**前に**データを事前取得する関数型 Resolver。ページ表示前にデータを準備できる。

**使いどころ**

- 詳細ページを開く前にそのエンティティをあらかじめ取得しておきたいとき
- データがないと意味をなさないページの「ローディング状態」を排除したいとき

```typescript
// core/resolvers/user-detail.resolver.ts
export const userDetailResolver: ResolveFn<User> = (route) => {
  const userApi = inject(UserApiService);
  const router = inject(Router);
  const id = route.paramMap.get('id')!;

  return userApi.getUserById(id).pipe(
    catchError(() => {
      // データ取得失敗時はリダイレクト
      router.navigate(['/not-found']);
      return EMPTY;  // 空を返すことでナビゲーションをキャンセル
    })
  );
};

// routing.module.ts
{
  path: ':id',
  resolve: { user: userDetailResolver },
  component: UserDetailPageComponent,
}

// page.ts でデータを受け取る
ngOnInit(): void {
  const user = this.activatedRoute.snapshot.data['user'] as User;
  this.store.dispatch(UserActions.loadUserSuccess({ user }));
}
```

**Resolver vs Page での dispatch の使い分け**

| 方法 | 向いている場面 |
|------|--------------|
| Resolver | データがないとページが成立しない場合（詳細ページ等）|
| Page の `ngOnInit` で dispatch | データの有無に関わらずページを表示し、ローディング状態で待つ場合 |

---

## HTTP・インターセプター系

---

### 14. HttpClient

**概要**

Angularが提供するHTTP通信クライアント。戻り値はすべて `Observable<T>`。

**使いどころ**

Service の中でのみ使う（Pages・Component では使わない）。

```typescript
@Injectable({ providedIn: 'root' })
export class UserApiService {
  private readonly http = inject(HttpClient);

  // 基本的なGET
  getUsers(params?: { page: number; limit: number }): Observable<PaginatedResponse<User>> {
    return this.http.get<PaginatedResponse<User>>('/api/users', { params });
  }

  // POST: ボディの型を明示
  createUser(payload: CreateUserRequest): Observable<User> {
    return this.http.post<User>('/api/users', payload);
  }

  // レスポンスヘッダーも取得したい場合
  getUserWithHeaders(id: string): Observable<HttpResponse<User>> {
    return this.http.get<User>(`/api/users/${id}`, { observe: 'response' });
  }

  // 進捗を取得したい場合（ファイルアップロード等）
  uploadAvatar(file: File): Observable<HttpEvent<void>> {
    const formData = new FormData();
    formData.append('file', file);
    return this.http.post<void>('/api/upload', formData, { reportProgress: true, observe: 'events' });
  }
}
```

---

### 15. HttpInterceptorFn（Interceptor）

**概要**

すべての HTTP リクエスト・レスポンスに横断的に割り込む関数型インターセプター。

**使いどころ**

- 認証ヘッダーの付与
- ローディング状態のグローバル制御
- レスポンスエラーのグローバルハンドリング
- リクエスト・レスポンスのロギング

```typescript
// core/interceptors/auth.interceptor.ts
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthTokenService).getToken();
  const authReq = token
    ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } })
    : req;
  return next(authReq);
};

// core/interceptors/loading.interceptor.ts
export const loadingInterceptor: HttpInterceptorFn = (req, next) => {
  const loadingService = inject(GlobalLoadingService);
  loadingService.show();
  return next(req).pipe(
    finalize(() => loadingService.hide())  // 成功・失敗問わず必ず実行
  );
};

// core/interceptors/error.interceptor.ts
export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const router = inject(Router);
  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401) {
        router.navigate(['/login']);
      }
      return throwError(() => error);  // 上位の catchError に伝播させる
    })
  );
};

// app.module.ts に登録
providers: [
  provideHttpClient(withInterceptors([authInterceptor, loadingInterceptor, errorInterceptor]))
]
```

---

### 16. HttpErrorResponse

**概要**

HTTP エラーレスポンスを表すクラス。`catchError` 内で型アサーションして使う。

**使いどころ**

Effects の `catchError` 内でエラーの内容を解析する。

```typescript
catchError((error: HttpErrorResponse) => {
  // ステータスコードで処理を分岐
  const message = (() => {
    switch (error.status) {
      case 400: return error.error?.message ?? '入力内容に誤りがあります';
      case 403: return 'アクセス権限がありません';
      case 404: return 'データが見つかりません';
      case 500: return 'サーバーエラーが発生しました';
      default:  return '通信エラーが発生しました';
    }
  })();

  return of(UserActions.loadUsersFailure({ error: message }));
})
```

---

## フォーム系

---

### 17. FormBuilder / FormGroup / FormControl

**概要**

リアクティブフォームを構築するためのクラス群。`FormBuilder` はボイラープレートを減らすヘルパー。

**使いどころ**

Pages でフォームを定義し、Component に `@Input` で渡す。

```typescript
export class UserCreatePageComponent {
  private readonly fb = inject(FormBuilder);

  // Typed FormGroup（v14以降）: 型から FormControl を推論する
  readonly form = this.fb.group({
    name: this.fb.control('', {
      validators: [Validators.required, Validators.maxLength(50)],
      nonNullable: true,  // reset() した時に null でなく初期値に戻る
    }),
    email: this.fb.control('', {
      validators: [Validators.required, Validators.email],
      nonNullable: true,
    }),
    role: this.fb.control<'admin' | 'user'>('user', { nonNullable: true }),
  });

  // 型安全にアクセス
  get nameControl() { return this.form.controls.name; }

  // FormArrayを使う動的フォーム
  readonly tagsForm = this.fb.group({
    tags: this.fb.array([this.fb.control('', { nonNullable: true })]),
  });

  get tags() { return this.tagsForm.controls.tags; }

  addTag(): void {
    this.tags.push(this.fb.control('', { nonNullable: true }));
  }
}
```

---

### 18. Validators / カスタムバリデータ

**概要**

フォームのバリデーションロジックを担う純粋関数。標準の `Validators` と、カスタム実装の2種類がある。

**使いどころ**

```typescript
// 標準バリデータ
Validators.required          // 必須
Validators.email             // メール形式
Validators.minLength(8)      // 最小文字数
Validators.maxLength(100)    // 最大文字数
Validators.pattern(/^\d+$/) // 正規表現

// カスタムバリデータ（純粋関数として実装 → テストが容易）
// shared/validators/password.validator.ts
export function passwordStrengthValidator(control: AbstractControl): ValidationErrors | null {
  const value: string = control.value ?? '';
  const hasUpperCase = /[A-Z]/.test(value);
  const hasNumber = /[0-9]/.test(value);
  const hasSpecial = /[!@#$%^&*]/.test(value);

  if (!hasUpperCase || !hasNumber || !hasSpecial) {
    return { passwordStrength: { hasUpperCase, hasNumber, hasSpecial } };
  }
  return null;  // null は「エラーなし」を意味する
}

// 非同期バリデータ（APIでのユニーク確認等）
export function uniqueEmailValidator(userApi: UserApiService): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> => {
    return userApi.checkEmailExists(control.value).pipe(
      debounceTime(300),
      map((exists) => exists ? { emailTaken: true } : null),
      catchError(() => of(null))  // API エラー時はバリデーション無効
    );
  };
}

// 使い方
email: this.fb.control('', {
  validators: [Validators.required, Validators.email],
  asyncValidators: [uniqueEmailValidator(this.userApi)],
})
```

---

### 19. AbstractControl

**概要**

`FormControl`・`FormGroup`・`FormArray` の共通基底クラス。カスタムバリデータの引数型として使う。

**使いどころ**

カスタムバリデータの型定義・フォームコントロールへの汎用的なアクセス。

```typescript
// バリデータの引数は AbstractControl で受け取ることで汎用化できる
export function rangeValidator(min: number, max: number): ValidatorFn {
  return (control: AbstractControl): ValidationErrors | null => {
    const value = Number(control.value);
    if (isNaN(value)) return { range: '数値を入力してください' };
    if (value < min || value > max) return { range: `${min}〜${max}の範囲で入力してください` };
    return null;
  };
}

// テンプレートでエラーを表示する場合
// form.get('age')?.hasError('range')
// form.get('age')?.getError('range')
```

---

## ライフサイクル系

---

### 20. OnInit / OnDestroy

**概要**

`ngOnInit`：コンポーネントの初期化時（`@Input` の値が確定した後）に実行。  
`ngOnDestroy`：コンポーネントが DOM から除去される直前に実行。

**使いどころ**

```typescript
export class UserListPageComponent implements OnInit, OnDestroy {
  private readonly store = inject(Store);
  private readonly destroy$ = new Subject<void>();

  ngOnInit(): void {
    // ここで初めて @Input の値が利用できる（コンストラクタでは未確定）
    this.store.dispatch(UserActions.loadUsers());

    // ActivatedRoute のパラメータ監視もここで開始
    this.activatedRoute.paramMap.pipe(
      takeUntil(this.destroy$)
    ).subscribe(...);
  }

  ngOnDestroy(): void {
    // takeUntil のトリガー → 全購読を解除
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

**注意点**

- コンストラクタで初期化処理を書かない（`@Input` がまだ未確定）
- `ngOnDestroy` を `implements OnDestroy` なしで書いても動くが、**型チェックが効かない**ので必ず implements する

---

### 21. AfterViewInit / AfterContentInit

**概要**

`AfterViewInit`：テンプレート（View）のレンダリングが完了した後に実行。`@ViewChild` で取得した要素にアクセスできる。  
`AfterContentInit`：投影コンテンツ（`ng-content`）の初期化後に実行。`@ContentChild` で取得した要素にアクセスできる。

**使いどころ**

```typescript
export class ChartComponent implements AfterViewInit {
  @ViewChild('canvas') canvasRef!: ElementRef<HTMLCanvasElement>;

  ngAfterViewInit(): void {
    // ここまでくると canvasRef が確定している
    // サードパーティライブラリの初期化など DOM が必要な処理はここで行う
    this.chart = new Chart(this.canvasRef.nativeElement, { ... });
  }
}
```

**使ってはいけない場面**

- 通常の初期化処理は `ngOnInit` に書く
- `AfterViewInit` は「DOM への直接アクセスが必要な場合のみ」

---

### 22. OnChanges / SimpleChanges

**概要**

`@Input` に渡された値が変更されるたびに実行されるライフサイクルフック。変更前後の値を `SimpleChanges` で取得できる。

**使いどころ**

`@Input` の変更に反応して派生データを計算する場合。

```typescript
export class UserCardComponent implements OnChanges {
  @Input({ required: true }) user!: UserCardViewModel;
  fullName = '';

  ngOnChanges(changes: SimpleChanges): void {
    // user Input が変わった場合のみ処理する
    if (changes['user']) {
      const prev = changes['user'].previousValue as UserCardViewModel | undefined;
      const curr = changes['user'].currentValue as UserCardViewModel;

      // firstChange: 最初の変更（コンポーネント初期化時）かどうか
      if (!changes['user'].firstChange) {
        console.log('user changed from', prev?.id, 'to', curr.id);
      }

      this.fullName = `${curr.lastName} ${curr.firstName}`;
    }
  }
}
```

**使ってはいけない場面**

- 単純な表示用の変換は Selector か `async` パイプ + `map` で対応する（`ngOnChanges` を増やすと変更の流れが追いにくくなる）
- HTTP 通信・Store dispatch は `ngOnChanges` に書かない

---

## NgRx 系

---

### 23. Store / inject(Store)

**概要**

NgRx のグローバル状態管理コンテナ。状態の読み取り（`select`）と変更のトリガー（`dispatch`）を行う。

**使いどころ**

Pages のみで使う（Component・Template・Service では使わない）。

```typescript
export class UserListPageComponent {
  private readonly store = inject(Store);

  // 状態の読み取り（Observable として返ってくる）
  readonly users$ = this.store.select(selectAllUsers);
  readonly isLoading$ = this.store.select(selectUsersLoading);

  // 状態の変更（Action を dispatch する）
  loadUsers(): void {
    this.store.dispatch(UserActions.loadUsers());
  }

  // 現在の値をスナップショットとして取得（非推奨。どうしても必要な場合のみ）
  // Observable でなく一度だけ値が欲しい場合: take(1) を使う
  getCurrentUsers(): void {
    this.users$.pipe(take(1)).subscribe((users) => {
      console.log('現在のユーザー数:', users.length);
    });
  }
}
```

---

### 24. createActionGroup / createAction

**概要**

型安全な NgRx Action を定義する関数。`createActionGroup` は関連アクションをグループ化する。

**使いどころ**

```typescript
// 推奨: createActionGroup でグループ化（DevToolsで [User] というプレフィックスが付く）
export const UserActions = createActionGroup({
  source: 'User',
  events: {
    'Load Users': emptyProps(),
    'Load Users Success': props<{ users: User[] }>(),
    'Load Users Failure': props<{ error: string }>(),
    'Select User': props<{ userId: string }>(),
    'Clear Selection': emptyProps(),
  },
});

// 使い方: UserActions.loadUsers() / UserActions.loadUsersSuccess({ users })
// 自動的に type が '[User] Load Users' のように生成される

// 単発の場合は createAction も使える
export const globalErrorAction = createAction(
  '[App] Global Error',
  props<{ message: string }>()
);
```

---

### 25. createReducer / on

**概要**

Action に応じて State を更新する純粋関数（Reducer）を定義する。

**使いどころ**

```typescript
// store/user.reducer.ts
const initialState: UserState = userAdapter.getInitialState({
  selectedUserId: null,
  loading: false,
  error: null,
});

export const userReducer = createReducer(
  initialState,
  on(UserActions.loadUsers, (state) => ({
    ...state,
    loading: true,
    error: null,
  })),
  on(UserActions.loadUsersSuccess, (state, { users }) =>
    // EntityAdapter のメソッドで Immutable に更新
    userAdapter.setAll(users, { ...state, loading: false })
  ),
  on(UserActions.loadUsersFailure, (state, { error }) => ({
    ...state,
    loading: false,
    error,
  })),
  on(UserActions.selectUser, (state, { userId }) => ({
    ...state,
    selectedUserId: userId,
  }))
);
```

**Reducer の鉄則**

Reducer は必ず**純粋関数**（同じ引数→同じ戻り値、副作用なし）でなければならない。HTTP 通信・乱数・`Date.now()` 等は書かない → Effects に書く。

---

### 26. createSelector / createFeatureSelector

**概要**

Store から派生データを取得する関数（Selector）を定義する。**メモ化**（前回と同じ入力なら計算をスキップ）されるためパフォーマンスに優れる。

**使いどころ**

```typescript
// store/user.selectors.ts

// Feature State への入口
export const selectUserState = createFeatureSelector<UserState>('user');

// EntityAdapter のセレクター
const { selectAll, selectEntities, selectTotal } = userAdapter.getSelectors();

export const selectAllUsers = createSelector(selectUserState, selectAll);
export const selectUserEntities = createSelector(selectUserState, selectEntities);
export const selectUsersLoading = createSelector(selectUserState, (s) => s.loading);
export const selectSelectedUserId = createSelector(selectUserState, (s) => s.selectedUserId);

// 複数のセレクターを合成（両方が変わらなければ再計算しない）
export const selectSelectedUser = createSelector(
  selectUserEntities,
  selectSelectedUserId,
  (entities, selectedId) => selectedId ? entities[selectedId] : null
);

// ViewModel 変換もここで行う（Component には表示用の型だけ渡す）
export const selectUserCardViewModels = createSelector(
  selectAllUsers,
  (users): UserCardViewModel[] => users.map(toUserCardViewModel)
);
```

**メモ化の効果**

`selectUserCardViewModels` は `selectAllUsers` の結果が変わらない限り再計算しません。`async` パイプと組み合わせると、計算結果が変わらなければ `OnPush` コンポーネントの変更検知もスキップされます。

---

### 27. createEffect / Actions / ofType

**概要**

非同期の副作用（HTTP通信・タイマー等）を Action の流れとして表現する。

**使いどころ**

```typescript
@Injectable()
export class UserEffects {
  private readonly actions$ = inject(Actions);
  private readonly userApi = inject(UserApiService);
  private readonly store = inject(Store);  // 他の State を参照したい場合のみ

  // HTTP通信の Effects
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

  // Action に反応して別の Action を dispatch するだけの Effects
  // dispatch: false にすると戻り値の Action を dispatch しない
  logError$ = createEffect(() =>
    this.actions$.pipe(
      ofType(UserActions.loadUsersFailure),
      tap(({ error }) => console.error('[UserEffects]', error))
    ),
    { dispatch: false }
  );

  // Store の状態と組み合わせる場合
  loadUserDetail$ = createEffect(() =>
    this.actions$.pipe(
      ofType(UserActions.selectUser),
      withLatestFrom(this.store.select(selectUserEntities)),
      switchMap(([{ userId }, entities]) => {
        // すでに Store にあればAPIコールしない
        if (entities[userId]) return of(UserActions.selectUserComplete());
        return this.userApi.getUserById(userId).pipe(
          map((user) => UserActions.loadUserDetailSuccess({ user })),
          catchError((error) => of(UserActions.loadUserDetailFailure({ error: error.message })))
        );
      })
    )
  );
}
```

---

### 28. EntityAdapter / EntityState

**概要**

NgRx が提供するコレクション管理のユーティリティ。エンティティを正規化して管理する。

**使いどころ**

```typescript
// store/user.state.ts
export interface UserState extends EntityState<User> {
  selectedUserId: string | null;
  loading: boolean;
  error: string | null;
}

// store/user.reducer.ts
export const userAdapter = createEntityAdapter<User>({
  selectId: (user) => user.id,  // IDフィールドが id 以外の場合に指定
  sortComparer: (a, b) => a.lastName.localeCompare(b.lastName),  // ソート順（任意）
});

// EntityAdapter が提供する操作（すべて Immutable に新しい State を返す）
userAdapter.addOne(user, state)          // 1件追加
userAdapter.addMany(users, state)        // 複数追加
userAdapter.setAll(users, state)         // 全件置き換え
userAdapter.setOne(user, state)          // 1件セット（なければ追加、あれば置き換え）
userAdapter.upsertOne(user, state)       // 差分更新（部分更新に対応）
userAdapter.upsertMany(users, state)     // 複数差分更新
userAdapter.updateOne({ id, changes }, state)  // 特定フィールドのみ更新
userAdapter.removeOne(id, state)         // 1件削除
userAdapter.removeMany(ids, state)       // 複数削除
userAdapter.removeAll(state)             // 全件削除
```

---

## ユーティリティ系

---

### 29. Pipe（カスタムパイプ）

**概要**

テンプレート内での値変換を宣言的に行う仕組み。Angular の組み込みパイプ（`date`・`currency`・`number`・`json`・`slice` 等）に加えて、プロジェクト固有のパイプを作れる。

**使いどころ**

- 表示フォーマット変換をテンプレート内に閉じたい場合
- 複数箇所で同じ変換ロジックを使い回す場合

```typescript
// shared/pipes/status-label.pipe.ts
@Pipe({ name: 'statusLabel' })
export class StatusLabelPipe implements PipeTransform {
  // pure: true（デフォルト）: 入力値が変わらなければキャッシュを返す → 高パフォーマンス
  // pure: false: レンダリングのたびに実行される → 配列・オブジェクトの変化検知が必要な場合
  transform(status: UserStatus): string {
    const labels: Record<UserStatus, string> = {
      active: '有効',
      inactive: '無効',
      suspended: '停止中',
    };
    return labels[status] ?? '不明';
  }
}

// テンプレートでの使用
// {{ user.status | statusLabel }}
// {{ user.createdAt | date: 'yyyy/MM/dd HH:mm' }}
// {{ price | currency: 'JPY' : 'symbol' : '1.0-0' }}
```

**使ってはいけない場面**

- 副作用（HTTP通信・Store dispatch）をパイプ内に書かない
- `pure: false` は意図的な場合のみ（パフォーマンス影響を理解した上で）

---

### 30. Directive（カスタムディレクティブ）

**概要**

DOM 要素の振る舞い・外観を制御する再利用可能な部品。コンポーネントとの違いは「テンプレートを持たない」点。

**使いどころ**

- DOM の振る舞い（ホバー・フォーカス・入力マスク等）を複数の要素で共有したい
- `@HostListener` でイベントを拾って処理したい
- 要素への属性・クラス・スタイルの動的制御

```typescript
// shared/directives/auto-focus.directive.ts
@Directive({ selector: '[appAutoFocus]' })
export class AutoFocusDirective implements AfterViewInit {
  private readonly el = inject(ElementRef);

  ngAfterViewInit(): void {
    this.el.nativeElement.focus();
  }
}

// shared/directives/click-outside.directive.ts
@Directive({ selector: '[appClickOutside]' })
export class ClickOutsideDirective {
  @Output() clickOutside = new EventEmitter<void>();
  private readonly el = inject(ElementRef);

  @HostListener('document:click', ['$event.target'])
  onDocumentClick(target: HTMLElement): void {
    if (!this.el.nativeElement.contains(target)) {
      this.clickOutside.emit();
    }
  }
}

// テンプレートでの使用
// <input appAutoFocus />
// <app-dropdown (clickOutside)="closeDropdown()" appClickOutside />
```

---

### 31. ErrorHandler

**概要**

アプリ全体で捕捉されなかったエラーの最終受け皿。`try-catch` や `catchError` で処理されなかったエラーはすべてここに届く。

**使いどころ**

```typescript
// core/handlers/global-error.handler.ts
@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  private readonly store = inject(Store);

  handleError(error: unknown): void {
    // エラーの種類に応じて処理を分ける
    if (error instanceof HttpErrorResponse) {
      // HTTPエラーはInterceptorで処理済みのはずなのでここには来ないはず
      console.error('Unhandled HTTP error:', error);
    } else if (error instanceof Error) {
      console.error('Runtime error:', error.message, error.stack);
    } else {
      console.error('Unknown error:', error);
    }

    this.store.dispatch(AppActions.globalError({
      message: 'システムエラーが発生しました。画面を更新してください。'
    }));
  }
}

// core.module.ts
providers: [
  { provide: ErrorHandler, useClass: GlobalErrorHandler }
]
```

---

### 32. DestroyRef

**概要**

Angular v16 以降で使える、コンポーネント破棄時のクリーンアップを登録する仕組み。`ngOnDestroy` と `Subject` を使った購読解除パターンの代替。

**使いどころ**

`OnDestroy` の実装と `destroy$` の管理を簡略化したい場合。

```typescript
import { DestroyRef, inject } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

export class UserListPageComponent implements OnInit {
  // DestroyRef を inject するだけで OnDestroy の実装が不要になる
  private readonly destroyRef = inject(DestroyRef);

  ngOnInit(): void {
    this.activatedRoute.paramMap.pipe(
      // takeUntilDestroyed にDestroyRefを渡すと自動解除
      takeUntilDestroyed(this.destroyRef)
    ).subscribe((params) => {
      this.store.dispatch(UserActions.selectUser({ userId: params.get('id')! }));
    });
  }
}

// クラスフィールドで直接使う場合（injection context 内なので引数不要）
export class AnotherComponent {
  readonly data$ = this.someService.getData().pipe(
    takeUntilDestroyed()  // inject コンテキスト内ではコンポーネントの DestroyRef を自動参照
  );
}
```

**使い分け：`DestroyRef` vs `Subject + ngOnDestroy`**

| 方法 | 推奨場面 |
|------|---------|
| `takeUntilDestroyed(this.destroyRef)` | v16以降のプロジェクト、新規実装 |
| `takeUntil(this.destroy$)` + `ngOnDestroy` | v15以前との互換性が必要な場合、既存コードとの統一 |

---

## まとめ：API 選択チートシート

| やりたいこと | 使う API |
|-------------|---------|
| 親→子のデータ受け渡し | `@Input` |
| 子→親のイベント通知 | `@Output` + `EventEmitter` |
| テンプレート内の DOM 参照 | `@ViewChild` / `@ViewChildren` |
| 投影コンテンツの参照 | `@ContentChild` / `@ContentChildren` |
| `Observable` をテンプレートで購読 | `async` パイプ |
| 条件付き表示 | `*ngIf` |
| リスト描画 | `*ngFor` + `trackBy` |
| 3択以上の切り替え | `*ngSwitch` |
| 余分なDOMなしで構造ディレクティブ | `ng-container` |
| DI でインスタンス取得 | `inject()` |
| 設定値・プリミティブな値のDI | `InjectionToken` |
| 起動時の初期化処理 | `APP_INITIALIZER` |
| ナビゲーション | `Router.navigate()` |
| ルートパラメータ取得 | `ActivatedRoute` |
| ルートアクセス制御 | `CanActivateFn` |
| ルート前のデータ取得 | `ResolveFn` |
| HTTP通信 | `HttpClient`（Service 内のみ） |
| 全リクエストへの横断処理 | `HttpInterceptorFn` |
| フォーム構築 | `FormBuilder` + Typed FormGroup |
| バリデーション | `Validators` / カスタム `ValidatorFn` |
| コンポーネント初期化 | `ngOnInit` |
| コンポーネント破棄 | `ngOnDestroy` / `DestroyRef` |
| DOM 操作が必要な初期化 | `ngAfterViewInit` |
| Input 変更の検知 | `ngOnChanges` |
| 状態の読み取り（NgRx） | `Store.select(selector)` |
| 状態の変更トリガー（NgRx） | `Store.dispatch(action)` |
| 非同期副作用（NgRx） | `createEffect` + `ofType` |
| リスト状態管理（NgRx） | `EntityAdapter` / `EntityState` |
| 表示値の変換 | カスタム `Pipe` |
| DOM の振る舞い制御 | カスタム `Directive` |
| Zone外からの変更通知 | `ChangeDetectorRef.markForCheck()` |
| 予期しないエラーの最終捕捉 | `ErrorHandler` |
