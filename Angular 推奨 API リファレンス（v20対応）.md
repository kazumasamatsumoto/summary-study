# Angular 推奨 API リファレンス（v20対応）
> **このドキュメントの読み方**  
> 現在のプロジェクトは NgModule・デコレータベースの構成で動いている。  
> v17以降、Angular は Standalone・Signal API・新制御フロー構文を推奨するようになった。  
> 本ドキュメントでは **「現在使っている書き方」と「今後移行していく書き方」を並べて記載する**。  
> 新規ファイルを作る際はなるべく「今後」の書き方を採用し、段階的に移行していく方針をとる。
>
> **重要：Signal API（`input()` / `output()` / `toSignal()` 等）は NgModule 管理下のコンポーネントでも使用できる。NgModule → Standalone の移行とは独立した別軸の作業として進められる。**

---

## 移行方針の全体像

Signal API と Standalone は**独立した2つの移行軸**として捉える。Signal API は NgModule 管理下のコンポーネントでも使えるため、Standalone 化を待たずに先行して進められる。

| カテゴリ | 現在（実績ある書き方） | 今後（v17〜v20 推奨） | NgModule のまま適用可 | 移行タイミング |
|---------|-------------------|-------------------|:-----------------:|--------------|
| テンプレート制御 | `*ngIf` / `*ngFor` / `*ngSwitch` | `@if` / `@for` / `@switch` | ✅ | 新規テンプレートから即適用可 |
| 購読解除 | `Subject` + `takeUntil` | `takeUntilDestroyed()` | ✅ | 新規ファイルから即適用可 |
| Input / Output | `@Input()` / `@Output()` | `input()` / `output()` 関数 | ✅ | 新規コンポーネントから順次 |
| DOM参照 | `@ViewChild` / `@ContentChild` | `viewChild()` / `contentChild()` 関数 | ✅ | 新規コンポーネントから順次 |
| 状態管理 | `async` パイプ + Store | `toSignal()` + Store | ✅ | 新規ファイルから順次 |
| コンポーネント管理 | NgModule + `declarations` | Standalone Component | ➖（これ自体が移行作業） | 別軸で計画的に進める |

> **判断の原則：既存ファイルを触る場合は現在の書き方を維持する。新規ファイルを作る場合は「今後」の書き方を検討する。**

---

## 目次

