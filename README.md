# FuelPHP 1.8 + PHP 7.x → PHP 8.5 移行キット

FuelPHP 1.8.2 / PHP 7.x を **PHP 8.5** へ上げる汎用手順。方針は 3 つ:

- **8.5 一本** — 途中版を経由しない
- **3 段で確かめる** — 機械検知 → 文脈判定 → 実機 + ログ
- **修正は `fuel/app` のみ** — core / packages / vendor は検知だけ、直すなら fork

## 手順（0〜6）

`<summary>` で流れ、開いて詳細。

<details>
<summary>0. 検証環境を先に作る</summary>

### PHP 8.5 Docker 環境
社内の参考環境:
- [apuro](https://nextstep.backlog.jp/git/APURO/apuro/tree/migration/php85/)
- [apuro_api](https://nextstep.backlog.jp/git/APURO/apuro_api/tree/migration/php85/)
- [ssdevelop](https://nextstep.backlog.jp/git/SS/ssdevelop/tree/main)

### deprecation を致命化させずログ化
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

### 実行時ディレクトリを用意
`fuel/app/storage/{cache,log,sync}` 等が無いと起動時 fatal。`.gitkeep` で追跡し fresh clone でも存在させる（`locale` 未インストールも同様）。

### 業務データ（seed）を投入
PII なし・マスタ系と業務系で分割し、ここで投入まで済ませる。作り方はアプリ型で変わる（手順5のゲート）:

| アプリ型 | 前提 |
|---|---|
| 自己完結（管理画面・帳票） | 合成 seed を DB 投入 |
| 連携 / プロキシ（外部 API 同期等） | 外部 API 到達 or 記録フィクスチャ。seed だけでは変換層に入れない |

### 詰まりどころ
- 検証 DB を migrations で組むなら**本番と列が一致するか先に確認**。列欠落だと 8.5 は PDO 既定 EXCEPTION で 500 になる。
- oil の CLI は `--env` でなく **`FUEL_ENV=docker php oil …`**（`--env` は別 host 接続にフォールバックする）。
</details>

<details>
<summary>1. composer / 依存を 8.5 対応</summary>

### 阻害要因を洗い出す
```bash
composer why-not php 8.5
```
制約が通る ≠ 動く。実際の壁は依存制約でなく framework のコードで、手順2/3 で出る。

### 依存の対応方針
| 項目 | 対応 |
|---|---|
| FuelPHP 本体 | 8.x 版なし → 共有 fork（PHP8 ブランチ）。案件ごとに作り直さない |
| ビルド再現性 | `composer.lock` を追跡して commit 固定 |
| 周辺ライブラリ | `composer/installers ^2` / `twig ^3` / `phpspreadsheet` 等 |

### fork を repositories に配線
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

### 対象はアプリ層のみ
`fuel/app` だけに静的解析。既存指摘が多ければ baseline 化して移行起因に集中。使うツールは `require-dev` に入れる（Rector ^2 はスコープ配布で lock を汚さない）。

### ツール
| ツール | 役割 | 備考 |
|---|---|---|
| `php -l` | 構文・削除構文 | 波括弧オフセット・暗黙 nullable（`Type $x = null`）等を確実に取れる |
| Rector dry-run | 自動修正候補 | fuel/app 移行の主力。適用は人が判定してから |
| PHPCompatibility | 非互換を列挙 | ROI 低（sniff ~8.1 上限）。使うなら global/隔離で |
| PHPStan / Psalm | 型・未定義・データフロー | アプリ層限定・baseline 運用 |

### 構文チェック
```bash
# 暗黙 nullable もここで出る
find fuel/app -name '*.php' -print0 | xargs -0 -n1 php -l | grep -v 'No syntax'
```

### Rector は「全セット ON + 近代化を skip」
移行必須ルールだけ挙げる方式は設定済み汎用ルールを取りこぼす。全 ON にして cosmetic だけ外せば、残ったものが移行必須になる。
```php
// rector_migrate.php
return RectorConfig::configure()
    ->withPaths([__DIR__ . '/fuel/app', __DIR__ . '/htdocs'])  // DOCROOT も対象
    ->withPhpSets(php85: true)                                  // 単一指定（複数 true はエラー）
    ->withSkip([
        \Rector\Php54\Rector\Array_\LongArrayToShortArrayRector::class,  // array()→[]
        \Rector\Php80\Rector\NotIdentical\StrContainsRector::class,      // strpos→str_contains
        // … 近代化ルールを列挙
    ]);
```
```bash
php -d xdebug.mode=off vendor/bin/rector process --config rector_migrate.php --dry-run
```

### その場で直す（代表例）
```diff
  // 8.1: null→文字列関数
- strlen(Arr::get($data,'memo'))
+ strlen((string) Arr::get($data,'memo'))

  // 8.4: fputcsv の escape を明示
- fputcsv($fp, $line)
+ fputcsv($fp, $line, escape: '\\')

  // 8.5: array_key_exists の null キー非推奨
- array_key_exists($key, $arr)
+ array_key_exists((string) $key, $arr)
```
</details>

<details>
<summary>3. 公式 UPGRADING 突合</summary>

### 版固有の Removed/Deprecated を grep
手順 2 が拾わない版固有を `upgrading/UPGRADING-*.txt` を元に grep。sniff 上限の補完。
```bash
grep -rnE 'each\(|create_function\(|money_format\(' fuel/app   # 8.0 削除関数
grep -rnE '\$\{' fuel/app                                      # 8.2 ${} 文字列補間
grep -rnE 'utf8_encode|utf8_decode' fuel/app                   # 8.2 非推奨関数
grep -rnE 'curl_close\(|curl_multi_close\(|curl_share_close\(' fuel/app  # 8.5 no-op化で非推奨
```

### no-op 化で非推奨になった関数に注意
`curl_close`・`finfo_close`・`imagedestroy` 等は呼んでも動くので見落としやすい。手順2の全セットでも拾えるが grep で二重に押さえる。

### その場で直す（例）
```diff
  // 8.2: ${} 文字列補間 → {$...}
- "Hello ${name}"
+ "Hello {$name}"
```
</details>

<details>
<summary>4. ツールが取れない非互換</summary>

### 2・3 をすり抜ける非互換
実機ログ・操作・レビューでしか出ず、手順 5 で炙って直す。9 種の詳細は [`manual-fix.md`](manual-fix.md)。
</details>

<details>
<summary>5. 実機チェック</summary>

### ログがゼロになるまで反復
deprecation ログがゼロになるまで手順 4〜5 を反復（2・3 は一度で完了）。`manual-fix.md` を片手に 9 種を炙る。`@` 抑制は巡回前に外す（でないとログが嘘になる）。

### 全入口を駆動する
「導線＝画面」とは限らない。入口の種類に応じた手段で叩く:

| 入口の種類 | 駆動手段 |
|---|---|
| 画面（管理UI・帳票） | ブラウザ巡回。一覧・検索・帳票・各ログイン種別・POST |
| API エンドポイント | HTTP クライアントで全 EP（本文は捨て**ステータス**だけ） |
| CLI / バッチ | `oil r <task>`（`FUEL_ENV` を忘れず） |

各入口を**入力・認証・分岐のマトリクス**で叩く（未認証 / 必須パラメータ欠落 / 不正値 / seed 済み値での DB 読込分岐）。

### ログをゼロにする
```bash
grep -irE 'deprecated|warning' fuel/app/logs/   # ゼロになるまで 4〜5 を反復
```

### 直し先はログの発生元で決まる
`fuel/app/…` ならアプリ、`fuel/core`・`fuel/packages/…` なら fork を直す（アプリ側に迂回パッチを当てない）。fork 修正は手順6の commit ref 更新で全案件に反映。

### 実機は使う経路しか炙れない
網羅でなくカバレッジ依存。外部 API 変換層などフィクスチャ無しで入れない経路や未使用機能は「未到達」と明記し、本番相当データのステージング検証に回す。
</details>

<details>
<summary>6. 仕上げ・リリース</summary>

### リリース手順
1. fork の追加修正は commit ref 更新で取り込み（全案件共有）。
2. `composer.json` の `php` 制約を `>=8.5` に、CI も 8.5 固定。
3. PHPStan 最終確認 → 段階リリース。

### php 制約を上げる
```jsonc
// composer.json
"require": { "php": ">=8.5" }
```

### lock の content-hash を同期
`php` 制約は lock の content-hash に含まれる。版は変えずハッシュだけ更新（でないと `composer validate` が落ちる）:
```bash
composer update --lock
```
</details>
