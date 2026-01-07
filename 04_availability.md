# 可用性・耐障害性・信頼性設計

## 冗長化とフェイルオーバーによるシステムを切り替え

### 概要

システムの冗長化により、単一障害点を排除し、高可用性を実現します。
フェイルオーバー機構により、障害発生時に自動的に待機系に切り替えます。

### システム設計図

```mermaid
graph TB
    subgraph "クライアント層"
        Client[クライアント]
    end
    
    subgraph "DNS/ロードバランサー"
        DNS[DNSフェイルオーバー<br/>ヘルスチェック]
        LB_Primary[ロードバランサー<br/>プライマリ]
        LB_Secondary[ロードバランサー<br/>セカンダリ]
        
        DNS -->|正常時| LB_Primary
        DNS -.->|障害時| LB_Secondary
        LB_Primary <-.->|ヘルスチェック| LB_Secondary
    end
    
    subgraph "アプリケーション層"
        App1[アプリケーション1]
        App2[アプリケーション2]
        App3[アプリケーション3]
        
        LB_Primary --> App1
        LB_Primary --> App2
        LB_Secondary -.-> App3
    end
    
    subgraph "データベース層"
        DB_Master[(マスターDB<br/>書き込み)]
        DB_Standby[(スタンバイDB<br/>同期レプリケーション)]
        DB_Replica[(レプリカDB<br/>読み取り)]
        
        DB_Master -->|同期レプリケーション| DB_Standby
        DB_Master -->|非同期レプリケーション| DB_Replica
        
        App1 -->|書き込み| DB_Master
        App2 -->|書き込み| DB_Master
        App1 -->|読み取り| DB_Replica
        App2 -->|読み取り| DB_Replica
    end
    
    DB_Master <-.->|自動フェイルオーバー| DB_Standby
```

```mermaid
sequenceDiagram
    participant Client as クライアント
    participant LB as ロードバランサー
    participant App1 as アプリ1(プライマリ)
    participant App2 as アプリ2(スタンバイ)
    participant DB_M as マスターDB
    participant DB_S as スタンバイDB

    Client->>LB: リクエスト
    LB->>App1: 転送
    App1->>DB_M: クエリ実行
    DB_M-->>App1: データ返却
    App1-->>Client: レスポンス
    
    Note over App1,DB_M: プライマリサーバー障害発生
    
    LB->>App1: ヘルスチェック
    App1--xLB: タイムアウト
    LB->>App1: ヘルスチェック(リトライ)
    App1--xLB: タイムアウト
    LB->>LB: App1を切り離し
    
    Client->>LB: リクエスト
    LB->>App2: 転送(フェイルオーバー)
    App2->>DB_M: クエリ実行
    DB_M-->>App2: データ返却
    App2-->>Client: レスポンス
    
    Note over DB_M,DB_S: マスターDB障害発生
    
    App2->>DB_M: クエリ実行
    DB_M--xApp2: 接続失敗
    App2->>App2: DB接続リトライロジック
    App2->>DB_S: フェイルオーバー接続
    DB_S->>DB_S: マスターに昇格
    DB_S-->>App2: データ返却
    App2-->>Client: レスポンス
```

### 設計のポイント

ヘルスチェックの間隔と閾値を適切に設定して、誤検知を防ぎます。
同期レプリケーションを使用して、データ損失を最小限に抑えます。
フェイルオーバー時のダウンタイムを最小化するため、自動フェイルオーバーを実装します。
スプリットブレイン問題を防ぐため、クォーラムベースの判定を使用します。
定期的にフェイルオーバーテストを実施して、機構が正常に動作することを確認します。

## レプリケーションによる耐障害性、可用性の向上

### 概要

データベースのレプリケーションにより、データの冗長性を確保し、障害時のデータ損失を防ぎます。
複数のレプリケーション方式を理解し、要件に応じて選択します。

### システム設計図

