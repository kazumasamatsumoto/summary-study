# テスト モック・スタブ定義（Angular v20 / Jest）

> **このドキュメントの目的**  
> Component・Service・Store それぞれのテストで「何をモック化・スタブ化するか」を定義する。  
> 「なぜそうするか」の理由とセットで記載し、判断に迷ったときの基準として使う。

---

## 用語の整理

まず「モック」「スタブ」「フェイク」の意味をこのドキュメント内で統一しておく。

| 用語 | 意味 | 主な用途 |
|------|------|---------|
| **スタブ** | 決まった値を返すだけの偽物 | 依存先の戻り値を固定して、テスト対象の動作を制御する |
| **モック** | 呼ばれたかどうかを検証できる偽物 | 依存先に期待通りの呼び出しがされたか確認する |
| **フェイク** | 本物に近い動作をする軽量な実装 | `MockStore`・`HttpTestingController` 等 |
| **スパイ** | 本物に割り込んで呼び出しを記録する | `jest.spyOn()` で既存メソッドを監視する |

実際のテストではこれらを組み合わせて使う。呼び出しの確認だけなら「モック」、返す値の固定だけなら「スタブ」と意識を分けると、`expect` の書き方が明確になる。

---

## 全体方針

テストの信頼性とメンテナンスコストのバランスをとるために、**テスト対象の責務だけを純粋に検証する**ことを原則とする。

```
テスト対象の責務の外にあるものは原則モック・スタブ化する
テスト対象の責務の中にあるものは本物を使う
```

各層の責務は開発ルールの通り。

| 層 | 責務 |
|----|------|
| Component | 受け取った値を表示する・イベントを発火する |
| Template | コンポーネントをレイアウトに従って組み立てる |
| Page | Store と接続してビジネスロジックを実行する |
| Service | HTTP通信を行いレスポンスを返す |
| Reducer | Action を受け取り新しい State を返す |
| Selector | State から派生データを計算して返す |
| Effects | Action を受け取り Service を呼び、結果を Action に変換する |

---

## 1. Component テスト

### テスト対象の責務

- `@Input()` で受け取った値が正しく DOM に表示されること
- ユーザー操作（クリック等）で `@Output()` から正しい値が発火されること

### モック・スタブの方針

**モックする：子コンポーネント（スタブ化）**

ComponentはUIパーツの最小単位なので子コンポーネントを持つことは少ないが、持つ場合は子コンポーネントをスタブ化する。本物の子コンポーネントを使うと、子コンポーネントのバグがこのテストを壊すため。

**モックしない：外部依存（Component は持たないルールのため不要）**

開発ルール上、ComponentはStore・Router・Serviceを持たない。依存が増えたらルール違反のサインとして捉える。

```typescript
// user-card.component.spec.ts

// 子コンポーネントがある場合のスタブ定義
@Component({ selector: 'app-avatar', template: '' })
class AvatarStubComponent {
  @Input() src!: string;
}

describe('UserCardComponent', () => {
  let fixture: ComponentFixture<UserCardComponent>;
  let component: UserCardComponent;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [
        UserCardComponent,
        AvatarStubComponent,  // 本物ではなくスタブを登録
      ],
    }).compileComponents();

    fixture = TestBed.createComponent(UserCardComponent);
    component = fixture.componentInstance;
  });

  describe('表示', () => {
    it('渡されたユーザー名が表示される', () => {
      // Arrange
      component.user = mockUserCardViewModel({ firstName: '太郎', lastName: '山田' });

      // Act
      fixture.detectChanges();

      // Assert
      const el = fixture.nativeElement.querySelector('[data-testid="user-name"]');
      expect(el.textContent).toContain('山田 太郎');
    });

    it('isHighlighted が true のとき highlighted クラスが付く', () => {
      component.user = mockUserCardViewModel();
      component.isHighlighted = true;
      fixture.detectChanges();

      const el = fixture.nativeElement.querySelector('[data-testid="card"]');
      expect(el.classList).toContain('highlighted');
    });
  });

  describe('イベント', () => {
    it('カードをクリックすると selected に userId が emit される', () => {
      // Arrange
      const userId = 'user-123';
      component.user = mockUserCardViewModel({ id: userId });
      fixture.detectChanges();

      const emittedValues: string[] = [];
      component.selected.subscribe((v) => emittedValues.push(v));

      // Act
      const card = fixture.nativeElement.querySelector('[data-testid="card"]');
      card.click();

      // Assert
      expect(emittedValues).toEqual([userId]);
    });
  });
});
```

