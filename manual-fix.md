# 機械的に対応できない非互換カタログ

機械検知（Rector / PHPStan / 公式 UPGRADING 突合）をすり抜ける非互換の一覧。

収録基準は「**UPGRADING に載っているか**」ではなく「**ルールを知っても該当箇所を機械的に特定・修正できないか**」。
UPGRADING にルールが書かれていても、戻り型 mixed で grep/静的解析が刺さらない、値・状態次第で発火する、
といった理由で**箇所の特定が静的にできない**ものはここに含める（例: 項目2・9）。

多くの項目に共通する見つけ方は **実機で動かして deprecation / warning をログに出す**こと:

1. `errors.continue_on` に対象のエラー型を追加し、**fatal 化を止めてログに流す**（巡回が途中で止まらない）。
2. 実データ入りで全導線を1パス巡回（一覧・検索・帳票・POST・各ログイン種別）。
3. 出たログを潰す。

以降の各項目「見つけ方」は、この上での個別の炙り方。

---

## 1. `@` エラー抑制（8.0〜）

<details>
<summary>`@` が deprecation を隠し、8.0 で致命エラーも顕在化</summary>

| 観点 | 内容 |
|---|---|
| 見つけ方 | `@$data['key']` 等の `@` は deprecation をログから消すため、**`errors.continue_on` ログ方式を使う項目（2・4・5）が無効化**される（「ログゼロ＝完了」が嘘になる）。移行の最初に `grep -rnE '@\$|@[a-zA-Z_]+\(' <app>` で棚卸しし、抑制を外してからログ収集する |
| 直し方 | `@` を使わず `?? ''` 等の明示ガードに置換。`@func()` は 8.0 で**致命エラーを隠さなくなった**ため、隠れていた fatal（削除済み関数など）が出ないか併せて確認 |
| 注意（カスタム error handler） | PHP 8.0 で `@` は **error_reporting を 0 にしなくなった**（特定マスクをセットするだけ）。`error_reporting() === 0` / `!== 0` で `@` を判定している**独自 `set_error_handler` は壊れる**（`@` が効かなくなる）。正しくは `error_reporting() & $severity` のビット判定。フレームワークの error handler に手を入れる場合（FuelPHP の `Errorhandler` 等）はここを確認 |

```diff
- $len = strlen(@$data['memo']);            // null/未定義を @ が握り潰す→ログにも出ない
+ $len = strlen((string) ($data['memo'] ?? ''));

  // カスタム error handler 側
- if (error_reporting() !== 0) { ... }       // 8.0 で @ 中でも非0 → @ を無視してしまう
+ if (error_reporting() & $severity) { ... } // @ で抑制された severity を正しく尊重
```
</details>

---

## 2. null を文字列関数へ渡す（8.1〜）

<details>
<summary>strlen / trim 等に null → "Passing null ... deprecated"</summary>

| 観点 | 内容 |
|---|---|
| 見つけ方 | 戻り型が `mixed`（`Session::get`/`Input::param`/DB結果）で Rector/PHPStan は素通り |
| 直し方 | `(string)` キャスト or `Input::param('no','')` のデフォルト付き取得 |

```diff
- strlen(Arr::get($data,'memo'))           // memo 無→null
+ strlen((string) Arr::get($data,'memo'))
```
</details>

---

## 3. 値依存の比較（8.0〜）

<details>
<summary>`0 == "foo"` が true→false に変化</summary>

| 観点 | 内容 |
|---|---|
| 見つけ方 | 壊れるかは実行時の値次第で静的に取れない（対象: `==` / `!=` / `switch` / loose `in_array`）。<br>手動レビューで比較両辺の型が一貫しているか（常に数値 or 常に文字列）を確認。混在だけが危険 |
| 直し方 | 危険な混在のみ `(int)$x === 0` 等で明示。**`===` 一括変換はしない**（数値文字列比較を壊す） |

```diff
- if ($status == 0)        // $status="foo" が 8 で false 化
+ if ((int) $status === 0)
```
</details>

---

## 4. 非配列・非Countable への操作（count / foreach / 配列アクセス・8.0〜）

<details>
<summary>配列以外（null・スカラー・bool）が来て `count()` は TypeError、`foreach`/配列アクセスは Warning</summary>

| 観点 | 内容 |
|---|---|
| 見つけ方 | 値が配列か否かは実行時依存（mixed 戻り値）で Rector・PHPStan は素通り。<br>`count(`/`foreach(`/`$x[` 箇所の入力源を辿り、常に配列化されているか確認 |
| 直し方 | 入力正規化層で必ず配列化。個別には `is_array()` / `is_countable()` ガード |

```diff
- $len = count($tmp);                 // $tmp が null/スカラーだと 8.0 で TypeError（fatal）
+ $len = is_countable($tmp) ? count($tmp) : 0;
- foreach ($templates as $t) { ... }  // $templates が非配列だと Warning（続行・空ループ扱い）
+ if (is_array($templates)) foreach ($templates as $t) { ... }
```
</details>

---

## 5. 未定義 / null 配列キー・未定義変数（Twig・PHP 両方）

<details>
<summary>`{{ errors['x'] }}` / `$row['id']` がデータ次第で 8.x Warning。typo も顕在化</summary>

| 観点 | 内容 |
|---|---|
| 見つけ方 | 該当カラムが実際に null/未定義のデータでしか発火しない。<br>**Twig**: PHPツールは `.twig` を見ない → `strict_variables=true` で実機レンダリングし RuntimeError で炙る。<br>**PHP**: 未定義キー・typo 由来の未定義変数（PHP7 Notice → PHP8 Warning）は 項目1（`@` 除去）後にログ巡回で炙る |
| 直し方 | Twig は `{{ errors['x'] ?? '' }}`、PHP は `isset()` / `??` ガード。typo は正しい変数名に修正 |