```mermaid
graph TB
    subgraph "同期レプリケーション"
        SyncMaster[(マスターDB)]
        SyncStandby1[(スタンバイDB1)]
        SyncStandby2[(スタンバイDB2)]
        
        SyncMaster -->|同期レプリケーション<br/>書き込み完了待機| SyncStandby1
        SyncMaster -->|同期レプリケーション<br/>書き込み完了待機| SyncStandby2
    end
    
    subgraph "非同期レプリケーション"
        AsyncMaster[(マスターDB)]
        AsyncReplica1[(レプリカDB1)]
        AsyncReplica2[(レプリカDB2)]
        AsyncReplica3[(レプリカDB3)]
        
        AsyncMaster -->|非同期レプリケーション<br/>即座に完了| AsyncReplica1
        AsyncMaster -->|非同期レプリケーション<br/>即座に完了| AsyncReplica2
        AsyncMaster -->|非同期レプリケーション<br/>即座に完了| AsyncReplica3
    end
    
    subgraph "準同期レプリケーション"
        SemiMaster[(マスターDB)]
        SemiStandby1[(スタンバイDB1<br/>同期対象)]
        SemiReplica1[(レプリカDB1<br/>非同期)]
        SemiReplica2[(レプリカDB2<br/>非同期)]
        
        SemiMaster -->|準同期<br/>1台以上の完了待機| SemiStandby1
        SemiMaster -->|非同期| SemiReplica1
        SemiMaster -->|非同期| SemiReplica2
    end
```

```mermaid
sequenceDiagram
    participant App as アプリケーション
    participant Master as マスターDB
    participant Standby1 as スタンバイ1
    participant Standby2 as スタンバイ2

    Note over App,Standby2: 同期レプリケーション
    
    App->>Master: UPDATE users SET name='太郎'
    Master->>Master: WALに書き込み
    
    par 同期レプリケーション
        Master->>Standby1: WAL送信
        Standby1->>Standby1: WAL適用
        Standby1-->>Master: ACK
    and
        Master->>Standby2: WAL送信
        Standby2->>Standby2: WAL適用
        Standby2-->>Master: ACK
    end
    
    Master->>Master: 全スタンバイのACK確認
    Master-->>App: 更新成功
    
    Note over App,Standby2: 準同期レプリケーション
    
    App->>Master: INSERT INTO orders VALUES(...)
    Master->>Master: WALに書き込み
    
    par 準同期レプリケーション
        Master->>Standby1: WAL送信
        Standby1->>Standby1: WAL適用
        Standby1-->>Master: ACK
    and
        Master->>Standby2: WAL送信(非同期)
        Note over Standby2: ACK待機なし
    end
    
    Master->>Master: 1台以上のACK確認
    Master-->>App: 挿入成功
```

### 設計のポイント

同期レプリケーションは、データ損失ゼロを保証しますが、パフォーマンスが低下します。
非同期レプリケーションは、高パフォーマンスですが、障害時にデータ損失のリスクがあります。
準同期レプリケーションは、両者のバランスを取った方式です。
レプリケーション遅延を監視して、閾値を超えた場合にアラートを発行します。
クロスリージョンレプリケーションにより、災害対策を実現します。

## ログによるデータベースの耐障害性の向上

### 概要

WAL(Write-Ahead Logging)やトランザクションログを使用して、障害時のデータ復旧を可能にします。
ログベースのレプリケーションにより、データの整合性を保ちます。

### システム設計図

```mermaid
graph TB
    subgraph "書き込みプロセス"
        Client[クライアント]
        App[アプリケーション]
        WAL[WALバッファ]
        WALFile[WALファイル<br/>ディスク]
        DataFile[データファイル<br/>ディスク]
        
        Client -->|書き込みリクエスト| App
        App -->|1. WAL書き込み| WAL
        WAL -->|2. ディスク永続化| WALFile
        WAL -.->|3. 非同期書き込み| DataFile
    end
    
    subgraph "レプリケーション"
        WALFile -->|WAL転送| ReplicaWAL[レプリカWAL]
        ReplicaWAL -->|WAL適用| ReplicaData[レプリカデータ]
    end
    
    subgraph "リカバリプロセス"
        CrashPoint[障害発生]
        WALReplay[WALリプレイ]
        Recovery[データ復旧]
        
        CrashPoint -->|起動時| WALReplay
        WALReplay -->|未反映の変更適用| Recovery
    end
```