### Component テストのチェックリスト

- [ ] `@Input()` の全パターン（正常値・境界値・必須チェック）を網羅しているか
- [ ] `@Output()` の発火タイミングと値を検証しているか
- [ ] 子コンポーネントはスタブ化されているか
- [ ] `data-testid` 属性でDOM要素を特定しているか（class や tag 名ではない）
- [ ] `fixture.detectChanges()` を `@Input()` のセット後に呼んでいるか

---

## 2. Template テスト

### テスト対象の責務

- `@Input()` で受け取ったデータが正しく子コンポーネントへバインドされること
- 子コンポーネントのイベントが `@Output()` を通じて正しく親へ伝達されること
- 条件分岐（`@if` / `*ngIf`）やリスト描画（`@for` / `*ngFor`）が正しく動作すること

### モック・スタブの方針

**モックする：子コンポーネント（スタブ化）**

TemplateテストはTemplateの「組み立てが正しいか」を検証する。子コンポーネントの内部動作はそれぞれのComponentテストが担保しているので、ここではスタブで十分。

**モックしない：外部依存（Template は持たないルールのため不要）**

```typescript
// user-list.template.spec.ts

@Component({ selector: 'app-user-card', template: '' })
class UserCardStubComponent {
  @Input() user!: UserCardViewModel;
  @Output() selected = new EventEmitter<string>();
}

@Component({ selector: 'app-loading-spinner', template: '<div>loading</div>' })
class LoadingSpinnerStubComponent {}

describe('UserListTemplateComponent', () => {
  let fixture: ComponentFixture<UserListTemplateComponent>;
  let component: UserListTemplateComponent;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [
        UserListTemplateComponent,
        UserCardStubComponent,
        LoadingSpinnerStubComponent,
      ],
    }).compileComponents();

    fixture = TestBed.createComponent(UserListTemplateComponent);
    component = fixture.componentInstance;
  });

  it('isLoading が true のときローディングスピナーが表示される', () => {
    component.isLoading = true;
    component.users = [];
    fixture.detectChanges();

    const spinner = fixture.nativeElement.querySelector('app-loading-spinner');
    expect(spinner).not.toBeNull();
  });

  it('users の件数分だけ app-user-card が描画される', () => {
    component.isLoading = false;
    component.users = [mockUserCardViewModel(), mockUserCardViewModel()];
    fixture.detectChanges();

    const cards = fixture.nativeElement.querySelectorAll('app-user-card');
    expect(cards.length).toBe(2);
  });

  it('app-user-card の selected イベントが userSelected として伝播される', () => {
    component.isLoading = false;
    component.users = [mockUserCardViewModel({ id: 'user-1' })];
    fixture.detectChanges();

    const emitted: string[] = [];
    component.userSelected.subscribe((v) => emitted.push(v));

    // スタブコンポーネントのインスタンスを取得してイベントを発火
    const cardDebugEl = fixture.debugElement.query(By.directive(UserCardStubComponent));
    cardDebugEl.componentInstance.selected.emit('user-1');

    expect(emitted).toEqual(['user-1']);
  });
});
```

---

## 3. Page テスト

### テスト対象の責務

- `ngOnInit` 等のタイミングで正しい Action が dispatch されること
- Store から取得した Observable がテンプレートに正しく渡されること
- ユーザーイベントのハンドラーが正しい Action を dispatch すること
- Router ナビゲーションが正しいパスで呼ばれること

