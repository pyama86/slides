# 成長期における、ユーザー領域の複雑さと整備の進め方

タクシー配車アプリ「GO」のバックエンドAPIサーバーは、クリーンアーキテクチャを参考にしつつ独自の命名規則を採用したGoで実装されている。本稿では、サービスの成長に伴い複数の層に散在していたユーザー関連のバリデーション処理をドメインサービスに集約した経緯、ケーパビリティ（capability）という概念を導入してエンドポイント単位のアクセス制御を実現した設計、そしてデータベーススキーマ変更を安全に自動化するツール「alterguard」の開発について解説する。

---

## このプロジェクトのアーキテクチャ

本題に入る前に、このプロジェクト固有のアーキテクチャについて説明しておく。一般的なDDDやクリーンアーキテクチャの知識を持つ読者ほど、最初に命名の違いに戸惑うためだ。

### 独自の命名規則

**表1 パッケージと一般的なDDDの対応**

| このプロジェクト | 一般的なDDDの呼び名 | 責務 |
|----------------|-------------------|------|
| `handler/` | Controller層 | HTTPリクエスト処理 |
| `controller/` | UseCase層 ⚠️ | ビジネスロジック |
| `usecase/` | Interface Adapter層 ⚠️ | DB操作の橋渡し |
| `domain/model` | Entity層 | ドメインモデル |
| `domain/service` | Domain Service層 | 複数controller間の調整 |
| `infra/` | Infrastructure層 | Repository実装 |

`controller/`がUseCaseに、`usecase/`がInterface Adapterに相当するという命名は、歴史的経緯によるものだ。一般的なMVCフレームワークのControllerとは異なるため注意が必要である。

### 呼び出しフロー

コードベースの大半は次の標準パターンを採用している。

```
handler → controller → usecase → infra → DB
```

一方、複数のcontrollerを組み合わせる必要がある複雑なビジネスロジックに対しては、`domain/service`を介するパターンを使用する。

```
handler → domain/service → controller → usecase → infra → DB
```

### domain/serviceの立ち位置

一般的なDDDにおけるドメインサービスは「ドメインモデルに書けない純粋なビジネスロジック」を指す。しかし本プロジェクトの`domain/service`は、**複数のcontrollerを組み合わせる必要があるロジック**の置き場所として機能する。

`domain/service`の使用は必要最小限に留め、「handlerが複数のcontrollerを直接newしている」状態が見受けられるときに移行を検討する。判断基準はポリシーとして明文化している。

---

## 第一の整備：バリデーションの散在をリファクタリングする

### 問題の状態

サービスが成長する過程で、ユーザーのバリデーション処理が複数のhandlerに重複して存在するようになっていた。配車作成・事前確定運賃・AI予約という3種類のhandlerそれぞれに、ほぼ同様の処理が記述されていた。

**リスト1 リファクタリング前のhandler（抜粋）**

```go
// 各handlerが複数のcontrollerをそれぞれnewして組み合わせていた
uc := controller.NewUserController(logger, h.Repo, h.ReplicaRepo, memRepo)
user, err := uc.Show(userID)
if err != nil { ... }

payment, familyLinkGroup, err :=
    h.FamilyLinkGroupController.ValidateCarRequestPaymentProfile(
        h.Ctx, user, payload.IsFamilyProfile(), payload.IsBusinessProfile(),
    )
if errorVM, ok := familylink.ConvertFamilyLinkErrors(err); ok { ... }

lc := controller.NewLimitationController(h.Repo)
if err := lc.ApplyPaymentRequiredLimitationIfNone(...); err != nil { ... }
```

コード自体は動作するが、バリデーション条件が3ファイルに散在することで、仕様変更時にすべての箇所を追わなければならない状態が生まれていた。また、「ユーザーはなぜここでチェックされているのか」という仕様の根拠が、コードから読み取りにくくなっていた。

**handlerが複数のcontrollerを直接newしている状態は、`domain/service`が必要なサインである。**

### domain/serviceへの集約

