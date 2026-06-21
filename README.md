# FuelPHP 1.8 + PHP 7.x → PHP 8.5 移行キット

FuelPHP 1.8.2 / PHP 7.x を **PHP 8.5** へ上げる汎用手順。方針は 3 つ:

- **8.5 一本** — 途中版は経由しない（deprecation は累積で 8.5 に出揃う）
- **3 段で確かめる** — 機械検知 → 文脈判定 → 実機 + ログ
- **修正は `fuel/app` のみ** — core / packages / vendor は検知だけ、直すなら fork

## 手順（0〜6）

`<summary>` で流れ、開いて詳細。

<details>
<summary>0. 検証環境を先に作る</summary>

修正前に「壊れたら気づける」土台を作る。ここで成否が決まる。

| 項目 | 要点 |
|---|---|
| PHP 8.5 Docker | 本番同等設定で起動。8.1〜8.5 の deprecation が出揃う |
| deprecation ログ化 | 致命化させず `continue_on` でログに流す（下記） |
| 実行時ディレクトリ | `fuel/app/storage/{cache,log,sync}` 等が無いと `File::create` が `Invalid basepath` で fatal。`.gitkeep` で追跡し fresh clone でも存在させる。`locale` 未インストールも起動時 fatal の定番 |
| 業務データ（seed） | **ここで作って投入まで済ませる**。PII なし・マスタ系と業務系で分割。生成スクリプト→`initdb/*.sql`→`compose up` で自動投入 |

**業務データの作り方はアプリ型で変わる**（手順5のゲート）:

| アプリ型 | 前提 |
|---|---|
| 自己完結（管理画面・帳票） | **合成 seed** を DB 投入 |
| 連携 / プロキシ（外部 API 同期等） | **外部 API 到達 or 記録フィクスチャ**。seed だけでは変換層に入れない |

ログ設定（致命化させない）:
```ini
; php.ini
error_reporting = E_ALL
log_errors = On
zend.exception_ignore_args = Off
```
```php
// fuel/app/config/<env>.php
'errors' => ['continue_on' => [E_DEPRECATED, E_USER_DEPRECATED, E_WARNING, E_NOTICE]],
```

**詰まりどころ**:
- 検証DBを migrations で組むなら**本番と列が一致するか先に確認**。未追従で列欠落だと Orm の全列 SELECT が `1054 Unknown column`→8.5 は PDO 既定 EXCEPTION で 500（manual-fix #9）。本番ダンプの `SHOW COLUMNS` と `comm` 差分照合し、欠け列は補完 migration（型・位置は本番 DDL で確認）。
- oil の CLI は `--env` でなく **`FUEL_ENV=docker php oil …`**（`--env` はフォールバックして別host接続）。
</details>

<details>
<summary>1. composer / 依存を 8.5 対応</summary>

まず阻害要因を洗い出す:
```bash
composer why-not php 8.5
```

> **制約が通る ≠ 動く。** `why-not` が阻害なしでも、実際の壁は依存制約でなく framework のコード（curly brace offset 等＝手順2/3 で出る）。

| 項目 | 対応 |
|---|---|
| FuelPHP 本体 | 8.x 版なし → 共有 fork（PHP8 ブランチ）。案件ごとに作り直さない |
| ビルド再現性 | `composer.lock` を追跡して commit 固定 |
| 周辺ライブラリ | `composer/installers ^2` / `twig ^3`（2→3 は API 破壊）/ `phpspreadsheet` 等 |

fork を `repositories` に配線:
```jsonc
// composer.json
"repositories": [
  { "type": "vcs", "url": "git@github.com:in-g-jp/fuel-core.git" }
  // auth / email / oil / orm / parser も同様
],
"require": {
  "fuel/core": "dev-1.9/develop-php85",
  "twig/twig": "^3"
}
```
</details>

<details>
<summary>2. 機械検知（静的解析ツール）</summary>

**アプリ層のみ**に静的解析。既存指摘が多ければ baseline 化して移行起因に集中。

| ツール | 役割 | 備考 |
|---|---|---|
| `php -l` | 構文・削除構文 | 波括弧オフセット・**暗黙 nullable**（`Type $x = null`→8.4 deprecated）等。これは静的に確実に取れる |
| Rector dry-run | 自動修正候補 | **fuel/app 移行の主力**。適用は人が判定してから |
| PHPCompatibility | 非互換を列挙 | **ROI 低**。sniff ~8.1 上限で、実績上 fuel/app は全バージョン 0 件（framework ノイズのみ）。入れるなら global/隔離で十分 |
| PHPStan / Psalm | 型・未定義・データフロー | アプリ層限定・baseline 運用 |

**ツールの入れ方**: 使うツールは `require-dev` に入れる。**Rector ^2 はスコープ配布で依存が実質 2 個**（symfony/php-parser を引かない）ので require-dev に入れても lock は汚れない。PHPCompatibility(phpcs)系は ROI が低く plugin 許可も要るので、使う場合だけ global/隔離で。

```bash
# 構文（暗黙 nullable もここで出る）
find fuel/app -name '*.php' -print0 | xargs -0 -n1 php -l | grep -v 'No syntax'
```

