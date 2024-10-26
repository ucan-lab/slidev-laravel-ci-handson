---
# You can also start simply with 'default'
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: bg.png
# some information about your slides (markdown enabled)
title: Laravelプロジェクトで学ぶ！GitHub ActionsによるCIの実践
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
# apply unocss classes to the current slide
class: text-center
# https://sli.dev/features/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
transition: slide-left
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
# take snapshot for each slide in the overview
overviewSnapshots: true
fonts:
  # basically the text
  sans: 'Helvetica Neue,Robot'
  # use with `font-serif` css class from windicss
  local: 'Helvetica Neue'
  # for code blocks, inline code, etc.
  mono: 'Fira Code'
---

# Laravelプロジェクトで学ぶ！GitHub ActionsでCI実践！

#ミライトデザイン #ペチオブ / ucan

---

# 自己紹介

- ucan / ゆうきゃん
  - X → https://x.com/ucan_lab
  - Qiita → https://qiita.com/ucan-lab
- 1988年生まれ(0x22歳) 長崎県西海市出身
- 2010/04 〜 エンジニア(4社目)
- ミライトデザイン所属

---
layout: center
---

# 趣味: HADO(ARスポーツ)

<p><img src="/hado.jpg" class="h-100"></p>

---

# この動画の目的

- GitHub Actionsを使ったCI/CDパイプラインの設定
- Laravelプロジェクトを例にした実践的な使い方
- GitHub ActionsのCIワークフロー構成や最適化手法
  - CDの実践例は今回は紹介しない

---
layout: two-cols
---

# GitHub Actionsとは

- CI/CD(継続的インテグレーション/継続的デプロイ)
- ワークフロー: ビルド、テスト、デプロイ等の処理
  - イベント: PRやIssueのオープン、push等
  - ランナー: 仮想マシンのOS
  - ジョブ: 順次または並列で実行
  - ステップ: アクションまたはスクリプト
- 料金
  - パブリックリポジトリは無料
  - プライベートリポジトリは無料枠
    - 2,000分/月(Free)
    - 3,000分/月(Pro, Team)
    - 超えるとワークフローが実行されない

::right::

<p><img src="/overview-actions-simple.png" class="h-40"></P>

---
layout: two-cols
---

### Hello World ワークフロー

- name: 任意のワークフロー名を指定
- on: ワークフローを実行するイベント
  - workflow_dispatch: GitHub上から手動実行
- jobs: ジョブは複数定義できます
  - say-hello: 任意のジョブ名を指定
    - runs-on: 実行するランナー(OS)を指定
    - steps: 具体的な処理を記述する
      - run: 実行したいコマンド

::right::

### .github/workflows/hello-world.yaml

```yaml
name: Hello World
on:
  workflow_dispatch:
jobs:
  say-hello:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Hello World!"
```

---
layout: center
---

## say-hello ワークフロー①

<p><img src="/say-hello1.png" class="h-100"></p>

---
layout: center
---

## say-hello ワークフロー②

<p><img src="/say-hello2.png" class="h-100"></p>

---
layout: center
---

## say-hello ワークフロー③

<p><img src="/say-hello3.png" class="h-100"></p>

---
layout: center
---

## say-hello ワークフロー④

<p><img src="/say-hello4.png" class="h-100"></p>

---
layout: cover
---

# 実践: Laravel Sail でCI

実際に使っているLaravel Sailで作ったプロジェクトでCI設定をご紹介

---

## .github/workflows/ci.yaml

