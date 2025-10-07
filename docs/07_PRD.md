
## 0. 目的・背景・ビジョン

### 0.1 目的

個人の入口（Gateway）として、プロフィール要点と3リンク（GitHub／LinkedIn／Email）を明快に提示。
問い合わせを最小構成で受け付け、RDS に 90 日保持。
週1で解析を閲覧可能（GA4：日次 UU／日次 PV／流入上位 5）。
運用は **月額 USD 10 上限・10 分以内のデプロイ／復帰** という軽量SLOを守る。

### 0.2 背景

就職・協業向けに、技術実証（ALB+EC2+RDS）と情報のワンストップ提示を同時に達成する小さなプロダクトを構築。

### 0.3 ビジョン

“軽く速く、誤解なく” を合言葉に、メンテと運用の手数が少ないゲートウェイを提供する。

### 0.4 対象読者

開発者／デザイナー／運用者／採用・協業の意思決定者。

---

## 1. スコープ

### 1.1 In-Scope（PoC 到達点）

* 画面：トップ（/）／リンク集（/links）／問い合わせ（/contact）／404（/404）／500（/500）／/health
* 独自ドメイン + HTTPS（ALB + ACM）、PC/モバイル対応
* 問い合わせ → RDS 保存（保持 90 日・日次削除ジョブ）
* 解析（GA4）：日次 UU・日次 PV・流入上位 5（週1で閲覧可）
* 運用：main→本番 ≤10 分、失敗→直前版復帰 ≤10 分、Budgets 80% 通知

### 1.2 Out-of-Scope（今回やらない）

認証／決済／CMS／多言語／商用 SLA／多段環境（stg/prod）
多AZ冗長化／APM／NAT Gateway 構成

---

## 2. 対象ユーザーと価値

採用担当／技術者／協業相手：一目で人物像と導線が掴める。
訪問者の仕事：プロフィール確認 → 外部プロフィールへ／問い合わせで連絡。

---

## 3. KPI（成功指標）

| 証跡                                   | カテゴリ | 指標                  | 基準                       |
| ------------------------------------ | ---- | ------------------- | ------------------------ |
| Budgets 通知ログ／`/docs/costs/yyyymm.md` | コスト  | 月額請求                | USD 10 以下                |
| 計測キャプチャ                              | 性能   | Lighthouse(Desktop) | 80+／CLS < 0.1／JS < 300KB |
| 計測ログ                                 | 運用   | デプロイ／復帰             | 各 ≤10 分                  |
| `/docs/analytics/`                   | 利用   | GA4                 | 日次 UU・PV・流入上位5を週1閲覧      |

---

## 4. アーキテクチャ

**構成：** ALB(443) → EC2(80) → RDS(PostgreSQL, Private, PublicAccess=false)
**リージョン：** ap-northeast-1（東京）
**OS/Runtime：** Amazon Linux 2023 / Node.js 20 LTS
**ネットワーク：**
VPC 10.10.0.0/16、Public（ALB/EC2）・Private（RDS）、IGWあり／NATなし
**管理：** SSM Session Manager（SSH 不可）
**CI/CD：** GitHub Actions → SSM Send-Command（artifact 展開→systemd再起動）

### 4.1 図版

<img width="1024" height="683" alt="Pasted image 20251005151352" src="https://github.com/user-attachments/assets/acf39b59-256e-4114-aa1c-d5a4596b7c3e" />


### 4.2 ASCII 概略

```
Internet
   |
 [ALB:443] -- ACM
   | (SG: ALB→APP:80)
 [EC2:80]  (Public Subnet)
   | 5432
 [RDS:PostgreSQL] (Private, Non-Public)
```

---

## 5. 機能要求（FR）

| ID   | 機能     | 仕様（要件）                                                                     | 受け入れ基準                                  |
| ---- | ------ | -------------------------------------------------------------------------- | --------------------------------------- |
| F-01 | トップ表示  | / にプロフィール要点＋主CTA（3リンクのいずれか）                                                | 200 応答／主要CTAから1クリックで遷移                  |
| F-02 | リンク集3点 | GitHub／LinkedIn／Email の3リンクのみ。PC横並び／モバイル縦。Tab順：GitHub→LinkedIn→Email       | 3リンクのみ可視／外部新規タブ／Emailはmailto件名付         |
| F-03 | 問い合わせ  | POST /api/contact：email/body≤1000/name≤50?/consent=true。Turnstile サーバー検証必須 | 201 {id, storedAt}／不正は400 + fieldErrors |
| F-04 | ヘルス    | GET /health → {"status":"ok"}                                              | ALB 経由で200                              |
| F-05 | 解析     | GA4 で日次 UU・日次 PV・流入上位 5 を週1で閲覧可                                            | `/docs/analytics/` に週次証跡                |
| F-06 | エラーページ | 404/500 の体裁                                                                | 不正URL→404／例外で500表示                      |