Rector の **検知は allow-list でなく「全セット ON + 近代化を skip」。** `withRules([...])` で移行必須ルールだけ挙げる方式は、設定済み汎用ルール（`RemoveFuncCallRector`=`curl_close` 削除 / `RenameFunctionRector`=`mysqli_execute`→… / `RenameCastRector`=`(integer)`→`(int)` 等）を取りこぼす。全 ON にして cosmetic だけ外せば、設定済み/新規ルールも必ず拾える:
```php
// rector_migrate.php — 8.5まで全セット ON、cosmetic だけ skip
return RectorConfig::configure()
    ->withPaths([__DIR__ . '/fuel/app', __DIR__ . '/htdocs'])  // ★ DOCROOT も対象
    ->withPhpSets(php85: true)                                  // 8.5 まで累積（単一指定。複数 true はエラー）
    ->withSkip([
        \Rector\Php54\Rector\Array_\LongArrayToShortArrayRector::class,  // array()→[]
        \Rector\Php80\Rector\NotIdentical\StrContainsRector::class,      // strpos→str_contains
        // … 近代化ルールを列挙（残ったものが「移行必須」）
    ]);
```
```bash
php -d xdebug.mode=off vendor/bin/rector process --config rector_migrate.php --dry-run
```
> 走査は **DOCROOT(`htdocs`/`public`)も対象**に。front controller にも非互換は出る（`strpos($response->body(),…)` の 8.1 null→文字列を fuel/app 限定走査で取りこぼした実例）。

検知できたものは**その場で直す**。代表例:
```diff
  // 8.1: null→文字列関数（動的 null は manual-fix #2）
- strlen(Arr::get($data,'memo'))
+ strlen((string) Arr::get($data,'memo'))

  // 8.4: fputcsv の escape を明示
- fputcsv($fp, $line)
+ fputcsv($fp, $line, escape: '\\')

  // 8.5: array_key_exists の null キー非推奨 → 空文字に（未定義キーは manual-fix #5）
- array_key_exists($key, $arr)
+ array_key_exists((string) $key, $arr)
```
</details>

<details>
<summary>3. 公式 UPGRADING 突合</summary>

手順 2 が拾わない**版固有の Removed/Deprecated** を `upgrading/UPGRADING-*.txt` を元に grep。sniff 上限の補完。

```bash
grep -rnE 'each\(|create_function\(|money_format\(' fuel/app   # 8.0 削除関数
grep -rnE '\$\{' fuel/app                                      # 8.2 ${} 文字列補間
grep -rnE 'utf8_encode|utf8_decode' fuel/app                   # 8.2 非推奨関数
grep -rnE 'curl_close\(|curl_multi_close\(|curl_share_close\(' fuel/app  # 8.5 no-op化で非推奨
```

> **8.5 で no-op 化＝非推奨になった関数（`curl_close`・`finfo_close`・`imagedestroy` 等）。** 呼んでも動くので目視では見落としやすい。手順2を全セットで回せば `RemoveFuncCallRector` が拾うが、grep でも二重に押さえる。

**その場で直す**。例:
```diff
  // 8.2: ${} 文字列補間 → {$...}
- "Hello ${name}"
+ "Hello {$name}"
```
</details>

<details>
<summary>4. ツールが取れない非互換</summary>

2・3 を**両方すり抜ける**非互換。実機ログ・操作・レビューでしか出ず、手順 5 で**炙って直す**。

→ 9 種の詳細は [`manual-fix.md`](manual-fix.md)。
</details>

<details>
<summary>5. 実機チェック</summary>

deprecation ログがゼロになるまで**手順 4〜5 を反復**（2・3 は一度で完了）。
`manual-fix.md` を片手に 9 種を炙る（`@` 抑制は巡回前に外す＝でないとログが嘘になる）。

**方法は導線の種類に依らず同じ三手**:
1. **ログを消す**（巡回ごとに新規分だけ見る）
2. **全入口を駆動する** ＝ 入口の種類に応じた手段で叩く。**「導線＝画面」とは限らない**:

   | 入口の種類 | 駆動手段 |
   |---|---|
   | 画面（管理UI・帳票） | ブラウザ巡回。一覧・検索・帳票・各ログイン種別・POST |
   | API エンドポイント | HTTP クライアントで全 EP（`curl` 等。本文は捨て**ステータス**だけ見れば十分） |
   | CLI / バッチ | `oil r <task>`（`FUEL_ENV` を忘れず） |

   各入口を **入力・認証・分岐のマトリクス**で叩いて経路を網羅する（例: 未認証 / 必須パラメータ欠落 / 不正値で外部到達 / **seed 済み値で DB 読込分岐**）。
3. **ログをゼロにする**:
   ```bash
   grep -irE 'deprecated|warning' fuel/app/logs/   # ゼロになるまで 4〜5 を反復
   ```

> 外部 API の応答を変換する層など、サンドボックス実データ/フィクスチャが無いと入れない経路は「未到達」と明記し、本番相当データのステージング検証に回す。
</details>

<details>
<summary>6. 仕上げ・リリース</summary>

1. fork の追加修正は commit ref 更新で取り込み（全案件共有）。
2. `composer.json` の `php` 制約を `>=8.5` に、CI も 8.5 固定。
3. PHPStan 最終確認 → 段階リリース。

```jsonc
// composer.json
"require": { "php": ">=8.5" }
```
`php` 制約は lock の content-hash に含まれるため、変更後は同期しないと `composer validate` が落ちる。版は変えずハッシュだけ更新:
```bash
composer update --lock
```
</details>
