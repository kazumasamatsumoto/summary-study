非常に良い質問です。Angularのテストを体系的に理解するために、**本質的な構造**から解説します。

## Angularテストの3つの本質的要素

Angularのテストで理解すべきは、たった3つの柱です:

### 1. **テスト対象の分離 (Isolation)**
### 2. **依存関係の制御 (Dependency Injection)**
### 3. **変更検知の管理 (Change Detection)**

この3つを理解すれば、どんなAngularコードもテストできます。

---

## テスト対象別の必須要素マトリックス

| テスト対象 | 分離 | DI | 変更検知 | 優先度 |
|---------|-----|----|---------|----|
| **Pure Pipe** | ✅ | ❌ | ❌ | 最簡単 |
| **Service (依存なし)** | ✅ | ❌ | ❌ | 簡単 |
| **Service (依存あり)** | ✅ | ✅ | ❌ | 中 |
| **Component (単純)** | ✅ | ✅ | ✅ | 中 |
| **Component (複雑)** | ✅ | ✅ | ✅✅✅ | 難 |
| **Directive** | ✅ | ✅ | ✅ | 中〜難 |

優先度順に解説します。

---

## 1. Pure Pipe のテスト（最もシンプル）

**必要な要素: 分離のみ**

```typescript
// pipe実装
@Pipe({ name: 'currencyFormat' })
export class CurrencyFormatPipe implements PipeTransform {
  transform(value: number, symbol: string = '¥'): string {
    return `${symbol}${value.toLocaleString()}`;
  }
}

// テスト
describe('CurrencyFormatPipe', () => {
  let pipe: CurrencyFormatPipe;

  beforeEach(() => {
    pipe = new CurrencyFormatPipe(); // ← new するだけ！
  });

  it('should format number with yen symbol', () => {
    expect(pipe.transform(1000)).toBe('¥1,000');
  });

  it('should format with custom symbol', () => {
    expect(pipe.transform(1000, '$')).toBe('$1,000');
  });
});
```

**ポイント:**
- TestBed不要
- DI不要
- 変更検知不要
- **純粋な関数のテストと同じ**

---

## 2. Service（依存なし）のテスト

**必要な要素: 分離のみ**

```typescript
// service実装
@Injectable({ providedIn: 'root' })
export class CalculatorService {
  add(a: number, b: number): number {
    return a + b;
  }

  subtract(a: number, b: number): number {
    return a - b;
  }
}

// テスト
describe('CalculatorService', () => {
  let service: CalculatorService;

  beforeEach(() => {
    service = new CalculatorService(); // ← new するだけ！
  });

  it('should add numbers', () => {
    expect(service.add(2, 3)).toBe(5);
  });
});
```

**ポイント:**
- 依存がなければPipeと同じくらい簡単
- TestBed不要（使ってもいいが不要）

---

## 3. Service（依存あり）のテスト

**必要な要素: 分離 + DI**

```typescript
// service実装
@Injectable({ providedIn: 'root' })
export class UserService {
  constructor(private http: HttpClient) {}

  getUser(id: number): Observable<User> {
    return this.http.get<User>(`/api/users/${id}`);
  }
}

// テスト - ここから重要！
describe('UserService', () => {
  let service: UserService;
  let httpMock: jasmine.SpyObj<HttpClient>;

  beforeEach(() => {
    // ★ ポイント1: 依存関係をモック化
    httpMock = jasmine.createSpyObj('HttpClient', ['get']);
    
    // ★ ポイント2: TestBedで依存注入を設定
    TestBed.configureTestingModule({
      providers: [
        UserService,
        { provide: HttpClient, useValue: httpMock }
      ]
    });
    
    service = TestBed.inject(UserService);
  });

  it('should get user by id', () => {
    const mockUser = { id: 1, name: 'Test' };
    httpMock.get.and.returnValue(of(mockUser));

    service.getUser(1).subscribe(user => {
      expect(user).toEqual(mockUser);
    });

    expect(httpMock.get).toHaveBeenCalledWith('/api/users/1');
  });
});
```

**ポイント:**
- **TestBedが必要になる理由 = 依存注入の制御**
- `jasmine.createSpyObj`でモック作成
- `{ provide: X, useValue: mock }`で依存差し替え

---

## 4. Component（単純）のテスト

**必要な要素: 分離 + DI + 変更検知**

