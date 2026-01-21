# provider_shopper: 処理フローと状態管理の完全ガイド

provider_shopperアプリの処理フローと状態管理の仕組みを包括的にまとめたドキュメントです。

---

## 📱 アプリ概要

**目的**: Providerパッケージを使った状態管理のデモアプリ

**機能**:
- 商品カタログの閲覧（無限スクロール）
- ショッピングカートへの追加/削除
- 合計金額の計算

**画面構成**:
1. ログイン画面（Login Screen）
2. カタログ画面（Catalog Screen）
3. カート画面（Cart Screen）

---

## 🏗️ アーキテクチャ全体図

```
┌─────────────────────────────────────────────────────┐
│                   main.dart                         │
│                                                     │
│  MultiProvider ← ここで状態を登録                    │
│    ├─ Provider<CatalogModel>                        │
│    └─ ChangeNotifierProxyProvider<CartModel>       │
│                     ↓                               │
│              MaterialApp.router                     │
└─────────────────────────────────────────────────────┘
                      ↓
        ┌─────────────┴─────────────┐
        ↓                           ↓
┌──────────────┐           ┌──────────────┐
│ models/      │           │ screens/     │
│              │           │              │
│ catalog.dart │←─────────→│ login.dart   │
│ cart.dart    │           │ catalog.dart │
│              │           │ cart.dart    │
└──────────────┘           └──────────────┘
```

---

## 📦 状態管理の構造

### 1. モデル層（lib/models/）

#### CatalogModel（商品カタログ）

```dart
class CatalogModel {
  static List<String> itemNames = [
    'Code Smell', 'Control Flow', 'Interpreter', ...
  ];

  // IDから商品を取得
  Item getById(int id) => Item(id, itemNames[id % itemNames.length]);
}
```

**特徴**:
- ✅ **不変データ** → `ChangeNotifier`を継承しない
- ✅ 15種類の商品名を無限ループで提供
- ✅ 通常の`Provider`で提供

**なぜChangeNotifierじゃない？**
→ カタログは起動後に変わらないから

---

#### CartModel（ショッピングカート）

```dart
class CartModel extends ChangeNotifier {
  late CatalogModel _catalog;  // Catalogに依存
  final List<int> _itemIds = [];  // カート内の商品ID

  // Catalogへの依存を設定
  set catalog(CatalogModel newCatalog) {
    _catalog = newCatalog;
    notifyListeners();
  }

  // 計算プロパティ
  List<Item> get items =>
    _itemIds.map((id) => _catalog.getById(id)).toList();

  int get totalPrice =>
    items.fold(0, (total, current) => total + current.price);

  // カートに追加
  void add(Item item) {
    _itemIds.add(item.id);
    notifyListeners();  // ← UI更新をトリガー
  }

  // カートから削除
  void remove(Item item) {
    _itemIds.remove(item.id);
    notifyListeners();  // ← UI更新をトリガー
  }
}
```

**特徴**:
- ✅ **可変データ** → `ChangeNotifier`を継承
- ✅ CatalogModelに依存（ProxyProviderで注入）
- ✅ IDだけを保存、商品情報はCatalogから取得
- ✅ `notifyListeners()`でUI自動更新

**なぜIDだけ保存？**
→ Item自体はCatalogから毎回生成されるから、IDで管理

---

#### Item（商品データ）

```dart
@immutable
class Item {
  final int id;
  final String name;
  final Color color;
  final int price = 42;  // 全商品42ドル

  Item(this.id, this.name)
    : color = Colors.primaries[id % Colors.primaries.length];

  @override
  int get hashCode => id;

  @override
  bool operator ==(Object other) => other is Item && other.id == id;
}
```

**特徴**:
- ✅ 不変（@immutable）
- ✅ `==`をオーバーライド → IDで等価性判定
- ✅ `hashCode`もオーバーライド → Set/Mapで正しく動作

---

### 2. Providerの設定（lib/main.dart）

