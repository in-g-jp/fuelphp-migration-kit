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
| 業務データ | 全導線を実データで踏む | 下記いずれか。**PII なし**・マスタ系と業務系でファイル分割 |

業務データは**アプリ型で手段が変わる**:

| アプリ型 | 「全導線を踏む」ための前提（手順5のゲート） |
|---|---|
| 自己完結（管理画面・帳票など内部ロジック中心） | **合成 seed**（DB に投入して一覧・検索・帳票・POST を巡回） |
| 連携 / プロキシ（外部 API を叩く同期 API 等） | **外部 API の到達性 or 記録済みフィクスチャ**。seed だけでは深い経路（外部レスポンスの変換層）に入れない。サンドボックスが叩けるなら実 ID、無ければ XML/JSON レスポンスを録って差し替える |

→ つまり「seed を作る」が常にゲートとは限らない。**自分のアプリの最深リスク経路が何の入力で発火するか**を先に見極める。

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
| `php -l` | 構文・削除構文 | 波括弧オフセット等 |
| PHPCompatibility | 非互換を列挙 | sniff は ~8.1 上限（8.2+ は `php -l` で補完） |
| Rector dry-run | 自動修正候補 | **適用は人が判定してから** |
| PHPStan / Psalm | 型・未定義・データフロー | アプリ層限定・baseline 運用 |

```bash
# 構文
find fuel/app -name '*.php' -print0 | xargs -0 -n1 php -l | grep -v 'No syntax'
# 非互換 sniff
phpcs --standard=PHPCompatibility --runtime-set testVersion 8.5 fuel/app
# 自動修正候補（適用前にレビュー）
vendor/bin/rector process fuel/app --dry-run
```

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

合成 seed で全導線を踏み、deprecation ログがゼロになるまで**フェーズ 4〜5 を反復**（2・3 は一度で完了）。<br>
`manual-fix.md` を片手に 9 種を炙る（`@` 抑制は巡回前に外す＝でないとログが嘘になる）。

```bash
# ゼロになるまで 4〜5 を繰り返す
grep -irE 'deprecated|warning' fuel/app/logs/
```
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
</details>