```mermaid
sequenceDiagram
    participant App as アプリケーション
    participant WAL as WALバッファ
    participant WALDisk as WALディスク
    participant DataDisk as データディスク
    participant Replica as レプリカ

    App->>WAL: トランザクション開始<br/>INSERT INTO users...
    WAL->>WAL: WALレコード生成<br/>(LSN: 1000)
    
    App->>WAL: COMMIT
    WAL->>WALDisk: WAL永続化(fsync)
    WALDisk-->>WAL: 完了
    WAL-->>App: COMMIT成功
    
    Note over WAL,DataDisk: バックグラウンド処理
    WAL->>DataDisk: チェックポイント<br/>WALをデータファイルに反映
    
    par レプリケーション
        WALDisk->>Replica: WAL転送(LSN: 1000)
        Replica->>Replica: WAL適用
    end
    
    Note over App,Replica: サーバー障害発生
    
    App->>WALDisk: サーバー再起動
    WALDisk->>WALDisk: リカバリ開始
    WALDisk->>DataDisk: 最後のチェックポイント検出
    WALDisk->>WALDisk: WALリプレイ<br/>(未反映のトランザクション適用)
    WALDisk->>WALDisk: リカバリ完了
    WALDisk-->>App: データベース利用可能
```

### 設計のポイント

WALファイルは、データファイルとは別のディスクに配置して、I/O性能を向上させます。
定期的にチェックポイントを実行して、リカバリ時間を短縮します。
WALアーカイブを別のストレージに保存して、Point-in-Time Recoveryを可能にします。
ログの肥大化を防ぐため、古いWALファイルを定期的に削除します。
レプリカはWALを受信してリプレイすることで、マスターと同じ状態を保ちます。

## オブジェクトストレージによる非構造化データの耐久性の向上

### 概要

S3などのオブジェクトストレージは、高い耐久性と可用性を提供します。
画像、動画、ログファイルなどの非構造化データを安全に保存します。

### システム設計図

```mermaid
graph TB
    subgraph "クライアント層"
        User[ユーザー]
        App[アプリケーション]
    end
    
    subgraph "オブジェクトストレージ(S3)"
        S3_Primary[S3プライマリリージョン<br/>自動レプリケーション]
        S3_Secondary[S3セカンダリリージョン<br/>クロスリージョンレプリケーション]
        
        S3_Primary -->|自動レプリケーション| S3_Secondary
    end
    
    subgraph "バージョニング"
        Version1[オブジェクト v1]
        Version2[オブジェクト v2]
        Version3[オブジェクト v3<br/>最新]
        VersionDeleted[削除マーカー]
        
        Version1 --> Version2
        Version2 --> Version3
        Version3 -.-> VersionDeleted
    end
    
    subgraph "ライフサイクル管理"
        Standard[Standardクラス<br/>頻繁アクセス]
        IA[Infrequent Accessクラス<br/>低頻度アクセス]
        Glacier[Glacierクラス<br/>アーカイブ]
        Delete[削除]
        
        Standard -->|30日後| IA
        IA -->|90日後| Glacier
        Glacier -->|365日後| Delete
    end
    
    User -->|ファイルアップロード| App
    App -->|PUT Object| S3_Primary
```