```yaml {*}{maxHeight: '400px'}
name: Continuous Integration
on:
  push:
    branches:
      - 'main'
  pull_request:
    branches:
      - 'main'
  workflow_dispatch:
env:
  DB_CONNECTION: mysql
  DB_HOST: 127.0.0.1
  DB_PORT: 3306
  DB_DATABASE: laravel
  DB_USERNAME: sail
  DB_PASSWORD: password
jobs:
  ci-backend:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    services:
      mysql:
        image: mysql/mysql-server:8.0
        ports:
          - 3306:3306
        env:
          MYSQL_DATABASE: ${{ env.DB_DATABASE }}
          MYSQL_USER: ${{ env.DB_USERNAME }}
          MYSQL_PASSWORD: ${{ env.DB_PASSWORD }}
        options: >-
          --health-cmd "mysqladmin ping"
          --health-start-period 30s
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - name: Setup PHP with composer v2
        uses: shivammathur/setup-php@v2
        with:
          php-version: 8.3
          tools: composer:v2
      - name: Cache Vendor
        id: cache-vendor
        uses: actions/cache@v4
        with:
          path: ./vendor
          key: ${{ runner.os }}-composer-${{ hashFiles('**/composer.lock') }}
          restore-keys: ${{ runner.os }}-composer-
      - name: Install Dependencies
        if: steps.cache-vendor.outputs.cache-hit != 'true'
        run: composer install --quiet --prefer-dist --no-progress --no-interaction --no-scripts --no-ansi
      - name: Composer Validate
        run: composer validate
      - name: Laravel Setting
        run: |
          cp .env.example .env
          php artisan optimize
          git config --local core.fileMode false
          chmod -R 777 storage bootstrap/cache
      - name: PHP Version
        run: php --version
      - name: Composer Version
        run: composer --version
      - name: Laravel Version
        run: php artisan --version
      - name: Run Migrate
        run: php artisan migrate
      - name: Run Migrate Refresh
        run: php artisan migrate:refresh
      - name: Run Seeding
        run: php artisan db:seed
      - name: Run IDE Helper Models
        run: |
          php artisan ide-helper:models --write --reset
          ./vendor/bin/pint app/Models
          if ! git diff --exit-code; then
            echo "Error: The phpdoc for the model ide-helper is not updated!"
            echo "Run: php artisan ide-helper:models --write --reset"
            exit 1
          fi
      - name: Cache Pint
        uses: actions/cache@v4
        with:
          path: ./.pint.cache
          key: ${{ runner.os }}-pint-${{ hashFiles('**/composer.lock') }}
          restore-keys: ${{ runner.os }}-pint-
      - name: Run Pint
        run: ./vendor/bin/pint --test
      - name: Cache Rector
        uses: actions/cache@v4
        with:
          path: ./storage/rector/cache
          key: ${{ runner.os }}-rector-${{ hashFiles('**/composer.lock') }}
          restore-keys: ${{ runner.os }}-rector-
      - name: Run Rector
        run: ./vendor/bin/rector process --dry-run
      - name: Cache PHPStan
        uses: actions/cache@v4
        with:
          path: ./storage/phpstan
          key: ${{ runner.os }}-phpstan-${{ hashFiles('**/composer.lock') }}
          restore-keys: ${{ runner.os }}-phpstan-
      - name: Run PHPStan
        run: ./vendor/bin/phpstan analyze
      - name: Cache Pest
        uses: actions/cache@v4
        with:
          path: ./storage/pest/cache
          key: ${{ runner.os }}-pest-${{ hashFiles('**/composer.lock') }}
          restore-keys: ${{ runner.os }}-pest-
      - name: Run Pest
        env:
          SESSION_DRIVER: array
          DB_CONNECTION: sqlite
          DB_DATABASE: ":memory:"
        run: |
          php artisan config:clear
          ./vendor/bin/pest --parallel --cache-directory storage/pest/cache
  ci-frontend:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with:
          version: 9
          run_install: false
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '>=20.17.0'
          cache: 'pnpm'
      - name: Install Dependencies
        run: pnpm install
      - name: Run Build
        run: pnpm run build
```

---
layout: two-cols
---

### イベントと環境変数

- on.push: 指定ブランチにコミット
- on.pull_request: 指定ブランチへのプルリクエスト
- env: サーバー環境変数
  - LaravelからCIのDBに繋ぐための接続設定

::right::

### .github/workflows/ci.yaml

```yaml
name: Continuous Integration
on:
  push:
    branches:
      - 'main'
  pull_request:
    branches:
      - 'main'
  workflow_dispatch:
env:
  DB_CONNECTION: mysql
  DB_HOST: 127.0.0.1
  DB_PORT: 3306
  DB_DATABASE: laravel
  DB_USERNAME: sail
  DB_PASSWORD: password
jobs:
# ...
```

---
layout: two-cols
---

### ジョブを分けて並列化しよう！

- jobs: ci-backend, ci-frontend とジョブを分割

依存関係のないものはジョブを分けて並列で実行して高速化する

::right::

### .github/workflows/ci.yaml

```yaml
jobs:
  ci-backend:
    # ...
  ci-frontend:
    # ...
```

---
layout: two-cols
---

### ci-backend ジョブ

- runs-on: 実行するランナー(OS)を指定
- timeout-minutes: デフォルトは6時間
- services: 使い捨てのDBを使ってテスト
  - ports: Ubuntu上でphpが実行されるため
  - env: DBの接続情報を設定
  - options: Dockerのヘルスチェック設定
    - `>-`: YAMLの折りたたみブロックスカラー
      - 複数行のテキストを1行に結合する

::right::

### .github/workflows/ci.yaml

