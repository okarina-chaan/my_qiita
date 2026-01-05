---
title: RailsのAPIテストで422エラー？CSRF保護が原因だった話
tags:
  - Rails
private: false
updated_at: '2026-01-05T10:44:54+09:00'
id: 2d57faae82c9f4aa5e61
organization_url_name: null
slide: false
ignorePublish: false
---
## はじめに
- この記事で分かること
  - CSRFトークンの重要性
  - 開発環境とテスト環境の使い分けの仕方

## 前提
本記事では以下の環境を想定しています。

- Rails 8.0
- RSpec 3.13
- Docker環境
- 認証方式：LINE Login（セッションベース）
- フロントエンド：React
- 実装した機能：日記アプリにて、外部API（OpenAI）と通信して週次振り返りを生成

### アプリケーション構成
- バックエンド：Rails APIコントローラー
- フロントエンド：ReactでボタンコンポーネントからAPI呼び出し
- 本番環境ではaxiosでCSRFトークンを送信済み

## 遭遇した問題
#### テストを書いたら422エラーが出た
weekly_insight機能のテストを書き始める。
まずは「未認証の場合は401エラーを返す」という基本的なケースから。
```ruby
context "未認証のとき" do
  before do
    # current_userがnilを返すようにモック
    allow_any_instance_of(Api::WeeklyInsightsController)
      .to receive(:current_user)
      .and_return(nil)
  end
  
  it "エラーコード401を返す" do
    post "/api/weekly_insights"
    expect(response).to have_http_status(:unauthorized)
  end
end
```
シンプルなテスト。
テストを走らせると・・・
```bash
  1) Api::WeeklyInsightsController POST /api/weekly_insights 未認証のとき エラーコード401を返す
     Failure/Error: expect(response).to have_http_status(:unauthorized)
       expected the response to have status code :unauthorized (401) but it was :unprocessable_content (422)
```
↑401って書いてあるけど、422が返ってきてるよと怒られている。

#### エラーメッセージ: `ActionController::InvalidAuthenticityToken`
上記コードに`puts response.body`を追記してレスポンスを見てみる。
```ruby
it "エラーコード401を返す" do
  post "/api/weekly_insights"
  
  puts response.body  # ← 追加
  
  expect(response).to have_http_status(:unauthorized)
end
```

```html


  ...
  ActionController
  ::
  InvalidAuthenticityToken
  ...

```
開発環境で使っていたはずのBetter ErrorsがCSRF保護エラーをキャッチして、
HTMLのエラー画面を返していた。
つまり、テスト環境だと思っていたら開発環境だった。

## なぜテストで問題になるのか
#### 本番環境とテスト環境の違い
本番環境では、React側でaxiosを使ってCSRFトークンを送信していた。：
```javascript
// Reactのaxios設定（本番）
axios.defaults.headers.common['X-CSRF-Token'] = 
  document.querySelector('meta[name="csrf-token"]').content;
```

これで本番は問題なく動作していた。

#### RSpecではCSRFトークンが自動生成されない
しかし、RSpecのリクエストテストではブラウザを模した挙動をしないため、
CSRFトークンの自動付与や検証フローが省略される。

- CSRFトークンは自動生成されない
- `post "/api/weekly_insights"` だけではトークンなし
- Railsのデフォルト設定ではCSRF保護が有効
- → `ActionController::InvalidAuthenticityToken` が発生

つまり、**テスト環境でCSRF保護をどう扱うか**を決める必要がある。

## 解決方法の比較
解決方法をいくつか調べてみた。
### 方法1: テスト環境でCSRF保護を無効化（推奨）
※ system spec では無効化しない
```ruby
# config/environments/test.rb
config.action_controller.allow_forgery_protection = false
```
- いつ使うべきか
これはCSRF保護の検証をしない設定になるので、
あくまでもユニットテストでのみ使いたい。

### 方法2: APIコントローラーでCSRF保護をスキップ
```ruby
class Api::WeeklyInsightsController < ApplicationController
  skip_before_action :verify_authenticity_token
end
```
**いつ使うべきか**

この方法が適切なケース：
- トークンベース認証（JWT、API Keyなど）を使っている
- 外部アプリ（モバイルアプリ等）からのAPI呼び出し

**今回は使わない理由**

本アプリはセッションベース認証（LINE Login + Cookie）を使用している。
この場合、ブラウザから直接APIを呼ぶため、CSRF攻撃のリスクがあるため、対策する必要がある。