```dart
MultiProvider(
  providers: [
    // 1️⃣ CatalogModel（不変）
    Provider(create: (context) => CatalogModel()),

    // 2️⃣ CartModel（可変 + Catalogに依存）
    ChangeNotifierProxyProvider<CatalogModel, CartModel>(
      create: (context) => CartModel(),
      update: (context, catalog, cart) {
        if (cart == null) throw ArgumentError.notNull('cart');
        cart.catalog = catalog;  // ← Catalogを注入
        return cart;
      },
    ),
  ],
  child: MaterialApp.router(...),
)
```

**ポイント**:

| Provider種類 | 使用モデル | 理由 |
|-------------|----------|------|
| `Provider` | CatalogModel | 不変データ、依存なし |
| `ChangeNotifierProxyProvider` | CartModel | 可変データ、Catalogに依存 |

**ProxyProviderの動作**:
1. `CatalogModel`が先に作られる
2. `CartModel`が作られる
3. `update`が呼ばれ、`cart.catalog = catalog`でCatalogを注入
4. CartModelが正しく動作可能に

---

## 🔄 アプリ起動フロー

```
1. main() 実行
   ↓
2. MultiProvider作成
   ├─ Provider: CatalogModel を1個作成
   │   └─ itemNames を準備
   └─ ChangeNotifierProxyProvider: CartModel を1個作成
       └─ update でCatalogを注入
   ↓
3. MaterialApp.router 起動
   ↓
4. GoRouter が初期ルート決定
   ↓ /login
5. MyLogin 画面表示
```

**重要**: モデルは**アプリ全体で1個だけ**作られる

---

## 📱 各画面の処理フロー

### 1️⃣ ログイン画面（login.dart）

```dart
class MyLogin extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: Column(
          children: [
            Text('Welcome'),
            TextFormField(decoration: InputDecoration(hintText: 'Username')),
            TextFormField(
              decoration: InputDecoration(hintText: 'Password'),
              obscureText: true,
            ),
            ElevatedButton(
              onPressed: () => context.pushReplacement('/catalog'),
              child: const Text('ENTER'),
            ),
          ],
        ),
      ),
    );
  }
}
```

**特徴**:
- ✅ Providerを一切使わない
- ✅ 認証機能なし（デモ用）
- ✅ ENTERボタンでカタログ画面へ遷移

**フロー**:
```
1. ユーザーが ENTER タップ
   ↓
2. context.pushReplacement('/catalog')
   ↓
3. GoRouter がカタログ画面を表示
```

---

### 2️⃣ カタログ画面（catalog.dart）

#### 全体構造

```dart
MyCatalog
  └─ CustomScrollView
      ├─ _MyAppBar（ヘッダー）
      └─ SliverList（無限リスト）
          └─ _MyListItem × 無限
              └─ _AddButton
```

#### _MyAppBar（ヘッダー）

```dart
class _MyAppBar extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return SliverAppBar(
      title: Text('Catalog'),
      floating: true,
      actions: [
        IconButton(
          icon: const Icon(Icons.shopping_cart),
          onPressed: () => context.go('/catalog/cart'),  // カート画面へ
        ),
      ],
    );
  }
}
```

**特徴**:
- ✅ Providerを使わない（静的UI）
- ✅ カートアイコンでカート画面へ遷移

---

#### _MyListItem（商品アイテム）

```dart
class _MyListItem extends StatelessWidget {
  final int index;

  @override
  Widget build(BuildContext context) {
    // ✅ context.select でCatalogから商品取得
    var item = context.select<CatalogModel, Item>(
      (catalog) => catalog.getByPosition(index),
    );

    return Row(
      children: [
        Container(color: item.color),  // 色付き四角
        Text(item.name),                // 商品名
        _AddButton(item: item),         // ADDボタン
      ],
    );
  }
}
```

**ポイント**:
- `context.select<CatalogModel, Item>` を使用
- Catalogから商品情報を取得
- 商品名が変わったら再描画（実際には変わらない）

