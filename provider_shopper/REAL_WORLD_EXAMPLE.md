# 実際のアプリでのProvider + DB構成例

このサンプルは教育目的でProviderのみを使っていますが、実際のアプリではバックエンド（DB）と組み合わせます。

## 構成

```
Flutter App (クライアント)
  ├─ Provider (メモリ上の一時状態)
  │   └─ CartModel
  │       - 即座にUI更新
  │       - アプリ起動中のみ有効
  │
  └─ API Client
      ↕ HTTP通信

Backend (サーバー)
  └─ Database (永続化)
      └─ carts テーブル
          - アプリ終了後も保持
          - 複数端末で同期
```

## 実装例

### 1. API Client

```dart
// lib/services/cart_api.dart
class CartApi {
  final String baseUrl = 'https://api.example.com';

  // カート取得
  Future<CartResponse> getCart() async {
    final response = await http.get(Uri.parse('$baseUrl/cart'));
    return CartResponse.fromJson(jsonDecode(response.body));
  }

  // カートに追加
  Future<void> addToCart(int itemId) async {
    await http.post(
      Uri.parse('$baseUrl/cart/items'),
      body: jsonEncode({'item_id': itemId}),
      headers: {'Content-Type': 'application/json'},
    );
  }

  // カートから削除
  Future<void> removeFromCart(int itemId) async {
    await http.delete(Uri.parse('$baseUrl/cart/items/$itemId'));
  }
}

class CartResponse {
  final List<int> itemIds;

  CartResponse({required this.itemIds});

  factory CartResponse.fromJson(Map<String, dynamic> json) {
    return CartResponse(
      itemIds: List<int>.from(json['item_ids']),
    );
  }
}
```

### 2. Provider + DBを組み合わせたCartModel

```dart
// lib/models/cart.dart
class CartModel extends ChangeNotifier {
  late CatalogModel _catalog;
  final List<int> _itemIds = [];
  final CartApi _api;

  bool _isLoading = false;
  String? _error;

  CartModel({required CartApi api}) : _api = api;

  CatalogModel get catalog => _catalog;
  set catalog(CatalogModel newCatalog) {
    _catalog = newCatalog;
    notifyListeners();
  }

  List<Item> get items =>
    _itemIds.map((id) => _catalog.getById(id)).toList();

  int get totalPrice =>
    items.fold(0, (total, current) => total + current.price);

  bool get isLoading => _isLoading;
  String? get error => _error;

  // アプリ起動時: DBから読み込み
  Future<void> loadFromBackend() async {
    _isLoading = true;
    _error = null;
    notifyListeners();

    try {
      final response = await _api.getCart();
      _itemIds.clear();
      _itemIds.addAll(response.itemIds);
      _error = null;
    } catch (e) {
      _error = 'カートの読み込みに失敗しました: $e';
    } finally {
      _isLoading = false;
      notifyListeners();
    }
  }

  // 商品追加: Optimistic Update パターン
  Future<void> add(Item item) async {
    // 1. まずローカル（Provider）で即更新
    _itemIds.add(item.id);
    notifyListeners();  // UI即座に更新！

    // 2. バックグラウンドでDBに保存
    try {
      await _api.addToCart(item.id);
    } catch (e) {
      // 失敗したら元に戻す（ロールバック）
      _itemIds.remove(item.id);
      _error = '追加に失敗しました: $e';
      notifyListeners();
    }
  }

  // 商品削除: 同じパターン
  Future<void> remove(Item item) async {
    // 1. まずローカルで即更新
    _itemIds.remove(item.id);
    notifyListeners();

    // 2. バックグラウンドでDBから削除
    try {
      await _api.removeFromCart(item.id);
    } catch (e) {
      // 失敗したら元に戻す
      _itemIds.add(item.id);
      _error = '削除に失敗しました: $e';
      notifyListeners();
    }
  }
}
```

### 3. main.dartでの初期化

```dart
// lib/main.dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // API Clientの作成
  final cartApi = CartApi();

  // CartModelの作成と初期化
  final cartModel = CartModel(api: cartApi);

  // DBからデータ読み込み（アプリ起動時）
  await cartModel.loadFromBackend();

  runApp(
    MultiProvider(
      providers: [
        Provider(create: (context) => CatalogModel()),
        ChangeNotifierProvider.value(value: cartModel),
      ],
      child: const MyApp(),
    ),
  );
}
```