```yaml
ci-backend:
  runs-on: ubuntu-latest
  timeout-minutes: 20
  services:
    mysql:
      image: mysql/mysql-server:8.0
      ports:
        - 3306:3306
      env:
        MYSQL_DATABASE: ${{ env.DB_DATABASE }}
        MYSQL_USER: ${{ env.DB_USERNAME }}
        MYSQL_PASSWORD: ${{ env.DB_PASSWORD }}
      options: >-
        --health-cmd "mysqladmin ping"
        --health-start-period 30s
        --health-interval 10s
        --health-timeout 5s
        --health-retries 5
```

---
layout: two-cols
---

### ci-backend ステップ

- uses: 特定のアクションを再利用
  - https://github.com/marketplace?type=actions
  - actions/checkout@v4
    - `actions/checkout` のバージョン `v4` を使用
    - リポジトリのコードをチェックアウト
    - GitHub公式アクションは `actions/` の名前空間
  - shivammathur/setup-php
    - PHP環境のセットアップ
    - プロジェクトのPHPバージョンに合わせる
    - Sailのコンテナをビルドするより高速

::right::

### .github/workflows/ci.yaml

```yaml
steps:
  - uses: actions/checkout@v4
  - name: Setup PHP with composer v2
    uses: shivammathur/setup-php@v2
    with:
      php-version: 8.3
      tools: composer:v2
```

---
layout: two-cols
---

### ci-backend ステップ

- id: cache-vendor
  - ステップの実行結果にアクセスするためのid
- uses: actions/cache@v4
  - 依存パッケージやビルド結果を保持して、実行結果を削減するキャッシュ
  - path: キャッシュ対象のディレクトリ
  - key: キャッシュの保存と検索に利用されるキー
  - restore-keys: key でヒットしなかった時に前方一致でヒットするキーを検索

::right::

### .github/workflows/ci.yaml

```yaml
- name: Cache Vendor
  id: cache-vendor
  uses: actions/cache@v4
  with:
    path: ./vendor
    key: ${{ runner.os }}-composer-${{ hashFiles('**/composer.lock') }}
    restore-keys: ${{ runner.os }}-composer-
- name: Install Dependencies
  if: steps.cache-vendor.outputs.cache-hit != 'true'
  run: composer install --quiet --prefer-dist --no-progress --no-interaction --no-scripts --no-ansi
- name: Composer Validate
  run: composer validate
```

---
layout: two-cols
---

### ci-backend ステップ

- if: キャッシュがあった場合、composer install を実行しない
- composer install オプション
  - `--quiet`: 通常のログ出力を非表示
  - `--prefer-dist`: zipでダウンロード(通信量が減る)
  - `--no-progress`: 進捗バーを非表示
  - `--no-interaction`: 対話式プロンプトを無効
  - `--no-scripts`: Composerスクリプトを無効
  - `--no-ansi`: 色付きの出力を無効
- composer validate
  - `composer.json`, `composer.lock` の検証

::right::

### .github/workflows/ci.yaml

```yaml
- name: Cache Vendor
  id: cache-vendor
  uses: actions/cache@v4
  with:
    path: ./vendor
    key: ${{ runner.os }}-composer-${{ hashFiles('**/composer.lock') }}
    restore-keys: ${{ runner.os }}-composer-
- name: Install Dependencies
  if: steps.cache-vendor.outputs.cache-hit != 'true'
  run: composer install --quiet --prefer-dist --no-progress --no-interaction --no-scripts --no-ansi
- name: Composer Validate
  run: composer validate
```

---
layout: two-cols
---

### ci-backend ステップ

- 環境変数ファイルのコピー
- キャッシュコマンドが成功するか
- パーミッション設定
- パーミッションの変更を差分表示しない設定
  - 後述のIDEヘルパー差分チェックに引っかからないようにするため
- 各種バージョン表示

::right::

### .github/workflows/ci.yaml

```yaml
- name: Laravel Setting
  run: |
    cp .env.example .env
    php artisan optimize
    chmod -R 777 storage bootstrap/cache
    git config --local core.fileMode false
- name: PHP Version
  run: php --version
- name: Composer Version
  run: composer --version
- name: Laravel Version
  run: php artisan --version
```

---
layout: two-cols
---

### ci-backend ステップ

- マイグレーションテスト
- ロールバックテスト
- シーディングテスト
- IDEヘルパー差分チェック
  - `barryvdh/laravel-ide-helper`
    - フレームワークのコードをIDEで補完してくれるライブラリ
  - `--write --reset`: モデルクラスのPHPDocへ記載
    - `_ide_helper_models.php` 別ファイルに書き出せるが、ジャンプする時に邪魔になる

::right::

### .github/workflows/ci.yaml