### モック・スタブの方針

**フェイク化する：Store → `MockStore` を使う**

本物の Store を使うと Reducer・Effects・Selector すべてが動いてしまい、Pageの責務だけを検証できない。`MockStore` を使えばStoreの動作を無視して「dispatch が呼ばれたか」「Select の結果がテンプレートに渡ったか」だけに集中できる。

**モックする：Router → `jest.fn()` でスパイ**

ナビゲーション先のルートやコンポーネントは関係ない。「`router.navigate` が正しい引数で呼ばれたか」だけを検証する。

**スタブ化する：Template・子コンポーネント**

PageテストはPageの責務（StoreとのやりとりとビジネスロジックのDispatch）を検証する。テンプレートの描画内容はTemplateテストが担保する。

```typescript
// user-list.page.spec.ts
import { MockStore, provideMockStore } from '@ngrx/store/testing';

describe('UserListPageComponent', () => {
  let fixture: ComponentFixture<UserListPageComponent>;
  let component: UserListPageComponent;
  let store: MockStore;
  let router: { navigate: jest.Mock };

  // Storeの初期状態
  const initialState = {
    user: userAdapter.getInitialState({
      loading: false,
      error: null,
      selectedUserId: null,
    }),
  };

  beforeEach(async () => {
    // RouterをシンプルなオブジェクトでモックするのでTestBedが不要
    router = { navigate: jest.fn() };

    await TestBed.configureTestingModule({
      declarations: [
        UserListPageComponent,
        UserListTemplateStubComponent,  // Templateはスタブ
      ],
      providers: [
        // MockStore: Storeのフェイク実装
        // selectors の結果を任意に上書きできる
        provideMockStore({ initialState }),
        // Router は jest.fn() を使ったモックで提供
        { provide: Router, useValue: router },
      ],
    }).compileComponents();

    store = TestBed.inject(MockStore);
    fixture = TestBed.createComponent(UserListPageComponent);
    component = fixture.componentInstance;
  });

  afterEach(() => {
    // MockStore のオーバーライドをリセット
    store.resetSelectors();
  });

  describe('初期化', () => {
    it('ngOnInit で loadUsers Action が dispatch される', () => {
      // Arrange
      const dispatchSpy = jest.spyOn(store, 'dispatch');

      // Act
      fixture.detectChanges();  // ngOnInit を実行

      // Assert
      expect(dispatchSpy).toHaveBeenCalledWith(UserActions.loadUsers());
    });
  });

  describe('Selector の値がテンプレートに渡る', () => {
    it('selectUserCardViewModels の結果がテンプレートに渡る', () => {
      // Arrange: Selector の返す値を MockStore でオーバーライド
      const mockUsers = [mockUserCardViewModel()];
      store.overrideSelector(selectUserCardViewModels, mockUsers);
      store.refreshState();  // Selector のキャッシュをリフレッシュ

      // Act
      fixture.detectChanges();

      // Assert: スタブに渡った値を検証
      const templateDebugEl = fixture.debugElement.query(
        By.directive(UserListTemplateStubComponent)
      );
      expect(templateDebugEl.componentInstance.users).toEqual(mockUsers);
    });

    it('selectUsersLoading が true のとき isLoading が渡る', () => {
      store.overrideSelector(selectUsersLoading, true);
      store.refreshState();
      fixture.detectChanges();

      const templateDebugEl = fixture.debugElement.query(
        By.directive(UserListTemplateStubComponent)
      );
      expect(templateDebugEl.componentInstance.isLoading).toBe(true);
    });
  });

  describe('イベントハンドラー', () => {
    it('onUserSelected を呼ぶと selectUser Action が dispatch される', () => {
      const dispatchSpy = jest.spyOn(store, 'dispatch');
      fixture.detectChanges();

      component.onUserSelected('user-123');

      expect(dispatchSpy).toHaveBeenCalledWith(
        UserActions.selectUser({ userId: 'user-123' })
      );
    });

    it('onEdit を呼ぶと /users/:id/edit にナビゲートする', () => {
      fixture.detectChanges();
      component.onEdit('user-123');

      expect(router.navigate).toHaveBeenCalledWith(['/users', 'user-123', 'edit']);
    });
  });
});
```