---

#### _AddButton（ADDボタン）**⭐ 最重要！**

```dart
class _AddButton extends StatelessWidget {
  final Item item;

  @override
  Widget build(BuildContext context) {
    // ✅ context.select でカート状態を監視
    var isInCart = context.select<CartModel, bool>(
      (cart) => cart.items.contains(item),
    );

    return TextButton(
      onPressed: isInCart ? null : () {
        // ✅ context.read でカートを取得（監視なし）
        var cart = context.read<CartModel>();
        cart.add(item);
      },
      child: isInCart ? Icon(Icons.check) : Text('ADD'),
    );
  }
}
```

**処理フロー**:

```
【初期表示時】
1. context.select<CartModel, bool>(...) 実行
   ↓
2. Provider が CartModel を探す
   ↓
3. cart.items.contains(item) を評価
   ↓
4. 結果: false（カートは空）
   ↓
5. ボタン表示: [ADD]

【ADDボタンタップ時】
1. onPressed コールバック実行
   ↓
2. context.read<CartModel>() でカート取得
   ↓
3. cart.add(item) 実行
   ↓
4. CartModel 内部:
   _itemIds.add(item.id)
   notifyListeners()  ← 全リスナーに通知
   ↓
5. Provider が全 select を再評価
   ↓
6. このボタンの select:
   cart.items.contains(item)
   false → true に変化！
   ↓
7. build() が再実行される
   ↓
8. ボタン表示: [✓]（無効化）
```

**重要ポイント**:
- `select`: 値を監視 → 変わったら再描画
- `read`: 監視なし → ただ取得して実行

---

### 3️⃣ カート画面（cart.dart）

#### 全体構造

```dart
MyCart
  └─ Scaffold
      └─ Column
          ├─ _CartList（上部: 商品リスト）
          └─ _CartTotal（下部: 合計金額）
```

#### _CartList（商品リスト）

```dart
class _CartList extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    // ✅ context.watch でカート全体を監視
    var cart = context.watch<CartModel>();

    return ListView.builder(
      itemCount: cart.items.length,
      itemBuilder: (context, index) => ListTile(
        leading: Icon(Icons.done),
        title: Text(cart.items[index].name),
        trailing: IconButton(
          icon: Icon(Icons.remove_circle_outline),
          onPressed: () {
            cart.remove(cart.items[index]);  // 削除
          },
        ),
      ),
    );
  }
}
```

**ポイント**:
- `context.watch<CartModel>()` を使用
- カートの変更があれば**常に**再描画
- `items`、`remove()`など複数のプロパティ/メソッドを使用

**なぜwatch？**
→ 商品が追加/削除されたら、リスト全体を更新する必要があるから

---

#### _CartTotal（合計金額）

```dart
class _CartTotal extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Row(
      children: [
        // ✅ Consumer で合計金額だけ監視
        Consumer<CartModel>(
          builder: (context, cart, child) =>
            Text('\$${cart.totalPrice}', style: hugeStyle),
        ),
        FilledButton(
          onPressed: () {
            ScaffoldMessenger.of(context).showSnackBar(
              SnackBar(content: Text('Buying not supported yet.')),
            );
          },
          child: Text('BUY'),
        ),
      ],
    );
  }
}
```

**ポイント**:
- `Consumer<CartModel>` を使用
- `Consumer`の**中だけ**が再描画される
- `BUY`ボタンは再描画されない（効率的）

**Consumerの動作**:
```
商品追加
  ↓
notifyListeners()
  ↓
Consumerのbuilder が再実行
  ↓
Text('\$${cart.totalPrice}') だけ更新
  ↓
BUYボタンは再描画されない ✅
```

---

## 🎯 Providerの使い分け完全ガイド

### context.read() - 取得のみ（監視なし）

```dart
// コールバック内で使う
onPressed: () {
  var cart = context.read<CartModel>();
  cart.add(item);
}
```

