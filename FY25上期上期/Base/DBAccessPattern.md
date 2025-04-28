# LaravelのDBアクセスパターン

## 1. 導入

Laravelを使用してWebアプリケーションを開発していると、データベースへのアクセス方法として複数の選択肢があることに気づきます。Eloquent ORMを使うこともあれば、Query Builderを使うこともあり、時にはRaw SQLを直接書くこともあるでしょう。

```php
// Eloquent ORMの例
$users = User::where('active', 1)->get();

// Query Builderの例
$users = DB::table('users')->where('active', 1)->get();

// Raw SQLの例
$users = DB::select('SELECT * FROM users WHERE active = ?', [1]);
```

これらのアプローチは一見すると単なる好みの問題のように思えますが、実際には各アプローチには長所と短所があり、適切な状況で適切なパターンを選択することが重要です。しかし、多くの開発者はこれらのパターンの使い分けを意識せずに開発を進めています。

本資料では、Martin Fowlerの「Patterns of Enterprise Application Architecture」（PoEAA）で紹介されているData Source Architectural Patternsを参照しながら、Laravelでのデータベースアクセスパターンについて整理し、それぞれのパターンがどのような状況で適しているのかを解説します。

## 2. PoEAAにおけるData Source Architectural Patternsの一覧

### PoEAAとは

PoEAA（Patterns of Enterprise Application Architecture）は、Martin Fowlerが2002年に出版した書籍「Patterns of Enterprise Application Architecture」の略称です。この書籍では、エンタープライズアプリケーション開発における様々なアーキテクチャパターンが紹介されています。

### Data Source Architectural Patterns

Data Source Architectural Patternsは、アプリケーションとデータベース間のインタラクションを扱うパターン群

1. **Table Data Gateway**: データベースの1つのテーブルに対するゲートウェイとして機能するクラス
2. **Row Data Gateway**: データベースの1つの行に対するゲートウェイとして機能するクラス
3. **Active Record**: データベースの行とビジネスロジックを1つのクラスに組み合わせたパターン
4. **Data Mapper**: データベースとドメインオブジェクト間のマッピングを担当する層を設けるパターン

## 3. 各パターンの詳細解説

### Table Data Gateway

#### 概念と特徴
Table Data Gatewayは、データベースの1つのテーブルに対するすべての操作（SELECT、INSERT、UPDATE、DELETE）を1つのクラスにまとめるパターンです。このクラスはテーブルに対するゲートウェイとして機能し、ビジネスロジックは含みません。

```php
class UserGateway {
    public function findAll() {
        return DB::select('SELECT * FROM users');
    }
    
    public function findById($id) {
        return DB::select('SELECT * FROM users WHERE id = ?', [$id]);
    }
    
    public function insert($data) {
        DB::insert('INSERT INTO users (name, email) VALUES (?, ?)', [$data['name'], $data['email']]);
    }
    
    // その他のメソッド
}
```

#### メリット
- シンプルで理解しやすい
- SQLの最適化が容易
- テーブルごとに明確な責任分担

#### デメリット
- ビジネスロジックとデータアクセスの分離が必要
- 複数テーブルを跨ぐ操作が複雑になる可能性がある

#### 適用シナリオ
- シンプルなCRUD操作が中心のアプリケーション
- パフォーマンスが重視される場合
- SQLの最適化が必要な場合

---

### Row Data Gateway

#### 概念と特徴
Row Data Gatewayは、データベースの1つの行に対応するオブジェクトを提供するパターンです。各オブジェクトはデータベースの1行を表し、その行に対する操作（取得、更新、削除）を提供します。Table Data Gatewayと異なり、各インスタンスは特定の行に対応します。

```php
class UserRow {
    private $id;
    private $name;
    private $email;
    
    public function __construct($id = null) {
        if ($id) {
            $result = DB::select('SELECT * FROM users WHERE id = ?', [$id]);
            if (count($result) > 0) {
                $this->id = $result[0]->id;
                $this->name = $result[0]->name;
                $this->email = $result[0]->email;
            }
        }
    }
    
    public function getName() { return $this->name; }
    public function setName($name) { $this->name = $name; }
    
    public function getEmail() { return $this->email; }
    public function setEmail($email) { $this->email = $email; }
    
    public function save() {
        if ($this->id) {
            DB::update('UPDATE users SET name = ?, email = ? WHERE id = ?', 
                [$this->name, $this->email, $this->id]);
        } else {
            DB::insert('INSERT INTO users (name, email) VALUES (?, ?)', 
                [$this->name, $this->email]);
            $this->id = DB::getPdo()->lastInsertId();
        }
    }
    
    public function delete() {
        if ($this->id) {
            DB::delete('DELETE FROM users WHERE id = ?', [$this->id]);
        }
    }
}
```