### `MockStore` で Selector をオーバーライドする理由

本物の Selector を使うと、Selector のロジックや Store の構造変更がPageテストを壊す可能性がある。`overrideSelector` を使うことで「このSelectorは〇〇を返す」という前提を固定し、Pageの責務（その値をどう使うか）だけを検証できる。

### Page テストのチェックリスト

- [ ] `MockStore` を使っているか（本物の Store を使っていないか）
- [ ] `Router` は `jest.fn()` のモックで提供しているか
- [ ] Template・子コンポーネントはスタブ化されているか
- [ ] `store.overrideSelector()` で Selector の返す値を固定しているか
- [ ] `afterEach` で `store.resetSelectors()` を呼んでいるか
- [ ] dispatch の検証は `jest.spyOn(store, 'dispatch')` を使っているか

---

## 4. Service テスト

### テスト対象の責務

- 正しいエンドポイント・HTTPメソッド・リクエストボディで通信が行われること
- レスポンスがそのまま Observable として返されること
- エラー時に Observable がエラーを発行すること

### モック・スタブの方針

**フェイク化する：HttpClient → `HttpClientTestingModule` + `HttpTestingController` を使う**

本物の HTTP 通信は外部依存（サーバーの状態）に左右されるためテストが不安定になる。`HttpTestingController` は「リクエストが来たことを確認して任意のレスポンスを返す」フェイク実装。

**モックしない：その他の依存**

開発ルール上、ServiceはStore・Router・他のServiceに依存しない。依存が増えたらルール違反のサインとして捉える。

```typescript
// user-api.service.spec.ts
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing';

describe('UserApiService', () => {
  let service: UserApiService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [UserApiService],
    });

    service = TestBed.inject(UserApiService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    // 未処理のリクエストが残っていればテストを失敗させる
    // → 「呼ばれるはずのないリクエストが走っていた」を検出できる
    httpMock.verify();
  });

  describe('getUsers()', () => {
    it('GET /api/users を呼び、レスポンスをそのまま返す', (done) => {
      // Arrange
      const mockResponse: UserResponse[] = [mockUserResponse()];

      // Act
      service.getUsers().subscribe({
        next: (users) => {
          // Assert: レスポンスがそのまま返ってきているか
          expect(users).toEqual(mockResponse);
          done();
        },
      });

      // Assert: リクエストの内容を検証してレスポンスを返す
      const req = httpMock.expectOne('/api/users');
      expect(req.request.method).toBe('GET');
      req.flush(mockResponse);
    });

    it('クエリパラメータを正しく付与する', (done) => {
      service.getUsers({ page: 2, limit: 10 }).subscribe({ next: () => done() });

      const req = httpMock.expectOne((r) => r.url === '/api/users');
      expect(req.request.params.get('page')).toBe('2');
      expect(req.request.params.get('limit')).toBe('10');
      req.flush([]);
    });

    it('サーバーエラー時に Observable がエラーを発行する', (done) => {
      service.getUsers().subscribe({
        next: () => done.fail('エラーが発生すべき'),
        error: (err: HttpErrorResponse) => {
          expect(err.status).toBe(500);
          done();
        },
      });

      const req = httpMock.expectOne('/api/users');
      req.flush('Server Error', { status: 500, statusText: 'Internal Server Error' });
    });
  });

  describe('createUser()', () => {
    it('POST /api/users を正しいボディで呼ぶ', (done) => {
      const payload: CreateUserRequest = { firstName: '太郎', lastName: '山田', email: 'taro@example.com' };
      const mockResponse = mockUserResponse();

      service.createUser(payload).subscribe({ next: () => done() });

      const req = httpMock.expectOne('/api/users');
      expect(req.request.method).toBe('POST');
      // リクエストボディが正しいか検証
      expect(req.request.body).toEqual(payload);
      req.flush(mockResponse);
    });
  });
});
```

