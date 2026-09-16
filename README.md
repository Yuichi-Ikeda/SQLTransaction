# Azure SQL Database の瞬断（再構成）と「コミット済みだが応答が来ない」事象への対処

> **免責事項**
>
> 本資料は AI が公開情報（Microsoft Learn 等）をもとに調査・考察した内容であり、マイクロソフトの公式見解・製品サポートの回答ではありません。記載内容の正確性・完全性・最新性について、いかなる保証も行いません。
>
> 掲載しているコードおよび構成例はあくまで参考実装であり、そのままでの本番利用を想定したものではありません。本資料の内容を実システムへ適用する場合は、**必ずお客様ご自身で十分な検証を実施のうえ、自己の責任においてご判断ください。** 本資料の利用により生じたいかなる損害についても、作成者は一切の責任を負いません。
>
> 記載内容は作成時点の情報に基づいています。Azure のサービス仕様は随時更新されるため、実装前に必ず最新の[公式ドキュメント](#10-参考資料)をご確認ください。

## 目次

1. [相談内容](#1-相談内容)
2. [結論サマリー](#2-結論サマリー)
3. [事象のメカニズム](#3-事象のメカニズム)
4. [発生頻度の知見](#4-発生頻度の知見)
5. [検知手段](#5-検知手段)
6. [推奨アーキテクチャ](#6-推奨アーキテクチャ)
7. [実装例（.NET Framework 4.8）](#7-実装例net-framework-48)
8. [実装・運用上の注意](#8-実装運用上の注意)
9. [ヒアリングで確認したい事項](#9-ヒアリングで確認したい事項)
10. [参考資料](#10-参考資料)

---

## 1. 課題

Azure SQL Database または Managed Instance ではメンテナンス時に瞬断が発生する前提で、DB サーバの呼び出し側（App Service 側）で DB サーバ瞬断時にリトライを行う仕組みの導入を検討している。しかし、DB サーバのコミット後〜App Service への応答というわずかなタイミングで瞬断した場合、**「DB がコミットされているにもかかわらず、AP サーバに応答が来ない」**　という現象が発生することを懸念している。

---

## 2. 結論サマリー

| 論点 | 結論 |
|---|---|
| 事象は起こり得るか | **起こり得る。** 「コミット完了後、成功応答がクライアントに届く前に接続が切れる」ケースは、クライアント/サーバ型 DB 共通の結果不確定問題で、プラットフォーム側で完全に防ぐ手段はない。**通常の HA 再構成ではコミット済みデータの RPO = 0** だが、地域災害やデータ損失を伴う地理フェールオーバーまでの無条件保証ではない。 |
| AP 側に何が返るか | プロセスが稼働し適切なタイムアウトが設定されていれば、切断エラーやタイムアウト（例: SqlException の -2）などとして検出する。ただし **Commit 中の通信例外だけでは「ロールバック済み」と「コミット済み」を区別できない**。タイムアウトの適用範囲は API ごとに異なる（3-3 参照）。 |
| Azure Monitor で検知できるか | Resource Health / 診断ログ / メトリックで障害や再構成の手掛かりを得てアラートを設定できるが、**すべての瞬断を網羅するものではない**。個々のトランザクションの成功応答が失われたかどうかは、これらだけでは判断できない。 |
| AP 側で検知できるか | **結果不確定を検知し、DB の記録から成功を確認する設計が可能。** EF6 `CommitFailureHandler`、EF Core `ExecuteInTransaction` が公式パターン。ただし、照会不能やジャーナル行の未観測を「ロールバック済み」と断定せず、**確定できない場合は不確定状態を維持**する。 |
| 発生頻度 | 計画メンテナンスは月 2 回以上あり得、平均 1.7 回の再構成/イベント、通常 30 秒以内（平均 8 秒）という公式値はある。一方、**「コミット直後の瞬断」そのものの公表発生率は確認できていない**。4-2 は仮定を置いた計算例であり、実環境の頻度予測や上限ではない。 |

---

## 3. 事象のメカニズム

### 3-1. 計画メンテナンス時に何が起きるか

出典: [Azure SQL Database および Azure SQL Managed Instance での Azure メンテナンス イベントの計画](https://learn.microsoft.com/azure/azure-sql/database/planned-maintenance?view=azuresql)

- Business Critical / Premium ではセカンダリ レプリカがプライマリに昇格、General Purpose では別の計算ノードへデータベース エンジン プロセスが移動する（GP はコールド キャッシュで一時的に性能低下あり）。
- 「計画メンテナンス イベント 1 回当たり、平均 1.7 回の再構成が発生する。再構成は通常 30 秒以内に完了し、平均は 8 秒である。」
- 既存接続は再接続が必要。再構成中の新規接続は **40613 (Database Unavailable)**。実行中のクエリは中断され、再実行が必要。
- Managed Instance も同様。[Azure SQL Managed Instance でユーザーが開始した手動フェールオーバー](https://learn.microsoft.com/azure/azure-sql/managed-instance/user-initiated-failover?view=azuresql)では「クライアント接続が失敗するのは短時間のみで、通常は 1 分未満」としている。

### 3-2. 通常の HA 再構成ではコミット済みデータは失われない

出典: [Azure SQL Database の高可用性](https://learn.microsoft.com/azure/azure-sql/database/high-availability-sla-local-zone-redundancy?view=azuresql)

- ローカル冗長で保護されるノード障害や通常のメンテナンス再構成では RPO はゼロ。ゾーン障害への耐性にはゾーン冗長の構成が必要であり、地域災害やデータ損失を許容する地理フェールオーバーは別の DR 設計として扱う。
- BC / Premium: 「**各トランザクションをコミットする前に**、十分な数のセカンダリ レプリカへデータが永続化されることを保証する」
- GP: ログ / データは Azure Blob Storage 上にあり、「ログ ファイル内のすべてのレコードは、データベース エンジン プロセスがクラッシュしても保持される」

→ 本資料で扱う通常の HA 再構成と完全永続化を前提にすれば、懸念は「データ消失」ではなく「**AP 側がコミット成否を知れない**」問題である。遅延持続性を有効化した場合などは、この前提を別途確認する。

### 3-3. AP 側に返る例外

出典: [Azure SQL Database の一般的な接続エラーのトラブルシューティング](https://learn.microsoft.com/azure/azure-sql/database/troubleshoot-common-errors-issues?view=azuresql)

- 再構成時には 40613 / 40197（内包コード 40020, 40143, 40166, 40540）などが返ることがある。40501 はサービス過負荷などでも発生する一時エラーであり、再構成が起きた証拠とは限らない。
- ネットワーク切断では TCP 系のエラー番号や 0 などが返ることがあり、番号・例外型はドライバーと失敗経路に依存する。
- `SqlCommand.Execute*` の応答待ちは `CommandTimeout` に従い、クライアント側タイムアウトは一般に **-2** として通知される。
- **`SqlTransaction.Commit()` は別の API**。確認した Microsoft.Data.SqlClient 6.1.0 の .NET Framework 向け実装は `ConnectTimeout` を参照しており、直前のコマンドの `CommandTimeout` では制御できない（[ExecuteTransaction2005 の実装](https://github.com/dotnet/SqlClient/blob/v6.1.0/src/Microsoft.Data.SqlClient/netfx/src/Microsoft/Data/SqlClient/SqlInternalConnectionTds.cs)）。実装依存のため、採用バージョンで確認する。タイムアウト値 0 による無期限待機にも注意する。
- これらの通信エラーやタイムアウトだけから「**コミットが通ったかどうか**」を判断しない。

### 3-4. なぜ「盲目的なリトライ」が危険か

出典: [EF Core の接続の回復性](https://learn.microsoft.com/ef/core/miscellaneous/connection-resiliency)（「トランザクションのコミット失敗と冪等性の問題」節）

> 「一般に、接続障害が発生すると、現在のトランザクションはロールバックされる。ただし、トランザクションのコミット中に接続が切断された場合、そのトランザクションの結果は不明となる。」

そのまま再実行すると「新しいデータベース状態と整合しない場合は例外が発生する可能性があり、たとえば自動生成キー値を使用して新しい行を挿入する場合には、**データ破損**につながる可能性がある」。

→ 「瞬断時にリトライする仕組み」を**冪等性の担保なしに入れると、二重登録・二重加算のリスクを新たに生む**。

同じ整理は EF6 の公式ドキュメント（[トランザクション コミット エラーの処理](https://learn.microsoft.com/ef/ef6/fundamentals/connection-resiliency/commit-failures)）にもある:

> 「トランザクションのコミット中に例外が発生する原因は 2 つある。サーバー上でトランザクションのコミットに失敗した場合と、サーバー上ではコミットに成功したものの、接続の問題によって成功通知がクライアントへ届かなかった場合である。前者ではアプリケーションまたはユーザーが操作を再試行できるが、後者では再試行を避ける必要があり、アプリケーションが自動的に回復できる。」

---

## 4. 発生頻度の知見

### 4-1. 公式に公表されている数値

| 項目 | 値 | 出典 |
|---|---|---|
| メンテナンス更新の頻度 | 「**1 か月に 2 回以上の更新**が行われる場合がある。既定のメンテナンス ウィンドウでは、より頻繁にメンテナンスが行われる可能性がある。」 | [Azure SQL Database のメンテナンス期間に関する FAQ](https://learn.microsoft.com/azure/azure-sql/database/maintenance-window-faq?view=azuresql) |
| 再構成が必要な場合の回数 | 平均 1.7 回/イベント（FAQ では「通常 1～2 回」） | planned-maintenance / FAQ |
| 再構成の所要時間 | 概ね 30 秒以内、平均 8 秒（MI は「通常 1 分未満」） | planned-maintenance / MI failover |
| 計画外の再構成 | ハードウェア障害、クラスタ負荷分散、SLO 変更などでメンテナンス ウィンドウ外にも発生 | [Azure SQL Database のメンテナンス期間](https://learn.microsoft.com/azure/azure-sql/database/maintenance-window?view=azuresql) |

「コミット直後の瞬断」そのものの発生率は Microsoft から公表されていない。

### 4-2. 概算モデル（仮定に基づく計算例）

対象事象は、DB のコミット完了から成功応答の受信までの間に通信が途切れることで起こり得る。この窓幅にはネットワーク遅延や処理待ちも関係し、**数ミリ秒と保証されるものではない**。

コミットの到着率が一定で、切断タイミングとの相関を無視でき、対象接続が各再構成で切断されると仮定した簡略モデル:

$$
E[N] \approx \lambda \times w \times r
$$

ここで、λ は再構成が起きる時間帯のコミット頻度（tx/秒）、$w$ は窓幅（秒）、$r$ は年間の対象切断回数である。

下表では **月 2 イベント × 12 か月 × 1.7 回 ≒ 年 41 回**、窓幅を 1 ms または 5 ms と仮定する。「月 2 回以上あり得る」は月 2 回が平均・上限という意味ではなく、すべての更新で再構成が必要とも限らない。**年 41 回は実測値でも上限寄りの推定でもない**。

| コミット頻度 | 窓幅 1 ms | 窓幅 5 ms |
|---|---|---|
| 1 tx/秒 | 約 0.04 件/年 | 約 0.2 件/年 |
| 10 tx/秒 | 約 0.4 件/年 | 約 2 件/年 |
| 100 tx/秒 | 約 4 件/年 | 約 20 件/年 |

**解釈:** この計算例だけから「年 1 回程度」「低頻度なので安全」と結論付けることはできない。実環境では計画外再構成、ネットワーク障害、時間帯別負荷も考慮する。頻度にかかわらず結果不確定に対応できる設計とし、運用では **不確定検知件数・成功確認件数・未解決件数を分けて**記録する（7-4 参照）。

---

## 5. 検知手段

### 5-1. Azure Monitor / プラットフォーム側 —「再構成が起きた事実と時刻」の把握

| 手段 | 分かること | 制約 |
|---|---|---|
| **Service Health 事前通知**（[SQL Database](https://learn.microsoft.com/azure/azure-sql/database/advance-notifications?view=azuresql) / [Managed Instance](https://learn.microsoft.com/azure/azure-sql/managed-instance/advance-notifications?view=azuresql)） | 計画メンテの事前、開始前後、終了前後に通知（メール / SMS / プッシュ / 音声） | SQL DB / MI 両対応。SQL DB では既定以外のメンテナンス ウィンドウを選択して通知を構成する。**厳密な 24 時間前の通知保証ではない**。新規作成・スケール・ウィンドウ変更直後などに事前通知が届かない例外がある（[FAQ](https://learn.microsoft.com/azure/azure-sql/database/maintenance-window-faq?view=azuresql)）。製品別の対応条件を確認する |
| **Resource Health**（[SQL Database の判定仕様](https://learn.microsoft.com/azure/azure-sql/database/resource-health-to-troubleshoot-connectivity?view=azuresql)） | Degraded / Unavailable と最大 30 日の履歴。判定できたダウンタイム理由（Planned maintenance / Reconfiguration）は通常約 45 分以内に公開。Resource Health アラート可 | SQL DB の状態更新は 1〜2 分間隔、ダウンタイム履歴は 2 分粒度。**ログイン失敗数などのしきい値未満の瞬断は検知されないことがある**。アラート時刻も数分遅れる。MI には製品別の判定仕様がある |
| **Azure Monitor メトリック** `connection_failed`（システム エラー） | 1 分粒度で接続失敗数 | SQL DB 向け（MI のメトリック名は未確認） |
| **診断ログ「Errors」カテゴリ** → Log Analytics（[診断テレメトリ](https://learn.microsoft.com/azure/azure-sql/database/metrics-diagnostic-telemetry-logging-streaming-export-configure?view=azuresql)） | 記録された SQL エラーを `error_number_d` などで調査できる | **SQL DB / MI のユーザー DB に対応**。DB ごとに診断設定の有効化が必要。すべてのクライアント側切断やタイムアウトの記録を保証しない |
| **`sys.event_log`**（master） | 接続成功 / 失敗の 5 分集計。保持は**最大** 30 日（DB 数と一意イベント数によってはそれ未満） | **SQL DB のみ**。再構成というイベント種別は無い。**データ反映は通常 1 時間以内、最大 24 時間**かかるため直後の時刻突合には使えない。master への負荷も高く補助的 |

**限界:** これらのテレメトリだけでは、個々のトランザクションの成功応答喪失やコミット成否は判定できない。AP 側ログの**裏取り（時刻の突合）**に使う位置づけであり、**記録がないことは「瞬断がなかった」証明にならない**。

### 5-2. AP サーバ側 —「どのトランザクションが不確定か」の特定（本命）

1. **例外の発生フェーズで 3 分類する**
    - 接続オープン時の失敗 → 一時エラーなら再試行
    - トランザクション開始時・文の実行中（Commit 前）の失敗 → ロールバックと接続破棄を行い、一時エラーなら **同じ冪等キーでトランザクション全体**を再実行。文のタイムアウトはセッション消失を意味せず、自動ロールバック済みとは限らない
    - **`Commit()` 中の失敗 → 「結果不確定」** → 古い接続を破棄して状態を検証。確定できなければ不確定として通知し、新しいキーで再実行しない
2. **不確定コミットを専用イベントとして記録**（業務キー / リクエスト ID 付きの構造化ログ → Application Insights / Log Analytics でアラート）
3. **ドライバのリトライ機能の限界を理解する**
    - SqlClient の Configurable Retry Logic: 「組み込みプロバイダーは、**接続にアクティブなトランザクションがある場合、コマンドを再試行しない**。…トランザクション全体をロールバックして再試行する。」（[SqlClient で再試行ロジックを設定](https://learn.microsoft.com/sql/connect/ado-net/configurable-retry-logic-sqlclient-introduction?view=sql-server-ver17)）
    - JDBC の Connection resiliency: 「次の場合、ドライバーは切断されたアイドル接続を復元できない。…**開いているトランザクションがある場合**。」（[JDBC ドライバーの接続回復性](https://learn.microsoft.com/sql/connect/jdbc/connection-resiliency?view=sql-server-ver17)）
   - → **ドライバ任せでは in-doubt は解決しない**。アプリ層の設計が必要。

この分類は **AP 側が管理する明示的なトランザクション**が前提。オートコミットの更新、内部で COMMIT するストアド プロシージャ、外部システムへの副作用、`TransactionScope` は同じ条件では扱えない。

### 5-3. DB 側で「実際にコミットされたか」を確定する方法

| パターン | 内容 | 適用 |
|---|---|---|
| **トランザクション追跡（ジャーナル）テーブル** | 業務更新と同じトランザクションの先頭で一意 ID の行を INSERT。Commit 失敗時にコミット済みの行を読めれば、その要求の成功を確認できる。**行が見えないだけではロールバック済みと断定しない** | **汎用的**。EF6 `CommitFailureHandler`（`__Transactions` 表）/ EF Core の「トランザクションを手動で追跡する」解法が同系統の考え方。独自実装では分離レベル・保持期間・キーの一意性を設計する |
| **状態検証** | EF Core `ExecuteInTransaction(operation, verifySucceeded)` の `verifySucceeded`（「トランザクションのコミット中に例外がスローされた場合でも、操作が成功したかどうかを検査する」）で業務データを再読取り | 業務キーで成否が判定できる場合 |
| **クライアント生成キー** | GUID 等を使い、二重実行は一意制約違反として「予測可能に失敗」させる（IDENTITY 依存を避ける） | INSERT 中心の処理 |
| （参考）SQL 監査 BATCH_COMPLETED | サーバ側の実行証跡は残るが負荷が大きい | 事後調査用途に限定 |

検証先は **更新と同じ DB の書き込み先プライマリ**とし、READ COMMITTED 以上の読み取りを使う。`NOLOCK` / READ UNCOMMITTED による未コミット行の読み取りや、読み取り専用レプリカの反映遅延を成功判定に使わない。

Azure SQL Database の既定の **RCSI** はステートメント開始時点のコミット済みデータを読むため、元のコミットがまだ進行中ならジャーナル行が見えない場合がある（[分離レベルの仕様](https://learn.microsoft.com/sql/t-sql/statements/set-transaction-isolation-level-transact-sql?view=sql-server-ver17)）。**RCSI 有効時は `WITH (READCOMMITTED)` ヒントを付けても行バージョン管理のまま**なので、この未観測は回避できない。元トランザクションの決着まで待って確定的な答え（コミット済み / ロールバック済み）を得たい場合は `WITH (READCOMMITTEDLOCK)` でロック待ちさせる選択肢もあるが、ブロッキングと待ち時間が発生する。7-3 の参考実装はヒントで挙動を変えず、照会を繰り返し、確認できなければ `InDoubt` とする保守的な方式を採用する。**実際にはロールバック済みでも不確定として残る場合がある**。

「同じキーの INSERT を再試行して一意制約で二重適用を防ぐこと」と「元の試行のロールバックを証明すること」は別である。再送時はキーと要求内容を維持し、ジャーナルの INSERT に成功した場合だけ業務処理を実行する。

---

## 6. 推奨アーキテクチャ

1. **リトライは「冪等前提」で設計**: 1 業務操作に一意のリクエスト ID（またはクライアント生成 GUID）を付与し、業務更新と同一トランザクション内でジャーナルに記録する。再送は同じキー・同じ要求内容に限定する。**増分更新でもこの方式で二重適用を防げる**。絶対値更新への置換だけでは並行更新の取りこぼしを防げず、`rowversion` による楽観ロックなどの競合制御は別途必要。
2. **Commit 例外は専用パスへ**: `Commit()` の例外を「in-doubt」として記録 → 旧接続を破棄 → ジャーナルで成功を確認。未観測なら再確認し、なお不明なら `InDoubtCommitException` で通知する。UTC タイムスタンプを残してプラットフォームの履歴と突合する（[一時的な接続エラーのトラブルシューティング](https://learn.microsoft.com/azure/azure-sql/database/troubleshoot-common-connectivity-issues?view=azuresql)）。
3. **リトライ間隔と時間予算**: アプリ層は初回 5 秒、指数バックオフで各待機を最大 60 秒とし、試行回数にも上限を設ける（同ドキュメントの推奨）。**待機時間の上限は処理全体の所要時間の上限ではない**。ドライバー内部の再試行、接続・コマンド・コミットの待ち時間を含めて業務の時間予算を設計する。
4. **運用面のリスク低減**
    - 対応するサービス レベル・リージョンで **既定以外のメンテナンス ウィンドウ**（平日または週末の現地時間 22:00〜6:00）を選択し、事前通知を有効化して作業計画に反映。厳密な通知時刻や全イベントの事前通知は保証されず、緊急のセキュリティ更新などでウィンドウが例外的に上書きされる場合もある。
   - 接続ポリシーは **Redirect** を推奨（ゲートウェイ メンテナンスの影響を回避）。
    - **手動フェールオーバーで実地試験**: SQL DB は `Invoke-AzSqlDatabaseFailover`、MI は `Invoke-AzSqlInstanceFailover`（いずれも **15 分に 1 回**の制限）。プライマリがオフラインになる動作を試験できるが、Commit の応答喪失を毎回再現できるとは限らない。障害注入試験と組み合わせ、試験結果を自然発生頻度と混同しない。

---

## 7. 実装例（.NET Framework 4.8）

### 7-0. 使うライブラリ

| 選択肢 | 位置づけ | 備考 |
|---|---|---|
| **Microsoft.Data.SqlClient**（NuGet） | **推奨** | .NET Framework 4.6.2 以降に対応。2026-09-16 確認時のサポート版は 7.0（STS）と 6.1（LTS、2028-08-14 まで）。**本例は 6.1 の最新修正パッチを前提**とする（3-3 の Commit タイムアウト挙動は `v6.1.0` タグの公開実装で確認したもので、パッチ間の差分は採用時に確認する）。7.0 で Managed Identity 等の Entra 認証を使う場合は `Microsoft.Data.SqlClient.Extensions.Azure` も必要 |
| System.Data.SqlClient（Framework 同梱） | 既存資産がある場合 | `using` の変更に加え、`RetryLogicProvider` の設定と `OpenRetryProvider` の定義（`SqlRetryLogicBaseProvider` / `SqlConfigurableRetryFactory` / `SqlRetryLogicOption`）を除去・置換する。**2 行の削除だけでは移植できない**。接続文字列の Managed Identity 認証もそのまま使えないため、トークン取得と `AccessToken` 設定など別の接続方法が必要 |
| **EF6（6.1 以降）** | ORM 利用時 | `CommitFailureHandler` がこの問題専用の公式機能（7-5 参照） |

バージョンの根拠: [サポート ライフサイクル](https://learn.microsoft.com/sql/connect/ado-net/sqlclient-driver-support-lifecycle?view=sql-server-ver17)、[SqlClient 7.0 の Entra 認証移行](https://learn.microsoft.com/sql/connect/ado-net/sql/azure-active-directory-authentication?view=sql-server-ver17#migrate-to-microsoftdatasqlclient-70)。

### 7-1. 設計の骨格（3 フェーズで例外を分類）

```text
[A] Open()            失敗 → 一時エラーなら再試行（安全）
[B] BEGIN TRAN        開始時の一時エラーも再試行対象
    INSERT ジャーナル行 (冪等キー)   ← この INSERT の重複だけを成功済みとして扱う
    業務 UPDATE/INSERT ...          失敗 → Rollback・破棄 → 一時エラーなら同じキーで全体を再試行
                                   業務テーブルの一意制約違反は成功扱いにしない
[C] Commit()          失敗 → 不確定 → 旧接続を破棄 → 新しい接続でジャーナルを確認
    行を確認できた                 → 成功を確認
    行が見えない / 一時的な照会失敗 → 上限まで再確認（業務更新の再実行ではない）
    なお未確認 / 非一時的な照会失敗 → InDoubt を維持して通知
```

### 7-2. DB 側: ジャーナル表（冪等キー）

```sql
CREATE TABLE dbo.TxnJournal
(
    IdempotencyKey UNIQUEIDENTIFIER NOT NULL
        CONSTRAINT PK_TxnJournal PRIMARY KEY NONCLUSTERED,   -- 二重実行はここで一意制約違反になる
    Operation      NVARCHAR(100)    NOT NULL,
    CreatedAtUtc   DATETIME2(3)     NOT NULL
        CONSTRAINT DF_TxnJournal_CreatedAtUtc DEFAULT SYSUTCDATETIME()
);
-- 追記型にして INSERT と期限切れ削除を軽くする
CREATE CLUSTERED INDEX CX_TxnJournal_CreatedAtUtc ON dbo.TxnJournal (CreatedAtUtc);

-- 保持期間 (例: 30 日) を過ぎた行を定期削除 (Elastic Jobs / Automation / アプリの定期処理)
DELETE TOP (5000) FROM dbo.TxnJournal
WHERE CreatedAtUtc < DATEADD(DAY, -30, SYSUTCDATETIME());
```

※ EF Core の例では成功時に行を削除するが、保持期間方式なら **保持中のキーについて**上流の遅延再送も検知でき、コミット直後の削除用ラウンドトリップを省ける。**30 日は例であり、削除後の再送は重複排除できない**。最大処理時間、上流の再送期限、障害解消までの時間、運用者による遅延再送を含めて保持期間を決め、期限外の要求を拒否するか別の照合手段を用意する。削除ジョブは上記の小分け削除を繰り返す方式とし、未解決要求の照合に必要な記録を失わないよう運用する。

### 7-3. AP 側: ADO.NET 実装（C# 7.3 / .NET Framework 4.8）

**前提と制限:** 1 キーは 1 業務操作に対応し、再送時も要求内容を変えない。別の操作・別のテナントでキーを使い回さない。この前提を上流で保証できない場合は、要求ハッシュなども保存・照合して不一致を拒否する。本例のジャーナルには他の一意制約や副作用を持つトリガーを追加しない。`work` は渡された接続・トランザクションだけを使い、自分で Commit / Rollback / Close を行わない。`TransactionScope` との併用は対象外。

Commit 前の一時エラーは再試行するが、**Commit 例外後は成功確認だけを行い、行が見えなくても自動で業務処理を再実行しない**。未確認なら不確定として扱うため、真にロールバック済みの要求でも運用上の照合が必要になる場合がある。照合後の再送でも元のキーを維持する。

**ブロッキングに注意:** 同じキーの要求が並行して走っている場合、後続のジャーナル INSERT は一意インデックスで**ロック待ち**になり、重複キー違反 (2627) は先行トランザクションが決着してから返る。Rollback / Dispose に失敗して孤立したトランザクションがサーバ側に残ると、再試行はコマンドタイムアウトまでブロックする。

```csharp
// NuGet: Microsoft.Data.SqlClient 6.1 (LTS) の最新修正パッチを想定
using System;
using System.Collections.Generic;
using System.Data;
using System.Threading;
using Microsoft.Data.SqlClient;

namespace Sample.DataAccess
{
    public enum CommitOutcome
    {
        Committed,          // 正常にコミット完了
        AlreadyCommitted,   // 前回試行が実はコミット済みだった (ジャーナル行で判明)
        RetryableBeforeCommit,
        InDoubt             // 成功を確認できず不確定。TryOnce の内部結果で、Execute では InDoubtCommitException に変換される
    }

    /// <summary>コミット結果を確定できなかったことを表す例外 (手動確認が必要)</summary>
    public sealed class InDoubtCommitException : Exception
    {
        public Guid IdempotencyKey { get; }
        public InDoubtCommitException(Guid key, string message, Exception inner) : base(message, inner)
        {
            IdempotencyKey = key;
        }
    }

    public sealed class ResilientTransactionRunner
    {
        private readonly string _connectionString;
        // 構造化ログの出口 (Application Insights の TrackEvent など)
        private readonly Action<string, IDictionary<string, object>> _logEvent;

        // Azure SQL の再構成時に返る代表的な一時エラー番号 (Microsoft Learn の一覧より)
        // 40143 / 40540 は通常 40197 の内包コードとしてメッセージ側に現れる (Number は 40197) が、念のため保持する
        private static readonly HashSet<int> TransientSqlErrors = new HashSet<int>
        {
            4060, 40197, 40501, 40613, 40143, 40540, 49918, 49919, 49920,
            10928, 10929, 4221, 615,
            10053, 10054, 10060, 233, 64, 20, 0,   // TCP 層の切断 (Number が 0 になることがある)
            -2                                      // タイムアウト
        };

        // [A] Open() の自動再試行 (Microsoft.Data.SqlClient 3.0+)。System.Data.SqlClient の場合はこの機能を外す
        private static readonly SqlRetryLogicBaseProvider OpenRetryProvider =
            SqlConfigurableRetryFactory.CreateExponentialRetryProvider(new SqlRetryLogicOption
            {
                NumberOfTries   = 5,
                DeltaTime       = TimeSpan.FromSeconds(5),
                MaxTimeInterval = TimeSpan.FromSeconds(60)
            });

        public ResilientTransactionRunner(string connectionString,
                                          Action<string, IDictionary<string, object>> logEvent)
        {
            _connectionString = connectionString;
            _logEvent = logEvent ?? throw new ArgumentNullException(nameof(logEvent));
        }

        /// <summary>
        /// 1 業務リクエスト = 1 トランザクションを、冪等キーで保護しながら実行する。
        /// work の中では渡された接続・トランザクションだけを使い、DB 外の副作用 (指令送信など) は行わないこと。
        /// </summary>
        public CommitOutcome Execute(Guid idempotencyKey, string operationName,
                                     Action<SqlConnection, SqlTransaction> work, int maxAttempts = 5)
        {
            if (idempotencyKey == Guid.Empty) throw new ArgumentException("冪等キーが必要です。", nameof(idempotencyKey));
            if (string.IsNullOrWhiteSpace(operationName) || operationName.Length > 100)
                throw new ArgumentException("操作名は 1〜100 文字で指定してください。", nameof(operationName));
            if (work == null) throw new ArgumentNullException(nameof(work));
            if (maxAttempts < 1) throw new ArgumentOutOfRangeException(nameof(maxAttempts));

            var delay = TimeSpan.FromSeconds(5);   // 初回 5 秒 → 指数的に最大 60 秒 (Learn の推奨値)

            for (int attempt = 1; ; attempt++)
            {
                Exception lastError;
                var outcome = TryOnce(idempotencyKey, operationName, work, attempt, out lastError);

                switch (outcome)
                {
                    case CommitOutcome.Committed:
                    case CommitOutcome.AlreadyCommitted:
                        return outcome;

                    case CommitOutcome.InDoubt:
                        throw new InDoubtCommitException(idempotencyKey,
                            "コミット結果を確定できませんでした。IdempotencyKey=" + idempotencyKey + " を確認してください。",
                            lastError);

                    case CommitOutcome.RetryableBeforeCommit:
                        if (attempt >= maxAttempts)
                            throw new InvalidOperationException(
                                "一時エラーの再試行上限 (" + maxAttempts + " 回) に達しました。", lastError);
                        Thread.Sleep(delay);
                        delay = TimeSpan.FromSeconds(Math.Min(delay.TotalSeconds * 2, 60));
                        break;
                }
            }
        }

        private CommitOutcome TryOnce(Guid key, string operationName,
                                      Action<SqlConnection, SqlTransaction> work,
                                      int attempt, out Exception error)
        {
            error = null;
            SqlConnection conn = null;
            SqlTransaction tx = null;
            bool committing = false;

            try
            {
                conn = new SqlConnection(_connectionString) { RetryLogicProvider = OpenRetryProvider };
                conn.Open();                                        // [A]
                tx = conn.BeginTransaction();

                // [B-1] ジャーナル行 (冪等キー) を同じトランザクション内で INSERT
                using (var cmd = new SqlCommand(
                    "INSERT INTO dbo.TxnJournal (IdempotencyKey, Operation) VALUES (@key, @op);", conn, tx))
                {
                    cmd.Parameters.Add("@key", SqlDbType.UniqueIdentifier).Value = key;
                    cmd.Parameters.Add("@op",  SqlDbType.NVarChar, 100).Value   = operationName;
                    try
                    {
                        cmd.ExecuteNonQuery();
                    }
                    catch (SqlException ex) when (IsDuplicateKey(ex))
                    {
                        SafeRollback(tx);
                        Log("SqlRequestAlreadyCommitted", key, operationName, attempt, ex, "AlreadyCommitted");
                        return CommitOutcome.AlreadyCommitted;
                    }
                }

                // [B-2] 業務更新
                work(conn, tx);

                // [C] コミット。ここで投げられた例外は「成否不明」
                committing = true;
                tx.Commit();

                Log("SqlCommitSucceeded", key, operationName, attempt);
                return CommitOutcome.Committed;
            }
            catch (Exception ex) when (!committing)
            {
                SafeRollback(tx);
                if (IsTransient(ex))
                {
                    error = ex;
                    Log("SqlTransientBeforeCommit", key, operationName, attempt, ex);
                    return CommitOutcome.RetryableBeforeCommit;
                }
                throw;
            }
            catch (Exception ex) when (committing)
            {
                error = ex;
                Log("SqlCommitInDoubt", key, operationName, attempt, ex);
            }
            finally
            {
                SafeDispose(tx, key, operationName, attempt);
                SafeDispose(conn, key, operationName, attempt);
            }

            return VerifyCommit(key, operationName, attempt, error, out error);
        }

        /// <summary>ジャーナル行で成功を確認する。行が見えない場合も未確定として確認を続ける。</summary>
        private CommitOutcome VerifyCommit(Guid key, string operationName, int attempt,
                                           Exception commitError, out Exception error)
        {
            var delay = TimeSpan.FromSeconds(5);
            Exception last = commitError;
            error = commitError;

            for (int verificationAttempt = 1; verificationAttempt <= 6; verificationAttempt++)
            {
                try
                {
                    using (var conn = new SqlConnection(_connectionString) { RetryLogicProvider = OpenRetryProvider })
                    // RCSI 有効時このヒントは行バージョン管理のまま。確定待ちが必要なら READCOMMITTEDLOCK を検討する
                    using (var cmd = new SqlCommand(
                        "SELECT COUNT(*) FROM dbo.TxnJournal WITH (READCOMMITTED) WHERE IdempotencyKey = @key;", conn))
                    {
                        cmd.Parameters.Add("@key", SqlDbType.UniqueIdentifier).Value = key;
                        conn.Open();
                        bool exists = (int)cmd.ExecuteScalar() > 0;

                        if (exists)
                        {
                            Log("SqlCommitInDoubtResolved", key, operationName, attempt, commitError, "Committed");
                            return CommitOutcome.Committed;
                        }
                    }
                }
                catch (Exception ex)
                {
                    last = ex;
                    if (!IsTransient(ex)) break;
                }

                if (verificationAttempt < 6)
                {
                    Thread.Sleep(delay);
                    delay = TimeSpan.FromSeconds(Math.Min(delay.TotalSeconds * 2, 60));
                }
            }

            error = last == commitError ? commitError :
                new AggregateException("コミットと結果確認でエラーが発生しました。", commitError, last);
            Log("SqlCommitInDoubtUnresolved", key, operationName, attempt, last);
            return CommitOutcome.InDoubt;
        }

        private static bool IsDuplicateKey(SqlException ex) => ex.Number == 2627 || ex.Number == 2601;

        private static bool IsTransient(Exception ex)
        {
            if (ex is TimeoutException) return true;
            var aggregate = ex as AggregateException;
            if (aggregate != null)
            {
                var errors = aggregate.Flatten().InnerExceptions;
                if (errors.Count == 0) return false;
                foreach (var inner in errors)
                    if (!IsTransient(inner)) return false;
                return true;
            }

            var sqlEx = ex as SqlException ?? ex.InnerException as SqlException;
            if (sqlEx == null) return false;
            foreach (SqlError sqlError in sqlEx.Errors)
                if (TransientSqlErrors.Contains(sqlError.Number)) return true;
            return false;
        }

        private static void SafeRollback(SqlTransaction tx)
        {
            try { if (tx != null && tx.Connection != null) tx.Rollback(); }
            catch (Exception) { /* Rollback 不能でもセッション終了時にサーバ側でロールバックされる */ }
        }

        private void SafeDispose(IDisposable resource, Guid key, string operation, int attempt)
        {
            try { resource?.Dispose(); }
            catch (Exception ex) { Log("SqlCleanupFailed", key, operation, attempt, ex); }
        }

        private void Log(string eventName, Guid key, string operation, int attempt,
                         Exception ex = null, string resolution = null)
        {
            var props = new Dictionary<string, object>
            {
                ["IdempotencyKey"] = key,
                ["Operation"]      = operation,
                ["Attempt"]        = attempt,
                ["TimestampUtc"]   = DateTime.UtcNow.ToString("O"),
                ["SqlErrorNumber"] = (ex as SqlException)?.Number,
                ["ExceptionType"]  = ex?.GetType().Name,
                ["Message"]        = ex?.Message,
                ["Resolution"]     = resolution
            };
            try
            {
                _logEvent(eventName, props);
            }
            catch (Exception logError)
            {
                try
                {
                    System.Diagnostics.Trace.TraceError("SQL transaction logging failed: {0}", logError);
                }
                catch (Exception) { }
            }
        }
    }
}
```

`RetryableBeforeCommit` は「この試行では Commit を呼んでいない一時エラー」であり、サーバー側ロールバックの完了を観測したという意味ではない。Rollback / Dispose は試みるが失敗し得るため、必ず新しい接続で、同じキーの INSERT を先頭に置いて再試行する。

`VerifyCommit` は最大 6 回照会し、試行間のアプリ層待機は 5 + 10 + 20 + 40 + 60 = **135 秒**。最終試行後には待機しない。これに接続・照会・ドライバー内部の再試行時間が加わるため、**処理全体が 135 秒以内に終わる保証ではない**。そもそも `VerifyCommit` に入る前に、3-3 のとおり `Commit()` の応答待ちで最大 `Connect Timeout` 分を消費する。この参考実装には総経過時間による打ち切りやキャンセルは含まれない。本番では Web 要求の時間予算と整合する上限を設け、超過時は不確定状態を維持してバックグラウンド確認へ移すなどの設計が必要。

#### 呼び出し例

```csharp
// 上流から渡ってきたリクエスト ID を冪等キーにする (無ければ生成し、再試行でも同じ値を使い続ける)
Guid key = request.RequestId;

var outcome = runner.Execute(key, "WorkPlan.UpdateStatus", (conn, tx) =>
{
    using (var cmd = new SqlCommand(
        "UPDATE dbo.WorkPlan SET Status = @status, UpdatedAtUtc = SYSUTCDATETIME() " +
        "WHERE PlanId = @id AND RowVer = @rowver;", conn, tx))        // rowversion による楽観ロック
    {
        cmd.Parameters.Add("@status", SqlDbType.Int).Value       = (int)request.NewStatus;
        cmd.Parameters.Add("@id",     SqlDbType.Int).Value       = request.PlanId;
        cmd.Parameters.Add("@rowver", SqlDbType.Binary, 8).Value = request.RowVersion;
        if (cmd.ExecuteNonQuery() != 1)
            throw new DBConcurrencyException("他の更新と競合しました。");   // 一時エラーではないので再試行されない
    }
});
```

#### 接続文字列（参考）

```text
Server=tcp:<server>.database.windows.net,1433;Database=<db>;
Authentication=Active Directory Managed Identity;Encrypt=True;
Connect Timeout=30;ConnectRetryCount=2;ConnectRetryInterval=10
```

- この例は App Service のシステム割り当て Managed Identity と SQL Database の書き込み先への接続を想定する。ID の有効化、DB ユーザーの作成と必要な権限付与、ネットワーク到達性は別途構成する。MI の場合はその接続先へ変更する。
- `ConnectRetryCount` / `ConnectRetryInterval` は、**初回の `Open()` の接続回復性と、切断されたアイドル接続の回復性の両方**に関係する。実行途中のクエリやアクティブなトランザクションを自動で再実行するものではない（[公式資料](https://learn.microsoft.com/azure/azure-sql/database/troubleshoot-common-connectivity-issues?view=azuresql#net-sqlconnection-parameters-for-connection-retry)）。
- **`Connect Timeout` は全再試行を賄える値にする**。公式の条件は `Connect Timeout >= ConnectRetryCount × ConnectRetryInterval` だが、同資料のタイムライン例では 0 回目の試行と障害検知の時間も加算される（検知 1 秒 + 10 秒 × 3 回 = 31 秒）。`ConnectRetryCount=3` だと 30 秒では 3 回目に到達しないため、上記例では 2 回に抑えている。
- **`Connect Timeout` は Commit の待ち時間も決める**。3-3 のとおり `SqlTransaction.Commit()` は `CommandTimeout` ではなく `ConnectTimeout` を参照するため、上記例では Commit の応答待ちが最大 30 秒になる。値を大きくすると接続再試行の余裕は増えるが、不確定状態の判明がその分遅れる。
- **`MultiSubnetFailover` は指定しない**。公式定義は「SQL Server 2012 以降の可用性グループ リスナーまたはフェールオーバー クラスター インスタンスに接続する場合は常に指定する」で、ゲートウェイ経由の Azure SQL Database / MI は対象外。指定すると `TransparentNetworkIPResolution` と `IPAddressPreference` が無視される（[接続文字列キーワードの仕様](https://learn.microsoft.com/dotnet/api/microsoft.data.sqlclient.sqlconnection.connectionstring)）。
- 接続回復性に加えて、`OpenRetryProvider` は `NumberOfTries = 5`、つまり **初回を含む最大 5 試行**を行う。さらにアプリ層にも再試行があるため、回数と待機が積み重なる。`MaxTimeInterval` は個々の待機の上限であり、総経過時間を制限しない。接続タイムアウト・コマンドタイムアウトにも各試行の実行時間が必要で、設定した全再試行を必ず消化できるわけではない。
- 本番では再試行を担当する層を整理する。例えば接続回復性を `ConnectRetryCount=0` で無効にする、`OpenRetryProvider` を外してアプリ層に集約する、または併用時の総時間予算を設定する。クライアント側の -2 は CRL の既定の一時エラー一覧には含まれず、本例ではアプリ層の一覧で扱う。
- ドライバのコマンド再試行は「組み込みのコマンド プロバイダーは、トランザクションがアクティブな場合には再試行を行わないため、複数ステートメントのトランザクションは、トランザクションを再作成できるアプリケーション コードで再試行する必要がある」とされている。このため、トランザクションの再試行は本コードのようにアプリ層で行う。

### 7-4. 検知: Application Insights → Azure Monitor アラート

```csharp
// Microsoft.ApplicationInsights (NuGet)
using System;
using System.Globalization;
using System.Linq;
using Microsoft.ApplicationInsights;
using Microsoft.ApplicationInsights.Extensibility;
using Sample.DataAccess;

var configuration = TelemetryConfiguration.CreateDefault();
configuration.ConnectionString = appInsightsConnectionString;

var telemetry = new TelemetryClient(configuration);
var runner = new ResilientTransactionRunner(connectionString, (name, props) =>
    telemetry.TrackEvent(name, props.ToDictionary(property => property.Key,
        property => Convert.ToString(property.Value, CultureInfo.InvariantCulture))));
```

`using` はファイル先頭、`var configuration` 以降はアプリケーションの初期化メソッド内に置く。`TelemetryConfiguration.Active` は非推奨のため使わず、DI などで管理された `TelemetryConfiguration` があればそれを使う。`TelemetryConfiguration` と `TelemetryClient` はアプリケーション全体で再利用し、要求ごとに生成しない。通常のログ出力例外はトランザクション制御から隔離し、独立した Trace 出力へフォールバックする。ただし Trace の出力先設定も必要であり、**両方の出力先の障害や AP プロセス停止時の記録まで保証するものではない**。未解決イベントをサンプリング対象から除外し、ログ収集経路自体も監視する。

イベント名の意味付け:

| イベント | 意味 | 運用 |
|---|---|---|
| `SqlCommitSucceeded` | Commit が正常復帰した | 正常完了件数 |
| `SqlTransientBeforeCommit` | Open / Begin / 業務処理の一時エラーで、Commit は未呼び出し | 再試行回数と上限到達を監視 |
| `SqlCommitInDoubt` | Commit 中の例外により成否が不確定になった | **不確定検知件数**。ロールバックされたケースも含み、コミット済み応答喪失の確定件数ではない |
| `SqlCommitInDoubtResolved`（Resolution = Committed） | 不確定検知後、同じキーのコミット済みジャーナル行を確認した | **成功確認件数**。再構成が原因だったことまで証明するものではない |
| `SqlRequestAlreadyCommitted`（Resolution = AlreadyCommitted） | ジャーナル INSERT が重複し、業務処理を実行せず成功済みとして扱った | 上流からの再送・同時要求を含む重複排除件数。不確定コミットと別集計 |
| `SqlCommitInDoubtUnresolved` | 行が見えないまま上限到達、または照会不能で `InDoubtCommitException` | **未解決件数**。アラート・手動照合の対象 |
| `SqlCleanupFailed` | トランザクションまたは接続の解放時に例外が発生した | 接続資源と後続処理への影響を調査 |

集計はキー・操作名・試行番号で相関させ、要求件数と試行件数を分ける。同じキーの同時要求がある場合、行の存在は **その業務要求の成功**を示すが、どの試行で応答が失われたかを一意に示すとは限らない。AP 停止やテレメトリ欠落があれば件数は過少になる。自然発生の計測期間・負荷条件と、障害注入試験の結果も分けて管理する。

アラート用 KQL（ログ検索アラート、結果 ≥ 1 件で通知）:

Application Insights リソースのログ画面で使うスキーマを示す。Log Analytics ワークスペースを直接クエリする場合は `AppEvents` などの対応するテーブル・列名へ合わせる。通知はログ取り込みとアラート評価の後であり、即時到達を保証しない。

```kusto
customEvents
| where name == "SqlCommitInDoubtUnresolved"
| extend key = tostring(customDimensions.IdempotencyKey),
         op  = tostring(customDimensions.Operation),
         err = tostring(customDimensions.SqlErrorNumber)
| project timestamp, key, op, err
```

`TimestampUtc` を残しているので、Resource Health のダウンタイム履歴（Planned maintenance / Reconfiguration）や診断ログ `Errors` の 40613 と時刻で突合できる。

### 7-5. EF6 を使っている場合（6.1 以降・ほぼ設定のみ）

出典: [トランザクション コミット エラーの処理（EF6）](https://learn.microsoft.com/ef/ef6/fundamentals/connection-resiliency/commit-failures)

この EF6 設定例は `System.Data.SqlClient` プロバイダー用。Microsoft.Data.SqlClient を採用する場合は EF6 用の対応プロバイダーと設定を別途確認し、名前空間の置換だけで流用しない。EF6 のトランザクション追跡は、上流からの業務要求の再送を永続的に重複排除する機能とは別である。

> 「EF 6.1 の新機能により、EF はトランザクションが成功したかどうかをデータベースに再確認し、適切な処理を透過的に実行できる。」

```csharp
using System;
using System.Data.Entity;
using System.Data.Entity.Infrastructure;
using System.Data.Entity.SqlServer;

public class AzureSqlDbConfiguration : DbConfiguration
{
    public AzureSqlDbConfiguration()
    {
        // コミット中の切断を __Transactions 表で自動検証
        SetTransactionHandler(SqlProviderServices.ProviderInvariantName, () => new CommitFailureHandler());
        // 一時エラーの自動再試行
        SetExecutionStrategy(SqlProviderServices.ProviderInvariantName,
            () => new SqlAzureExecutionStrategy(5, TimeSpan.FromSeconds(60)));
    }
}
```

仕組みは 7-2 のジャーナル方式と同じ:

> 「この機能を有効にすると、EF は `__Transactions` という新しいテーブルをデータベースへ自動的に追加する。EF がトランザクションを作成するたびにこのテーブルへ新しい行が挿入され、コミット中にトランザクション エラーが発生した場合は、その行が存在するか確認される。不要になった行は EF が可能な限り削除するが、アプリケーションが途中で終了するとテーブルが増大する可能性があるため、状況によっては手動でテーブルをクリーンアップする必要がある。」

**注意:** 複数の `SaveChanges` を 1 トランザクションにまとめる場合、再試行戦略が有効だと `Database.BeginTransaction()` はそのままでは使えない（[接続の回復性と再試行ロジック（EF6）](https://learn.microsoft.com/ef/ef6/fundamentals/connection-resiliency/retry-logic)）:

> 「解決策は、実行戦略を手動で使用し、実行する一連のロジック全体を渡すことである。そうすれば、いずれかの操作が失敗した場合にすべてを再試行できる。…コンテキストは、再試行されるコード ブロック内で構築する必要がある。これにより、再試行のたびにクリーンな状態から開始できる。」

```csharp
new SqlAzureExecutionStrategy().Execute(() =>
{
    using (var db = new WorkPlanContext())            // コンテキストは再試行ブロック内で生成
    using (var tx = db.Database.BeginTransaction())
    {
        // ... 複数の更新と SaveChanges ...
        tx.Commit();
    }
});
```

### 7-6. `TransactionScope` を使っている場合

`System.Transactions.TransactionInDoubtException` を「in-doubt」として同じ検証パスに流す（[TransactionInDoubtException クラス](https://learn.microsoft.com/dotnet/api/system.transactions.transactionindoubtexception)）:

> 「トランザクションの状態を判定できない場合、そのトランザクションは不確定（in-doubt）となる。具体的には、コミットされたか中止されたかという最終結果を確認できない状態である。この例外は、トランザクションをコミットしようとした際に、そのトランザクションが `InDoubt` 状態になった場合にもスローされる。これは回復可能なエラーである。」

`Complete()` は成功の投票であり、実際のコミットに伴う例外はスコープの `Dispose()` 時にも発生するため、`using` 全体を例外処理の対象にする。検証は元のスコープを抜けた後、または ambient transaction を抑制して新しい接続で行う。7-3 はローカルトランザクション用の参考実装であり、分散トランザクションへそのまま適用せず、参加リソースとトランザクション マネージャーの回復手順を確認する。

---

## 8. 実装・運用上の注意

1. **検証は旧接続を破棄してから、新しい接続で行う**。RCSI による未観測や照会不能は、ロールバックの証明ではない。例のアプリ層待機合計は 135 秒だが、**接続等を含む総所要時間には別の上限設計が必要**。上限到達・権限不足等の非一時エラーでも不確定状態とアラート経路を維持する。
2. **`work` 内に DB 外の副作用を入れない**: 指令送信・外部 API は業務更新と同一トランザクションに Outbox 行として記録し、別プロセスで送信する。これは DB 更新と送信予定の原子性を担保するが、**Outbox 単独では二重送信を防げない**。送信成功後・送信済み記録前の停止で再送されるため、下流にも同じ冪等キーを伝え、受信側の処理と重複排除記録を原子的に確定するなどの対策が必要（[冪等コンシューマー パターン](https://learn.microsoft.com/azure/architecture/patterns/idempotent-consumer)）。
3. **冪等性と並行更新の制御を分ける**。増分更新もジャーナル方式で再送時の二重適用を防げる。絶対値更新だけでは並行更新の取りこぼしが起こり得るため、`rowversion` 等の競合制御を業務要件に応じて使う。IDENTITY 主キーの INSERT も、同じトランザクションのジャーナルで保護できる。
4. **.NET Framework 4.8 の `SqlTransaction.Commit()` は同期 API**。非同期の待機を採用する場合は `await Task.Delay(...)` を使えるよう呼び出し元まで async 化する。単に `Thread.Sleep` を未待機の `Task.Delay` に置き換えてはならず、長い再確認処理はバックグラウンド化も検討する。
5. **DB の `DELAYED_DURABILITY` は既定の DISABLED のまま**にする（有効化すると「ACK 済みでも未永続化」が起こり得るため前提が崩れる）。
6. **メンテナンス ウィンドウと通知の適用条件を確認する**。選択可能なサービス レベル・リージョンで既定以外のウィンドウと事前通知を構成する。通知漏れの例外や緊急メンテナンスによる上書きがあり、ハードウェア障害・負荷分散・SLO 変更などによる再構成もウィンドウ外で発生し得る。
7. **テストは専用の検証環境で行う**。連続更新中の手動フェールオーバーで再接続・再試行を確認し、Commit 応答喪失は必要に応じて障害注入で再現する。単発のフェールオーバーで不確定イベントが出ないことは不具合の否定にならず、試験中の件数は通常運用の発生率ではない。

最低限の検証ケース:

| ケース | 確認する結果 |
|---|---|
| ジャーナルのキー重複 | 業務処理を呼ばず `AlreadyCommitted`。`SqlRequestAlreadyCommitted` として別集計 |
| 業務テーブルの一意制約違反 | 成功扱いにせず例外を返す。業務更新と今回のジャーナルをロールバック |
| Open / Begin / Commit 前の一時エラー | 同じキーで新しい接続・トランザクションを使い再試行 |
| Commit 応答喪失後、行が見える | `SqlCommitInDoubt` の後に `SqlCommitInDoubtResolved`。業務更新を再実行しない |
| 行が遅れて見える / 最後まで見えない | 照会を継続し、成功確認または `InDoubtCommitException`。0 件をロールバック確定としない |
| 検証時の権限エラー・接続障害 | 非一時エラーまたは確認上限で `SqlCommitInDoubtUnresolved` を記録し、不確定を維持 |
| ログ出力・解放処理の例外 | 通常のログ例外が業務結果・不確定確認を変更しない。フォールバックと資源解放失敗も監視 |
| 同時再送・保持期限を超えた再送 | 保持中の同じキーでは二重適用しない。期限外は拒否または別途照合する運用を検証 |

コードのコンパイルや制御フローのテストだけでは、実 DB のロック・分離レベル・ドライバー切断処理・通知経路を検証したことにはならない。採用するサービス レベルとドライバー版で上記の障害試験を実施する。

---

## 9. ヒアリングで確認したい事項

- 実装言語 / フレームワーク（.NET Framework 4.8 + ADO.NET / EF6 か、.NET (Core) + EF Core か、Java / JDBC か）— 適用できる公式パターンが変わる
- SQL Database か Managed Instance か、サービス レベル（General Purpose / Business Critical）
- メンテナンス時間帯を含む時間帯別のコミット頻度、応答遅延の実測値
- 更新要求の一意キーと要求内容を再送時も維持できるか、同時再送・最大再送期間・手動再送の扱い、増分更新の有無
- トランザクション内に DB 外の副作用（外部 API 呼び出し、指令送信）があるか
- `TransactionScope` / 分散トランザクションの利用有無

---

## 10. 参考資料

### Azure SQL の動作・メンテナンス

- [Azure SQL Database および Azure SQL Managed Instance での Azure メンテナンス イベントの計画](https://learn.microsoft.com/azure/azure-sql/database/planned-maintenance?view=azuresql)
- [Azure SQL Database のメンテナンス期間](https://learn.microsoft.com/azure/azure-sql/database/maintenance-window?view=azuresql)
- [Azure SQL Database のメンテナンス期間に関する FAQ](https://learn.microsoft.com/azure/azure-sql/database/maintenance-window-faq?view=azuresql)
- [Azure SQL Database の計画メンテナンス イベントの事前通知](https://learn.microsoft.com/azure/azure-sql/database/advance-notifications?view=azuresql)
- [Azure SQL Database の高可用性](https://learn.microsoft.com/azure/azure-sql/database/high-availability-sla-local-zone-redundancy?view=azuresql)
- [Azure SQL Managed Instance でユーザーが開始した手動フェールオーバー](https://learn.microsoft.com/azure/azure-sql/managed-instance/user-initiated-failover?view=azuresql)

### 検知・監視

- [Resource Health を使用した Azure SQL Database の接続のトラブルシューティング](https://learn.microsoft.com/azure/azure-sql/database/resource-health-to-troubleshoot-connectivity?view=azuresql)
- [Azure SQL Database の sys.event_log](https://learn.microsoft.com/sql/relational-databases/system-catalog-views/sys-event-log-azure-sql-database?view=azuresqldb-current)
- [Azure SQL Database の一時的な接続エラーのトラブルシューティング](https://learn.microsoft.com/azure/azure-sql/database/troubleshoot-common-connectivity-issues?view=azuresql)
- [Azure SQL Database の一般的な接続エラーとエラー コードのトラブルシューティング](https://learn.microsoft.com/azure/azure-sql/database/troubleshoot-common-errors-issues?view=azuresql)
- [Azure SQL のメトリックとリソース ログのストリーミング エクスポートの構成](https://learn.microsoft.com/azure/azure-sql/database/metrics-diagnostic-telemetry-logging-streaming-export-configure?view=azuresql)
- [Azure SQL Database の Azure Monitor リファレンス](https://learn.microsoft.com/azure/azure-sql/database/monitoring-sql-database-azure-monitor-reference?view=azuresql)

### AP 側の実装

- [SQL Server 用 Microsoft.Data.SqlClient（概要・本番向けベースライン）](https://learn.microsoft.com/sql/connect/ado-net/microsoft-ado-net-sql-server?view=sql-server-ver17)
- [SqlClient ドライバーのサポート ライフサイクル](https://learn.microsoft.com/sql/connect/ado-net/sqlclient-driver-support-lifecycle?view=sql-server-ver17)
- [SqlConnection.ConnectionString（接続文字列キーワードの仕様・MultiSubnetFailover の適用範囲）](https://learn.microsoft.com/dotnet/api/microsoft.data.sqlclient.sqlconnection.connectionstring)
- [SqlClient の構成可能な再試行ロジック](https://learn.microsoft.com/sql/connect/ado-net/configurable-retry-logic?view=sql-server-ver17)
- [SqlClient で再試行ロジックを設定](https://learn.microsoft.com/sql/connect/ado-net/configurable-retry-logic-sqlclient-introduction?view=sql-server-ver17)
- [SqlClient の Microsoft Entra 認証と 7.0 への移行](https://learn.microsoft.com/sql/connect/ado-net/sql/azure-active-directory-authentication?view=sql-server-ver17)
- [SqlClient 6.1.0 の .NET Framework 向けトランザクション実装（ExecuteTransaction2005）](https://github.com/dotnet/SqlClient/blob/v6.1.0/src/Microsoft.Data.SqlClient/netfx/src/Microsoft/Data/SqlClient/SqlInternalConnectionTds.cs)
- [SET TRANSACTION ISOLATION LEVEL（RCSI と読み取りの仕様）](https://learn.microsoft.com/sql/t-sql/statements/set-transaction-isolation-level-transact-sql?view=sql-server-ver17)
- [冪等コンシューマー パターン（Outbox と下流の重複排除）](https://learn.microsoft.com/azure/architecture/patterns/idempotent-consumer)
- [SQL Server 用 JDBC ドライバーの接続回復性](https://learn.microsoft.com/sql/connect/jdbc/connection-resiliency?view=sql-server-ver17)
- [EF Core の接続の回復性（トランザクションのコミット失敗と冪等性の問題）](https://learn.microsoft.com/ef/core/miscellaneous/connection-resiliency)
- [EF Core の ExecutionStrategyExtensions.ExecuteInTransaction](https://learn.microsoft.com/dotnet/api/microsoft.entityframeworkcore.executionstrategyextensions.executeintransaction)
- [EF6 のトランザクション コミット エラーの処理](https://learn.microsoft.com/ef/ef6/fundamentals/connection-resiliency/commit-failures)
- [EF6 の接続の回復性と再試行ロジック](https://learn.microsoft.com/ef/ef6/fundamentals/connection-resiliency/retry-logic)
- [TransactionInDoubtException クラス（System.Transactions）](https://learn.microsoft.com/dotnet/api/system.transactions.transactionindoubtexception)
- [一時的な障害の処理（Azure Architecture Center）](https://learn.microsoft.com/azure/architecture/best-practices/transient-faults)

---

### 調査上の注記

- 仕様・サポート版の確認日: **2026-09-16**。SqlClient の初回 Open とアイドル接続の回復性は ADO.NET 向け公式資料で確認した。Commit のタイムアウト参照先は 6.1.0 の公開実装を根拠とするため、別バージョンでの同一動作を保証しない。
- Resource Health の具体的な判定条件とメンテナンス通知の例外は SQL Database 向け資料に基づく。Managed Instance 固有のウィンドウ詳細・通知条件・接続メトリック名は採用構成で別途確認する。
- 発生頻度の概算モデル（4-2）は到着率・窓幅・切断回数を仮定した計算例。公開情報だけでは実環境の本事象の頻度を確定できず、AP 側の不確定例外件数もそのまま確定発生件数にはならない。
