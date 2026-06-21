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

| 項目 | 目的 | 要点 |
|---|---|---|
| PHP 8.5 Docker | 実機検証の土台 | 本番同等の設定で起動。8.1〜8.5 の deprecation が出揃う |
| deprecation ログ化 | 非推奨を一括収集 | 致命化させず `continue_on` でログに流す |
| 実行時ディレクトリ | 巡回を 500 で止めない | `fuel/app/storage/{cache,log,sync}` 等が無いと `File::create`（既存dir前提）が `Invalid basepath` で fatal。**fresh clone でも存在するよう `.gitkeep` で追跡**（`storage/cache` は gitignore されがち→中身無視・`.gitkeep` 残しに）。`locale` 未インストールも同様に起動時 fatal の定番 |
| 業務データ（seed） | 全導線を実データで踏む | **ここで作って投入まで済ませる**（手順5で慌てて作らない）。下記いずれか。**PII なし**・マスタ系と業務系でファイル分割。生成スクリプト（例 `docker/db/gen_dummy_seed.php`）→ `initdb/*.sql` に吐き、fresh な `compose up` で自動投入できる形に |

> **oil を CLI で回すときの env 指定。** 検証DBを migrations で組む/タスクを叩くとき、
> `oil refine migrate --env=docker` の **`--env` フラグは効かないことがある**（development 設定にフォールバックし別hostで名前解決失敗）。
> **環境変数で渡すのが確実**: `FUEL_ENV=docker php oil refine migrate`。

業務データは**アプリ型で手段が変わる**:

| アプリ型 | 「全導線を踏む」ための前提（手順5のゲート） |
|---|---|
| 自己完結（管理画面・帳票など内部ロジック中心） | **合成 seed**（DB に投入して一覧・検索・帳票・POST を巡回） |
| 連携 / プロキシ（外部 API を叩く同期 API 等） | **外部 API の到達性 or 記録済みフィクスチャ**。seed だけでは深い経路（外部レスポンスの変換層）に入れない。サンドボックスが叩けるなら実 ID、無ければ XML/JSON レスポンスを録って差し替える |

→ つまり「seed を作る」が常にゲートとは限らない。**自分のアプリの最深リスク経路が何の入力で発火するか**を先に見極める。

> **検証スキーマ＝本番、を先に保証する。** 検証DBを migrations で組むと、**migrations が本番に未追従だと列が欠落**する。
> Orm モデル(`$_properties` 全列を SELECT)はその列で `1054 Unknown column`→**8.5 は PDO 既定 `ERRMODE_EXCEPTION` で 500**（PHP7 は黙って false のことも＝manual-fix #9）。
> seed 投入＋全テーブルを 1 回 Orm で読む(=巡回)だけでドリフトが炙れる。本番ダンプがあれば
> `SHOW COLUMNS` の集合を本番 DDL と `comm` で差分照合し、欠けた列は補完 migration を足す（本番 DDL で型・位置を確認してから）。
> 「実機巡回で出た 500 が、移行起因か・元から壊れていたか」を切り分けるためにも、この一致確認を seed 投入前に済ませる。

ログに落とす（致命化させない）:
```ini
; php.ini
error_reporting = E_ALL
log_errors = On
zend.exception_ignore_args = Off
```
```php
// fuel/app/config/<env>.php — fatal 化を止める
'errors' => [
    'continue_on' => [E_DEPRECATED, E_USER_DEPRECATED, E_WARNING, E_NOTICE],
],
```
</details>

<details>
<summary>1. composer / 依存を 8.5 対応</summary>

まず阻害要因を洗い出す:
```bash
composer why-not php 8.5
```

> **「制約が通る ≠ 動く」。** `why-not` が阻害なしでも安心しない。インストール済み依存の php 制約は緩く 8.x を許容しがちで、
> 実際の壁は依存制約ではなく **framework 自身のコード**（curly brace offset・required-after-optional 等＝手順2/3 で出る）。why-not は出発点で、合否判定ではない。

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

**ツールの入れ方**: 解析ツールは「一度回して結果を反映したら終わり」なので、原則プロジェクトの `composer.json` を汚さない。
ただし **Rector ^2 はスコープ配布で依存が実質 2 個**（symfony/php-parser を引かない）→ `require-dev` に入れても lock は汚れずクリーン。PHPCompatibility(phpcs)系は plugin 許可も要るので global/隔離が無難。

```bash
# 構文（暗黙 nullable もここで出る）
find fuel/app -name '*.php' -print0 | xargs -0 -n1 php -l | grep -v 'No syntax'
# 自動修正候補（適用前にレビュー）。xdebug は切ると速い
php -d xdebug.mode=off vendor/bin/rector process fuel/app --dry-run
```