### 方法3: テストでCSRFトークンを送る（非推奨）
```ruby
it "新しいインサイトを生成する" do
  # 実際には hidden_field から取得する必要があるが、ここでは簡略的に
  get "/some_page"
  csrf_token = response.headers['X-CSRF-Token']
  
  # そのトークンを使ってPOST
  post "/api/weekly_insights",
    headers: { 'X-CSRF-Token' => csrf_token }
  
  expect(response).to have_http_status(:created)
end
```

**なぜ推奨しないのか**

- 実装が複雑になる
- テストコードが冗長になる
- `allow_any_instance_of` でモックしている時点で、実際の認証フローを飛ばしてしまっている
- CSRF保護の検証は、モックを使わない統合テストで行うべき

つまり、**ユニットテストでCSRF保護を検証する意味がない**ため、
テスト環境で無効化する方が合理的である。

## 採用した解決方法

`config/environments/test.rb`にて、以下の設定を追記した。
```ruby
config.action_controller.allow_forgery_protection = false
```

## Docker環境での追加のハマりポイント
最初はテストを実行してもエラーが変わらず、沼にハマっていた。
デバッグしてみると...
```ruby
puts "Rails.env: #{Rails.env}"
# => development  # ← テスト環境のはずが開発環境
```

今回は開発環境ではCSRF保護しているが、テスト環境ではしていない。
そのため、テストを走らせるときには確実にテスト環境で行うように
Docker環境で指定する必要がある。

`compose.yml`
```ruby
environment:
  RAILS_ENV: ${RAILS_ENV:-development}
  PORT: 3000
```
このように環境変数を柔軟に設定した。
これでデフォルトでは`development`を設定し、
テストを走らせるときに`RAILS_ENV=`として環境変数を渡すことができる。
```bash
# 開発サーバー起動
docker compose up

# テスト実行
RAILS_ENV=test docker compose exec web bundle exec rspec
```

## まとめ
### 学んだこと

1. **CSRF保護は重要だが、テスト環境では柔軟に対応すべき**
   - 本番：CSRF保護は必須（特にセッションベース認証）
   - テスト：実用性を優先して無効化してOK

2. **認証方式によって対応が変わる**
   - セッションベース → 本番でCSRF保護必要
   - トークンベース → CSRF保護不要な場合が多い

3. **Docker環境では環境変数の指定を忘れずに**
   - `RAILS_ENV=test` を明示的に指定
   - `docker-compose.yml` で柔軟な設定にしておく

4. **セキュリティテストは適切な場所で**
   - ユニットテスト：ロジックのテスト（CSRF保護は無効化）
   - 統合テスト：実際の認証フロー全体をテスト（CSRF保護を有効化）

### この記事が役立つ人

- Rails + RSpec でAPIテストを書いている
- Docker環境で開発している
- CSRF関連のエラーで困っている
- セッションベース認証を使っている

## 【補足】SameSite属性について

最近のRailsアプリでは、セッションCookieに `SameSite=Lax` が設定されている。

これにより、他のサイトからのPOSTリクエストではCookieが送られないため、
**CSRF攻撃のリスクが大幅に減少**している。
```ruby
# Rails 7以降のデフォルト
Rails.application.config.session_store :cookie_store,
  same_site: :lax
```

### それでもCSRFトークンを使う理由

1. **古いブラウザ対応**（IE11など）
2. **多層防御**（複数の対策を組み合わせる）
3. **Railsのデフォルト動作**（標準で有効）

本番環境ではSameSiteとCSRFトークンの**両方**で守られているため、
テスト環境では効率性を優先してCSRF保護を無効化しても問題ないと判断した。

## 参考資料
- [Rails Guides - セキュリティガイド](https://railsguides.jp/security.html#csrf%E3%81%B8%E3%81%AE%E5%AF%BE%E5%BF%9C%E7%AD%96)
- [RSpec Rails - Request Specs](https://rspec.info/features/6-0/rspec-rails/request-specs/)
- [Better Errors](https://github.com/BetterErrors/better_errors)
- [Rails で CSRF トークンの検証を制御する](https://qiita.com/kurashita/items/d1c8f6d79daec89c368c)
- [Web API の CSRF 対策まとめ【追記あり】](https://qiita.com/okamoai/items/044c03680766f0609d41)
- [令和時代の API 実装のベースプラクティスと CSRF 対策](https://blog.jxck.io/entries/2024-04-26/csrf.html)
- [使えるRSpec入門・その3「ゼロからわかるモック（mock）を使ったテストの書き方」](https://qiita.com/jnchito/items/640f17e124ab263a54dd)