#### メリット
- データベースの行とオブジェクトの対応が明確
- ビジネスロジックとデータアクセスの分離が可能

#### デメリット
- 各行に対して別々のオブジェクトが必要
- リレーションシップの扱いが複雑になる可能性がある

#### 適用シナリオ
- データベースの行とオブジェクトの1対1マッピングが必要な場合
- ビジネスロジックとデータアクセスを分離したい場合

---

### Active Record

#### 概念と特徴
Active Recordは、データベースの行とビジネスロジックを1つのクラスに組み合わせたパターンです。各Active Recordオブジェクトはデータベースの1行に対応し、データアクセスとビジネスロジックの両方を担当します。

```php
class User {
    private $id;
    private $name;
    private $email;
    
    public function __construct($id = null) {
        if ($id) {
            $this->load($id);
        }
    }
    
    private function load($id) {
        $result = DB::select('SELECT * FROM users WHERE id = ?', [$id]);
        if (count($result) > 0) {
            $this->id = $result[0]->id;
            $this->name = $result[0]->name;
            $this->email = $result[0]->email;
        }
    }
    
    public function getName() { return $this->name; }
    public function setName($name) { $this->name = $name; }
    
    public function getEmail() { return $this->email; }
    public function setEmail($email) { $this->email = $email; }
    
    public function save() {
        if ($this->id) {
            DB::update('UPDATE users SET name = ?, email = ? WHERE id = ?', 
                [$this->name, $this->email, $this->id]);
        } else {
            DB::insert('INSERT INTO users (name, email) VALUES (?, ?)', 
                [$this->name, $this->email]);
            $this->id = DB::getPdo()->lastInsertId();
        }
    }
    
    public function delete() {
        if ($this->id) {
            DB::delete('DELETE FROM users WHERE id = ?', [$this->id]);
        }
    }
    
    // ビジネスロジックメソッド
    public function isValidEmail() {
        return filter_var($this->email, FILTER_VALIDATE_EMAIL) !== false;
    }
    
    public function sendWelcomeEmail() {
        // メール送信ロジック
    }
    
    // 静的メソッド（ファインダーメソッド）
    public static function findAll() {
        $users = [];
        $results = DB::select('SELECT id FROM users');
        foreach ($results as $result) {
            $users[] = new User($result->id);
        }
        return $users;
    }
    
    public static function findByEmail($email) {
        $result = DB::select('SELECT id FROM users WHERE email = ?', [$email]);
        if (count($result) > 0) {
            return new User($result[0]->id);
        }
        return null;
    }
}
```

#### メリット
- シンプルで直感的なAPI
- データアクセスとビジネスロジックが統合されている
- 開発速度が速い

#### デメリット
- 単一責任の原則に違反する可能性がある
- テストが難しくなる可能性がある
- 複雑なドメインモデルには適さない場合がある

#### 適用シナリオ
- シンプルなドメインモデル
- RAD（Rapid Application Development）が重視される場合
- データベースのスキーマとドメインモデルが密接に関連している場合

---

### Data Mapper

#### 概念と特徴
Data Mapperは、データベースとドメインオブジェクト間のマッピングを担当する層を設けるパターンです。ドメインオブジェクトはデータベースの存在を意識せず、Data Mapperがデータの変換と永続化を担当します。