```mermaid
sequenceDiagram
    participant User as ユーザー
    participant App as アプリケーション
    participant S3 as S3プライマリ
    participant S3_CRR as S3セカンダリ
    participant DB as データベース

    User->>App: ファイルアップロード<br/>(10MB画像)
    App->>App: Pre-signed URL生成<br/>(有効期限: 15分)
    App-->>User: Upload URL返却
    
    User->>S3: PUT /bucket/images/photo.jpg<br/>(直接アップロード)
    S3->>S3: オブジェクト保存
    S3->>S3: 3つのAZに自動レプリケーション
    S3->>S3: MD5チェックサム検証
    S3-->>User: 200 OK, ETag
    
    par クロスリージョンレプリケーション
        S3->>S3_CRR: オブジェクトレプリケーション
        S3_CRR->>S3_CRR: セカンダリリージョンに保存
    end
    
    User->>App: アップロード完了通知<br/>(Object Key)
    App->>DB: メタデータ保存<br/>(file_path, size, etag)
    
    Note over User,DB: ファイル取得
    
    User->>App: ファイル取得リクエスト
    App->>DB: メタデータ取得
    App->>App: Pre-signed URL生成<br/>(GET用、有効期限: 1時間)
    App-->>User: Download URL返却
    User->>S3: GET /bucket/images/photo.jpg
    S3-->>User: ファイルダウンロード
```

### 設計のポイント

S3の耐久性は99.999999999%(イレブンナイン)です。
バージョニングを有効化して、誤削除や上書きから保護します。
クロスリージョンレプリケーションにより、災害対策を実現します。
Pre-signed URLを使用して、アプリケーションサーバーを経由せずに直接アップロード/ダウンロードします。
ライフサイクルポリシーを設定して、古いデータを自動的に低コストのストレージクラスに移行します。

## リクエストのリトライ設計について

### 概要

一時的なネットワーク障害やサービス過負荷に対応するため、適切なリトライ戦略を実装します。
指数バックオフとジッターを使用して、サーバーへの負荷を分散します。

### システム設計図

```mermaid
graph TB
    subgraph "リトライ戦略"
        Request[リクエスト送信]
        Success{成功?}
        RetryDecision{リトライ可能?}
        BackOff[バックオフ計算]
        Retry[リトライ実行]
        Fail[失敗処理]
        
        Request --> Success
        Success -->|成功| Done[完了]
        Success -->|失敗| RetryDecision
        RetryDecision -->|Yes| BackOff
        RetryDecision -->|No| Fail
        BackOff --> Retry
        Retry --> Success
    end
    
    subgraph "リトライ可能な条件"
        HTTP5xx[HTTPステータス<br/>500, 502, 503, 504]
        Timeout[タイムアウト<br/>接続タイムアウト]
        Network[ネットワークエラー<br/>DNS解決失敗]
        RateLimit[レート制限<br/>429 Too Many Requests]
    end
    
    subgraph "リトライ不可の条件"
        HTTP4xx[HTTPステータス<br/>400, 401, 403, 404]
        Invalid[無効なリクエスト<br/>バリデーションエラー]
        Auth[認証エラー<br/>トークン無効]
    end
```

```mermaid
sequenceDiagram
    participant Client as クライアント
    participant API as APIサーバー

    Note over Client,API: 1回目の試行
    Client->>API: リクエスト送信
    API--xClient: 503 Service Unavailable
    
    Client->>Client: リトライ判定(503はリトライ可能)
    Client->>Client: バックオフ計算<br/>wait = min(base * 2^attempt, max)<br/>wait = min(1 * 2^0, 32) = 1秒
    Client->>Client: ジッター追加<br/>wait = 1 + random(0, 1) = 1.3秒
    
    Note over Client,API: 1.3秒待機
    
    Note over Client,API: 2回目の試行
    Client->>API: リクエスト送信
    API--xClient: タイムアウト
    
    Client->>Client: リトライ判定(タイムアウトはリトライ可能)
    Client->>Client: バックオフ計算<br/>wait = min(1 * 2^1, 32) = 2秒
    Client->>Client: ジッター追加<br/>wait = 2 + random(0, 2) = 3.1秒
    
    Note over Client,API: 3.1秒待機
    
    Note over Client,API: 3回目の試行
    Client->>API: リクエスト送信
    API-->>Client: 200 OK
    Client->>Client: 成功、処理完了
```

### 設計のポイント