### コンポーネント・テンプレート系
1. [モジュール管理（NgModule → Standalone）](#1-モジュール管理ngmodule--standalone)
2. [@Input / @Output → input() / output()](#2-input--output--input--output)
3. [@ViewChild / @ViewChildren → viewChild() / viewChildren()](#3-viewchild--viewchildren--viewchild--viewchildren)
4. [@ContentChild / @ContentChildren → contentChild()](#4-contentchild--contentchildren--contentchild)
5. [テンプレート制御構文（*ngIf → @if / *ngFor → @for）](#5-テンプレート制御構文ngif--if--ngfor--for)
6. [AsyncPipe / toSignal()](#6-asyncpipe--tosignal)
7. [ng-content / ng-container / ng-template](#7-ng-content--ng-container--ng-template)
8. [ChangeDetectorRef](#8-changedetectorref)

### DI・サービス系
9. [inject()](#9-inject)
10. [InjectionToken](#10-injectiontoken)
11. [APP_INITIALIZER](#11-app_initializer)

### ルーティング系
12. [Router / ActivatedRoute](#12-router--activatedroute)
13. [CanActivateFn（Guard）](#13-canactivatefnguard)
14. [ResolveFn（Resolver）](#14-resolvefnresolver)

### HTTP系
15. [HttpClient](#15-httpclient)
16. [HttpInterceptorFn（Interceptor）](#16-httpinterceptorfninterceptor)
17. [HttpErrorResponse](#17-httperrorresponse)

### フォーム系
18. [FormBuilder / FormGroup / FormControl](#18-formbuilder--formgroup--formcontrol)
19. [Validators / カスタムバリデータ](#19-validators--カスタムバリデータ)

### ライフサイクル系
20. [OnInit / OnDestroy / DestroyRef](#20-oninit--ondestroy--destroyref)
21. [AfterViewInit / AfterContentInit](#21-afterviewinit--aftercontentinit)
22. [OnChanges / SimpleChanges](#22-onchanges--simplechanges)

### NgRx系
23. [Store / dispatch / select](#23-store--dispatch--select)
24. [createActionGroup / createAction](#24-createactiongroup--createaction)
25. [createReducer / on](#25-createreducer--on)
26. [createSelector / createFeatureSelector](#26-createselector--createfeatureselector)
27. [createEffect / Actions / ofType](#27-createeffect--actions--oftype)
28. [EntityAdapter / EntityState](#28-entityadapter--entitystate)

### ユーティリティ系
29. [Pipe（カスタムパイプ）](#29-pipeカスタムパイプ)
30. [Directive（カスタムディレクティブ）](#30-directiveカスタムディレクティブ)
31. [ErrorHandler](#31-errorhandler)

---

## コンポーネント・テンプレート系

---

### 1. モジュール管理（NgModule → Standalone）

**概要**

現在は NgModule でコンポーネントを `declarations` に登録して管理している。v15以降、NgModule 不要の Standalone コンポーネントが正式サポートされ、v17以降は CLI のデフォルトになっている。

---

#### 現在の書き方（NgModule）

```typescript
// feature.module.ts
@NgModule({
  declarations: [
    UserListPageComponent,
    UserListTemplateComponent,
    UserCardComponent,
  ],
  imports: [
    CommonModule,
    SharedModule,
    ReactiveFormsModule,
    StoreModule.forFeature(userFeatureKey, userReducer),
    EffectsModule.forFeature([UserEffects]),
    UserRoutingModule,
  ],
})
export class UserModule {}
```

```typescript
// user-card.component.ts（NgModule管理下のコンポーネント）
@Component({
  selector: 'app-user-card',
  templateUrl: './user-card.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class UserCardComponent {
  @Input({ required: true }) user!: UserCardViewModel;
}
// → NgModule の declarations に登録しないと使えない
```

---

#### 今後の書き方（Standalone）

```typescript
// user-card.component.ts（Standalone）
@Component({
  selector: 'app-user-card',
  templateUrl: './user-card.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
  standalone: true,                    // これだけで NgModule 不要
  imports: [CommonModule],             // 必要なものをコンポーネント自身が宣言
})
export class UserCardComponent {
  user = input.required<UserCardViewModel>();
}
```

```typescript
// feature の routing だけで Standalone を束ねられる
// feature-routing.module.ts
const routes: Routes = [
  {
    path: '',
    loadComponent: () =>
      import('./pages/user-list/user-list.page').then(m => m.UserListPageComponent),
  }
];
```

---

#### 移行戦略

NgModule と Standalone は**共存できる**。段階的に移行可能。

```typescript
// NgModule から Standalone コンポーネントを使う
@NgModule({
  imports: [
    UserCardComponent,  // Standalone コンポーネントは declarations ではなく imports に書く
  ],
})
export class UserModule {}
```

移行の推奨順序は次の通り。

1. 末端の表示専用 Component（`UserCardComponent` 等）から Standalone 化する
2. Template Component を Standalone 化する
3. Page Component と Routing を Standalone に切り替える
4. Module ファイル自体を削除する

---

### 2. @Input / @Output → input() / output()

**概要**

`@Input()` / `@Output()` デコレータは v17.1 で Signal ベースの `input()` / `output()` 関数に置き換わった。Signal API は変更を自動追跡できるため、`ChangeDetectorRef` を手動で呼ぶ必要がなくなる。

---

#### 現在の書き方

```typescript
@Component({ ... })
export class UserCardComponent {
  @Input({ required: true }) user!: UserCardViewModel;
  @Input() isHighlighted = false;
  @Output() selected = new EventEmitter<string>();

  onSelect(): void {
    this.selected.emit(this.user.id);
  }
}
```

---

#### 今後の書き方（Signal Input / Output）

```typescript
import { input, output, computed } from '@angular/core';

@Component({ ... })
export class UserCardComponent {
  // input() は Signal を返す。値を取り出すには () が必要
  user = input.required<UserCardViewModel>();
  isHighlighted = input(false);  // デフォルト値あり

  selected = output<string>();

  // Signal の派生値は computed() で定義できる
  // user() の値が変わると自動で再計算される
  fullName = computed(() => `${this.user().lastName} ${this.user().firstName}`);

  onSelect(): void {
    this.selected.emit(this.user().id);  // Signal なので () で値を取り出す
  }
}
```

```html
<!-- テンプレート側の書き方は変わらない -->
<app-user-card [user]="user" (selected)="onSelect($event)" />
```

---

#### 使い分けの判断基準

| 状況 | 書き方 |
|------|--------|
| 既存ファイルを修正する | `@Input()` / `@Output()` のまま維持 |
| 新規コンポーネントを作る | `input()` / `output()` を使う（NgModule 管理下でも使用可） |
| 1ファイル内で混在させる | 避ける（デコレータか Signal かどちらかに統一） |

> **注意：** `input()` は Signal なのでテンプレート外で値を読む場合は必ず `this.user()` と `()` をつける。`this.user` だと Signal オブジェクト自体を参照してしまう。

---

### 3. @ViewChild / @ViewChildren → viewChild() / viewChildren()

**概要**

Signal Input と同様に、DOM参照も Signal ベースの関数に移行している。

---

#### 現在の書き方

```typescript
export class ChartComponent implements AfterViewInit {
  @ViewChild('canvas') canvasRef!: ElementRef<HTMLCanvasElement>;
  @ViewChildren(UserCardComponent) cards!: QueryList<UserCardComponent>;

  ngAfterViewInit(): void {
    // AfterViewInit 以降でないと参照が確定しない
    this.initChart(this.canvasRef.nativeElement);
  }
}
```

---

#### 今後の書き方（Signal ViewChild）

```typescript
import { viewChild, viewChildren } from '@angular/core';

export class ChartComponent implements AfterViewInit {
  // Signal として宣言。required は必ず存在することを保証する
  canvasRef = viewChild.required<ElementRef<HTMLCanvasElement>>('canvas');
  cards = viewChildren(UserCardComponent);

  ngAfterViewInit(): void {
    // Signal なので () で値を取り出す
    this.initChart(this.canvasRef().nativeElement);
  }
}
```

---

### 4. @ContentChild / @ContentChildren → contentChild()

**概要**

投影コンテンツの参照も Signal ベースに移行している。用途（汎用コンテナの ng-content スロット参照）は変わらない。

---

#### 現在の書き方

```typescript
export class TabsComponent implements AfterContentInit {
  @ContentChildren(TabPanelComponent) panels!: QueryList<TabPanelComponent>;

  ngAfterContentInit(): void {
    this.panels.first?.activate();
  }
}
```

---

#### 今後の書き方

```typescript
import { contentChildren } from '@angular/core';

export class TabsComponent implements AfterContentInit {
  panels = contentChildren(TabPanelComponent);

  ngAfterContentInit(): void {
    this.panels()[0]?.activate();  // Signal なので () で配列を取り出す
  }
}
```

---

### 5. テンプレート制御構文（*ngIf → @if / *ngFor → @for）

**概要**

v17 で新しいビルトイン制御フロー構文が導入された。`*ngIf` / `*ngFor` / `*ngSwitch` を使ったディレクティブベースの構文から、`@if` / `@for` / `@switch` という言語構文に移行する。

新構文は `CommonModule` のインポートが不要で、型推論の精度が高く、`@for` では `track` が必須なためパフォーマンス上の書き忘れも防げる。

---

#### 現在の書き方

```html
<!-- 条件分岐 -->
<div *ngIf="isLoading; else content">
  <app-loading-spinner />
</div>
<ng-template #content>
  <app-user-list [users]="users" />
</ng-template>

<!-- リスト描画 -->
<app-user-card
  *ngFor="let user of users; trackBy: trackById; let i = index; let last = last"
  [user]="user"
/>

<!-- 複数分岐 -->
<ng-container [ngSwitch]="status">
  <app-loading *ngSwitchCase="'loading'" />
  <app-error   *ngSwitchCase="'error'" />
  <app-list    *ngSwitchDefault />
</ng-container>
```

---

#### 今後の書き方（v17〜 ビルトイン制御フロー）

```html
<!-- 条件分岐：as でローカル変数を作れる -->
@if (isLoading) {
  <app-loading-spinner />
} @else if (error) {
  <app-error-message [message]="error" />
} @else {
  <app-user-list [users]="users" />
}

<!-- async パイプと組み合わせた典型パターン -->
@if (users$ | async; as users) {
  <app-user-list [users]="users" />
}

<!-- リスト描画：track が必須（関数ではなくプロパティ式で書く） -->
@for (user of users; track user.id) {
  <app-user-card [user]="user" />
} @empty {
  <!-- データが空のときの表示（ngFor にはなかった機能） -->
  <p>ユーザーが存在しません</p>
}

<!-- index や last 等のローカル変数 -->
@for (user of users; track user.id; let i = $index; let last = $last) {
  <app-user-card [user]="user" [isLast]="last" />
}

<!-- 複数分岐 -->
@switch (status) {
  @case ('loading') { <app-loading /> }
  @case ('error')   { <app-error /> }
  @default          { <app-user-list /> }
}
```

---

#### `trackBy` 関数が不要になった理由

旧構文では `trackBy` に**関数**を渡す必要があり、コンポーネントに関数を定義する手間があった。新構文の `track` は**プロパティ式**を直接書けるため、関数定義が不要になった。

```typescript
// 旧: コンポーネントに関数を定義する必要があった
trackById(_index: number, item: { id: string }): string {
  return item.id;
}
```

```html
<!-- 旧 -->
<li *ngFor="let user of users; trackBy: trackById">

<!-- 新: 式で直接書ける。関数定義が不要 -->
@for (user of users; track user.id) {
```

---

#### 移行の進め方

新構文と旧構文は**1つのプロジェクト内で混在できる**。ファイルを新規作成するときや、既存ファイルを大幅に修正するタイミングで新構文に切り替えていく。

Angular CLI には自動マイグレーションコマンドも用意されている。

```bash
# 旧構文を新構文に自動変換するマイグレーション
ng generate @angular/core:control-flow
```

---

### 6. AsyncPipe / toSignal()

**概要**

`async` パイプはテンプレートで `Observable` を購読してきた主役だが、Signal ベースの開発では `toSignal()` に移行していく。ただし NgRx との統合では当面 `async` パイプの運用が現実的。

---

#### 現在の書き方（async パイプ）

```typescript
// page.ts
readonly users$ = this.store.select(selectAllUsers);
readonly isLoading$ = this.store.select(selectUsersLoading);

// 複数の Observable をまとめる
readonly vm$ = combineLatest({
  users: this.store.select(selectAllUsers),
  isLoading: this.store.select(selectUsersLoading),
  error: this.store.select(selectUsersError),
});
```

```html
<!-- page.html -->
@if (vm$ | async; as vm) {
  @if (vm.isLoading) { <app-loading /> }
  @else {
    <app-user-list [users]="vm.users" />
  }
}
```

---

#### 今後の書き方（toSignal）

`Observable` を Signal に変換する。変換後はテンプレートで `()` を使って値を取り出せる。`async` パイプが不要になる。

```typescript
import { toSignal } from '@angular/core/rxjs-interop';

export class UserListPageComponent {
  private readonly store = inject(Store);

  // toSignal で Observable を Signal に変換
  // initialValue: コンポーネント初期化時の初期値（型安全のために指定推奨）
  readonly users = toSignal(this.store.select(selectAllUsers), { initialValue: [] });
  readonly isLoading = toSignal(this.store.select(selectUsersLoading), { initialValue: false });

  // computed で派生値を作れる
  readonly userCount = computed(() => this.users().length);
}
```

```html
<!-- async パイプが不要になる -->
@if (isLoading()) {
  <app-loading />
} @else {
  <app-user-list [users]="users()" />
}
<p>{{ userCount() }} 件</p>
```

---

#### 移行の判断基準

| 状況 | 推奨 |
|------|------|
| 既存ファイルを修正する | そのまま `async` パイプを使う |
| 新規ファイルを作る | `toSignal()` を使う（NgModule 管理下でも使用可） |
| `combineLatest` で複数 Observable を合成している | `toSignal()` + `computed()` に移行すると読みやすくなる |
| Effects / Store の Observable をテンプレートで使う | `toSignal()` で Signal に変換してから渡す |

> **現実的な移行ゴール：** まず新制御フロー構文（`@if` / `@for`）と `DestroyRef` を新規ファイルから導入する。次に Signal Input（`input()` / `output()`）と `toSignal()` を新規コンポーネントで採用する。いずれも NgModule のまま進められるため Standalone 化とは切り離して計画する。

---

### 7. ng-content / ng-container / ng-template

**概要**

この3つは v20 でも書き方は変わらない。汎用コンポーネントを作るときに使う。

```html
<!-- ng-content: 外から渡されたコンテンツを投影するスロット -->
<!-- 汎用カードコンポーネントの内側で定義する -->
<div class="card">
  <ng-content select="[card-header]"></ng-content>
  <ng-content></ng-content>
</div>

<!-- 使う側 -->
<app-card>
  <h2 card-header>タイトル</h2>
  <p>本文テキスト</p>
</app-card>
```

```html
<!-- ng-container: DOMに要素を追加せずディレクティブを適用するラッパー -->
<!-- 新構文では @if / @for で代替できる場面が多い -->
<ng-container *ngTemplateOutlet="myTemplate; context: { $implicit: user }">
</ng-container>
```

```html
<!-- ng-template: 即時描画されないテンプレートの定義 -->
<!-- 旧構文の else 節で使っていたが、新構文では @else で代替できる -->
<!-- ngTemplateOutlet での動的描画や ViewContainerRef での操作では引き続き使う -->
<ng-template #loadingTpl>
  <app-loading-spinner />
</ng-template>
```

---

### 8. ChangeDetectorRef

**概要**

`OnPush` 環境で Zone の外から変更が来たときに手動で変更検知をトリガーする。Signal ベースに移行すると Signal の値更新が自動的に検知されるため、`ChangeDetectorRef` が不要になるケースが増える。

---

#### 現在の書き方

```typescript
export class RealtimeComponent {
  private readonly cdr = inject(ChangeDetectorRef);
  data: number[] = [];

  ngOnInit(): void {
    // Zone の外のイベント（WebSocket等）
    socket.on('update', (data: number[]) => {
      this.data = data;
      this.cdr.markForCheck();  // 変更を Angular に通知
    });
  }
}
```

---

#### 今後の書き方（Signal で ChangeDetectorRef が不要になる）

```typescript
import { signal } from '@angular/core';

export class RealtimeComponent {
  // Signal は値の更新を自動で変更検知に伝えるため markForCheck() が不要
  readonly data = signal<number[]>([]);

  ngOnInit(): void {
    socket.on('update', (incoming: number[]) => {
      this.data.set(incoming);  // set() で更新するだけでテンプレートに反映される
    });
  }
}
```

`markForCheck()` は引き続き NgModule + async パイプ環境では必要。完全に Signal ベースに移行した後は不要になる。

---

## DI・サービス系

---

### 9. inject()

**概要**

v14 から使える DI の新しいスタイル。コンストラクタに引数を並べる代わりに、フィールド定義の場所で直接インジェクションできる。書き方は v20 でも変わらない。

```typescript
// 現在 & 今後（書き方は変わらない）
@Component({ ... })
export class UserListPageComponent {
  private readonly store = inject(Store);
  private readonly router = inject(Router);
  private readonly activatedRoute = inject(ActivatedRoute);
}

// 関数型インターセプター・ガードでも inject() が使える
export const authGuard: CanActivateFn = () => {
  const authService = inject(AuthService);
  return authService.isAuthenticated$;
};
```

コンストラクタインジェクションは旧来の書き方として動くが、新規ファイルでは `inject()` を統一して使う。

---

### 10. InjectionToken

**概要**

文字列・数値・設定オブジェクト等のプリミティブ値を DI コンテナに登録するトークン。書き方は v20 でも変わらない。

```typescript
// tokens/app-config.token.ts
export interface AppConfig {
  apiUrl: string;
  featureFlags: { newDashboard: boolean };
}

export const APP_CONFIG = new InjectionToken<AppConfig>('APP_CONFIG');

// app.module.ts（現在）または provideAppConfig 関数（今後）でも登録できる
providers: [
  {
    provide: APP_CONFIG,
    useValue: { apiUrl: environment.apiUrl, featureFlags: environment.featureFlags },
  },
]

// service.ts での使い方
export class UserApiService {
  private readonly config = inject(APP_CONFIG);
  private readonly baseUrl = `${this.config.apiUrl}/users`;
}
```

---

### 11. APP_INITIALIZER

**概要**

アプリ起動前に必ず実行したい非同期処理（認証状態の復元・設定値の取得等）を登録する。書き方は v20 でも変わらない。

```typescript
// core.module.ts
function initializeApp(authService: AuthService): () => Observable<void> {
  // 関数を返す関数にする（ファクトリー）
  return () => authService.restoreSession();
}

@NgModule({
  providers: [
    {
      provide: APP_INITIALIZER,
      useFactory: initializeApp,
      deps: [AuthService],
      multi: true,  // 複数登録できる。全部完了するまでアプリが起動しない
    },
  ],
})
export class CoreModule {}
```

重い処理を入れると初期表示が遅くなるため、絶対に必要な処理だけに絞る。

---

## ルーティング系

---

### 12. Router / ActivatedRoute

**概要**

`Router` によるプログラムナビゲーションと `ActivatedRoute` によるルートパラメータ取得。Pages のみで使う。書き方は v20 でも変わらない。

```typescript
export class UserDetailPageComponent implements OnInit {
  private readonly router = inject(Router);
  private readonly activatedRoute = inject(ActivatedRoute);
  private readonly store = inject(Store);
  private readonly destroyRef = inject(DestroyRef);  // 今後: ngOnDestroy 不要に

  ngOnInit(): void {
    this.activatedRoute.paramMap.pipe(
      map((params) => params.get('id')!),
      takeUntilDestroyed(this.destroyRef),  // 今後の書き方
    ).subscribe((id) => {
      this.store.dispatch(UserActions.loadUserById({ id }));
    });
  }

  onEdit(userId: string): void {
    this.router.navigate(['/users', userId, 'edit']);
  }
}
```

**`snapshot` vs `Observable` の選び方**

| 方法 | 使い場面 |
|------|---------|
| `activatedRoute.snapshot.paramMap` | ページロード時に1回取得すれば十分な場合 |
| `activatedRoute.paramMap`（Observable）| 同じコンポーネントがパラメータ違いで再利用される場合 |

---

### 13. CanActivateFn（Guard）

**概要**

ルートアクセス制御。クラスベースの `CanActivate` は v15 以降非推奨。現在も今後も関数型で書く。

```typescript
// core/guards/auth.guard.ts
export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  return authService.isAuthenticated$.pipe(
    take(1),
    map((isAuth) =>
      isAuth || router.createUrlTree(['/login'], { queryParams: { returnUrl: state.url } })
    )
  );
};

// routing.module.ts
{
  path: 'admin',
  canActivate: [authGuard, adminGuard],
  loadChildren: () => import('./features/admin/admin.module').then(m => m.AdminModule),
}
```

---

### 14. ResolveFn（Resolver）

**概要**

ページ表示前にデータを事前取得する。クラスベースの `Resolve` は v15 以降非推奨。現在も今後も関数型で書く。

```typescript
// core/resolvers/user-detail.resolver.ts
export const userDetailResolver: ResolveFn<User> = (route) => {
  const userApi = inject(UserApiService);
  const router = inject(Router);
  const id = route.paramMap.get('id')!;

  return userApi.getUserById(id).pipe(
    catchError(() => {
      router.navigate(['/not-found']);
      return EMPTY;
    })
  );
};

// page.ts でデータを受け取る
ngOnInit(): void {
  const user = this.activatedRoute.snapshot.data['user'] as User;
  this.store.dispatch(UserActions.setUser({ user }));
}
```

---

## HTTP系

---

### 15. HttpClient

**概要**

Service 内のみで使う。戻り値は常に `Observable<T>`。書き方は v20 でも変わらない。

```typescript
@Injectable({ providedIn: 'root' })
export class UserApiService {
  private readonly http = inject(HttpClient);

  getUsers(params?: { page: number; limit: number }): Observable<PaginatedResponse<User>> {
    return this.http.get<PaginatedResponse<User>>('/api/users', { params });
  }

  createUser(payload: CreateUserRequest): Observable<User> {
    return this.http.post<User>('/api/users', payload);
  }

  updateUser(id: string, changes: Partial<User>): Observable<User> {
    return this.http.patch<User>(`/api/users/${id}`, changes);
  }

  deleteUser(id: string): Observable<void> {
    return this.http.delete<void>(`/api/users/${id}`);
  }
}
```

---

### 16. HttpInterceptorFn（Interceptor）

**概要**

クラスベースの `HttpInterceptor` は v15 以降非推奨。現在も今後も関数型（`HttpInterceptorFn`）で書く。認証・ローディング・エラーの横断処理を一元管理する。

```typescript
// core/interceptors/auth.interceptor.ts
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthTokenService).getToken();
  const authReq = token
    ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } })
    : req;
  return next(authReq);
};

// core/interceptors/error.interceptor.ts
export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const router = inject(Router);
  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401) router.navigate(['/login']);
      return throwError(() => error);
    })
  );
};

// app.module.ts に登録
providers: [
  provideHttpClient(withInterceptors([authInterceptor, errorInterceptor]))
]
```

---

### 17. HttpErrorResponse

**概要**

HTTP エラーを表すクラス。Effects の `catchError` 内でステータスコードに応じた処理をする。

```typescript
catchError((error: HttpErrorResponse) => {
  const message = (() => {
    switch (error.status) {
      case 400: return error.error?.message ?? '入力内容に誤りがあります';
      case 401: return '認証が必要です';
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

### 18. FormBuilder / FormGroup / FormControl

**概要**

Reactive Forms は v20 でも変わらない。v14 以降の Typed Forms（型安全フォーム）を標準として使う。Pages でフォームを定義して Component に `@Input` で渡す。

```typescript
export class UserCreatePageComponent {
  private readonly fb = inject(FormBuilder);
  private readonly store = inject(Store);

  // Typed FormGroup: nonNullable: true で reset() 時も null にならない
  readonly form = this.fb.group({
    firstName: this.fb.control('', {
      validators: [Validators.required, Validators.maxLength(50)],
      nonNullable: true,
    }),
    lastName: this.fb.control('', {
      validators: [Validators.required, Validators.maxLength(50)],
      nonNullable: true,
    }),
    email: this.fb.control('', {
      validators: [Validators.required, Validators.email],
      nonNullable: true,
    }),
    role: this.fb.control<'admin' | 'user'>('user', { nonNullable: true }),
  });

  onSubmit(): void {
    if (this.form.invalid) return;
    // getRawValue() は disabled コントロールの値も含めて取得する（value だと undefined になる）
    this.store.dispatch(UserActions.createUser({ payload: this.form.getRawValue() }));
  }
}
```

---

### 19. Validators / カスタムバリデータ

**概要**

バリデーションは純粋関数として実装する。書き方は v20 でも変わらない。

```typescript
// 標準バリデータ
Validators.required
Validators.email
Validators.minLength(8)
Validators.maxLength(100)
Validators.pattern(/^\d+$/)

// カスタムバリデータ（純粋関数で実装 → テスト容易）
// shared/validators/password.validator.ts
export function passwordStrengthValidator(control: AbstractControl): ValidationErrors | null {
  const value: string = control.value ?? '';
  const hasUpperCase = /[A-Z]/.test(value);
  const hasNumber = /[0-9]/.test(value);
  const hasSpecial = /[!@#$%^&*]/.test(value);

  if (!hasUpperCase || !hasNumber || !hasSpecial) {
    return { passwordStrength: { hasUpperCase, hasNumber, hasSpecial } };
  }
  return null;  // null は「エラーなし」
}

// 非同期バリデータ（例：メールアドレスの重複確認）
export function uniqueEmailValidator(userApi: UserApiService): AsyncValidatorFn {
  return (control: AbstractControl): Observable<ValidationErrors | null> =>
    userApi.checkEmailExists(control.value).pipe(
      debounceTime(300),
      map((exists) => exists ? { emailTaken: true } : null),
      catchError(() => of(null))  // API エラー時はバリデーション無効
    );
}
```

---

## ライフサイクル系

---

### 20. OnInit / OnDestroy / DestroyRef

**概要**

`ngOnInit` は変わらない。`ngOnDestroy` + `Subject` での購読解除パターンを、v16 以降は `DestroyRef` + `takeUntilDestroyed()` に移行できる。

---

#### 現在の書き方（ngOnDestroy + Subject）

```typescript
export class UserListPageComponent implements OnInit, OnDestroy {
  private readonly destroy$ = new Subject<void>();

  ngOnInit(): void {
    this.store.dispatch(UserActions.loadUsers());
    this.activatedRoute.paramMap.pipe(
      takeUntil(this.destroy$)
    ).subscribe(...);
  }

  ngOnDestroy(): void {
    this.destroy$.next();
    this.destroy$.complete();
  }
}
```

---

#### 今後の書き方（DestroyRef + takeUntilDestroyed）

```typescript
import { DestroyRef, inject } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

export class UserListPageComponent implements OnInit {
  private readonly destroyRef = inject(DestroyRef);
  // OnDestroy の実装が不要になる

  ngOnInit(): void {
    this.store.dispatch(UserActions.loadUsers());
    this.activatedRoute.paramMap.pipe(
      takeUntilDestroyed(this.destroyRef)
    ).subscribe(...);
  }
}

// クラスフィールド宣言（injection context）内では引数なしで使える
export class AnotherComponent {
  readonly data$ = this.service.getData().pipe(
    takeUntilDestroyed()  // injection context 内は DestroyRef が自動解決される
  );
}
```

---

### 21. AfterViewInit / AfterContentInit

**概要**

DOM や投影コンテンツへのアクセスが必要な初期化処理に使う。書き方は v20 でも変わらない。ただし Signal ViewChild に移行すると `AfterViewInit` が不要になるケースがある。

```typescript
export class ChartComponent implements AfterViewInit {
  // 現在: @ViewChild
  @ViewChild('canvas') canvasRef!: ElementRef<HTMLCanvasElement>;

  ngAfterViewInit(): void {
    // AfterViewInit 以降で canvasRef が確定する
    this.initChart(this.canvasRef.nativeElement);
  }
}

// 今後: Signal viewChild に移行すると effect() で反応できる
export class ChartComponent {
  canvasRef = viewChild<ElementRef<HTMLCanvasElement>>('canvas');

  constructor() {
    // effect() は Signal の変化に反応して実行される
    effect(() => {
      const canvas = this.canvasRef();
      if (canvas) this.initChart(canvas.nativeElement);
    });
  }
}
```

---

### 22. OnChanges / SimpleChanges

**概要**

`@Input()` の変更に反応して処理を実行する。Signal Input に移行すると `ngOnChanges` の代わりに `computed()` や `effect()` で対応できる。

---

#### 現在の書き方

```typescript
export class UserCardComponent implements OnChanges {
  @Input({ required: true }) user!: UserCardViewModel;
  fullName = '';

  ngOnChanges(changes: SimpleChanges): void {
    if (changes['user']) {
      this.fullName = `${this.user.lastName} ${this.user.firstName}`;
    }
  }
}
```

---

#### 今後の書き方（Signal + computed）

```typescript
export class UserCardComponent {
  user = input.required<UserCardViewModel>();

  // computed は user() が変わると自動で再計算される
  // ngOnChanges が不要になる
  readonly fullName = computed(() =>
    `${this.user().lastName} ${this.user().firstName}`
  );
}
```

---

## NgRx系

NgRx の API（`createReducer` / `createEffect` / `createSelector` 等）は v20 でも書き方が変わらない。現在の実装をそのまま継続して問題ない。

---

### 23. Store / dispatch / select

**概要**

Pages のみで使う。`select` は `Observable` を返す。`toSignal()` と組み合わせると Signal として扱える。

```typescript
// 現在（async パイプ）
export class UserListPageComponent {
  private readonly store = inject(Store);

  readonly users$ = this.store.select(selectAllUsers);
  readonly isLoading$ = this.store.select(selectUsersLoading);

  ngOnInit(): void {
    this.store.dispatch(UserActions.loadUsers());
  }
}

// 今後（toSignal）
export class UserListPageComponent {
  private readonly store = inject(Store);

  readonly users = toSignal(this.store.select(selectAllUsers), { initialValue: [] });
  readonly isLoading = toSignal(this.store.select(selectUsersLoading), { initialValue: false });

  ngOnInit(): void {
    this.store.dispatch(UserActions.loadUsers());
  }
}
```

---

### 24. createActionGroup / createAction

```typescript
// 現在 & 今後（書き方は変わらない）
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

---

### 25. createReducer / on

```typescript
// 現在 & 今後（書き方は変わらない）
export const userReducer = createReducer(
  initialState,
  on(UserActions.loadUsers, (state) => ({ ...state, loading: true, error: null })),
  on(UserActions.loadUsersSuccess, (state, { users }) =>
    userAdapter.setAll(users, { ...state, loading: false })
  ),
  on(UserActions.loadUsersFailure, (state, { error }) => ({
    ...state, loading: false, error,
  }))
);
```

Reducer の鉄則：純粋関数として書く。HTTP通信・`Date.now()`・乱数等の副作用は書かない。

---

### 26. createSelector / createFeatureSelector

**概要**

メモ化されたセレクター。`async` パイプ・`toSignal()` のどちらとも組み合わせて使える。

```typescript
// 現在 & 今後（書き方は変わらない）
export const selectUserState = createFeatureSelector<UserState>('user');

const { selectAll, selectEntities } = userAdapter.getSelectors();

export const selectAllUsers = createSelector(selectUserState, selectAll);
export const selectUsersLoading = createSelector(selectUserState, (s) => s.loading);
export const selectSelectedUserId = createSelector(selectUserState, (s) => s.selectedUserId);

// ViewModel変換をSelectorに閉じ込める（Componentに表示用の型だけを渡す）
export const selectUserCardViewModels = createSelector(
  selectAllUsers,
  (users): UserCardViewModel[] => users.map(toUserCardViewModel)
);

// 複数のSelectorを合成
export const selectSelectedUser = createSelector(
  selectEntities,
  selectSelectedUserId,
  (entities, selectedId) => selectedId ? entities[selectedId] : null
);
```

---

### 27. createEffect / Actions / ofType

```typescript
// 現在 & 今後（書き方は変わらない）
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

  // dispatch: false → 戻り値を Action として dispatch しない（ログ・副作用のみ）
  logError$ = createEffect(() =>
    this.actions$.pipe(
      ofType(UserActions.loadUsersFailure),
      tap(({ error }) => console.error('[UserEffects]', error))
    ),
    { dispatch: false }
  );
}
```

`catchError` で `of(FailureAction)` を使う理由：エラーを throw すると Effects の Stream が終了して以降のアクションを受け取れなくなるため、`of()` で1件のアクションに変換して Stream を維持する。

---

### 28. EntityAdapter / EntityState

```typescript
// 現在 & 今後（書き方は変わらない）
export const userAdapter = createEntityAdapter<User>({
  selectId: (user) => user.id,
  sortComparer: (a, b) => a.lastName.localeCompare(b.lastName),
});

// Reducer での操作
userAdapter.setAll(users, state)
userAdapter.upsertOne(user, state)
userAdapter.removeOne(id, state)
```

---

## ユーティリティ系

---

### 29. Pipe（カスタムパイプ）

**概要**

テンプレート内の値変換に使う。v20 で Standalone Pipe が使いやすくなっているが、NgModule 環境では SharedModule に登録する方針を維持する。

```typescript
// 現在（NgModule 管理）
@Pipe({ name: 'statusLabel' })
export class StatusLabelPipe implements PipeTransform {
  transform(status: UserStatus): string {
    const labels: Record<UserStatus, string> = {
      active: '有効',
      inactive: '無効',
      suspended: '停止中',
    };
    return labels[status] ?? '不明';
  }
}
// → SharedModule の declarations と exports に追加して使う

// 今後（Standalone Pipe）
@Pipe({
  name: 'statusLabel',
  standalone: true,  // コンポーネントの imports に直接追加できる
})
export class StatusLabelPipe implements PipeTransform {
  transform(status: UserStatus): string { ... }
}
```

---

### 30. Directive（カスタムディレクティブ）

**概要**

DOM の振る舞い制御に使う。Pipe と同様に NgModule 環境では SharedModule で管理し、Standalone 移行後はコンポーネントの `imports` に追加する。

```typescript
// 現在（NgModule 管理）
@Directive({ selector: '[appAutoFocus]' })
export class AutoFocusDirective implements AfterViewInit {
  private readonly el = inject(ElementRef);

  ngAfterViewInit(): void {
    this.el.nativeElement.focus();
  }
}

// 今後（Standalone Directive）
@Directive({
  selector: '[appClickOutside]',
  standalone: true,
})
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
```

---

### 31. ErrorHandler

**概要**

アプリ全体で捕捉されなかったエラーの最終受け皿。書き方は v20 でも変わらない。

```typescript
@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  private readonly store = inject(Store);

  handleError(error: unknown): void {
    if (error instanceof HttpErrorResponse) {
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
providers: [{ provide: ErrorHandler, useClass: GlobalErrorHandler }]
```

---

## まとめ：移行ロードマップ

Signal API の移行と Standalone 化は**独立した別軸**として進める。Signal API はすべて NgModule 管理下のままで適用できるため、Standalone 化を待つ必要はない。

### 軸A：Signal API への移行（NgModule のままで進められる）

**ステップ1：今すぐ新規ファイルから適用する**

- `*ngIf` / `*ngFor` / `*ngSwitch` → `@if` / `@for` / `@switch`
- `ngOnDestroy` + `Subject` → `DestroyRef` + `takeUntilDestroyed()`
- クラス型 Guard / Resolver → 関数型 `CanActivateFn` / `ResolveFn`（すでに非推奨）
- クラス型 Interceptor → `HttpInterceptorFn`（すでに非推奨）

**ステップ2：新規コンポーネント作成時から適用する**

- `@Input()` / `@Output()` → `input()` / `output()`
- `@ViewChild` / `@ContentChild` → `viewChild()` / `contentChild()`
- `ngOnChanges` → `computed()`
- `async` パイプ → `toSignal()`

**ステップ3：既存コンポーネントを修正するタイミングで順次適用する**

- 上記ステップ1・2の内容を、既存ファイルに触れる機会に合わせて書き換えていく

---

### 軸B：Standalone 化（別途計画して進める）

NgModule から Standalone への移行は影響範囲が大きいため、機能単位で計画的に進める。Signal API 移行の進捗とは切り離して判断する。

1. 末端の表示専用 Component から Standalone 化する
2. Template Component を Standalone 化する
3. Page Component と Routing を Standalone に切り替える
4. Module ファイルを削除する

---

### 変わらないもの（書き直し不要）

NgRx の `createReducer` / `createEffect` / `createSelector` / `EntityAdapter`、`HttpClient`（Service 内）、Reactive Forms、`InjectionToken`、`APP_INITIALIZER`、`ErrorHandler` は v20 でも書き方が変わらないためそのまま継続する。
