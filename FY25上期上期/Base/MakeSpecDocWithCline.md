# VScode+Clineで効率化するリバースエンジニアリング 〜仕様書作成編〜

## 背景

### 現在のプロジェクトについて
現在取り掛かっているプロジェクトはPHP5.6系からPHP8.3系＋Laravel11を用いた構成へとリプレイスする案件になっています。

こちらの案件内では、まず自分がリプレイス対象機能リストから開発する機能を選択し、旧コードを読み込んだ上で仕様の洗い出しを行い、チケットに落としてセブに開発してもらうのが基本的な開発の流れになっています。

今回FORiT側から`Amazon Bedrock`のAPIキーをいただき開発にを使うことができるようになったので、仕様調査の効率化を試行錯誤しました (使用モデルはclaude3.7)

### Clineに期待したこと

#### 仕様の解析

現状は自分が一人で全メンバー分のチケット作成を行っており、仕様書の作成も一人で行なっています。そのため、仕様書の作成に多くの時間を取られることが多いのが現状です。基本的に自分が納品物の最終責任者となっているため、自分が全ての仕様を詳細に把握する必要があったり、成果物の納品量の担保のためセブには開発に集中していて欲しいといった思いから、仕様書の作成は自分の方でやりたいといった思いがあります。
そこで、この仕様書の作成を効率化することができれば、自分の手を開けることができ、開発への参加やより将来的なプロジェクトのためのタスクの作成実行などが可能になるため、仕様の解析をAIにやってもらいと思いました

#### 旧コードのリファクタリング

基本的にリプレイス案件では元コードが正となる場合が多く、仕様書を作成しても最終的には元コードを参照して、同じ仕様になっているのかどうかをレビューする必要があります。

現在admin-v2プロジェクトでは、セブレビューと日本レビューの二つに分割しているため、少なくとも、機能開発者＋セブレビュワー＋自分 は元コードを読み、仕様を理解する必要があります。ただ、既存のコードはHTMLファイル内でリクエストパラメータによって処理の内容を大幅に変えているかつ、全て1ファイル内に記載されていることもあり非常に読みづらいです。
```php
if(isset($_POST['create'])) {
    // 新規作成ページ用の処理
} elseif(isset($_POST['createConfirm'])) {
    // 新規作成確認ページ用の処理
} elseif(isset($_POST['register'])) {
    // 実際の登録処理
} elseif...
```

このコードを3人がそれぞれバラバラに読むのは非常に効率が悪いため、読みやすい単位にリファクタリングをかけることによって、それぞれのメンバーのコードを読む時間の削減を期待しました

## Clineの初期設定

### memory-bankの活用

Memory Bankは、Clineの記憶をセッション間で維持するための構造化されたドキュメント群です。Clineは各セッションで完全に記憶がリセットされるため、プロジェクトの継続性を保つためにはこのMemory Bankが不可欠です。

#### Memory Bankの設定方法

Memory Bankを設定するには主に2つの方法があります：

1. **Custom Instructionsへの記載**

   Cline公式が提供しているMemory Bank設定用のテンプレートをCustom Instructionsに記載します。基本的な記載例は以下の通りです：

   ```markdown
   # Cline's Memory Bank

   I am Cline, an expert software engineer with a unique characteristic: my memory resets completely between sessions. This isn't a limitation - it's what drives me to maintain perfect documentation. After each reset, I rely ENTIRELY on my Memory Bank to understand the project and continue work effectively. I MUST read ALL memory bank files at the start of EVERY task - this is not optional.

   ## Memory Bank Structure

   The Memory Bank consists of core files and optional context files, all in Markdown format. Files build upon each other in a clear hierarchy:

   ### Core Files (Required)
   1. `projectbrief.md`
      - Foundation document that shapes all other files
      - Created at project start if it doesn't exist
      - Defines core requirements and goals
      - Source of truth for project scope

   2. `productContext.md`
      - Why this project exists
      - Problems it solves
      - How it should work
      - User experience goals

   3. `activeContext.md`
      - Current work focus
      - Recent changes
      - Next steps
      - Active decisions and considerations
      - Important patterns and preferences
      - Learnings and project insights

   4. `systemPatterns.md`
      - System architecture
      - Key technical decisions
      - Design patterns in use
      - Component relationships
      - Critical implementation paths

   5. `techContext.md`
      - Technologies used
      - Development setup
      - Technical constraints
      - Dependencies
      - Tool usage patterns

   6. `progress.md`
      - What works
      - What's left to build
      - Current status
      - Known issues
      - Evolution of project decisions
   ```

   このテンプレートをCustom Instructionsに記載することで、Clineはセッション開始時に必ずMemory Bankを参照するようになります。