指数バックオフを使用して、リトライ間隔を徐々に延ばします(1秒、2秒、4秒、8秒...)。
ジッターを追加して、複数のクライアントが同時にリトライするのを防ぎます。
最大リトライ回数を設定して、無限ループを防ぎます(例: 3回)。
冪等性を保証して、リトライによる重複処理を安全にします。
リトライ可能なエラーと不可能なエラーを明確に区別します。

## サーキットブレーカーでリトライによる過負荷を防ぐ

### 概要

サーキットブレーカーパターンにより、障害が発生しているサービスへのリクエストを一時的に遮断します。
システム全体の連鎖的な障害を防ぎ、障害からの復旧を促進します。

### システム設計図

```mermaid
stateDiagram-v2
    [*] --> Closed
    Closed --> Open: 失敗率が閾値超過
    Open --> HalfOpen: タイムアウト経過
    HalfOpen --> Closed: テストリクエスト成功
    HalfOpen --> Open: テストリクエスト失敗
    
    note right of Closed
        通常状態
        すべてのリクエストを通過
        失敗をカウント
    end note
    
    note right of Open
        遮断状態
        リクエストを即座に拒否
        Fast Fail
        タイムアウトまで待機
    end note
    
    note right of HalfOpen
        半開状態
        一部のリクエストを試行
        サービス復旧確認
    end note
```

```mermaid
sequenceDiagram
    participant App as アプリケーション
    participant CB as サーキットブレーカー
    participant Service as 外部サービス
    participant Fallback as フォールバック処理

    Note over App,Fallback: Closed状態(正常)
    
    App->>CB: リクエスト
    CB->>Service: 転送
    Service-->>CB: 成功
    CB-->>App: レスポンス
    
    Note over App,Fallback: サービス障害発生
    
    loop 失敗が増加
        App->>CB: リクエスト
        CB->>Service: 転送
        Service--xCB: 失敗(タイムアウト)
        CB->>CB: 失敗カウント増加
        CB->>Fallback: フォールバック実行
        Fallback-->>App: デフォルトレスポンス
    end
    
    CB->>CB: 失敗率50%超過を検知
    CB->>CB: Open状態に遷移
    
    Note over App,Fallback: Open状態(遮断中)
    
    App->>CB: リクエスト
    CB->>CB: 即座に拒否(Fast Fail)
    CB->>Fallback: フォールバック実行
    Fallback-->>App: デフォルトレスポンス
    
    Note over App,Fallback: 30秒経過
    
    CB->>CB: HalfOpen状態に遷移
    
    Note over App,Fallback: HalfOpen状態(テスト中)
    
    App->>CB: テストリクエスト
    CB->>Service: 転送(1回目)
    Service-->>CB: 成功
    CB-->>App: レスポンス
    
    App->>CB: テストリクエスト
    CB->>Service: 転送(2回目)
    Service-->>CB: 成功
    CB-->>App: レスポンス
    
    App->>CB: テストリクエスト
    CB->>Service: 転送(3回目)
    Service-->>CB: 成功
    CB-->>App: レスポンス
    
    CB->>CB: 連続成功を確認
    CB->>CB: Closed状態に遷移
```

### 設計のポイント

失敗率の閾値は、サービスの特性に応じて設定します(例: 50%)。
Open状態のタイムアウト時間は、サービスの復旧時間を考慮します(例: 30秒)。
フォールバック処理を実装して、サーキットブレーカーOpen時にもユーザーに何らかのレスポンスを返します。
メトリクスを監視して、サーキットブレーカーの状態遷移をダッシュボードで確認できるようにします。
各サービスごとに独立したサーキットブレーカーを設定します。

## メッセージキューによる信頼性向上

### 概要

メッセージキューを使用して、非同期処理の信頼性を向上させます。
メッセージの永続化、リトライ、Dead Letter Queueにより、処理の失敗に対応します。

### システム設計図