### Service テストのチェックリスト

- [ ] `HttpClientTestingModule` を使っているか
- [ ] `afterEach` で `httpMock.verify()` を呼んでいるか
- [ ] エンドポイント・HTTPメソッド・リクエストボディの3点を検証しているか
- [ ] 正常系・エラー系（404・500）のケースを網羅しているか
- [ ] レスポンスのデータ変換がある場合、変換後の値を検証しているか

---

## 5. Store テスト

Storeのテストは3種類（Reducer・Selector・Effects）に分かれる。それぞれ性質が違うのでモック戦略も異なる。

---

### 5-1. Reducer テスト

#### テスト対象の責務

- 各 Action に対して State が正しく更新されること
- State の不変性（Immutability）が保たれること

#### モック・スタブの方針

**何もモックしない**

Reducerは `(state, action) => newState` の純粋関数。外部依存が一切ないため、モックする対象がない。TestBedも不要。

```typescript
// store/user.reducer.spec.ts
describe('UserReducer', () => {
  // initialState を毎回フレッシュに作る（テスト間の状態汚染を防ぐ）
  const getInitialState = (): UserState =>
    userAdapter.getInitialState({
      loading: false,
      error: null,
      selectedUserId: null,
    });

  describe('loadUsers', () => {
    it('loading が true になり error がクリアされる', () => {
      const state = { ...getInitialState(), error: '前回のエラー' };

      const result = userReducer(state, UserActions.loadUsers());

      expect(result.loading).toBe(true);
      expect(result.error).toBeNull();
    });
  });

  describe('loadUsersSuccess', () => {
    it('ユーザーが State に追加され loading が false になる', () => {
      const state = { ...getInitialState(), loading: true };
      const users = [mockUser({ id: 'user-1' }), mockUser({ id: 'user-2' })];

      const result = userReducer(state, UserActions.loadUsersSuccess({ users }));

      expect(result.loading).toBe(false);
      expect(result.error).toBeNull();
      // EntityAdapterのセレクターで全件取得して検証
      expect(userAdapter.getSelectors().selectAll(result)).toHaveLength(2);
    });

    it('既存の State に影響を与えない（Immutability の確認）', () => {
      const state = getInitialState();
      const stateBefore = { ...state };

      userReducer(state, UserActions.loadUsersSuccess({ users: [mockUser()] }));

      // 元の State オブジェクトが変更されていないことを確認
      expect(state).toEqual(stateBefore);
    });
  });

  describe('loadUsersFailure', () => {
    it('error が State にセットされ loading が false になる', () => {
      const state = { ...getInitialState(), loading: true };

      const result = userReducer(state, UserActions.loadUsersFailure({ error: 'Network Error' }));

      expect(result.loading).toBe(false);
      expect(result.error).toBe('Network Error');
    });
  });

  describe('selectUser', () => {
    it('selectedUserId が更新される', () => {
      const state = getInitialState();

      const result = userReducer(state, UserActions.selectUser({ userId: 'user-123' }));

      expect(result.selectedUserId).toBe('user-123');
    });
  });
});
```

---

### 5-2. Selector テスト

#### テスト対象の責務

- State から正しい派生データが返されること
- ViewModel への変換が正しく行われること
- メモ化が機能しているか（同じ入力で再計算が起きないか）

#### モック・スタブの方針

**何もモックしない**

SelectorはReducerと同様に純粋関数。`projector` 関数を直接呼ぶことで TestBed なしでテストできる。