```diff
- {{ errors['seminar_date'] }}
+ {{ errors['seminar_date'] ?? '' }}
- if ($row['id'] == '')              // 'id' キー欠落で 8.0 Warning
+ if (!isset($row['id']) || $row['id'] == '')
```
</details>

---

## 6. MIME タイプ判定（アップロード）

<details>
<summary>CSV が `text/plain` 等で来て whitelist を通らない</summary>

| 観点 | 内容 |
|---|---|
| 見つけ方 | 送られる MIME はブラウザ/OS 依存で静的に取れない。<br>実機で実ファイルをアップロードして弾かれないか確認 |
| 直し方 | `mime_whitelist` に実際に来る型だけを追加（`text/plain` は CSV 以外も通すので必要な型に絞る） |

```diff
- 'mime_whitelist' => ['text/csv'],
+ 'mime_whitelist' => ['text/csv', 'application/csv'],
```
</details>

---

## 7. 動的プロパティ（8.2〜）

<details>
<summary>未宣言プロパティへの代入が非推奨（magic 経由は遮蔽）</summary>

| 観点 | 内容 |
|---|---|
| 見つけ方 | Orm/Model_Crud は `__get/__set` 経由で非推奨が発火しない。直接代入だけが危険。<br>実機監視で未宣言プロパティへの直接代入を探す |
| 直し方 | クラスに明示宣言、または `#[\AllowDynamicProperties]` を付与 |

```diff
  class Model_Foo extends \Orm\Model {
+     protected $extra;   // 未宣言で直接代入すると 8.2 で deprecated
  }
```
</details>

---

## 8. ライブラリのメジャー更新に伴う挙動変更（PhpSpreadsheet・mPDF・Twig 等）

<details>
<summary>同じ呼び出しのまま結果だけ変わる（列インデックス 0→1始まり等）。多くは deprecation を出さず黙って誤出力</summary>

| 観点 | 内容 |
|---|---|
| 見つけ方 | **2種類に分かれ、検知可否が逆**。<br>(a) **API名・シグネチャの変化**（クラス/メソッドの改名・削除: `Twig_Environment`→`\Twig\Environment`、`setCellValueByColumnAndRow` 廃止）＝ **新ライブラリを入れて PHPStan にかければ `undefined method/class` で取れる**（厳密には機械検知できる側）。<br>(b) **シグネチャは同じで挙動だけ変わる**（列インデックス 0→1始まり・デフォルト値・戻り型・数値セルの既定型）＝ **静的に取れず、多くは deprecation も出さず黙って誤出力**。共通の `continue_on` ログ方式では炙れない。<br>→ (b) は **ライブラリ公式の UPGRADING / CHANGELOG を一次情報として読み**、**実機で帳票を生成して出力物を目視・前バージョンと比較**するのが唯一の発見手段 |
| 直し方 | CHANGELOG の該当変更に追従（例: 列番号を 0→1 始まりに直す）。可能なら**挙動変更の影響を受けない安定APIに寄せる**（座標文字列指定の `setCellValue('A1', ...)` 等。これで列インデックス系の非互換を回避できる） |

```diff
- $sheet->setCellValueByColumnAndRow(0, 1, $v);  // PhpSpreadsheet 旧: 列0始まり
+ $sheet->setCellValueByColumnAndRow(1, 1, $v);  // 新: 列1始まり（無修正だと1列ズレて黙って誤出力）
```
</details>

---

## 9. 従来 false / 警告だった内部処理が例外化（8.0〜）

<details>
<summary>戻り値で握っていた箇所が黙って例外パスに化ける。シグネチャ不変で静的に取れない</summary>

| 観点 | 内容 |
|---|---|
| 見つけ方 | 代表例は **PDO 既定エラーモードが `ERRMODE_SILENT`→`ERRMODE_EXCEPTION`**（[RFC pdo_default_errmode](https://wiki.php.net/rfc/pdo_default_errmode)、UPGRADING-8.0 に記載）。<br>**ルールは UPGRADING で分かるが、自コードのどの `=== false` 分岐が壊れるかは静的に特定できない**（項目2と同性質）。シグネチャ不変で Rector/PHPStan 素通り、deprecation も出ず **fatal/例外として落ちる**ため `continue_on` ログ方式でも炙れない。<br>実機で全 DB 導線を踏み、握っていた失敗が例外に化けていないか確認 |
| 直し方 | `=== false` 前提の分岐を `try/catch` に直す、または明示的に errmode を設定。<br>**FuelPHP 系アプリは直 PDO を書かず DBAL/ORM 経由が普通 → 多くはフレームワーク層で顕在化するので、直すなら fork 側**。<br>関連: `PDO::inTransaction()` が implicit commit を反映するようになり（UPGRADING-8.0 PDO MySQL）、トランザクション終了後の `commit`/`rollBack` が例外を投げる。core 側の commit/rollback に `inTransaction()` ガードを足す |

```diff
- $stmt = $pdo->query($sql);
- if ($stmt === false) { ... }              // 8.0 で false にならず PDOException → この分岐に来ない
+ try { $stmt = $pdo->query($sql); } catch (\PDOException $e) { ... }

  // core 側: implicit commit 後の二重 commit/rollback 防御
+ if ( ! $this->_connection->inTransaction()) { return true; }
  return $this->_connection->commit();
```
</details>