```mermaid
graph LR
    subgraph "プロデューサー"
        API[APIサーバー]
    end
    
    subgraph "メッセージキュー"
        Queue[メインキュー<br/>Visibility Timeout: 30s]
        DLQ[Dead Letter Queue<br/>最大3回リトライ後]
    end
    
    subgraph "コンシューマー"
        Worker1[ワーカー1]
        Worker2[ワーカー2]
    end
    
    subgraph "監視・再処理"
        Monitor[監視ダッシュボード]
        Replay[手動再処理]
    end
    
    API -->|Enqueue| Queue
    Queue -->|Dequeue| Worker1
    Queue -->|Dequeue| Worker2
    
    Worker1 -.->|処理失敗3回| DLQ
    Worker2 -.->|処理失敗3回| DLQ
    
    DLQ --> Monitor
    Monitor --> Replay
    Replay -.-> Queue
```

```mermaid
sequenceDiagram
    participant API as APIサーバー
    participant Queue as メッセージキュー
    participant Worker as ワーカー
    participant DLQ as Dead Letter Queue
    participant DB as データベース

    API->>Queue: メッセージ送信<br/>(Order ID: 123)
    Queue->>Queue: メッセージ永続化
    Queue-->>API: 送信成功
    
    Worker->>Queue: メッセージ取得
    Queue->>Queue: Visibility Timeout設定(30秒)
    Queue-->>Worker: メッセージ返却(Receive Count: 1)
    
    Worker->>DB: 注文処理
    DB--xWorker: データベースエラー
    Worker->>Worker: 処理失敗
    
    Note over Worker,Queue: Visibility Timeout経過(30秒)
    
    Queue->>Queue: メッセージ再表示
    Worker->>Queue: メッセージ取得(再試行)
    Queue-->>Worker: メッセージ返却(Receive Count: 2)
    
    Worker->>DB: 注文処理
    DB--xWorker: データベースエラー
    
    Note over Worker,Queue: Visibility Timeout経過(30秒)
    
    Queue->>Queue: メッセージ再表示
    Worker->>Queue: メッセージ取得(再試行)
    Queue-->>Worker: メッセージ返却(Receive Count: 3)
    
    Worker->>DB: 注文処理
    DB--xWorker: データベースエラー
    
    Queue->>Queue: Receive Count > 3を検知
    Queue->>DLQ: メッセージ移動
    
    Note over Worker,DLQ: 手動調査と修正
    
    DLQ->>Worker: 原因調査
    Worker->>DB: データ修正
    Worker->>Queue: メッセージ再投入
    Worker->>Queue: メッセージ取得
    Worker->>DB: 注文処理
    DB-->>Worker: 成功
    Worker->>Queue: メッセージ削除(ACK)
```

### 設計のポイント

メッセージの永続化により、キューサーバー障害時もメッセージを失いません。
Visibility Timeoutは、処理時間より長く設定します。
Receive Countを監視して、一定回数失敗したメッセージはDLQに移動します。
DLQのメッセージは定期的に確認し、根本原因を解決してから再処理します。
メッセージの順序保証が必要な場合は、FIFOキューを使用します。

## 大量のリクエストを制限する

### 概要

レート制限(Rate Limiting)により、システムを過負荷から保護します。
複数のアルゴリズムを理解し、ユースケースに応じて選択します。

### システム設計図

```mermaid
graph TB
    subgraph "レート制限アルゴリズム"
        FixedWindow[Fixed Window<br/>固定ウィンドウ]
        SlidingWindow[Sliding Window Log<br/>スライディングウィンドウ]
        TokenBucket[Token Bucket<br/>トークンバケット]
        LeakyBucket[Leaky Bucket<br/>リーキーバケット]
    end
    
    subgraph "実装レイヤー"
        Client[クライアント層<br/>クライアント側制限]
        Gateway[API Gateway層<br/>グローバル制限]
        Application[アプリケーション層<br/>ユーザー単位制限]
        Database[データベース層<br/>接続数制限]
    end
    
    subgraph "制限基準"
        IP["IPアドレス単位"]
        User["ユーザーID単位"]
        API["APIエンドポイント単位"]
        Global["グローバル全体"]
    end
```