```php
// ドメインオブジェクト
class User {
    private $id;
    private $name;
    private $email;
    
    public function __construct($id = null, $name = null, $email = null) {
        $this->id = $id;
        $this->name = $name;
        $this->email = $email;
    }
    
    public function getId() { return $this->id; }
    
    public function getName() { return $this->name; }
    public function setName($name) { $this->name = $name; }
    
    public function getEmail() { return $this->email; }
    public function setEmail($email) { $this->email = $email; }
    
    // ビジネスロジック
    public function isValidEmail() {
        return filter_var($this->email, FILTER_VALIDATE_EMAIL) !== false;
    }
    
    public function sendWelcomeEmail() {
        // メール送信ロジック
    }
}

// Data Mapper
class UserMapper {
    public function find($id) {
        $result = DB::select('SELECT * FROM users WHERE id = ?', [$id]);
        if (count($result) > 0) {
            return new User(
                $result[0]->id,
                $result[0]->name,
                $result[0]->email
            );
        }
        return null;
    }
    
    public function findAll() {
        $users = [];
        $results = DB::select('SELECT * FROM users');
        foreach ($results as $result) {
            $users[] = new User(
                $result->id,
                $result->name,
                $result->email
            );
        }
        return $users;
    }
    
    public function save(User $user) {
        if ($user->getId()) {
            DB::update('UPDATE users SET name = ?, email = ? WHERE id = ?', 
                [$user->getName(), $user->getEmail(), $user->getId()]);
        } else {
            DB::insert('INSERT INTO users (name, email) VALUES (?, ?)', 
                [$user->getName(), $user->getEmail()]);
            // IDの設定は通常このようには行わないが、例としてシンプルにしている
            $id = DB::getPdo()->lastInsertId();
            $user = new User($id, $user->getName(), $user->getEmail());
            return $user;
        }
        return $user;
    }
    
    public function delete(User $user) {
        if ($user->getId()) {
            DB::delete('DELETE FROM users WHERE id = ?', [$user->getId()]);
        }
    }
}
```

#### メリット
- ドメインオブジェクトとデータベースの完全な分離
- 複雑なドメインモデルに適している
- テストが容易

#### デメリット
- 実装が複雑
- 開発速度が遅くなる可能性がある
- 小規模なアプリケーションではオーバーエンジニアリングになる可能性がある

#### 適用シナリオ
- 複雑なドメインモデル
- ドメイン駆動設計（DDD）を採用している場合
- データベースとドメインモデルの構造が大きく異なる場合

## 4. 各パターンの比較

![](../images/db-access-pattern/compare_data_source_architectual_patterns.png)

- Domain Logic: ドメインロジック。アプリケーションの中で意味を持つ処理
- Entity: 意味あるデータのまとまりを管理するオブジェクト
- Data Access: Queryの発行など実際にDBとやり取りをするクラス 

## 5. Laravelで実装可能なDBアクセス方法の特徴・違い・ユースケース

| 方法           | 使いやすさ | 柔軟性 | 可読性 | パフォーマンス |
|----------------|------------|--------|--------|----------------|
| **Eloquent**     | ◎         | ◯     | ◎     | △（複雑だと遅い）|
| **QueryBuilder** | ◯         | ◎     | ◯     | ◯              |
| **Raw SQL**      | △         | ◎     | △     | ◎              |

#### Eloquent（Active Record）
- Laravel標準のORM。
- モデルクラスとテーブルが1対1。
- リレーション・アクセサ・ミューテータ等が使える。

#### Query Builder
- SQLに近い記法でクエリ構築。
- モデルは使わず、テーブルベースで操作。

#### Raw SQL
- 生SQLをそのまま実行。

## 6. Laravelで実装可能なData Source Architectural Patterns

1. **Active Record**
   - Eloquent ORMとして既に実装されている
   - Laravelの主要なデータアクセスパターン
   - 小〜中規模のアプリケーションに最適

2. **Table Data Gateway**
   - Query Builderを使用して実装可能
   - パフォーマンスが重視される場合に有用

3. **Row Data Gateway**
   - Eloquent ORMがActive Recordパターンを採用しているため、純粋なRow Data Gatewayの実装は難しい
   - カスタム実装は可能だが、Laravelの哲学と合わない

4. **Data Mapper**
   - カスタム実装が必要
      - Query Builderを使用して実装可能
   - 複雑なドメインモデルに適している
   - Doctrine ORMのようなライブラリを統合することも可能

## 7. まとめ

### どんな時にどのパターンを使うべきか

1. Eloquent ORM
   - Active Record
      - シンプルかつドメインオブジェクトとテーブルが1対1で紐付いている時
      - 理由：開発速度が速く、学習曲線が緩やか

2. Query Builder
   - Table Data Gateway
      - パフォーマンスが重視される場合
      - 理由：SQLの最適化が容易、オーバーヘッドが少ない
   - Data Mapper (+ Repository)
      - 複雑なドメインロジックを持つ場合
      - 理由：ドメインモデルとデータアクセスの分離、テスト容易性

2. Raw SQL (基本的にQuery Builderを優先)
   - Table Data Gateway
      - パフォーマンスが重視される場合
      - 理由：SQLの最適化が容易、オーバーヘッドが少ない