### 5.1 API 契約（F-03）

```
入力：JSON email, body, name?, consent(true)
返却：201 { id, storedAt } / 400 { fieldErrors }
Bot：Cloudflare Turnstile 検証（失敗時 400）
```

### 5.2 DB スキーマ（F-03）

```sql
CREATE TABLE inquiries(
  id BIGSERIAL PRIMARY KEY,
  email VARCHAR(254) NOT NULL,
  body TEXT NOT NULL CHECK (char_length(body) <= 1000),
  name VARCHAR(50),
  consent BOOLEAN NOT NULL,
  user_agent VARCHAR(255),
  referer VARCHAR(255),
  created_at TIMESTAMPTZ DEFAULT now()
);
CREATE INDEX idx_inquiries_created_at ON inquiries(created_at);
```

---

## 6. 非機能要求（NFR）

| 区分       | 要件                                                                                                         |
| -------- | ---------------------------------------------------------------------------------------------------------- |
| セキュリティ   | 常時 HTTPS／RDS PublicAccess=false／秘密情報は SSM Parameter Store／SSH 無効・SSM のみ／Turnstile 必須／CSP, HSTS, nosniff 有効 |
| 可用性      | 単一 EC2（ALB Target Healthy=1 維持）                                                                            |
| 性能       | Lighthouse(Desktop) 80+、CLS < 0.1、JS < 300KB                                                               |
| 運用 SLO   | main→本番 ≤10 分／失敗→直前版復帰 ≤10 分                                                                               |
| 観測性      | CloudWatch Logs 30 日保持                                                                                     |
| コスト      | 月上限 USD 10／80% 通知                                                                                          |
| 法令/規約    | プライバシーポリシー掲出・同意取得                                                                                          |
| アクセシビリティ | Tab到達／alt必須／コントラスト比4.5:1以上                                                                                 |

---

## 7. 機能構成（v1.2 準拠）

### 7.1 機能ツリー（Must/Should）

```
[公開閲覧](M)：トップ／リンク集／アクセシビリティ／/health
[問い合わせ](M)：入力→検証→Turnstile→保存
[エラー](M)：404／500
[解析(GA4)](S)：日次UU/PV/流入上位5
[運用](M)：デプロイ自動反映・ロールバック・削除ジョブ
[ドキュメント](S)：/docs 集約
```

### 7.2 機能テーブル（抜粋）

| 親機能   | 子機能      | 説明            | 優先度 | 依存             |
| ----- | -------- | ------------- | --- | -------------- |
| 公開閲覧  | トップ表示    | ルート表示と概要・主CTA | M   | –              |
| 問い合わせ | 保存       | RDS 1行保存      | M   | RDS            |
| 運用    | デプロイ自動反映 | GA→SSM        | M   | GitHub Actions |

---

## 8. ユースケース（Station04 完成版）

### 8.1 アクター

訪問者／運用者／スケジューラ
外部：Email クライアント／GitHub・LinkedIn／Turnstile／GA4／Budgets／ALB

### 8.2 主要ユースケース

U1: トップ閲覧／U2: リンク集利用／U3: 問い合わせ送信／U4: エラーページ確認
O1: デプロイ／O2: ロールバック／O3: 解析確認／O4: コスト通知／O5: ドキュメント更新
S1: 問い合わせ削除

### 8.3 PlantUML

```plantuml
@startuml
...（略）...
@enduml
```

---

## 9. 画面設計（概要）

![Uploading Pasted image 20251005151835.png…]()

![Uploading Pasted image 20251005151849.png…]()

### 9.1 要約

* **Top**：見出し／説明文／CTA3／フッター
* **Links**：3リンクカード（PC横／SP縦）
* **Contact**：Name?／Email／Body／consent／Turnstile／送信
* **404/500**：簡潔メッセージ＋戻る導線
* **/health**：`{"status":"ok"}`（UI外）

### 9.2 デザインルール

| 項目    | 内容                          |
| ----- | --------------------------- |
| タイポ   | 見出し600+、本文1.6行間             |
| 配色    | 白地＋黒文字＋青アクセント               |
| 余白    | 上下80px／左右32px以上             |
| 動き    | opacity / transform（≤300ms） |
| レイアウト | 1カラム中心（レスポンシブ）              |
| 共通UI  | フッター・©表記・/docs/リンク          |