```mermaid
sequenceDiagram
    participant Client as クライアント
    participant Gateway as API Gateway
    participant Redis as Redis(カウンター)
    participant API as APIサーバー

    Note over Client,API: Token Bucketアルゴリズム
    
    Client->>Gateway: リクエスト1(user_123)
    Gateway->>Redis: GET rate_limit:user_123
    Redis-->>Gateway: tokens: 10, last_refill: 1000
    Gateway->>Gateway: 経過時間に基づきトークン補充<br/>tokens = 10 + (1200-1000)/1 = 10
    Gateway->>Gateway: トークン消費(tokens = 9)
    Gateway->>Redis: SET rate_limit:user_123<br/>tokens: 9, last_refill: 1200
    Gateway->>API: リクエスト転送
    API-->>Client: レスポンス<br/>X-RateLimit-Limit: 10<br/>X-RateLimit-Remaining: 9
    
    loop 高速リクエスト
        Client->>Gateway: リクエスト2~11
        Gateway->>Redis: トークン確認・消費
        Gateway->>API: リクエスト転送
        API-->>Client: レスポンス
    end
    
    Client->>Gateway: リクエスト12(トークン枯渇)
    Gateway->>Redis: GET rate_limit:user_123
    Redis-->>Gateway: tokens: 0
    Gateway->>Gateway: トークン不足
    Gateway-->>Client: 429 Too Many Requests<br/>Retry-After: 60<br/>X-RateLimit-Limit: 10<br/>X-RateLimit-Remaining: 0
    
    Note over Client,API: 60秒待機
    
    Client->>Gateway: リクエスト13
    Gateway->>Redis: GET rate_limit:user_123
    Redis-->>Gateway: tokens: 0, last_refill: 1200
    Gateway->>Gateway: トークン補充<br/>tokens = 0 + (1260-1200)/1 = 10
    Gateway->>API: リクエスト転送
    API-->>Client: レスポンス
```

### 設計のポイント

Token Bucketアルゴリズムは、バースト的なトラフィックを許容しながら、長期的なレートを制限します。
Sliding Window Logアルゴリズムは、より正確なレート制限を実現しますが、メモリ使用量が多いです。
レート制限の設定は、APIの種類(読み取り/書き込み)やユーザーのティア(無料/有料)に応じて変更します。
429レスポンスには、Retry-Afterヘッダーを含めて、クライアントに再試行タイミングを伝えます。
分散環境では、Redisなどの共有ストレージを使用してカウンターを管理します。

## 監視とオブザーバビリティ(Prometheus, Grafana)

### 概要

システムの可用性を維持するため、メトリクス、ログ、トレースを収集・分析して、問題を早期に検知します。
Prometheus、Grafana、Jaegerなどのツールを組み合わせて、包括的な監視基盤を構築します。

### システム設計図

```mermaid
graph TB
    subgraph "アプリケーション層"
        App1[アプリ1<br/>/metrics]
        App2[アプリ2<br/>/metrics]
        App3[アプリ3<br/>/metrics]
    end

    subgraph "メトリクス収集"
        Prometheus[Prometheus<br/>時系列DB]
        Exporter1[Node Exporter<br/>システムメトリクス]
        Exporter2[Blackbox Exporter<br/>外形監視]
        AlertManager[AlertManager<br/>アラート管理]

        Prometheus -->|scrape| App1
        Prometheus -->|scrape| App2
        Prometheus -->|scrape| App3
        Prometheus -->|scrape| Exporter1
        Prometheus -->|scrape| Exporter2
        Prometheus -->|alerts| AlertManager
    end

    subgraph "ログ収集"
        Fluentd[Fluentd/Fluent Bit<br/>ログ収集]
        Elasticsearch[(Elasticsearch<br/>ログストレージ)]
        Kibana[Kibana<br/>ログ可視化]

        App1 -->|logs| Fluentd
        App2 -->|logs| Fluentd
        App3 -->|logs| Fluentd
        Fluentd --> Elasticsearch
        Elasticsearch --> Kibana
    end

    subgraph "分散トレーシング"
        Jaeger[Jaeger/Zipkin<br/>トレース収集]
        TraceStore[(トレースストレージ)]

        App1 -->|traces| Jaeger
        App2 -->|traces| Jaeger
        App3 -->|traces| Jaeger
        Jaeger --> TraceStore
    end

    subgraph "可視化・アラート"
        Grafana[Grafana<br/>統合ダッシュボード]
        Slack[Slack]
        PagerDuty[PagerDuty]

        Prometheus --> Grafana
        Elasticsearch --> Grafana
        Jaeger --> Grafana
        AlertManager --> Slack
        AlertManager --> PagerDuty
    end
```