### 4. UIでの使用

```dart
// lib/screens/catalog.dart
class _AddButton extends StatelessWidget {
  final Item item;

  @override
  Widget build(BuildContext context) {
    // カートの状態を監視
    var isInCart = context.select<CartModel, bool>(
      (cart) => cart.items.contains(item),
    );

    // エラー状態も監視
    var error = context.select<CartModel, String?>(
      (cart) => cart.error,
    );

    // エラー表示
    if (error != null) {
      WidgetsBinding.instance.addPostFrameCallback((_) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text(error)),
        );
      });
    }

    return TextButton(
      onPressed: isInCart ? null : () async {
        var cart = context.read<CartModel>();
        await cart.add(item);  // 非同期だけどUIは即更新される
      },
      child: isInCart ? Icon(Icons.check) : Text('ADD'),
    );
  }
}
```

## バックエンド例（参考）

### Node.js + PostgreSQL

```javascript
// server.js
const express = require('express');
const { Pool } = require('pg');

const app = express();
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
});

app.use(express.json());

// カート取得
app.get('/cart', async (req, res) => {
  const userId = req.user.id;  // 認証から取得
  const result = await pool.query(
    'SELECT item_ids FROM carts WHERE user_id = $1',
    [userId]
  );

  if (result.rows.length === 0) {
    return res.json({ item_ids: [] });
  }

  res.json({ item_ids: result.rows[0].item_ids });
});

// カートに追加
app.post('/cart/items', async (req, res) => {
  const userId = req.user.id;
  const { item_id } = req.body;

  await pool.query(
    `INSERT INTO carts (user_id, item_ids, updated_at)
     VALUES ($1, ARRAY[$2], NOW())
     ON CONFLICT (user_id)
     DO UPDATE SET
       item_ids = array_append(carts.item_ids, $2),
       updated_at = NOW()`,
    [userId, item_id]
  );

  res.json({ success: true });
});

// カートから削除
app.delete('/cart/items/:itemId', async (req, res) => {
  const userId = req.user.id;
  const itemId = parseInt(req.params.itemId);

  await pool.query(
    `UPDATE carts
     SET item_ids = array_remove(item_ids, $2),
         updated_at = NOW()
     WHERE user_id = $1`,
    [userId, itemId]
  );

  res.json({ success: true });
});

app.listen(3000);
```

### データベーススキーマ

```sql
-- PostgreSQL
CREATE TABLE carts (
  user_id INTEGER PRIMARY KEY,
  item_ids INTEGER[],
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_carts_user_id ON carts(user_id);
```

## 重要なパターン

### 1. Optimistic Update（楽観的更新）

```dart
// まずUIを更新（楽観的に成功すると仮定）
_itemIds.add(item.id);
notifyListeners();

// 後でAPIに送信
try {
  await _api.addToCart(item.id);
} catch (e) {
  // 失敗したら元に戻す
  _itemIds.remove(item.id);
  notifyListeners();
}
```

**メリット**: UIが即座に反応 = ユーザー体験が良い

### 2. キャッシュファースト

```dart
// 1. まずキャッシュ（Provider）から表示
var items = cart.items;

// 2. バックグラウンドで最新データを取得
cart.loadFromBackend();

// 3. 取得完了したら自動的にUI更新
```

**メリット**: オフラインでも動作、即座に表示

### 3. エラーハンドリング

```dart
try {
  await _api.addToCart(item.id);
} catch (e) {
  if (e is NetworkException) {
    _error = 'ネットワークエラー: 後で再試行します';
    // ローカルキューに保存して後で再送
  } else {
    _error = 'エラーが発生しました';
  }
  notifyListeners();
}
```

## まとめ

| 要素 | 役割 | データの寿命 |
|------|------|-------------|
| **Provider** | UI状態管理、キャッシュ | アプリ起動中のみ |
| **Database** | 永続化、複数端末同期 | 永続的 |
| **API** | Provider ↔ DB の橋渡し | - |

**実際のアプリ**: Provider（速い）+ DB（永続）= 最高のUX