```yaml
- name: Run Migrate
  run: php artisan migrate
- name: Run Migrate Refresh
  run: php artisan migrate:refresh
- name: Run Seeding
  run: php artisan db:seed
- name: Run IDE Helper Models
  run: |
    php artisan ide-helper:models --write --reset
    ./vendor/bin/pint app/Models
    if ! git diff --exit-code; then
      echo "Error: The phpdoc for the model ide-helper is not updated!"
      echo "Run: php artisan ide-helper:models --write --reset"
      exit 1
    fi
```

---
layout: two-cols
---

### ci-backend ステップ

- Pint: コードフォーマッター
  - php-cs-fixerのラッパーライブラリ
  - PHPコードのスタイルや書式を統一
- Rector: アップグレード&リファクタリングツール
  - 冗長な部分を削除、古い記法のリファクタリング
  - 非推奨の機能や互換性のないコードの変更
  - フレームワークの新しいバージョンに合わせて変更を適用

::right::

### .github/workflows/ci.yaml

```yaml
- name: Cache Pint
  uses: actions/cache@v4
  with:
    path: ./.pint.cache
    key: ${{ runner.os }}-pint-${{ hashFiles('**/composer.lock') }}
    restore-keys: ${{ runner.os }}-pint-
- name: Run Pint
  run: ./vendor/bin/pint --test
- name: Cache Rector
  uses: actions/cache@v4
  with:
    path: ./storage/rector/cache
    key: ${{ runner.os }}-rector-${{ hashFiles('**/composer.lock') }}
    restore-keys: ${{ runner.os }}-rector-
- name: Run Rector
  run: ./vendor/bin/rector process --dry-run
```

---
layout: two-cols
---

### ci-backend ステップ

- PHPStan: PHPの静的解析ツール
  - 型の不一致やデッドコード等をチェック
  - PHPStanの拡張のlarastan/larastanツール
    - Laravel特有のコードを解釈
- Pest: PHPのテストフレームワーク
  - PHPUnitをベースでシンプルな記述が可能
  - 並列テストが標準オプションで搭載
  - env
    - SESSION_DRIVER: インメモリで高速
    - DB_CONNECTION: SQLiteメモリ内DBで高速
    - 本番とDBが異なるとカバー範囲が限定
    - 統合テストでは実際のDBで使い分けが必要

::right::

### .github/workflows/ci.yaml

```yaml
- name: Cache PHPStan
  uses: actions/cache@v4
  with:
    path: ./storage/phpstan
    key: ${{ runner.os }}-phpstan-${{ hashFiles('**/composer.lock') }}
    restore-keys: ${{ runner.os }}-phpstan-
- name: Run PHPStan
  run: ./vendor/bin/phpstan analyze
- name: Cache Pest
  uses: actions/cache@v4
  with:
    path: ./storage/pest/cache
    key: ${{ runner.os }}-pest-${{ hashFiles('**/composer.lock') }}
    restore-keys: ${{ runner.os }}-pest-
- name: Run Pest
  env:
    SESSION_DRIVER: array
    DB_CONNECTION: sqlite
    DB_DATABASE: ":memory:"
  run: |
    php artisan config:clear
    ./vendor/bin/pest --parallel --cache-directory storage/pest/cache
```

---
layout: two-cols
---

### ci-frontend ステップ

- uses: pnpm/action-setup@v4
  - pnpm はperformant npmの略。Node.jsのパッケージマネージャ
  - npm, yarn, bun の選択肢がある
  - pnpm は高速、ディスクスペース、厳格さで優位
- uses: actions/setup-node@v4
  - Node.jsのセットアップ
  - cache設定は pnpm/action-setup 公式参考

::right::

### .github/workflows/ci.yaml

```yaml
ci-frontend:
  runs-on: ubuntu-latest
  timeout-minutes: 10
  steps:
    - uses: actions/checkout@v4
    - uses: pnpm/action-setup@v4
      with:
        version: 9
        run_install: false
    - name: Setup Node
      uses: actions/setup-node@v4
      with:
        node-version: '>=20.17.0'
        cache: 'pnpm'
    - name: Install Dependencies
      run: pnpm install
    - name: Run Build
      run: pnpm run build
```

---
layout: center
---

## ci ワークフロー①

<p><img src="/ci1.png" class="h-100"></p>

---
layout: center
---

## ci ワークフロー②

<p><img src="/ci2.png" class="h-100"></p>

---
layout: center
---

## ci ワークフロー③

<p><img src="/ci3.png" class="h-100"></p>

---
layout: center
---

## ci ワークフロー④

<p><img src="/ci4.png" class="h-100"></p>

---
layout: cover
---

# おしまい

### チャンネル登録&高評価よろしくお願いします！