`UserValidationService`として、3つのhandlerに散在していたバリデーション処理を一箇所に集約した。

**リスト2 UserValidationService（概略）**

```go
type UserValidationService struct {
    userController            UserControllerInterface
    limitationController      LimitationControllerInterface
    familyLinkGroupController FamilyLinkGroupControllerInterface
    // ...
}

func (s *UserValidationService) ValidateUserForCarRequest(
    ctx context.Context,
    userID uint,
    // ...
) (*UserValidationResult, error) {
    user, err := s.userController.Show(userID)
    if err != nil { return nil, err }

    if err := s.validateUserLimitations(user); err != nil {
        return nil, err
    }
    // ファミリー連携・支払いプロファイルの検証もここで完結する
    // ...
}
```

エラーの種別も`domain/service`に集約し、呼び出し元が意味を持って処理できるよう整理した。

**リスト3 集約後のhandler（抜粋）**

```go
userValidationService := service.NewUserValidationService(
    h.Repo, h.ReplicaRepo, memRepo, h.FamilyLinkGroupController, logger,
)

validationResult, err := userValidationService.ValidateUserForCarRequest(
    h.Ctx, userID, ...
)
if err != nil {
    if service.IsOutstandingPaymentError(err) { ... }
    if service.IsTooManyCancellationsError(err) { ... }
    if service.IsPaymentRequiredError(err) { ... }
    ...
}
user := validationResult.User
```

handlerは「何をするか」のみを知ればよくなり、複数controllerの組み合わせ方という知識は`domain/service`が持つようになった。「ユーザーの配車依頼前バリデーション仕様を知りたければ`UserValidationService`を読む」という居場所ができたことが、この整備の本質的な成果である。

---

## 第二の整備：新機能を安全に乗せる仕組みを作る

### ニセコ展開という文脈

北海道のスキーリゾートであるニセコへのサービス展開を検討した際、技術的な課題が生じた。ニセコは外国人観光客が多いエリアであり、日本の携帯番号を持たないユーザーがSMS認証を完了できないという問題である。

解決策として、SMS認証なしでもGOを利用できる仕組みを導入した。ただし認証の緩和はリスクを伴うため、機能を限定することでリスクをコントロールするという設計判断を行った。

**表2 ニセコユーザーに設けた制約**

| 制約 | 内容 | 理由 |
|------|------|------|
| 電話番号登録不可 | SMS認証の代わりに制限 | 本人確認の代替 |
| クレカ3DS認証必須 | 決済時の本人確認を強化 | 不正利用の抑止 |
| Email MFA有効 | 代替認証手段として | SMS不要の補完 |
| 1日3件上限 | 配車数の制限 | 不正利用の抑止 |
| ニセコエリア外利用不可 | 現在地・乗車地・目的地すべて | サービス範囲の限定 |
| AI予約・チャーター等禁止 | 機能グループ単位で制限 | リスクの最小化 |
| ファミリー連携禁止 | 電話番号未登録のため | 機能依存関係の整理 |

### ナイーブな実装の問題

こうした制約を個別に実装していくと、ユーザー種別の判定が複数箇所に散在するという、第一の整備で解決したのと同種の問題が発生する。

```go
// あちこちのhandlerやcontrollerに生えていく
if profile.Name == "niseko" {
    limit = 3
}
if user.IsNiseko && !card.Is3DSVerified {
    return errors.New("3DS認証が必要です")
}
```

次のユーザー種別（別エリアの展開、法人向けプラン、など）が追加されるたびに同じパターンが繰り返され、仕様の把握が難しくなる。

### UserCapabilityインターフェース

「このユーザーは何ができるか」をインターフェースとして定義することで、ユーザー種別の仕様をコードとして表現した。

**リスト4 UserCapabilityインターフェース**