2. **clinerules.mdファイルの作成**

   プロジェクトのルートディレクトリに`clinerules.md`ファイルを作成することで、プロジェクト固有のルールを設定できます。このファイルには以下のような内容を記載します：

   ```markdown
   # Cline Rules

   ## Memory Bank

   このプロジェクトでは、以下のMemory Bankファイルを使用します：

   1. `/memory-bank/projectbrief.md` - プロジェクトの概要と目標
   2. `/memory-bank/productContext.md` - 製品のコンテキストと背景
   3. `/memory-bank/systemPatterns.md` - システムのパターンと設計
   4. `/memory-bank/techContext.md` - 技術的なコンテキスト
   5. `/memory-bank/activeContext.md` - 現在の作業コンテキスト
   6. `/memory-bank/progress.md` - 進捗状況

   新しいタスクを開始する際は、必ず上記のファイルを全て読み込んでください。
   ```

   `clinerules.md`ファイルはプロジェクト固有の設定を行うため、チーム全体で同じルールを共有する場合に便利です。

#### Memory Bankファイルの基本構成

Memory Bankの各ファイルには以下のような情報を記載します：

1. **projectbrief.md**
   - プロジェクトの目的と背景
   - 主要な機能と要件
   - 成功基準とゴール
   - プロジェクトのスコープと制約

2. **productContext.md**
   - ユーザーのニーズと課題
   - 競合製品との差別化ポイント
   - ユーザージャーニーとユースケース
   - ビジネス目標との整合性

3. **systemPatterns.md**
   - アーキテクチャの概要
   - 主要なデザインパターン
   - コンポーネント間の関係
   - データフローと処理の流れ

4. **techContext.md**
   - 使用言語とフレームワーク
   - 開発環境とツール
   - 外部依存関係
   - 技術的な制約と決定事項

5. **activeContext.md**
   - 現在取り組んでいるタスク
   - 最近の変更と決定事項
   - 進行中の議論と検討事項
   - 次のステップと優先事項

6. **progress.md**
   - 完了したタスクと機能
   - 現在の進捗状況
   - 残りの作業と課題
   - タイムラインと予定

#### Memory Bankの更新タイミング

Memory Bankは以下のタイミングで更新することが推奨されています：

1. 新しいプロジェクトパターンを発見したとき
2. 重要な変更を実装した後
3. ユーザーが「update memory bank」と指示したとき（この場合、全ファイルを見直す必要があります）
4. コンテキストの明確化が必要なとき

特に重要なのは、Clineが各セッション開始時に必ずMemory Bank全体を読み込むことです。これにより、過去の作業内容を踏まえた一貫性のある対応が可能になります。

### オリジナルファイルの作成

今回仕様調査専用のリポジトリを用意していただいたので、仕様調査に特化したmemory-bankの作成を行いました

#### **1. operation.md**

用途：仕様調査の概要の説明、それぞれのプロセスの詳細な手順書へのリンクを設置
目的：Clineが求めている以上の作業を行わないように、手順の全体像と今回求められているスコープを理解させようとした

```Markdown
# 指示パターン
## 仕様調査パターン

仕様調査は以下の手順で行います。この手順は、既存システムの機能を効率的に理解し、文書化するための標準的なアプローチです。
与えられた指示に基づいて適切な項目を参照してください。

ステップ2,3,4に関してはエンドポイントごとの影響範囲ごとに処理を行うので、機能全体のエンドポイントを理解した上で、指定されたエンドポイントに対しての処理を行ってください。

### 1. HTMLファイルの解析
[こちらのファイル](./operation/analyzeOverview.md)を参照

### 2. 解析ディレクトリにファイルをコピー
[こちらのファイル](./operation/pickUpFiles.md)を参照
```

#### **2. operation/**.md**