```typescript
// component実装
@Component({
  selector: 'app-counter',
  template: `
    <div>Count: {{ count }}</div>
    <button (click)="increment()">+</button>
  `
})
export class CounterComponent {
  count = 0;

  increment() {
    this.count++;
  }
}

// テスト
describe('CounterComponent', () => {
  let component: CounterComponent;
  let fixture: ComponentFixture<CounterComponent>;

  beforeEach(() => {
    // ★ ポイント1: TestBedでコンポーネント設定
    TestBed.configureTestingModule({
      declarations: [CounterComponent]
    });

    // ★ ポイント2: fixtureを作成
    fixture = TestBed.createComponent(CounterComponent);
    component = fixture.componentInstance;
  });

  it('should increment count', () => {
    // ★ ポイント3: 変更検知を実行
    fixture.detectChanges();
    
    expect(component.count).toBe(0);
    
    component.increment();
    fixture.detectChanges(); // ← ビュー更新
    
    expect(component.count).toBe(1);
    
    const compiled = fixture.nativeElement;
    expect(compiled.textContent).toContain('Count: 1');
  });
});
```

**ポイント:**
- `ComponentFixture`が登場
- `fixture.detectChanges()`で変更検知
- DOMへのアクセスは`fixture.nativeElement`

---

## 5. Component（依存あり）のテスト

**必要な要素: 分離 + DI + 変更検知**

```typescript
// component実装
@Component({
  selector: 'app-user-list',
  template: `
    <div *ngFor="let user of users">{{ user.name }}</div>
  `
})
export class UserListComponent implements OnInit {
  users: User[] = [];

  constructor(private userService: UserService) {}

  ngOnInit() {
    this.userService.getUsers().subscribe(users => {
      this.users = users;
    });
  }
}

// テスト - すべての要素が必要！
describe('UserListComponent', () => {
  let component: UserListComponent;
  let fixture: ComponentFixture<UserListComponent>;
  let mockUserService: jasmine.SpyObj<UserService>;

  beforeEach(() => {
    // ★ 依存関係をモック化
    mockUserService = jasmine.createSpyObj('UserService', ['getUsers']);

    TestBed.configureTestingModule({
      declarations: [UserListComponent],
      providers: [
        { provide: UserService, useValue: mockUserService }
      ]
    });

    fixture = TestBed.createComponent(UserListComponent);
    component = fixture.componentInstance;
  });

  it('should load users on init', () => {
    const mockUsers = [
      { id: 1, name: 'Alice' },
      { id: 2, name: 'Bob' }
    ];
    mockUserService.getUsers.and.returnValue(of(mockUsers));

    // ★ ngOnInitを発火
    fixture.detectChanges();

    expect(component.users).toEqual(mockUsers);
    expect(mockUserService.getUsers).toHaveBeenCalled();
  });
});
```

---

## 実務で本当に重要な3つの柱（詳細）

### 柱1: 依存関係の制御（最重要）

**なぜ重要か:**
- 外部APIを呼ばずにテスト
- データベースに接続せずにテスト
- 他のコンポーネント/サービスに依存せずにテスト

**実践パターン:**

```typescript
// パターン1: Serviceのモック
const mockService = jasmine.createSpyObj('MyService', ['method1', 'method2']);
mockService.method1.and.returnValue(of('mocked data'));

// パターン2: HttpClientのモック
TestBed.configureTestingModule({
  imports: [HttpClientTestingModule]
});
const httpMock = TestBed.inject(HttpTestingController);

// パターン3: 子コンポーネントのモック
@Component({ 
  selector: 'app-child', 
  template: '' 
})
class MockChildComponent {
  @Input() data: any;
}

TestBed.configureTestingModule({
  declarations: [ParentComponent, MockChildComponent]
});
```

### 柱2: 変更検知の管理

**いつ`detectChanges()`を呼ぶか:**

```typescript
it('change detection timing', () => {
  // タイミング1: 初期化時
  fixture.detectChanges(); // → ngOnInit実行

  // タイミング2: プロパティ変更後
  component.value = 'new value';
  fixture.detectChanges(); // → ビュー更新

  // タイミング3: イベント発火後
  button.click();
  fixture.detectChanges(); // → イベントハンドラ実行後のビュー更新

  // タイミング4: 非同期処理後
  tick(); // 非同期完了
  fixture.detectChanges(); // → 結果のビュー反映
});
```

**detectChanges()の自動化:**

```typescript
// 自動化したい場合
beforeEach(() => {
  fixture = TestBed.createComponent(MyComponent);
  component = fixture.componentInstance;
  fixture.autoDetectChanges(); // ← 自動変更検知ON
});

// 細かく制御したい場合
beforeEach(() => {
  fixture = TestBed.createComponent(MyComponent);
  component = fixture.componentInstance;
  // detectChanges()を手動で呼ぶ
});
```

### 柱3: 非同期処理のハンドリング

**3つのパターン:**

```typescript
// パターン1: done() - 古いスタイル
it('async with done', (done) => {
  service.getData().subscribe(data => {
    expect(data).toBeDefined();
    done();
  });
});

// パターン2: fakeAsync/tick - 推奨
it('async with fakeAsync', fakeAsync(() => {
  let result: any;
  service.getData().subscribe(d => result = d);
  
  tick(); // 非同期処理を「進める」
  
  expect(result).toBeDefined();
}));

// パターン3: async/whenStable
it('async with async', async(() => {
  fixture.detectChanges();
  
  fixture.whenStable().then(() => {
    expect(component.data).toBeDefined();
  });
}));
```