```go
type UserCapability interface {
    DailyDispatchLimit() *int            // 日次配車数制限（nilなら無制限）
    CanBatchDispatch() bool              // 複数台同時配車の可否
    CanRegisterPhoneNumber() bool        // 電話番号登録の可否
    SupportsEmailMFA() bool              // Email MFAの対応可否
    IsEndpointGroupAllowed(string) bool  // エンドポイントグループ単位の制御
    RequiredAreaIDForPickup() []GeoIDType // 乗車地のエリア制限
    Is3DSRequired() bool                 // 3DS認証の必須可否
    CanUseFamilyLink() bool              // ファミリー連携の可否
    // ...
}
```

通常ユーザー用の`DefaultUserCapability`はすべてのメソッドでデフォルト値（制限なし）を返す。ニセコ向けの`NisekoUserCapability`は制約に応じたメソッドをオーバーライドする。

**リスト5 NisekoUserCapability（抜粋）**

```go
func (NisekoUserCapability) CanRegisterPhoneNumber() bool { return false }
func (NisekoUserCapability) Is3DSRequired() bool          { return true }
func (NisekoUserCapability) SupportsEmailMFA() bool       { return true }
func (NisekoUserCapability) CanBatchDispatch() bool       { return false }
func (NisekoUserCapability) DailyDispatchLimit() *int {
    limit := 3
    return &limit
}
func (NisekoUserCapability) RequiredAreaIDForPickup() []GeoIDType {
    return []GeoIDType{GetNisekoAreaID()}
}
func (NisekoUserCapability) IsEndpointGroupAllowed(group string) bool {
    return !nisekoUserRestrictedEndpointGroups[group]
}
```

`NisekoUserCapability`を読むだけでニセコユーザーの仕様が全量把握できる。`DefaultUserCapability`との差分を見れば、通常ユーザーとの違いも明確だ。

新しいユーザー種別が必要になった場合は、`UserCapability`を実装した新しいstructを1ファイル追加するだけでよく、既存コードへの変更は不要である。

### エンドポイントグループとの連携

`IsEndpointGroupAllowed`が引数として受け取る文字列は、エンドポイントのグループ名である。`router/endpoint_groups.go`で全エンドポイントを機能単位にグルーピングし、リクエストのメソッドとパスから対応するグループ名を解決する仕組みを用意した。

具体的には、次のようにエンドポイントのスライスをグループ名に対応付ける。

**リスト6 エンドポイントグループの定義（例示）**

```go
// 配車関連のエンドポイント群
var dispatchEndpoints = []endpointPattern{
    {"POST", "/v1", "/car_requests"},
    {"GET",  "/v1", "/car_requests/{id}"},
    {"POST", "/v1", "/car_requests/{id}/confirm"},
    {"POST", "/v1", "/car_requests/{id}/cancel"},
}

// AI予約関連のエンドポイント群
var reservationEndpoints = []endpointPattern{
    {"POST",   "/v1", "/reservations"},
    {"GET",    "/v1", "/reservations/{id}"},
    {"DELETE", "/v1", "/reservations/{id}"},
}

// チャーター関連のエンドポイント群
var charterEndpoints = []endpointPattern{
    {"POST",   "/v1", "/charters"},
    {"GET",    "/v1", "/charters/{id}"},
    {"DELETE", "/v1", "/charters/{id}"},
}

// 車内決済関連のエンドポイント群
var walletEndpoints = []endpointPattern{
    {"POST", "/v1", "/wallet/sessions"},
    {"GET",  "/v1", "/wallet/balance"},
}
```

初期化時にこれらをグループ名へ対応付けるマップを構築し、リクエストのメソッドとパスからグループ名を解決する。

**リスト7 グループ名の解決**

```go
func init() {
    registerGroup("dispatch",    dispatchEndpoints)
    registerGroup("reservation", reservationEndpoints)  // ← ニセコは禁止
    registerGroup("charter",     charterEndpoints)      // ← ニセコは禁止
    registerGroup("wallet",      walletEndpoints)       // ← ニセコは禁止
    // ...
}

// リクエストが来たとき、グループ名を解決してケーパビリティで判定する
group := GetEndpointGroup(r.Method, prefix, r.URL.Path)
if !capability.IsEndpointGroupAllowed(group) {
    response.Forbidden(w)
    return
}
```