**Rector は「移行必須ルールだけ」を別 config で回して素の件数を数える。**
`withPhpSets(php85)` をそのまま回すと `array()→[]` 等の**近代化ルールに埋もれて**実移行課題が見えない。
移行必須ルールだけ列挙した `rector_migrate.php` を作り、件数とファイルを確定させる:
```php
// rector_migrate.php — fuel/app 限定・移行必須ルールのみ（近代化ノイズを混ぜない）
return RectorConfig::configure()
    ->withPaths([__DIR__ . '/fuel/app'])
    ->withRules([
        \Rector\Php81\Rector\FuncCall\NullToStrictStringFuncCallArgRector::class,   // 8.1 null→文字列関数
        \Rector\Php84\Rector\FuncCall\AddEscapeArgumentRector::class,               // 8.4 fputcsv 等の escape 既定
        \Rector\Php85\Rector\FuncCall\ArrayKeyExistsNullToEmptyStringRector::class, // 8.5 array_key_exists(null,…)
    ]);
```
```bash
php -d xdebug.mode=off vendor/bin/rector process --config rector_migrate.php --dry-run
```
> **限定 config の盲点**: 列挙したルールしか見ないので、**Rector にルールが無い非互換（例 `curl_close`）は当然出ない**。
> それは手順3の grep で取る（後述）。「Rector で 0 件＝移行課題なし」ではない。

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

フェーズ 2 が拾わない**版固有の Removed/Deprecated** を `upgrading/UPGRADING-*.txt` を元に grep。sniff 上限の補完。

```bash
grep -rnE 'each\(|create_function\(|money_format\(' fuel/app   # 8.0 削除関数
grep -rnE '\$\{' fuel/app                                      # 8.2 ${} 文字列補間
grep -rnE 'utf8_encode|utf8_decode' fuel/app                   # 8.2 非推奨関数
grep -rnE 'curl_close\(|curl_multi_close\(|curl_share_close\(' fuel/app  # 8.5 no-op化で非推奨
```

> **8.5 で「no-op 化された結果 deprecated」になった関数（`curl_close` 等）に注意。**
> シグネチャ不変・呼んでも動く・**Rector に該当ルールが無い**ため 2・4 を素通りし、
> 実機ログにしか出ない。だが UPGRADING-8.5 に明記＝**この grep に列挙すれば静的に取れる**。
> 「Rector に無い ＝ 機械検知不能」ではない。grep 対象に入れ忘れるな（実例: `curl_close` の取りこぼし）。

**その場で直す**。例:
```diff
  // 8.2: ${} 文字列補間 → {$...}
- "Hello ${name}"
+ "Hello {$name}"
```
</details>

<details>
<summary>4. ツールが取れない非互換</summary>

2・3 を**両方すり抜ける**非互換。実機ログ・操作・レビューでしか出ず、フェーズ 5 で**炙って直す**。

→ 9 種の詳細は [`manual-fix.md`](manual-fix.md)。
</details>

<details>
<summary>5. 実機チェック</summary>

deprecation ログがゼロになるまで**フェーズ 4〜5 を反復**（2・3 は一度で完了）。
`manual-fix.md` を片手に 9 種を炙る（`@` 抑制は巡回前に外す＝でないとログが嘘になる）。

**方法は導線の種類に依らず同じ三手**:
1. **ログを消す**（巡回ごとに新規分だけ見る）
2. **全入口を駆動する** ＝ 入口の種類に応じた手段で叩く。**「導線＝画面」とは限らない**:

   | 入口の種類 | 駆動手段 |
   |---|---|
   | 画面（管理UI・帳票） | ブラウザ巡回。一覧・検索・帳票・各ログイン種別・POST |
   | API エンドポイント | HTTP クライアントで全 EP（`curl` 等。本文は捨て**ステータス**だけ見れば十分） |
   | CLI / バッチ | `oil r <task>`（`FUEL_ENV` を忘れず） |

   各入口を **入力・認証・分岐のマトリクス**で叩いて経路を網羅する（例: 未認証 / 必須パラメータ欠落 / 不正値で外部到達 / **seed 済み値で DB 読込分岐**）。1 つの URL を 1 回叩くだけでは `?? ''` ガードのない分岐に入らない。
3. **ログをゼロにする**:
   ```bash
   grep -irE 'deprecated|warning' fuel/app/logs/   # ゼロになるまで 4〜5 を反復
   ```

> **外部依存で踏めない最深経路は正直に切り分ける。** 外部 API の応答を変換する層など、サンドボックス実データ/フィクスチャが無いと入れない経路は「未到達」と明記し、本番相当データのステージング検証に回す。「巡回したのに 0 件」と「そもそも入れていない」を混同しない。
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
`php` 制約は **`composer.lock` の content-hash に含まれる**ため、変更後は lock を同期しないと
`composer validate` が「lock is not up to date」で落ちる。**版は変えずハッシュだけ**更新する:
```bash
composer update --lock   # パッケージ版は不変・content-hash のみ再生成
```
</details>