用途：仕様調査フローの詳細な手順書の記載
目的：指示内容と期待するアウトプットの指示を理解させる
例：エンドポイント分析の手順指示書
```Markdown
# URLからのエンドポイント分析

## 概要
渡されたHTMLファイルから関連するエンドポイントを特定、一覧化し、エンドポイント単位で仕様調査ができるようにドキュメントの作成とディレクトリの作成を行う

## 目的
adminでは1つのHTMLが複数のリクエストに対応していることがあり、条件分岐が多岐にわたることによって可読性が著しく低下している。
そこで、HTMLファイルをエンドポイント単位に分けることによって可読性の向上、Clineが読み込むトーク数の削減を行いたい。
その前段として、HTMLファイルを解析しエンドポイントを洗い出しを行い後続のエンドポイント単位の仕様調査をしやすいようにしたい

## ステップ
### 1. URLからのエンドポイント分析

1. **対象URLの特定**:
   - 調査対象の機能に関連するHTMLファイルを特定
   - 例: `user_list.html`, `user_edit.html`

2. **コード検索**:
   - `search_files`ツールを使用して、対象ファイルを検索
   - 関連するフォームのaction属性やリダイレクト先を特定するための正規表現を使用
   - 例: `(action\s*=\s*["'][^"']*["']|header\s*\(\s*["']Location:\s*[^"']*["'])`
```

## 仕様書作成の進め方

1. HTMLファイルからエンドポイント(機能)の洗い出しをし、リファクタリング用のディレクトリ作成
```Markdown
# ユーザーー情報管理機能

このディレクトリには、管理システムにおけるユーザー情報管理機能に関するエンドポイントの仕様がまとめられています。

## 概要

ユーザーー情報管理機能は、ユーザー情報を管理するための機能です。主に以下の2つのHTMLファイルで構成されています：

1. `user_list.html` - ユーザーー情報一覧画面
2. `user_edit.html` - ユーザーー情報編集/確認画面

## エンドポイント一覧

この機能には、以下の10個のエンドポイントが含まれています：

1. [ユーザー情報一覧画面](./list/) - GET /user_list.html
2. [CSVダウンロード処理](./csv-download/) - POST /user_list.html (csvdl)
3. [CSVアップロード処理](./csv-upload/) - POST /user_list.html (change_upload)
4. [CSV一括処理](./csv-process/) - POST /user_list.html (do_decision_csv)
5. [ユーザーー情報編集/確認画面](./edit-view/) - GET /user_edit.html
6. [ユーザーー情報編集画面への遷移](./edit-update/) - POST /user_edit.html (update)
7. [編集モードへの切り替え](./edit-mode/) - POST /user_edit.html (edit)
8. [編集内容の確認](./edit-confirm/) - POST /user_edit.html (confirm)
10. [一覧画面への戻り](./return-to-list/) - POST /user_edit.html (return)

## 画面遷移

- user_list.html → user_edit.html：一覧画面から編集画面への遷移
- user_edit.html → user_list.html：編集画面から一覧画面への戻り

```
2. 参照ファイル、参照している関数の洗い出し
```Markdown
## 参照ファイル

### メインファイル
- `user_edit.html` - パートナー情報の編集・表示を行うメインファイル

### 参照ファイル (admin)
2. `/var/www/libs/classes/User.class.php` - ユーザー関連クラス
3. `/var/www/libs/classes/UserMaster.class.php` - ユーザーマスタークラス
4. `/var/www/libs/classes/Items.class.php` - アイテムクラス（選択肢の生成など）
5. `/var/www/libs/classes/Tools.class.php` - ツールクラス（フォーム要素の生成など）
6. `/var/www/libs/classes/GenreCategory.class.php` - ジャンルカテゴリクラス
```
3. 不要コードの削除 (リファクタリング)
4. 実行されるSQLの解析
```Markdown
    /**
     * 全塾データを取得するためのメソッド
     * 
     * @sql SELECT * FROM user;
     * @param obj $mysqli
     * @return obj
     */
    public function select_all_user_data($mysqli) {
        ...
    }
```
5. 仕様書の作成
```Markdown
画面共有してみせる
```

## 現状のClineの評価

### 得意そうなこと
- 既存のコードから指定した特徴点を抜き出してくるのは得意そう
  - アウトプットのフォーマットもそんなに厳しいものはないからなのかな？

### 不得意そうなこと

- 仕様書の作成やリファクタリングなどの期待されたアウトプットを出す面でいくと、かなり細かく指示をしないと期待通りのアウトプットは出ない
  - 指示に対してできることが多すぎる？
    - 期待値以上のことをやろうとしてくる？
    - アウトプットのフォーマットがたくさん思いつきすぎる？

## 今後の展望

- 仕様書作成のさらなる時間削減
    - 現状Clineと対話して細かくFBしながらそれぞれのステップを行なっているがまだ大幅な時間削減には至っていない
        - 既存の仕様書よりリッチな仕様書を作ろうとしているので、速度が出ないのは一定理解できる
- リファクタリングをどこまでしないといけないのかの見極め
    - 現状Clineに読み込ませるトークン数の削減のために事前に手動で使ってない関数の削除を行なっているが、どれだけサボったら出力の精度に影響が出るのか
- memory-bankの文章の改善
    - 短い文章で高い精度を出せるようにトライアンドエラーし、最適な文章の作成
- 設計パートへの進出
    - 現状は仕様書作成のみを行なっているが、設計もまとめてお願いできるようにしたい