**特徴**:
- ❌ 監視しない
- ❌ 再描画しない
- ✅ 値を変更する時に使う
- ✅ コールバック内で使う

**使用場所**: catalog.dart:58（_AddButton）

---

### context.select() - 値を監視（効率的）

```dart
// build内で使う
var isInCart = context.select<CartModel, bool>(
  (cart) => cart.items.contains(item),
);
```

**特徴**:
- ✅ 選択した値だけ監視
- ✅ 値が変わった時だけ再描画
- ✅ パフォーマンス最適化
- ✅ build内で使う

**使用場所**:
- catalog.dart:45（_AddButton - カート状態）
- catalog.dart:102（_MyListItem - 商品情報）

**再描画の仕組み**:
```
notifyListeners()
  ↓
全selectを再評価
  ↓
商品#5のボタン:
  前回: false
  今回: false
  → 変化なし、再描画しない ✅

商品#3のボタン:
  前回: false
  今回: true
  → 変化あり、再描画！✅
```

---

### context.watch() - モデル全体を監視

```dart
// build内で使う
var cart = context.watch<CartModel>();
print(cart.items);
print(cart.totalPrice);
cart.remove(item);
```

**特徴**:
- ✅ モデル全体を監視
- ✅ notifyListeners()で必ず再描画
- ✅ 複数のプロパティを使う時に便利
- ✅ build内で使う

**使用場所**: cart.dart:48（_CartList）

**いつ使う？**
- 複数のプロパティを使う
- リスト全体を表示する
- selectだと面倒な時

---

### Consumer - 部分的な再描画

```dart
Consumer<CartModel>(
  builder: (context, cart, child) =>
    Text('\$${cart.totalPrice}'),
)
```

**特徴**:
- ✅ Consumerの中だけ再描画
- ✅ 周りのウィジェットは再描画されない
- ✅ パフォーマンス最適化

**使用場所**: cart.dart:85（_CartTotal）

---

### 使い分け表

| 方法 | 監視 | 再描画 | 使う場所 | 用途 |
|------|------|--------|---------|------|
| **read()** | ❌ | ❌ | コールバック | 値を変更 |
| **select()** | ✅ 値のみ | 値変更時 | build | 1つの値を表示 |
| **watch()** | ✅ 全体 | 常に | build | 複数の値を表示 |
| **Consumer** | ✅ 全体 | 常に | build | 部分的に再描画 |

---

## 🔄 状態変更の完全フロー

### シナリオ: 商品#5をカートに追加

```
【ステップ1: ユーザー操作】
ユーザーが商品#5の[ADD]ボタンをタップ

【ステップ2: イベント発火】
_AddButton.onPressed() 実行
  ↓
context.read<CartModel>() でカート取得
  ↓
cart.add(Item(5, 'Recursion')) 実行

【ステップ3: CartModel内部】
CartModel.add() 内:
  _itemIds.add(5)
  // _itemIds = [5]
  notifyListeners()  ← 全リスナーに通知！

【ステップ4: Provider の反応】
Providerシステム:
  「notifyListenersが呼ばれた！」
  「全てのselect/watch/Consumerを再評価しよう」

【ステップ5: カタログ画面のボタン再評価】
商品#1のボタン:
  select: cart.items.contains(Item(1))
  結果: false → false（変化なし）
  → 再描画しない

商品#5のボタン:
  select: cart.items.contains(Item(5))
  結果: false → true（変化あり！）
  → build()再実行！

商品#7のボタン:
  select: cart.items.contains(Item(7))
  結果: false → false（変化なし）
  → 再描画しない

【ステップ6: 商品#5のボタン再描画】
_AddButton.build() 実行:
  isInCart = true
  ↓
TextButton表示:
  child: Icon(Icons.check)  ← [✓]に変わる
  onPressed: null  ← 無効化

【ステップ7: カート画面（もし開いていたら）】
_CartList (watch使用):
  notifyListeners() → 必ず再描画
  ListView.builder が再構築
  itemCount: 1
  商品#5が表示される

_CartTotal (Consumer使用):
  Consumer内だけ再描画
  Text('\$42') → 合計金額更新
  BUYボタンは再描画されない ✅
```

