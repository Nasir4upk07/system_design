# コミュニケーション設計

## リクエストを送信(HTTP)

### 概要

HTTPは、クライアントとサーバー間でデータをやり取りする際の標準的なプロトコルです。
RESTful APIを実装する場合、HTTP methodsを適切に使い分け、ステートレスな設計を実現します。

### システム設計図

```mermaid
sequenceDiagram
    participant Client as クライアント
    participant LB as ロードバランサー
    participant API as APIサーバー
    participant Cache as キャッシュ(Redis)
    participant DB as データベース

    Client->>LB: HTTPリクエスト(GET/POST/PUT/DELETE)
    LB->>API: リクエスト転送
    API->>Cache: キャッシュ確認
    alt キャッシュヒット
        Cache-->>API: キャッシュデータ
    else キャッシュミス
        API->>DB: クエリ実行
        DB-->>API: データ取得
        API->>Cache: キャッシュ保存
    end
    API-->>LB: HTTPレスポンス(JSON)
    LB-->>Client: レスポンス返却
```

### 設計のポイント

ロードバランサーを配置することで、複数のAPIサーバーに負荷を分散します。
キャッシュレイヤーを導入することで、データベースへの負荷を軽減し、レスポンス速度を向上させます。
HTTPステータスコードを適切に使用し、エラーハンドリングを明確にします。
冪等性を保証することで、ネットワーク障害時のリトライを安全に実行できます。

## サーバー側の更新をリアルタイムに受け取る(Polling / Long Polling / SSE)

### 概要

サーバー側の状態変更をクライアントに通知する方法として、Polling、Long Polling、Server-Sent Events(SSE)があります。
それぞれの特性を理解し、ユースケースに応じて選択します。

### システム設計図

```mermaid
sequenceDiagram
    participant Client as クライアント
    participant LB as ロードバランサー
    participant API as APIサーバー
    participant MQ as メッセージキュー
    participant DB as データベース

    Note over Client,DB: Short Polling方式
    loop 定期的なポーリング(例: 5秒ごと)
        Client->>API: データ取得リクエスト
        API->>DB: 最新データ確認
        DB-->>API: データ返却
        API-->>Client: レスポンス(データまたは304)
    end

    Note over Client,DB: Long Polling方式
    Client->>API: データ取得リクエスト
    API->>MQ: メッセージ待機(タイムアウト60秒)
    alt 新しいイベント発生
        MQ-->>API: イベント通知
        API-->>Client: イベントデータ返却
        Client->>API: 再接続リクエスト
    else タイムアウト
        API-->>Client: 空レスポンス
        Client->>API: 再接続リクエスト
    end

    Note over Client,DB: Server-Sent Events方式
    Client->>API: SSE接続確立
    API->>Client: 接続確立(text/event-stream)
    loop イベント発生時
        MQ->>API: イベント通知
        API->>Client: data: イベントデータ送信
    end
```

### 設計のポイント

Short Pollingは、実装がシンプルですが、無駄なリクエストが多く発生します。
Long Pollingは、リアルタイム性が向上しますが、コネクション管理が必要です。
SSEは、サーバーからクライアントへの一方向通信に適しており、自動再接続機能を持ちます。
タイムアウト値は、ネットワーク環境やユースケースに応じて調整します。

## リアルタイム双方向通信(WebSocket)

### 概要

WebSocketは、クライアントとサーバー間で双方向のリアルタイム通信を実現するプロトコルです。
チャットアプリケーションやリアルタイムコラボレーションツールなどで使用されます。

### システム設計図

```mermaid
sequenceDiagram
    participant Client1 as クライアント1
    participant Client2 as クライアント2
    participant LB as ロードバランサー
    participant WS1 as WebSocketサーバー1
    participant WS2 as WebSocketサーバー2
    participant Redis as Redis Pub/Sub
    participant DB as データベース

    Client1->>LB: WebSocketハンドシェイク
    LB->>WS1: 接続転送(Sticky Session)
    WS1-->>Client1: 接続確立

    Client2->>LB: WebSocketハンドシェイク
    LB->>WS2: 接続転送(Sticky Session)
    WS2-->>Client2: 接続確立

    Client1->>WS1: メッセージ送信
    WS1->>DB: メッセージ保存
    WS1->>Redis: Publish(channel: room_id)
    Redis->>WS1: Subscribe通知
    Redis->>WS2: Subscribe通知
    WS1->>Client1: メッセージ配信
    WS2->>Client2: メッセージ配信
```

### 設計のポイント