```mermaid
graph LR
    subgraph "メトリクスの種類"
        Counter[Counter<br/>累積値<br/>例: リクエスト数]
        Gauge[Gauge<br/>現在値<br/>例: メモリ使用量]
        Histogram[Histogram<br/>分布<br/>例: レイテンシ]
        Summary[Summary<br/>パーセンタイル<br/>例: P99レイテンシ]
    end

    subgraph "4つのゴールデンシグナル"
        Latency[Latency<br/>応答時間]
        Traffic[Traffic<br/>リクエスト数]
        Errors[Errors<br/>エラー率]
        Saturation[Saturation<br/>リソース使用率]
    end
```

```mermaid
sequenceDiagram
    participant App as アプリケーション
    participant Prom as Prometheus
    participant Alert as AlertManager
    participant Grafana as Grafana
    participant OnCall as オンコール担当者

    Note over App,OnCall: 通常の監視フロー

    loop 15秒ごと
        Prom->>App: GET /metrics
        App-->>Prom: http_requests_total{status="200"} 1000<br/>http_requests_total{status="500"} 5<br/>http_request_duration_seconds{quantile="0.99"} 0.5
        Prom->>Prom: メトリクス保存(時系列DB)
    end

    Grafana->>Prom: PromQL: rate(http_requests_total[5m])
    Prom-->>Grafana: 時系列データ
    Grafana->>Grafana: グラフ描画

    Note over App,OnCall: 障害検知とアラート

    App->>Prom: エラー率上昇<br/>http_requests_total{status="500"} 100

    Prom->>Prom: アラートルール評価<br/>rate(http_requests_total{status="500"}[5m])<br/>> 0.05

    Prom->>Alert: アラート発火<br/>ErrorRateHigh: 10%

    Alert->>Alert: グルーピング<br/>抑制ルール適用

    Alert->>OnCall: Slack通知<br/>PagerDuty呼び出し

    OnCall->>Grafana: ダッシュボード確認
    Grafana->>Prom: 関連メトリクス取得
    Prom-->>Grafana: CPU、メモリ、ディスク使用率

    OnCall->>OnCall: 原因調査・対応

    Note over App,OnCall: 分散トレーシング

    App->>App: リクエスト受信<br/>trace_id: abc123

    App->>Prom: メトリクス記録<br/>レイテンシ: 500ms

    OnCall->>Grafana: 遅いリクエストを調査
    Grafana->>Jaeger: trace_id: abc123
    Jaeger-->>Grafana: スパン一覧<br/>API: 50ms<br/>DB: 400ms<br/>外部API: 50ms

    OnCall->>OnCall: DBクエリが遅い原因を特定
```

### 設計のポイント

Prometheusは、Pull型でメトリクスを収集し、PromQLで柔軟なクエリが可能です。
4つのゴールデンシグナル(Latency、Traffic、Errors、Saturation)を監視することで、システムの健全性を把握します。
アラートは、症状ベース(エラー率が高い)で設定し、原因ベース(CPU使用率が高い)は避けます。
アラート疲れを防ぐため、適切な閾値設定と抑制ルールを設定します。
分散トレーシングにより、マイクロサービス間のリクエストフローを可視化し、ボトルネックを特定します。
SLI(Service Level Indicator)とSLO(Service Level Objective)を定義して、可用性の目標を明確にします。