```typescript
// store/user.selectors.spec.ts
describe('UserSelectors', () => {
  // テスト用の State を直接生成する
  const buildState = (overrides?: Partial<UserState>): UserState =>
    userAdapter.setAll(
      [mockUser({ id: 'user-1', firstName: '太郎', lastName: '山田' })],
      userAdapter.getInitialState({
        loading: false,
        error: null,
        selectedUserId: null,
        ...overrides,
      })
    );

  describe('selectAllUsers', () => {
    it('State からユーザーの配列を返す', () => {
      const state = buildState();

      // projector: Selector の変換ロジックだけを直接テストできる
      const result = selectAllUsers.projector(state);

      expect(result).toHaveLength(1);
      expect(result[0].id).toBe('user-1');
    });
  });

  describe('selectUsersLoading', () => {
    it('loading フラグを返す', () => {
      const state = buildState({ loading: true });

      const result = selectUsersLoading.projector(state);

      expect(result).toBe(true);
    });
  });

  describe('selectUserCardViewModels', () => {
    it('User を UserCardViewModel にマッピングして返す', () => {
      const users = [mockUser({ id: 'user-1', firstName: '太郎', lastName: '山田' })];

      // 複数のSelectorを合成している場合、projectorの引数は合成元の結果
      const result = selectUserCardViewModels.projector(users);

      expect(result).toHaveLength(1);
      expect(result[0]).toEqual<UserCardViewModel>({
        id: 'user-1',
        name: '山田 太郎',  // ViewModelの変換が正しいか
        avatarUrl: '/assets/default-avatar.png',
      });
    });
  });

  describe('selectSelectedUser', () => {
    it('selectedUserId に対応するユーザーを返す', () => {
      const entities = { 'user-1': mockUser({ id: 'user-1' }) };
      const selectedId = 'user-1';

      const result = selectSelectedUser.projector(entities, selectedId);

      expect(result?.id).toBe('user-1');
    });

    it('selectedUserId が null のとき null を返す', () => {
      const entities = { 'user-1': mockUser({ id: 'user-1' }) };

      const result = selectSelectedUser.projector(entities, null);

      expect(result).toBeNull();
    });
  });
});
```

---

### 5-3. Effects テスト

#### テスト対象の責務

- 特定の Action を受け取ったとき正しい Service メソッドが呼ばれること
- Service の成功時に Success Action が dispatch されること
- Service の失敗時に Failure Action が dispatch されること
- `switchMap` の場合、前のリクエストがキャンセルされること

#### モック・スタブの方針

**モックする：Service → `jest.fn()` でスタブ化**

EffectsのテストはEffectsの「ActionとServiceの橋渡し」ロジックを検証する。本物のServiceを使うと HTTP 通信が走ってしまうため、Service の各メソッドを `jest.fn()` でスタブ化する。

**フェイク化する：Actions → `provideMockActions()` を使う**

NgRxが提供するEffectsテスト用のユーティリティ。テストからActionを任意に流せる。

**モックしない：Store（Effectsで Store から State を取得する場合は MockStore を使う）**

