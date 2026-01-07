# アプリケーション設計パターン

## ファイルをオブジェクトストレージにアップロードする

### 概要

大容量ファイルをアップロードする際は、Pre-signed URLを使用してアプリケーションサーバーを経由せずに直接S3にアップロードします。
マルチパートアップロードにより、大きなファイルを分割して並列アップロードし、パフォーマンスと信頼性を向上させます。

### システム設計図

```mermaid
graph TB
    subgraph "クライアント"
        User[ユーザー]
        Browser[ブラウザ]
    end
    
    subgraph "アプリケーション層"
        API[APIサーバー]
        Validator[ファイル検証<br/>サイズ、タイプ]
        URLGenerator[Pre-signed URL生成]
    end
    
    subgraph "ストレージ層"
        S3[S3バケット]
        CloudFront[CloudFront CDN]
    end
    
    subgraph "データベース"
        DB[(メタデータDB)]
    end
    
    subgraph "後処理"
        Lambda[Lambda関数<br/>S3イベントトリガー]
        Resize[画像リサイズ]
        Thumbnail[サムネイル生成]
        Virus[ウイルススキャン]
    end
    
    User --> Browser
    Browser -->|1. アップロード要求| API
    API --> Validator
    Validator --> URLGenerator
    URLGenerator -->|2. Pre-signed URL| Browser
    Browser -->|3. 直接アップロード| S3
    S3 -->|4. 完了通知| API
    API -->|5. メタデータ保存| DB
    
    S3 -->|イベント通知| Lambda
    Lambda --> Resize
    Lambda --> Thumbnail
    Lambda --> Virus
    
    CloudFront --> S3
```

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant App as アプリケーション
    participant S3 as S3ストレージ
    participant DB as データベース
    participant Lambda as Lambda関数

    User->>App: POST /api/upload/initiate<br/>filename: "photo.jpg", size: 10MB
    
    App->>App: ファイル検証<br/>(サイズ < 100MB, type: image/*)
    
    App->>S3: Generate Pre-signed URL<br/>PUT /bucket/uploads/uuid/photo.jpg<br/>expires: 15分
    S3-->>App: Pre-signed URL
    
    App->>DB: INSERT INTO files<br/>(id, status: 'uploading', filename, size)
    
    App-->>User: Pre-signed URL返却
    
    Note over User,S3: マルチパートアップロード
    
    User->>S3: POST /bucket/uploads/uuid/photo.jpg<br/>?uploads (開始)
    S3-->>User: Upload ID
    
    par ファイルを5MBずつ分割アップロード
        User->>S3: PUT part1 (0-5MB)
        S3-->>User: ETag1
    and
        User->>S3: PUT part2 (5-10MB)
        S3-->>User: ETag2
    end
    
    User->>S3: POST complete<br/>[{PartNumber: 1, ETag: ETag1}, ...]
    S3-->>User: アップロード完了
    
    User->>App: POST /api/upload/complete<br/>file_id: uuid
    
    App->>DB: UPDATE files<br/>SET status='completed'
    App-->>User: 成功レスポンス
    
    Note over S3,Lambda: S3イベントトリガー
    
    S3->>Lambda: ObjectCreated通知
    Lambda->>S3: 元ファイル取得
    Lambda->>Lambda: 画像リサイズ(800x600)
    Lambda->>S3: リサイズ画像保存<br/>/bucket/resized/uuid/photo.jpg
    Lambda->>Lambda: サムネイル生成(200x200)
    Lambda->>S3: サムネイル保存<br/>/bucket/thumbnails/uuid/photo.jpg
    Lambda->>DB: UPDATE files<br/>SET processed=true
```

### 設計のポイント

Pre-signed URLにより、アプリケーションサーバーの負荷を削減し、直接S3にアップロードします。
マルチパートアップロードにより、大きなファイルを効率的にアップロードできます。
ファイルのバリデーション(サイズ、タイプ、拡張子)をサーバー側で実行します。
S3のライフサイクルポリシーを設定して、一時ファイルを自動削除します。
アップロード完了後の後処理(画像リサイズ、ウイルススキャン等)は非同期で実行します。

## 遅延キューを使って処理を遅らせる

### 概要

遅延キュー(Delay Queue)を使用して、指定した時間後に処理を実行します。
リマインダー、予約投稿、一時的なアカウントロックの解除などに活用します。

### システム設計図

```mermaid
graph TB
    subgraph "遅延キュー実装"
        Producer[プロデューサー]
        DelayQueue[遅延キュー<br/>SQS Delay Queue]
        Worker[ワーカー]
        
        Producer -->|メッセージ投入<br/>delay: 300秒| DelayQueue
        DelayQueue -->|300秒後に可視化| Worker
    end
    
    subgraph "Redis Sorted Set実装"
        App[アプリケーション]
        Redis[(Redis<br/>Sorted Set)]
        Scheduler[スケジューラー<br/>定期ポーリング]
        Executor[実行ワーカー]
        
        App -->|ZADD tasks score=timestamp| Redis
        Scheduler -->|ZRANGEBYSCORE<br/>-inf now| Redis
        Scheduler --> Executor
    end
    
    subgraph "ユースケース"
        Reminder[リマインダー通知<br/>30分後]
        ScheduledPost[予約投稿<br/>指定時刻]
        TempBan[一時的なBAN解除<br/>24時間後]
        OrderCancel[注文自動キャンセル<br/>15分後]
    end
```

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant API as APIサーバー
    participant Redis as Redis Sorted Set
    participant Scheduler as スケジューラー
    participant Worker as ワーカー
    participant Notification as 通知サービス

    Note over User,Notification: リマインダー登録(30分後)
    
    User->>API: POST /api/reminders<br/>message: "会議開始", delay: 1800秒
    
    API->>API: タスクID生成: task_123
    API->>API: 実行時刻計算<br/>execute_at = now + 1800
    
    API->>Redis: ZADD delayed_tasks<br/>score: execute_at<br/>member: task_123
    Redis-->>API: 成功
    
    API->>Redis: SET task:task_123<br/>{type: "reminder", user_id: 456, message: "会議開始"}
    
    API-->>User: タスクID: task_123
    
    Note over Scheduler,Worker: スケジューラーが定期的にポーリング(10秒ごと)
    
    loop 実行時刻まで待機
        Scheduler->>Redis: ZRANGEBYSCORE delayed_tasks<br/>-inf now LIMIT 100
        Redis-->>Scheduler: []（まだ時刻未到来)
    end
    
    Note over Scheduler,Notification: 30分経過後
    
    Scheduler->>Redis: ZRANGEBYSCORE delayed_tasks<br/>-inf now LIMIT 100
    Redis-->>Scheduler: [task_123]
    
    Scheduler->>Redis: ZREM delayed_tasks task_123
    Redis-->>Scheduler: 削除成功
    
    Scheduler->>Redis: GET task:task_123
    Redis-->>Scheduler: {type: "reminder", user_id: 456, ...}
    
    Scheduler->>Worker: タスク実行依頼(task_123)
    
    Worker->>Notification: リマインダー送信<br/>user_id: 456, message: "会議開始"
    Notification-->>Worker: 送信完了
    
    Worker->>Redis: DEL task:task_123
    Worker-->>Scheduler: タスク完了
    
    Note over User,Notification: 予約投稿(特定時刻)
    
    User->>API: POST /api/posts/schedule<br/>content: "新製品発表", publish_at: "2025-01-01 09:00"
    
    API->>Redis: ZADD delayed_tasks<br/>score: 1735689600(2025-01-01 09:00)<br/>member: post_789
    
    API->>Redis: SET task:post_789<br/>{type: "publish_post", post_id: 789}
    
    API-->>User: 予約投稿ID: post_789
```

### 設計のポイント

SQS Delay Queueは、最大15分までの遅延をサポートします。
それ以上の遅延が必要な場合は、Redis Sorted SetやDynamoDB TTLを使用します。
スケジューラーは複数インスタンスで実行して可用性を確保し、分散ロックで重複実行を防ぎます。
タスクの実行失敗時は、リトライポリシーを設定します。
タスクのキャンセル機能を実装して、不要になったタスクを削除できるようにします。

## ページネーションを設計する

### 概要

大量のデータを効率的に取得するため、ページネーションを実装します。
オフセットベース、カーソルベース、キーセットページネーションの特性を理解し、適切に選択します。

### システム設計図

```mermaid
graph TB
    subgraph "オフセットベースページネーション"
        Offset[OFFSET方式<br/>ページ番号指定]
        OffsetQuery[SELECT * FROM posts<br/>ORDER BY created_at DESC<br/>LIMIT 20 OFFSET 40]
        OffsetPros[利点: ページ番号でジャンプ可能]
        OffsetCons[欠点: 大きなOFFSETで遅い<br/>データ追加で重複/欠落]
        
        Offset --> OffsetQuery
        Offset --> OffsetPros
        Offset --> OffsetCons
    end
    
    subgraph "カーソルベースページネーション"
        Cursor[CURSOR方式<br/>前ページの最後のID指定]
        CursorQuery[SELECT * FROM posts<br/>WHERE id < last_id<br/>ORDER BY id DESC<br/>LIMIT 20]
        CursorPros[利点: 安定した結果<br/>高速]
        CursorCons[欠点: ページ番号でジャンプ不可]
        
        Cursor --> CursorQuery
        Cursor --> CursorPros
        Cursor --> CursorCons
    end
    
    subgraph "キーセットページネーション"
        Keyset[KEYSET方式<br/>複数カラムで範囲指定]
        KeysetQuery[SELECT * FROM posts<br/>WHERE created_at, id < last_timestamp, last_id<br/>ORDER BY created_at DESC, id DESC<br/>LIMIT 20]
        KeysetPros[利点: 非常に高速<br/>安定]
        KeysetCons[欠点: 実装複雑]
    end
```

```mermaid
sequenceDiagram
    participant Client as クライアント
    participant API as APIサーバー
    participant Cache as キャッシュ
    participant DB as データベース

    Note over Client,DB: オフセットベースページネーション
    
    Client->>API: GET /api/posts?page=1&limit=20
    API->>DB: SELECT * FROM posts<br/>ORDER BY created_at DESC<br/>LIMIT 20 OFFSET 0
    DB-->>API: 20件のデータ
    API-->>Client: {data: [...], page: 1, total_pages: 50}
    
    Client->>API: GET /api/posts?page=3&limit=20
    API->>DB: SELECT * FROM posts<br/>ORDER BY created_at DESC<br/>LIMIT 20 OFFSET 40
    DB-->>API: 20件のデータ
    API-->>Client: {data: [...], page: 3, total_pages: 50}
    
    Note over Client,DB: カーソルベースページネーション
    
    Client->>API: GET /api/posts?limit=20
    API->>Cache: GET posts_cache:first_page
    alt キャッシュヒット
        Cache-->>API: データ + cursor
    else キャッシュミス
        API->>DB: SELECT * FROM posts<br/>ORDER BY id DESC<br/>LIMIT 20
        DB-->>API: データ(id: 1000~981)
        API->>Cache: SET posts_cache:first_page
    end
    API-->>Client: {data: [...], next_cursor: "981", has_more: true}
    
    Client->>API: GET /api/posts?cursor=981&limit=20
    API->>DB: SELECT * FROM posts<br/>WHERE id < 981<br/>ORDER BY id DESC<br/>LIMIT 20
    DB-->>API: データ(id: 980~961)
    API-->>Client: {data: [...], next_cursor: "961", has_more: true}
    
    Note over Client,DB: キーセットページネーション(複合キー)
    
    Client->>API: GET /api/posts?limit=20
    API->>DB: SELECT * FROM posts<br/>ORDER BY created_at DESC, id DESC<br/>LIMIT 20
    DB-->>API: データ(last: created_at=2025-01-01, id=500)
    API-->>Client: {data: [...],<br/>next: "2025-01-01:500"}
    
    Client->>API: GET /api/posts?after=2025-01-01:500&limit=20
    API->>DB: SELECT * FROM posts<br/>WHERE (created_at, id)<br/>< ('2025-01-01', 500)<br/>ORDER BY created_at DESC, id DESC<br/>LIMIT 20
    DB-->>API: 20件のデータ
    API-->>Client: {data: [...], next: "2024-12-31:450"}
```

### 設計のポイント

オフセットベースは、ページ番号でのジャンプが必要な場合に使用します(検索結果等)。
カーソルベースは、無限スクロールやフィードに適しています。
キーセットページネーションは、大規模データで高速な取得が必要な場合に使用します。
総件数の取得は、COUNT(*)クエリがコストが高いため、キャッシュまたは概算値を使用します。
ページネーションのメタデータ(has_more、total_pages等)を適切に返却します。

## 全文検索を導入する

### 概要

全文検索エンジン(Elasticsearch、Algolia等)を使用して、高速で柔軟な検索機能を実現します。
データベースのLIKE検索では性能が出ない場合に導入します。

### システム設計図

```mermaid
graph TB
    subgraph "検索アーキテクチャ"
        Client[クライアント]
        API[APIサーバー]
        Search[検索エンジン<br/>Elasticsearch]
        DB[(プライマリDB)]
    end
    
    subgraph "データ同期"
        CDC[Change Data Capture]
        Indexer[インデクサー]
        
        DB -->|データ変更| CDC
        CDC --> Indexer
        Indexer -->|インデックス更新| Search
    end
    
    subgraph "検索機能"
        FullText[全文検索<br/>形態素解析]
        Fuzzy[あいまい検索<br/>typo許容]
        Filter[フィルタリング<br/>カテゴリ、価格帯]
        Facet[ファセット<br/>集計]
        Highlight[ハイライト<br/>検索語強調]
        Suggest[サジェスト<br/>オートコンプリート]
    end
    
    Client -->|検索クエリ| API
    API -->|検索実行| Search
    Search -->|結果返却| API
    API -->|詳細データ取得| DB
    API -->|結果返却| Client
```

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant API as APIサーバー
    participant ES as Elasticsearch
    participant DB as データベース
    participant Indexer as インデクサー

    Note over User,Indexer: 商品データのインデックス作成
    
    DB->>Indexer: データ変更通知<br/>(新商品追加)
    Indexer->>ES: PUT /products/_doc/123<br/>{title: "iPhone 15 Pro", price: 159800, ...}
    ES->>ES: 形態素解析<br/>["iPhone", "15", "Pro"]
    ES->>ES: 転置インデックス作成
    ES-->>Indexer: インデックス完了
    
    Note over User,Indexer: 全文検索実行
    
    User->>API: GET /api/search?q=アイフォン 15&size=20
    
    API->>ES: POST /products/_search<br/>{<br/>  "query": {<br/>    "multi_match": {<br/>      "query": "アイフォン 15",<br/>      "fields": ["title^3", "description"],<br/>      "fuzziness": "AUTO"<br/>    }<br/>  },<br/>  "highlight": {"fields": {"title": {}}}<br/>}
    
    ES->>ES: クエリ解析<br/>["アイフォン", "15"]
    ES->>ES: あいまい検索<br/>("アイフォン" -> "iPhone")
    ES->>ES: スコアリング<br/>(TF-IDF, BM25)
    
    ES-->>API: {<br/>  hits: [<br/>    {_id: 123, _score: 8.5,<br/>     title: "iPhone 15 Pro",<br/>     highlight: "<em>iPhone</em> <em>15</em> Pro"}<br/>  ]<br/>}
    
    API->>DB: SELECT * FROM products<br/>WHERE id IN (123, 456, ...)
    DB-->>API: 詳細データ(在庫、レビュー等)
    
    API-->>User: 検索結果(ハイライト付き)
    
    Note over User,Indexer: ファセット検索(絞り込み)
    
    User->>API: GET /api/search?q=スマートフォン<br/>&category=electronics&price_min=100000
    
    API->>ES: POST /products/_search<br/>{<br/>  "query": {<br/>    "bool": {<br/>      "must": {"match": {"title": "スマートフォン"}},<br/>      "filter": [<br/>        {"term": {"category": "electronics"}},<br/>        {"range": {"price": {"gte": 100000}}}<br/>      ]<br/>    }<br/>  },<br/>  "aggs": {<br/>    "brands": {"terms": {"field": "brand"}},<br/>    "price_ranges": {"range": {"field": "price", ...}}<br/>  }<br/>}
    
    ES-->>API: {<br/>  hits: [...],<br/>  aggregations: {<br/>    brands: {Apple: 15, Samsung: 10, ...},<br/>    price_ranges: {...}<br/>  }<br/>}
    
    API-->>User: 検索結果 + ファセット情報
```

### 設計のポイント

データベースとElasticsearchの同期は、CDCまたはアプリケーションレイヤーで実装します。
Elasticsearchはプライマリデータソースではなく、検索専用として使用します。
日本語の形態素解析には、kuromoji analyzerを使用します。
検索結果のスコアリングアルゴリズム(TF-IDF、BM25)を理解し、適切に調整します。
ファセット検索により、ユーザーが結果を絞り込みやすくします。
インデックスのシャーディングとレプリケーションで、スケーラビリティと可用性を確保します。

## ユーザーを認証・認可する

### 概要

JWT(JSON Web Token)を使用して、ステートレスな認証・認可を実現します。
OAuth 2.0とOpenID Connectにより、サードパーティ認証を統合します。

### システム設計図

```mermaid
graph TB
    subgraph "認証フロー"
        Login[ログイン画面]
        AuthServer[認証サーバー]
        ResourceServer[リソースサーバー]
        
        Login -->|ID/パスワード| AuthServer
        AuthServer -->|JWT発行| Login
        Login -->|JWT付きリクエスト| ResourceServer
    end
    
    subgraph "JWT構造"
        Header["Header<br/>alg: HS256, typ: JWT"]
        Payload["Payload<br/>user_id: 123, role: admin, exp: ..."]
        Signature["Signature<br/>HMACSHA256"]
        
        Header --- Payload
        Payload --- Signature
    end
    
    subgraph "OAuth 2.0フロー"
        Client[クライアント]
        AuthZ[認可サーバー]
        Resource[リソースサーバー]
        
        Client -->|1. 認可リクエスト| AuthZ
        AuthZ -->|2. 認可コード| Client
        Client -->|3. トークンリクエスト| AuthZ
        AuthZ -->|4. アクセストークン| Client
        Client -->|5. リソースアクセス| Resource
    end
```

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant Client as クライアント
    participant Auth as 認証サーバー
    participant API as APIサーバー
    participant DB as データベース
    participant Redis as Redis(ブラックリスト)

    Note over User,Redis: ログインフロー
    
    User->>Client: ログイン(email, password)
    Client->>Auth: POST /auth/login<br/>{email, password}
    
    Auth->>DB: SELECT * FROM users<br/>WHERE email = ?
    DB-->>Auth: ユーザー情報(id, password_hash)
    
    Auth->>Auth: パスワード検証<br/>bcrypt.compare(password, hash)
    
    alt パスワード正しい
        Auth->>Auth: JWT生成<br/>payload: {user_id: 123, role: "user"}<br/>exp: now + 1時間
        Auth->>Auth: 署名(secret_key)
        Auth->>Redis: SET refresh_token:uuid user_id<br/>EX 7日
        Auth-->>Client: {access_token: jwt, refresh_token: uuid}
        Client->>Client: トークン保存(localStorage)
    else パスワード間違い
        Auth-->>Client: 401 Unauthorized
    end
    
    Note over User,Redis: APIアクセス
    
    User->>Client: プロフィール取得
    Client->>API: GET /api/profile<br/>Authorization: Bearer jwt
    
    API->>API: JWT検証<br/>1. 署名確認<br/>2. 有効期限確認<br/>3. ブラックリスト確認
    
    API->>Redis: GET blacklist:jwt_id
    alt ブラックリストに存在
        API-->>Client: 401 Unauthorized(ログアウト済み)
    end
    
    API->>API: ペイロード抽出<br/>{user_id: 123, role: "user"}
    API->>DB: SELECT * FROM users WHERE id = 123
    DB-->>API: ユーザー情報
    API-->>Client: プロフィールデータ
    
    Note over User,Redis: トークンリフレッシュ
    
    Client->>Auth: POST /auth/refresh<br/>{refresh_token: uuid}
    Auth->>Redis: GET refresh_token:uuid
    Redis-->>Auth: user_id: 123
    
    Auth->>Auth: 新しいアクセストークン生成
    Auth-->>Client: {access_token: new_jwt}
    
    Note over User,Redis: ログアウト
    
    User->>Client: ログアウト
    Client->>Auth: POST /auth/logout<br/>{access_token: jwt}
    Auth->>Redis: SET blacklist:jwt_id 1<br/>EX remaining_ttl
    Auth->>Redis: DEL refresh_token:uuid
    Auth-->>Client: ログアウト成功
    Client->>Client: トークン削除
```

### 設計のポイント

パスワードは必ずハッシュ化(bcrypt、argon2)して保存します。
JWTのペイロードには機密情報を含めません(Base64エンコードで可読)。
アクセストークンは短い有効期限(15分~1時間)、リフレッシュトークンは長い有効期限(7日~30日)を設定します。
ログアウト時は、JWTをブラックリストに追加します(Redis、有効期限まで保持)。
RBAC(Role-Based Access Control)またはABAC(Attribute-Based Access Control)で認可を実装します。
CSRF対策として、SameSite cookieまたはCSRFトークンを使用します。

## SaaSのマルチテナントを設計する

### 概要

マルチテナントアーキテクチャにより、複数の顧客(テナント)が同じアプリケーションインスタンスを共有します。
データ分離、セキュリティ、リソース分離の方法を適切に選択します。

### システム設計図

```mermaid
graph TB
    subgraph "テナント分離方式"
        SharedDB[共有データベース<br/>tenant_id カラムで分離]
        SharedSchema[共有スキーマ<br/>テナントごとにテーブル]
        SeparateDB[個別データベース<br/>テナントごとにDB]
    end
    
    subgraph "共有データベース方式"
        App1[アプリケーション]
        DB1[(データベース)]
        
        App1 --> DB1
        
        Table1["users テーブル<br/>tenant_id, user_id, name"]
        Table2["orders テーブル<br/>tenant_id, order_id, amount"]
        
        DB1 --> Table1
        DB1 --> Table2
    end
    
    subgraph "個別データベース方式"
        App2[アプリケーション]
        TenantRouter[テナントルーター]
        
        DB_A[(DB: tenant_a)]
        DB_B[(DB: tenant_b)]
        DB_C[(DB: tenant_c)]
        
        App2 --> TenantRouter
        TenantRouter --> DB_A
        TenantRouter --> DB_B
        TenantRouter --> DB_C
    end
    
    subgraph "ハイブリッド方式"
        SharedMetaDB[(共有メタデータDB<br/>テナント情報、設定)]
        TenantDataDB[(個別データDB<br/>ビジネスデータ)]
    end
```

```mermaid
sequenceDiagram
    participant User as ユーザー(tenant_a)
    participant LB as ロードバランサー
    participant App as アプリケーション
    participant Router as テナントルーター
    participant Cache as Redis
    participant DB_A as DB(tenant_a)
    participant DB_B as DB(tenant_b)

    Note over User,DB_B: 共有データベース方式
    
    User->>LB: GET /api/orders<br/>Authorization: Bearer jwt
    LB->>App: リクエスト転送
    
    App->>App: JWT検証<br/>tenant_id: tenant_a, user_id: 123
    
    App->>Cache: GET tenant_a:orders:user_123
    alt キャッシュミス
        App->>DB_A: SELECT * FROM orders<br/>WHERE tenant_id = 'tenant_a'<br/>AND user_id = 123
        DB_A-->>App: 注文データ
        App->>Cache: SET tenant_a:orders:user_123
    end
    
    App-->>User: 注文一覧
    
    Note over User,DB_B: Row Level Securityで強制分離
    
    App->>DB_A: SET app.current_tenant = 'tenant_a'
    App->>DB_A: SELECT * FROM orders<br/>WHERE user_id = 123
    Note over DB_A: RLS Policy適用<br/>WHERE tenant_id = current_setting('app.current_tenant')
    DB_A-->>App: tenant_aのデータのみ
    
    Note over User,DB_B: 個別データベース方式
    
    User->>App: POST /api/products<br/>Authorization: Bearer jwt
    
    App->>App: JWT検証<br/>tenant_id: tenant_a
    
    App->>Router: テナント識別<br/>tenant_id: tenant_a
    Router->>Router: DB接続情報取得<br/>host: db-tenant-a.example.com
    
    Router->>DB_A: INSERT INTO products<br/>VALUES (...)
    DB_A-->>Router: 挿入成功
    Router-->>App: 成功
    App-->>User: 商品作成成功
    
    Note over User,DB_B: テナント間のデータ分離保証
    
    User->>App: GET /api/orders<br/>tenant_id: tenant_b(不正)
    
    App->>App: JWT検証<br/>token tenant_id: tenant_a<br/>request tenant_id: tenant_b
    App->>App: テナント不一致検知
    App-->>User: 403 Forbidden
```

### 設計のポイント

共有データベース方式は、コスト効率が高いですが、テナント間のデータ漏洩リスクがあります。
個別データベース方式は、完全な分離を実現しますが、運用コストが高くなります。
Row Level Security(RLS)を使用して、データベースレベルでテナント分離を強制します。
テナントIDは、JWTに含め、すべてのクエリで必ずフィルタリングします。
テナントごとのリソース使用量を監視して、公平なリソース配分を実現します(Rate Limiting等)。
大口顧客には個別データベースを割り当て、小規模顧客は共有データベースを使用するハイブリッド方式も有効です。

## マイクロサービスとモノリスの比較

### 概要

アプリケーションアーキテクチャとして、モノリス(一枚岩)とマイクロサービスの選択は重要な決定です。
それぞれの特性とトレードオフを理解し、システムの規模やチーム構成に応じて選択します。

### システム設計図

```mermaid
graph TB
    subgraph "モノリスアーキテクチャ"
        MonoClient[クライアント]
        MonoApp[モノリスアプリケーション]
        MonoUser[ユーザー機能]
        MonoOrder[注文機能]
        MonoPayment[決済機能]
        MonoInventory[在庫機能]
        MonoDB[(単一データベース)]

        MonoClient --> MonoApp
        MonoApp --> MonoUser
        MonoApp --> MonoOrder
        MonoApp --> MonoPayment
        MonoApp --> MonoInventory
        MonoUser --> MonoDB
        MonoOrder --> MonoDB
        MonoPayment --> MonoDB
        MonoInventory --> MonoDB
    end

    subgraph "マイクロサービスアーキテクチャ"
        MSClient[クライアント]
        Gateway[API Gateway]
        UserSvc[ユーザーサービス]
        OrderSvc[注文サービス]
        PaymentSvc[決済サービス]
        InventorySvc[在庫サービス]
        UserDB[(User DB)]
        OrderDB[(Order DB)]
        PaymentDB[(Payment DB)]
        InventoryDB[(Inventory DB)]
        MQ[メッセージキュー]

        MSClient --> Gateway
        Gateway --> UserSvc
        Gateway --> OrderSvc
        Gateway --> PaymentSvc
        Gateway --> InventorySvc
        UserSvc --> UserDB
        OrderSvc --> OrderDB
        PaymentSvc --> PaymentDB
        InventorySvc --> InventoryDB
        OrderSvc --> MQ
        MQ --> PaymentSvc
        MQ --> InventorySvc
    end
```

```mermaid
graph LR
    subgraph "モノリスの特徴"
        MonoPros["利点<br/>シンプルなデプロイ<br/>低レイテンシ - プロセス内呼び出し<br/>トランザクション管理容易<br/>デバッグ容易"]
        MonoCons["欠点<br/>スケーリングの粒度が粗い<br/>技術スタックの制約<br/>チーム間の依存関係<br/>デプロイ影響範囲が大きい"]
    end

    subgraph "マイクロサービスの特徴"
        MSPros["利点<br/>独立したスケーリング<br/>技術選択の自由度<br/>障害分離<br/>チームの自律性"]
        MSCons["欠点<br/>分散システムの複雑性<br/>ネットワークレイテンシ<br/>データ整合性の課題<br/>運用コスト増加"]
    end
```

```mermaid
sequenceDiagram
    participant Client as クライアント
    participant Mono as モノリス
    participant Gateway as API Gateway
    participant Order as 注文サービス
    participant Payment as 決済サービス
    participant Inventory as 在庫サービス

    Note over Client,Inventory: モノリスでの注文処理

    Client->>Mono: 注文リクエスト
    Mono->>Mono: BEGIN TRANSACTION
    Mono->>Mono: 在庫確認(プロセス内呼び出し)
    Mono->>Mono: 決済処理(プロセス内呼び出し)
    Mono->>Mono: 注文作成(プロセス内呼び出し)
    Mono->>Mono: COMMIT
    Mono-->>Client: 注文完了

    Note over Client,Inventory: マイクロサービスでの注文処理

    Client->>Gateway: 注文リクエスト
    Gateway->>Order: 注文作成
    Order->>Inventory: 在庫確認(HTTP/gRPC)
    Inventory-->>Order: 在庫あり
    Order->>Payment: 決済処理(HTTP/gRPC)
    Payment-->>Order: 決済成功
    Order->>Order: 注文保存
    Order->>Order: イベント発行(OrderCreated)
    Order-->>Gateway: 注文ID返却
    Gateway-->>Client: 注文完了

    Note over Order,Inventory: 非同期処理(Saga)
    Order->>Inventory: 在庫引当イベント
    Order->>Payment: 決済確定イベント
```

### 設計のポイント

モノリスは、小規模なチームや初期段階のプロダクトに適しています。
マイクロサービスは、大規模なチームや複雑なドメインを持つシステムに適しています。
モノリスからマイクロサービスへの移行は、Strangler Figパターンで段階的に行います。
マイクロサービスでは、サービス間通信、データ整合性、分散トレーシングなどの課題を考慮する必要があります。
モジュラーモノリスは、モノリスの利点を維持しながら、将来の分割に備えた設計です。
「マイクロサービスプレミアム」を払う準備ができていない場合は、モノリスから始めることを推奨します。

## セキュリティ設計(暗号化、TLS/SSL)

### 概要

システムのセキュリティを確保するため、データの暗号化、通信の保護、セキュリティ対策を適切に設計します。
TLS/SSL、暗号化アルゴリズム、OWASP Top 10対策などを理解し、実装します。

### システム設計図

```mermaid
graph TB
    subgraph "通信の暗号化"
        Client[クライアント]
        LB[ロードバランサー<br/>TLS終端]
        App[アプリケーション]
        DB[(データベース)]

        Client -->|HTTPS<br/>TLS 1.3| LB
        LB -->|HTTP or TLS| App
        App -->|TLS| DB
    end

    subgraph "データ暗号化"
        AtRest[保存時暗号化<br/>Encryption at Rest]
        InTransit[転送時暗号化<br/>Encryption in Transit]
        AppLevel[アプリケーション暗号化<br/>End-to-End Encryption]
    end

    subgraph "暗号化アルゴリズム"
        Symmetric[対称鍵暗号<br/>AES-256-GCM]
        Asymmetric[非対称鍵暗号<br/>RSA-2048, ECDSA]
        Hash[ハッシュ関数<br/>SHA-256, bcrypt]
        KDF[鍵導出関数<br/>PBKDF2, Argon2]
    end

    subgraph "鍵管理"
        KMS[KMS<br/>AWS KMS/HashiCorp Vault]
        Rotation[鍵ローテーション]
        HSM[HSM<br/>ハードウェアセキュリティモジュール]
    end
```

```mermaid
graph LR
    subgraph "TLSハンドシェイク"
        ClientHello[1. Client Hello<br/>対応暗号スイート提示]
        ServerHello[2. Server Hello<br/>暗号スイート選択<br/>証明書送信]
        KeyExchange[3. Key Exchange<br/>プリマスターシークレット]
        Finished[4. Finished<br/>セッション確立]

        ClientHello --> ServerHello
        ServerHello --> KeyExchange
        KeyExchange --> Finished
    end

    subgraph "証明書チェーン"
        Root[ルートCA証明書]
        Intermediate[中間CA証明書]
        Server[サーバー証明書]

        Root --> Intermediate
        Intermediate --> Server
    end
```

```mermaid
sequenceDiagram
    participant Client as クライアント
    participant LB as ロードバランサー
    participant App as アプリケーション
    participant KMS as KMS
    participant DB as データベース

    Note over Client,DB: TLS接続確立

    Client->>LB: Client Hello<br/>(TLS 1.3, 暗号スイート)
    LB->>Client: Server Hello<br/>(証明書, 公開鍵)
    Client->>Client: 証明書検証<br/>(CA署名確認)
    Client->>LB: Key Exchange<br/>(プリマスターシークレット)
    LB->>LB: セッションキー生成
    Client->>LB: Finished(暗号化)
    LB->>Client: Finished(暗号化)

    Note over Client,DB: 機密データの暗号化

    Client->>LB: POST /api/users<br/>{"ssn": "123-45-6789"}
    LB->>App: リクエスト転送

    App->>KMS: GenerateDataKey<br/>(CMK-ID)
    KMS-->>App: {<br/>plaintext_key: "...",<br/>encrypted_key: "..."<br/>}

    App->>App: データ暗号化<br/>AES-256-GCM(ssn, plaintext_key)

    App->>DB: INSERT INTO users<br/>(encrypted_ssn, encrypted_key)

    Note over Client,DB: 機密データの復号

    App->>DB: SELECT encrypted_ssn, encrypted_key<br/>FROM users
    DB-->>App: 暗号化データ

    App->>KMS: Decrypt(encrypted_key)
    KMS-->>App: plaintext_key

    App->>App: データ復号<br/>AES-256-GCM-Decrypt

    App-->>Client: {"ssn": "***-**-6789"}<br/>(マスキング)
```

```mermaid
graph TB
    subgraph "OWASP Top 10対策"
        Injection[1. インジェクション<br/>SQLi, XSS対策<br/>パラメータ化クエリ<br/>入力サニタイズ]

        BrokenAuth[2. 認証の不備<br/>MFA導入<br/>セッション管理<br/>パスワードポリシー]

        SensitiveData[3. 機密データ露出<br/>暗号化<br/>TLS必須<br/>最小権限]

        XXE[4. XXE<br/>XML外部エンティティ無効化]

        BrokenAccess[5. アクセス制御の不備<br/>RBAC/ABAC<br/>認可チェック]

        Misconfig[6. セキュリティ設定ミス<br/>デフォルト設定変更<br/>不要機能無効化]

        XSS[7. XSS<br/>出力エスケープ<br/>CSP設定]

        Deserialization[8. 安全でないデシリアライズ<br/>入力検証<br/>署名付きデータ]

        Components[9. 脆弱なコンポーネント<br/>依存関係監査<br/>定期更新]

        Logging[10. ログ・監視不足<br/>セキュリティログ<br/>異常検知]
    end
```

### 設計のポイント

TLS 1.3を使用し、TLS 1.0/1.1は無効化します。
証明書は信頼できるCAから取得し、自己署名証明書は本番環境で使用しません。
機密データ(PII、クレジットカード番号等)は、保存時も暗号化(Encryption at Rest)します。
暗号鍵は、KMS(Key Management Service)で一元管理し、定期的にローテーションします。
パスワードは、bcryptやArgon2でハッシュ化して保存し、平文で保存しません。
SQLインジェクション対策として、パラメータ化クエリを使用し、ユーザー入力を直接SQLに含めません。
CSP(Content Security Policy)ヘッダーを設定して、XSS攻撃を軽減します。