---

## テスト対象別チェックリスト

### Pipe のテスト

```typescript
✅ new でインスタンス化
✅ transform() メソッドをテスト
✅ 各引数パターンをテスト
❌ TestBed 不要（Pure Pipeの場合）
```

### Service のテスト

```typescript
✅ 依存サービスをモック化
✅ TestBed.configureTestingModule
✅ TestBed.inject でインスタンス取得
✅ HttpClientTestingModule（HTTP使用時）
❌ fixture 不要
❌ detectChanges 不要
```

### Component のテスト

```typescript
✅ TestBed.configureTestingModule
✅ TestBed.createComponent
✅ fixture.detectChanges()
✅ 依存サービスをモック化
✅ 子コンポーネントをモック化（必要に応じて）
✅ DOMアクセス（fixture.nativeElement）
✅ 非同期処理のハンドリング
```

### Directive のテスト

```typescript
✅ ホストコンポーネントを作成
✅ fixture.detectChanges()
✅ DebugElement.query でディレクティブ検索
✅ 属性変更のテスト
```

---

## 100%カバレッジ達成のための戦略マップ

### Phase 1: 簡単なものから（20%の労力で80%達成）

```typescript
// 1. Pure Pipe → 最優先
describe('MyPipe', () => {
  it('should work', () => {
    const pipe = new MyPipe();
    expect(pipe.transform(input)).toBe(expected);
  });
});

// 2. 依存なしService → 次に簡単
describe('UtilService', () => {
  it('should work', () => {
    const service = new UtilService();
    expect(service.calculate(1, 2)).toBe(3);
  });
});

// 3. 単純なComponent → 基本パターン確立
describe('SimpleComponent', () => {
  beforeEach(() => {
    fixture = TestBed.createComponent(SimpleComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should create', () => {
    expect(component).toBeTruthy();
  });
});
```

### Phase 2: 依存関係のパターン化（残り80%達成）

```typescript
// パターンA: Service依存Component
beforeEach(() => {
  mockService = jasmine.createSpyObj('Service', ['method']);
  
  TestBed.configureTestingModule({
    declarations: [MyComponent],
    providers: [{ provide: MyService, useValue: mockService }]
  });
});

// パターンB: HTTP依存Service
beforeEach(() => {
  TestBed.configureTestingModule({
    imports: [HttpClientTestingModule],
    providers: [MyService]
  });
  
  httpMock = TestBed.inject(HttpTestingController);
});

// パターンC: Router依存Component
beforeEach(() => {
  TestBed.configureTestingModule({
    declarations: [MyComponent],
    providers: [
      { provide: Router, useValue: mockRouter },
      { provide: ActivatedRoute, useValue: mockActivatedRoute }
    ]
  });
});
```

---

## あなたの状況での優先順位

### マイクロフロントエンド + レガシーコード修正

**最優先で習得すべき3つ:**

1. **依存関係のモック化**
   ```typescript
   const mock = jasmine.createSpyObj('Service', ['methods']);
   { provide: RealService, useValue: mock }
   ```
   **理由:** マイクロフロントエンド間の依存を分離するため

2. **fixture.detectChanges()のタイミング**
   ```typescript
   fixture.detectChanges(); // 初期化
   component.prop = 'new';
   fixture.detectChanges(); // 更新
   ```
   **理由:** コンポーネントテストの基本

3. **非同期処理（fakeAsync/tick）**
   ```typescript
   fakeAsync(() => {
     fixture.detectChanges();
     tick();
     expect(component.data).toBeDefined();
   })
   ```
   **理由:** API呼び出しが多いビジネスアプリケーションのため

---

## まとめ: テストで本当に必要な要素

### 絶対必要（これだけは必須）

1. **依存関係の制御** - モック化、DI設定
2. **変更検知の理解** - fixture.detectChanges()
3. **TestBedの基本** - configureTestingModule, createComponent

### あると便利（徐々に習得）

4. **非同期処理** - fakeAsync, tick, async
5. **DOM操作** - nativeElement, debugElement, By.css
6. **ライフサイクル** - 問題が起きた時に学ぶ

### 知らなくてもOK（必要になったら）

7. **高度なテスト技法** - Marble Testing, Component Harness
8. **E2Eテスト** - Protractor/Cypress（単体テストと別物）
9. **カスタムマッチャー** - Jasmineの拡張

**結論:** Angularのテストは複雑に見えますが、**依存関係・変更検知・TestBedの3つ**を理解すれば、どんなコードもテストできます。あなたの100%カバレッジ目標には、この3つの習得で十分です。