```typescript
// store/user.effects.spec.ts
import { provideMockActions } from '@ngrx/effects/testing';
import { provideMockStore } from '@ngrx/store/testing';

describe('UserEffects', () => {
  let effects: UserEffects;
  let actions$: Observable<Action>;
  // Service全体をモックオブジェクトで置き換える
  let userApiService: jest.Mocked<UserApiService>;

  beforeEach(() => {
    // jest.Mocked<T> で全メソッドを jest.fn() にした型安全なモックを作る
    const mockUserApiService: jest.Mocked<UserApiService> = {
      getUsers: jest.fn(),
      getUserById: jest.fn(),
      createUser: jest.fn(),
    } as jest.Mocked<UserApiService>;

    TestBed.configureTestingModule({
      providers: [
        UserEffects,
        // Actions を外部から制御できるようにする
        provideMockActions(() => actions$),
        // MockStore: withLatestFrom 等でStoreを参照する場合に必要
        provideMockStore(),
        // 本物のServiceの代わりにモックを注入
        { provide: UserApiService, useValue: mockUserApiService },
      ],
    });

    effects = TestBed.inject(UserEffects);
    userApiService = TestBed.inject(UserApiService) as jest.Mocked<UserApiService>;
  });

  describe('loadUsers$', () => {
    it('loadUsers Action で getUsers() が呼ばれ Success Action が返る', (done) => {
      // Arrange
      const mockUsers = [mockUser()];
      // Service のスタブが返す値を定義
      userApiService.getUsers.mockReturnValue(of(mockUsers));

      // Act: テストから Action を流す
      actions$ = of(UserActions.loadUsers());

      // Assert
      effects.loadUsers$.subscribe((resultAction) => {
        expect(userApiService.getUsers).toHaveBeenCalledTimes(1);
        expect(resultAction).toEqual(UserActions.loadUsersSuccess({ users: mockUsers }));
        done();
      });
    });

    it('getUsers() がエラーを返したとき Failure Action が返る', (done) => {
      // Arrange
      const error = new HttpErrorResponse({ status: 500, statusText: 'Internal Server Error' });
      userApiService.getUsers.mockReturnValue(throwError(() => error));

      actions$ = of(UserActions.loadUsers());

      // Assert
      effects.loadUsers$.subscribe((resultAction) => {
        expect(resultAction).toEqual(
          UserActions.loadUsersFailure({ error: error.message })
        );
        done();
      });
    });

    it('loadUsers が連続して来た場合、最後のもだけが有効になる（switchMap の検証）', () => {
      // Arrange: 1回目は遅延、2回目は即時返す
      userApiService.getUsers
        .mockReturnValueOnce(of([mockUser({ id: 'first' })]).pipe(delay(100)))
        .mockReturnValueOnce(of([mockUser({ id: 'second' })]));

      const results: Action[] = [];

      // ReplaySubject で複数のActionを流す
      const actions = new ReplaySubject<Action>();
      actions$ = actions.asObservable();

      effects.loadUsers$.subscribe((action) => results.push(action));

      // 1回目のAction → 2回目のAction（1回目はキャンセルされる）
      actions.next(UserActions.loadUsers());
      actions.next(UserActions.loadUsers());

      // switchMap により、最後のリクエストの結果だけが流れる
      expect(results).toHaveLength(1);
      expect((results[0] as ReturnType<typeof UserActions.loadUsersSuccess>).users[0].id)
        .toBe('second');
    });
  });

  describe('loadUserById$', () => {
    it('Storeにキャッシュがある場合 API を呼ばない', () => {
      // MockStore でEntityが存在する状態を作る
      const store = TestBed.inject(MockStore);
      store.overrideSelector(selectUserEntities, { 'user-1': mockUser({ id: 'user-1' }) });
      store.refreshState();

      actions$ = of(UserActions.selectUser({ userId: 'user-1' }));

      effects.loadUserById$.subscribe();

      // キャッシュがあるので API は呼ばれない
      expect(userApiService.getUserById).not.toHaveBeenCalled();
    });
  });
});
```

### Effects テストのチェックリスト

- [ ] Service は `jest.Mocked<T>` でモック化されているか
- [ ] `provideMockActions()` を使っているか
- [ ] `withLatestFrom` 等でStoreを参照する場合 `MockStore` を追加しているか
- [ ] 成功・失敗の両方のケースをテストしているか
- [ ] `switchMap` を使っている場合、キャンセル動作もテストしているか
- [ ] `jest.fn()` のスタブに戻り値を設定しているか（`mockReturnValue` / `mockReturnValueOnce`）

---

## 6. モック・スタブの使い分け早見表