---

## 📊 データの流れ図

```
┌─────────────────────────────────────────────┐
│           Provider (DIコンテナ)              │
│                                             │
│  CatalogModel (不変)                        │
│  ├─ itemNames: ['Code Smell', ...]         │
│  └─ getById(id) → Item                     │
│                    ↓ 依存                   │
│  CartModel (可変)                           │
│  ├─ _catalog: CatalogModel  ← 注入          │
│  ├─ _itemIds: [1, 5, 7]                    │
│  ├─ items: List<Item>  ← Catalogから取得   │
│  ├─ totalPrice: int                         │
│  ├─ add(item)                               │
│  └─ remove(item)                            │
│                                             │
└─────────────────────────────────────────────┘
         ↓ read/select/watch/Consumer
┌─────────────────────────────────────────────┐
│              UI Layer                       │
│                                             │
│  Catalog Screen                             │
│    ├─ _AddButton (select)                  │
│    │   └─ isInCart                          │
│    └─ _MyListItem (select)                 │
│        └─ item                              │
│                                             │
│  Cart Screen                                │
│    ├─ _CartList (watch)                    │
│    │   └─ cart.items                        │
│    └─ _CartTotal (Consumer)                │
│        └─ cart.totalPrice                   │
│                                             │
└─────────────────────────────────────────────┘
```

---

## 🎓 重要な設計パターン

### 1. 依存性注入（Dependency Injection）

```dart
// CartModelはCatalogModelに依存
ChangeNotifierProxyProvider<CatalogModel, CartModel>(
  update: (context, catalog, cart) {
    cart!.catalog = catalog;  // ← 依存を注入
    return cart;
  },
)
```

**メリット**:
- テストしやすい
- 疎結合
- 実装を入れ替え可能

---

### 2. 単一責任の原則

```
CatalogModel: 商品情報の提供
CartModel: カートの管理
Item: 商品データの保持
```

各クラスが1つの責任だけを持つ

---

### 3. 不変性（Immutability）

```dart
@immutable
class Item {
  final int id;
  final String name;
  final Color color;
  final int price = 42;
}
```

**メリット**:
- 予期せぬ変更を防ぐ
- 並行処理で安全
- デバッグしやすい

---

### 4. リアクティブプログラミング

```dart
notifyListeners()  // 変更を通知
  ↓
UI自動更新  // リアクティブに反応
```

**メリット**:
- UIと状態が自動で同期
- 手動でsetStateする必要なし

---

## 💡 まとめ

### Providerの役割

```
Provider = 依存性注入コンテナ + 状態管理

機能:
1. モデルの登録・管理
2. 依存関係の解決
3. ライフサイクル管理
4. 変更通知の配信
5. スコープ管理
```

### 状態管理のフロー

```
1. ユーザー操作
   ↓
2. イベントハンドラ (context.read)
   ↓
3. モデル更新 (add/remove)
   ↓
4. notifyListeners()
   ↓
5. Provider が検知
   ↓
6. select/watch/Consumer 再評価
   ↓
7. 変更があったウィジェットだけ再描画
   ↓
8. UI更新完了
```

### 使い分けの原則

```
値を変更する → read()
1つの値を表示 → select()
複数の値を表示 → watch()
部分的に再描画 → Consumer
```

### provider_shopperの設計思想

```
✅ シンプル: 最小限のコードで最大の学習効果
✅ 実用的: 実際のアプリで使えるパターン
✅ 効率的: パフォーマンス最適化の実例
✅ 拡張可能: DBやAPIと組み合わせ可能
```

---

このアプリは小さいですが、Providerの全ての重要概念が詰まっています。この理解があれば、大規模なFlutterアプリでも状態管理ができます！