エンドポイントが増えても`registerGroup`にパターンを追加するだけでよく、ケーパビリティ側の実装には変更が不要である。

APIエンドポイントは200本を超えており、機能単位への分類は相応の作業量を伴う。実際にはAIツールにエンドポイントの一覧を渡し「機能単位に分類せよ」と指示することで、分類の草案を短時間で得た。分類の妥当性判断はエンジニアが行い、整理・分類の作業はAIと分担するというアプローチが有効だった。

---

## 第三の整備：DB変更を安全に自動化する

### 以前の作業の実態

MySQLの大きなテーブルに対するスキーマ変更は、`ALTER TABLE`を直接実行するとテーブルロックが発生し、サービスに影響を与える。そのため、行数が多いテーブルに対してはPercona Toolkitの`pt-online-schema-change`（以下pt-osc）を使用する必要がある。

しかし以前は、その判断と実行手順がすべて人間の手作業に依存していた。変更のたびに手順書をゼロから（実態はテンプレートをコピーして）作成し、担当者2名が深夜2時から作業を行うという運用が常態化していた。

実際の手順書には以下が含まれる。

1. bastionサーバへのSSH接続手順
2. sqldefによるdry runと期待値の目視確認
3. pt-oscのdry runと、期待されるログ出力の目視確認
4. pt-osc実行とログ監視、メトリクス（レプリ遅延・CPU）の監視手順
5. 中止が必要な場合のプロセスkillとtrigger削除手順
6. （数日後）テーブルswapとtrigger削除の手順
7. old tableのpt-archiverによるレコード削除とDROP手順

これを変更のたびにコピーして書き直すこと自体がtoi lであり、「ALTER vs pt-oscのどちらを使うか」という判断が暗黙知として属人化していた。

### alterguardの設計

こうした手作業をコードに落とすツールとしてalterguardを開発した。中核となる設計思想は**「ALTER vs pt-oscの選択をコードで判断させる」**ことである。

**図1 alterguardの処理フロー**

```
PRマージ
    ↓
GitHub Actions 起動
    ↓
alterguardが対象テーブルの行数を取得
    ├─ 閾値以下 ──→ ALTER TABLE を実行
    └─ 閾値超え ──→ pt-online-schema-change を実行
                          ↓
                    完了後 Slack へ通知
```

閾値・チャンクサイズ・pt-oscのオプションは設定ファイルで管理し、dry runモードも備えている。Kubernetes Jobとして実行されるため、CI/CDとの統合も容易だ。

### 残る課題

すべてを自動化できたわけではない。pt-oscのメタデータロック問題は依然として存在し、トラフィックが高い一部のテーブルについては深夜実行を余儀なくされる場合がある。

正確に言えば「toil を削減した」であり「完全自動化した」ではない。それでも、変更のたびに手順書をコピーして2人で深夜に集まるという運用からは脱却できた。

---

## まとめ

3つの整備を通じて共通する考え方がある。

**散らばっているものを集める。** バリデーションが複数のhandlerに散在していた状態を`domain/service`に集約し、「ここを読めばわかる」場所を作った。仕様の居場所が定まることで、変更コストが下がる。

**次の変更が安全に乗せられる抽象を作る。** `UserCapability`インターフェースにより、新しいユーザー種別の追加が既存コードへの変更なしに行えるようになった。インターフェースによって「仕様」をコードとして表現することが、拡張性の鍵である。

**繰り返しの人間の判断をコードに落とす。** スキーマ変更の判断と手順が人間の暗黙知だった状態から、alterguardによる自動化でtoilを削減した。

成長期のプロダクトを整備する際、すべてを一度に直す必要はない。まず「居場所を作る」こと、そして「次の変更が乗せやすい抽象を置く」ことを繰り返すことで、プロダクトは徐々に変えやすい状態へと近づいていく。