| テスト対象 | 何をモック・スタブ化するか | 使うツール |
|-----------|--------------------------|-----------|
| **Component** | 子コンポーネント（スタブ） | `@Component({ template: '' })` のスタブクラス |
| **Template** | 子コンポーネント（スタブ） | `@Component({ template: '' })` のスタブクラス |
| **Page** | Store（フェイク） | `MockStore` / `provideMockStore()` |
| **Page** | Router（モック） | `{ navigate: jest.fn() }` |
| **Page** | Template・子コンポーネント（スタブ） | `@Component({ template: '' })` のスタブクラス |
| **Service** | HttpClient（フェイク） | `HttpClientTestingModule` + `HttpTestingController` |
| **Reducer** | なし | 純粋関数のためモック不要 |
| **Selector** | なし | `projector` で直接テスト可能 |
| **Effects** | Service（モック） | `jest.Mocked<T>` + `mockReturnValue()` |
| **Effects** | Actions（フェイク） | `provideMockActions()` |
| **Effects** | Store（フェイク、必要な場合のみ） | `MockStore` / `provideMockStore()` |

---

## 7. 共通ルール

### モックファクトリーは `testing/` ディレクトリに集約する

```typescript
// src/testing/mock-factories.ts

export const mockUser = (override?: Partial<User>): User => ({
  id: 'user-1',
  firstName: '太郎',
  lastName: '山田',
  email: 'taro@example.com',
  profile: null,
  ...override,
});

export const mockUserCardViewModel = (override?: Partial<UserCardViewModel>): UserCardViewModel => ({
  id: 'user-1',
  name: '山田 太郎',
  avatarUrl: '/assets/default-avatar.png',
  ...override,
});

export const mockUserResponse = (override?: Partial<UserResponse>): UserResponse => ({
  id: 'user-1',
  firstName: '太郎',
  lastName: '山田',
  email: 'taro@example.com',
  ...override,
});
```

ファクトリーを一箇所に集めると、型が変わったときの修正が1点で済む。テストごとにオブジェクトリテラルを書くと、型変更のたびに全テストファイルを修正することになる。

### スタブコンポーネントは `testing/stubs/` に集約する

```typescript
// src/testing/stubs/user-card.stub.ts
@Component({ selector: 'app-user-card', template: '' })
export class UserCardStubComponent {
  @Input() user!: UserCardViewModel;
  @Output() selected = new EventEmitter<string>();
}

// src/testing/stubs/loading-spinner.stub.ts
@Component({ selector: 'app-loading-spinner', template: '' })
export class LoadingSpinnerStubComponent {}
```

スタブを再利用することで「スタブが本物のコンポーネントの API と乖離したまま放置される」リスクを減らせる。

### `jest.fn()` の戻り値は必ず設定する

```typescript
// ❌ 戻り値を設定していない
userApiService.getUsers = jest.fn();
// → 戻り値が undefined になり、subscribe でエラーになる

// ✅ 戻り値を明示する
userApiService.getUsers = jest.fn().mockReturnValue(of([]));
userApiService.getUsers = jest.fn().mockReturnValue(throwError(() => new Error('fail')));

// 複数回呼ばれるケースでは mockReturnValueOnce を連鎖させる
userApiService.getUsers
  .mockReturnValueOnce(of([mockUser()]))  // 1回目
  .mockReturnValueOnce(of([]));           // 2回目
```

### `data-testid` でDOM要素を特定する

```html
<!-- ✅ 推奨 -->
<div data-testid="user-name">{{ user.name }}</div>
<button data-testid="select-btn" (click)="onSelect()">選択</button>

<!-- ❌ 避ける -->
<div class="user-name">{{ user.name }}</div>  <!-- クラス名はスタイル変更で壊れる -->
<button>選択</button>                          <!-- 要素の特定が曖昧 -->
```

```typescript
// テスト側
const el = fixture.nativeElement.querySelector('[data-testid="user-name"]');
```

`data-testid` はテスト専用の識別子として明示的に分離することで、スタイル変更やHTML構造の変更でテストが壊れるのを防げる。