Sticky Sessionを使用して、同じクライアントを同じサーバーに接続し続けます。
Redis Pub/Subを使用して、複数のWebSocketサーバー間でメッセージを同期します。
コネクション数の上限を考慮し、サーバーのスケーリングを計画します。
ハートビートを定期的に送信して、接続の生存確認を行います。
再接続時のメッセージ取得ロジックを実装し、メッセージの欠落を防ぎます。

## サービス間通信の設計

### 概要

マイクロサービスアーキテクチャにおいて、サービス間の通信方法を適切に設計します。
同期通信(REST、gRPC)と非同期通信(メッセージキュー)を使い分けます。

### システム設計図

```mermaid
graph TB
    subgraph "同期通信"
        Client[クライアント]
        Gateway[API Gateway]
        UserService[ユーザーサービス]
        OrderService[注文サービス]
        PaymentService[決済サービス]
        
        Client -->|REST| Gateway
        Gateway -->|gRPC| UserService
        Gateway -->|gRPC| OrderService
        OrderService -->|gRPC| PaymentService
    end
    
    subgraph "非同期通信"
        OrderServiceAsync[注文サービス]
        Queue[メッセージキュー]
        InventoryService[在庫サービス]
        NotificationService[通知サービス]
        
        OrderServiceAsync -->|Publish| Queue
        Queue -->|Subscribe| InventoryService
        Queue -->|Subscribe| NotificationService
    end
    
    subgraph "サービスディスカバリ"
        ServiceRegistry[サービスレジストリ]
        UserService -.->|登録| ServiceRegistry
        OrderService -.->|登録| ServiceRegistry
        PaymentService -.->|登録| ServiceRegistry
        Gateway -.->|検索| ServiceRegistry
    end
```

### 設計のポイント

同期通信は、即座にレスポンスが必要な場合に使用します。
非同期通信は、処理の完了を待つ必要がない場合や、複数サービスへの通知が必要な場合に使用します。
サービスディスカバリを導入して、動的なサービス検出とロードバランシングを実現します。
Circuit Breakerパターンを実装して、障害の連鎖を防ぎます。
タイムアウト値とリトライポリシーを適切に設定します。

## REST vs gRPC vs GraphQL

### 概要

API設計において、REST、gRPC、GraphQLは主要な選択肢です。
それぞれの特性を理解し、ユースケースに応じて選択します。

### システム設計図

```mermaid
graph TB
    subgraph "REST API"
        RESTClient[クライアント]
        RESTEndpoint1[GET /users/123]
        RESTEndpoint2[POST /users]
        RESTEndpoint3[GET /users/123/orders]

        RESTClient -->|HTTP/1.1 or HTTP/2| RESTEndpoint1
        RESTClient -->|JSON| RESTEndpoint2
        RESTClient -->|複数リクエスト必要| RESTEndpoint3
    end

    subgraph "gRPC"
        gRPCClient[クライアント]
        gRPCService[UserService.GetUser]
        gRPCStream[OrderService.StreamOrders]

        gRPCClient -->|HTTP/2| gRPCService
        gRPCClient -->|Protocol Buffers| gRPCService
        gRPCClient -->|双方向ストリーミング| gRPCStream
    end

    subgraph "GraphQL"
        GQLClient[クライアント]
        GQLEndpoint[POST /graphql]
        GQLResolver1[User Resolver]
        GQLResolver2[Order Resolver]

        GQLClient -->|単一エンドポイント| GQLEndpoint
        GQLEndpoint --> GQLResolver1
        GQLEndpoint --> GQLResolver2
    end
```

```mermaid
sequenceDiagram
    participant Client as クライアント
    participant REST as REST API
    participant gRPC as gRPCサービス
    participant GraphQL as GraphQL API
    participant DB as データベース

    Note over Client,DB: REST: Over-fetching/Under-fetching問題

    Client->>REST: GET /users/123
    REST->>DB: SELECT * FROM users
    DB-->>REST: 全カラム取得
    REST-->>Client: {id, name, email, address, ...}<br/>(不要なデータも含む)

    Client->>REST: GET /users/123/orders
    REST->>DB: SELECT * FROM orders
    DB-->>REST: 注文データ
    REST-->>Client: {orders: [...]}<br/>(2回目のリクエスト)

    Note over Client,DB: gRPC: 高速なバイナリ通信

    Client->>gRPC: GetUser(user_id: 123)<br/>Protocol Buffers
    gRPC->>DB: クエリ実行
    DB-->>gRPC: ユーザーデータ
    gRPC-->>Client: UserResponse(バイナリ)<br/>シリアライズ高速

    Note over Client,DB: GraphQL: 必要なデータのみ取得

    Client->>GraphQL: query {<br/>  user(id: 123) {<br/>    name<br/>    orders { id, amount }<br/>  }<br/>}
    GraphQL->>DB: 必要なフィールドのみ取得
    DB-->>GraphQL: ユーザー + 注文データ
    GraphQL-->>Client: {user: {name, orders}}<br/>(1回のリクエストで完結)
```

