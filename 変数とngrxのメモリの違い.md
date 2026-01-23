# 変数とステート管理のメモリ・GCの違い

## 結論

**保存領域(ヒープメモリ)は同じ。違いは参照の保持方法とGC対象になるタイミング。**

## 比較表

| 項目 | コンポーネントローカル変数 | NgRx Store |
|------|------------------------|-----------|
| **メモリ領域** | JavaScriptヒープ | JavaScriptヒープ |
| **参照の管理** | コンポーネントインスタンス経由 | ルートインジェクター経由 |
| **ライフサイクル** | コンポーネントと同期 | アプリケーション全体 |
| **GC対象** | コンポーネント破棄時 | アプリ終了まで対象外 |
| **ページ遷移後** | メモリ解放 | メモリ保持 |

## 詳細なメモリ動作

### 1. コンポーネントローカル変数

```typescript
@Component({...})
class UserComponent implements OnDestroy {
  private users: User[] = []; // ①
  
  ngOnDestroy() {
    // ②この時点でusersへの参照が切れる準備
  }
}
// ③コンポーネント破棄 → GC対象 → メモリ解放
```

**参照チェーン**: 
```
ルート → コンポーネントツリー → UserComponent → users配列
                ↓破棄
              参照消失 → GC
```

### 2. NgRx Store

```typescript
@Injectable({ providedIn: 'root' })
class Store<T> {
  private _state: T; // ①シングルトン
  
  constructor() {
    // ②ルートインジェクターに登録
  }
}
```

**参照チェーン**:
```
ルートインジェクター → Store → _state
        ↓常に保持
   参照維持 → GC対象外
```

## V8エンジンのメモリ管理

### メモリ空間の構造

```
[V8 Heap]
├─ New Space (若い世代)
│   └─ 短命なオブジェクト
│       - コンポーネントローカル変数
│       - 一時的なオブジェクト
│
└─ Old Space (古い世代)
    └─ 長期生存オブジェクト
        - Storeの状態
        - シングルトンサービス
```

### GCのタイミング

1. **Scavenge GC** (New Space): 頻繁に実行、コンポーネント変数を回収
2. **Mark-Sweep GC** (Old Space): 低頻度、Storeは対象外

## 実践例

```typescript
// ケース1: メモリリーク(意図せず参照保持)
@Component({...})
class LeakComponent {
  private subscription: Subscription;
  
  ngOnInit() {
    this.subscription = this.store.select(...).subscribe();
    // ngOnDestroyでunsubscribeしないと参照が残る
  }
}

// ケース2: 適切な管理
@Component({...})
class GoodComponent {
  users$ = this.store.select(selectUsers); // Observableで管理
  
  // async pipeが自動的にunsubscribe
}
```

## 公式ドキュメント

### JavaScriptエンジン・メモリ管理
- **V8 メモリ管理**: https://v8.dev/blog/trash-talk
- **MDN - メモリ管理**: https://developer.mozilla.org/ja/docs/Web/JavaScript/Memory_management
- **Chrome DevTools メモリプロファイリング**: https://developer.chrome.com/docs/devtools/memory-problems/

### Angular
- **Angular ライフサイクルフック**: https://angular.dev/guide/components/lifecycle
- **Angular 依存性注入**: https://angular.dev/guide/di
- **Angular メモリリーク対策**: https://angular.dev/best-practices/runtime-performance

### NgRx
- **NgRx 公式ドキュメント**: https://ngrx.io/
- **NgRx Store**: https://ngrx.io/guide/store
- **NgRx ベストプラクティス**: https://ngrx.io/guide/store/best-practices

### TypeScript
- **TypeScript ハンドブック**: https://www.typescriptlang.org/docs/handbook/intro.html

---

**重要なポイント**: ステート管理ツールを使う主な理由は「メモリの場所が違うから」ではなく、「**アプリケーション全体で状態を共有し、予測可能な方法で管理するため**」です。メモリの保持はその副次的な効果です。