### 9.3 画面構成図（ASCII）

```
                     ┌───────────────┐
                     │ トップページ（/） │
                     └─────┬───────┘
                           │
         ┌──────────────────┴──────────────────┐
         │                                     │
┌───────────────┐                ┌────────────────┐
│ リンク集（/links） │                │ 問い合わせ（/contact） │
└───────────────┘                └────────────────┘
                           │
                    ┌──────┴───────────────┐
                    │ エラーページ（/404・/500） │
                    └────────────────────────┘
```

---

## 10. 画面遷移（Station06 完成版）

### 10.1 ルール表

| #  | イベント              | 遷移元      | 条件                   | 遷移先      | 備考              |
| -- | ----------------- | -------- | -------------------- | -------- | --------------- |
| R1 | 「Links」クリック       | トップ      | —                    | /links   | 主要導線            |
| R2 | GitHub/LinkedIn   | トップ/リンク集 | —                    | 外部タブ     | target="_blank" |
| R3 | Emailクリック         | トップ/リンク集 | メールクライアントあり          | mailto起動 | 件名付き            |
| R4 | 「Contact」クリック     | トップ      | —                    | /contact | —               |
| R5 | POST /api/contact | 問い合わせ    | 入力OK & Turnstile OK  | 同一画面     | Thanks表示        |
| R6 | POST /api/contact | 問い合わせ    | 入力NG or Turnstile NG | 同一画面     | fieldErrors表示   |
| R7 | 不正URL             | 任意       | 存在しないパス              | /404     | —               |

---

## 11. 運用要件 / SOP

* **CI/CD**：GitHub Actions → SSM Send-Command で `/opt/gateway/deploy.sh` 実行
* **ロールバック**：直前版へ symlink 切替
* **片付け順**：ALB → EC2 → RDS → ACM → Route53

### 11.1 IAM

* EC2：`AmazonSSMManagedInstanceCore`＋`Parameter Store` 読取
* GitHub OIDC：`ssm:SendCommand`（タグ限定）＋`s3:GetObject`

### 11.2 Parameter Store

| Key                            | 用途     |
| ------------------------------ | ------ |
| /gateway/prod/DB_*             | RDS 接続 |
| /gateway/prod/TURNSTILE_SECRET | Bot検証  |
| /gateway/prod/GA4_ID           | 解析     |

---

## 12. セキュリティ実装要件（抜粋）

* CSP 最小許可（自己ドメイン + GA4 + Turnstile）
* HSTS 有効化、nosniff 付与
* SSM 管理（SSH 禁止）
* RDS は EC2 SG のみ許可
* Turnstile 検証失敗時は保存前に400応答

---

## 13. 自動ジョブ

* **日次 04:00 JST**
  `DELETE FROM inquiries WHERE created_at < now() - interval '90 days';`
* **週次 月曜 09:00 JST**
  GA4 確認 → `/docs/analytics/` 保存

---

## 14. 受け入れ試験（PowerShell）

| 観点         | 手順                                                 | 合格  |
| ---------- | -------------------------------------------------- | --- |
| トップ200     | `(Invoke-WebRequest https://<domain>/).StatusCode` | 200 |
| CTA導線      | 3リンク確認／Tab順確認                                      | OK  |
| 問い合わせ正常    | Invoke-RestMethod POST                             | 201 |
| 問い合わせ異常    | consent=false 等                                    | 400 |
| ヘルス        | `/health` = {"status":"ok"}                        | OK  |
| Lighthouse | Desktop 80+                                        | OK  |
| デプロイSLO    | main→本番 ≤10分                                       | OK  |
| 復帰SLO      | 自動ロールバック ≤10分                                      | OK  |

---

## 15. トレーサビリティ

| 機能ID | 画面      | ユースケース | API/DB            | 備考          |
| ---- | ------- | ------ | ----------------- | ----------- |
| F-01 | Top     | U1     | —                 | —           |
| F-03 | Contact | U3     | POST /api/contact | Turnstile検証 |
| F-04 | —       | HX     | GET /health       | —           |

---

## 16. マイルストーン

| ID | 内容                          |
| -- | --------------------------- |
| M1 | ドメイン + TLS 設定／/health 外部可視化 |
| M2 | 問い合わせ→RDS 保存／削除ロジック         |
| M3 | CI/CD + ロールバック実証            |
| M4 | GA4・Budgets・SOP 集約完了        |

---

### 付録A：用語

* **CTA**：Call To Action
* **UU**：Unique Users
* **PV**：Page Views
* **SLO**：Service Level Objective

---

こ