```mermaid
graph LR
    subgraph "比較表"
        REST_Pros["REST<br/>シンプル、キャッシュ容易<br/>広く普及、ツール豊富"]
        REST_Cons["REST<br/>Over/Under-fetching<br/>複数リクエスト必要な場合あり"]

        gRPC_Pros["gRPC<br/>高速 - バイナリ<br/>型安全、コード生成<br/>ストリーミング対応"]
        gRPC_Cons["gRPC<br/>ブラウザ直接利用不可<br/>デバッグ困難"]

        GraphQL_Pros["GraphQL<br/>必要なデータのみ取得<br/>単一エンドポイント<br/>型システム"]
        GraphQL_Cons["GraphQL<br/>キャッシュ複雑<br/>N+1問題<br/>学習コスト"]
    end

    subgraph "ユースケース"
        RESTCase["REST: 公開API<br/>シンプルなCRUD"]
        gRPCCase["gRPC: マイクロサービス間<br/>低レイテンシ要求"]
        GraphQLCase["GraphQL: モバイルアプリ<br/>複雑なデータ要求"]
    end
```

### 設計のポイント

RESTは、HTTPの標準的なメソッドとステータスコードを使用し、リソース指向の設計に適しています。
gRPCは、Protocol Buffersによるスキーマ定義とコード生成により、マイクロサービス間の高速通信に適しています。
GraphQLは、クライアントが必要なデータを正確に指定でき、モバイルアプリなど帯域幅が制限される環境に適しています。
gRPCはブラウザから直接利用できないため、grpc-webまたはAPI Gatewayで変換が必要です。
GraphQLのN+1問題はDataLoaderパターンで解決します。
公開APIにはREST、内部通信にはgRPC、複雑なクエリ要求にはGraphQLという組み合わせが一般的です。

## Pub/Subで複数のサービスにメッセージ配信

### 概要

Publish/Subscribeパターンを使用して、1つのイベントを複数のサービスに非同期で配信します。
サービス間の疎結合を実現し、システムの拡張性を向上させます。

### システム設計図

```mermaid
graph LR
    subgraph "Publisher"
        OrderService[注文サービス]
    end
    
    subgraph "Message Broker"
        Topic1[注文完了トピック]
        Topic2[在庫更新トピック]
    end
    
    subgraph "Subscribers"
        InventoryService[在庫サービス]
        NotificationService[通知サービス]
        AnalyticsService[分析サービス]
        BillingService[請求サービス]
    end
    
    OrderService -->|Publish: OrderCompleted| Topic1
    OrderService -->|Publish: InventoryChanged| Topic2
    
    Topic1 -->|Subscribe| InventoryService
    Topic1 -->|Subscribe| NotificationService
    Topic1 -->|Subscribe| AnalyticsService
    Topic1 -->|Subscribe| BillingService
    
    Topic2 -->|Subscribe| AnalyticsService
```

```mermaid
sequenceDiagram
    participant Order as 注文サービス
    participant Broker as メッセージブローカー
    participant Inventory as 在庫サービス
    participant Notification as 通知サービス
    participant Analytics as 分析サービス

    Order->>Broker: Publish(OrderCompleted Event)
    Note over Broker: イベントをトピックに保存
    
    par 並列配信
        Broker->>Inventory: OrderCompleted Event
        Inventory->>Inventory: 在庫更新処理
        Inventory-->>Broker: ACK
    and
        Broker->>Notification: OrderCompleted Event
        Notification->>Notification: メール送信処理
        Notification-->>Broker: ACK
    and
        Broker->>Analytics: OrderCompleted Event
        Analytics->>Analytics: 分析データ保存
        Analytics-->>Broker: ACK
    end
```

### 設計のポイント

メッセージブローカー(Kafka、RabbitMQ、AWS SNS/SQSなど)を使用して、信頼性の高いメッセージ配信を実現します。
各サブスクライバーは独立して動作し、他のサブスクライバーの障害に影響されません。
メッセージの順序保証が必要な場合は、パーティションキーを使用します。
メッセージの重複配信に対応するため、冪等性を考慮した処理を実装します。
Dead Letter Queueを設定して、処理に失敗したメッセージを別途管理します。
