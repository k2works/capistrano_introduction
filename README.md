Capistrano入門
===
# 目的
Capistranoバージョン3系の操作方法を修得する。  
[公式サイトの解説](http://capistranorb.com/documentation/overview/what-is-capistrano/)をざっくり超訳。

# 前提
| ソフトウェア     | バージョン    | 備考         |
|:---------------|:-------------|:------------|
| OS X           |10.8.5        |             |
| Capistrano　　　| 3            |             |
| Vagrant  　　 　| 1.6.0        |             |
| Ubuntu   　　 　| 14.04LTS     |             |

# 構成
+ [環境セットアップ](#0)
+ [はじめに](#1)
+ [進んだ機能](#2)
+ [フレームワーク拡張](#3)

# 詳細
## <a name="0">環境セットアップ</a>
### クックブック作成
```bash
$ knife cookbook create ops -o .
WARNING: No knife configuration file found
** Creating cookbook ops
** Creating README for cookbook: ops
** Creating CHANGELOG for cookbook: ops
** Creating metadata for cookbook: ops
** Creating specs for cookbook: ops
$ cd ops
$ berks init .
      create  Berksfile
      create  Thorfile
      create  chefignore
      create  .gitignore
         run  git init from "."
      create  Gemfile
      create  .kitchen.yml
      append  Thorfile
      create  test/integration/default
      append  .gitignore
      append  .gitignore
      append  Gemfile
      append  Gemfile
You must run `bundle install' to fetch any new gems.
      create  Vagrantfile
Successfully initialized
$ bundle
```
### プロビジョニング
#### レシピ作成
_ops/Berksfile_  
_ops/recipes/default.rb_  
_ops/attributes/default.rb_  
_ops/templates/default/nginx.conf.erb_

#### Vagrantfile編集
_Vagrantfile_
```ruby
・・・
config.vm.provision :shell, inline: "apt-get update"
・・・
config.vm.network "private_network", ip: "192.168.33.10"
・・・
config.vm.provision :chef_solo do |chef|
  chef_gem_path    = "/opt/chef/embedded/lib/ruby/gems/1.9.1"
  chef.binary_env  = "GEM_PATH=#{chef_gem_path} GEM_HOME=#{chef_gem_path}"
  chef.binary_path = "/opt/chef/bin"

   chef.json = {
        nginx: {
          env: ["ruby"]
        }
      }

  chef.run_list = [
      "recipe[rvm::vagrant]",
      "recipe[rvm::system]",
      "recipe[ops::default]"
  ]
end
・・・
```
#### プロビジョニング実行
```bash
$ vagrant up --provision
```
#### 動作確認
_http://192.168.33.10_にアクセスできたらOK
```bash
$ git commit -am "環境セットアップ"
$ cd ..
```

## <a name="1">はじめに</a>
### Capistranoとは
Capistranoはリモートサーバ自動化ツール

+ 一度にたくさんのマシンにwebアプリケーションを安定してデプロイ
+ 多くのマシンの認証を自動化
+ SSH経由で任意のスクリプトを実行
+ ソフトウェアチームの共通タスクを自動化
+ chef-soloやAnsibleなどインフラプロビジョニングツールを駆動する

その他には

+ 出力フォーマット互換
+ 他のソースコード管理ソフトの簡単な追加
+ 対話的にCapistranoを実行する簡単なマルチコンソール
+ ホストとロールで部分的デプロイまたは部分的クラスターマシンをフィルターする
+ Railsのアセットパイプラインとデータベースマイグレーション用レシピ
+ 複雑な環境のサポート
+ 健全で表現力に富んだAPI

### インストール
```bash
$ gem install capistrano
```
Railsの場合はGemfileに以下を追加する
```
group :development do
  gem 'capistrano-rails', '~> 1.1.1'
end
```
### アプリケーション準備
1. Railsアプリケーションの作成
```bash
$ rails new dev
$ cd dev
$ bundle
```
以下のコメントを解除してbundle  
_dev/Gemfile_
```ruby
gem 'therubyracer',  platforms: :ruby
gem 'unicorn'
gem 'capistrano-rails', group: :development
```

1. 外部の利用可能なソースコード管理サービスにアプリケーションをコミットする
```bash
$ git add .
$ git commit -am "アプリケーションの準備"
$ git push origin master
$ cd dev
```

1. レポジトリから秘密を取り除く
```bash
$ cp config/database.yml{,.example}
$ echo config/database.yml >> .gitignore
```

1. アプリケーション用Capistranoの初期化
```bash
$ cap install
mkdir -p config/deploy
create config/deploy.rb
create config/deploy/staging.rb
create config/deploy/production.rb
mkdir -p lib/capistrano/tasks
Capified
```

1. 生成されたファイルにサーバーアドレスの設定をする  
_dev/config/deploy/staging.rb_
```ruby
role :app, %w{vagrant@192.168.33.10}
role :web, %w{vagrant@192.168.33.10}
role :db,  %w{vagrant@192.168.33.10}
・・・
server '192.168.33.10', user: 'vagrant', roles: %w{web app}, my_property: :my_value
・・・
```

1. **deploy.rb**に共通情報を設定する
_dev/config/deploy.rb_
```ruby
set :application, 'capistrano_introduction'
set :repo_url, 'git@github.com:k2works/capistrano_introduction.git'
```

### 認証と委任
### コールドスタート
### フロー
### ロールバック

## <a name="2">進んだ機能</a>
## <a name="3">フレームワーク拡張</a>


# 参照
+ [Capistrano](http://capistranorb.com/documentation/overview/what-is-capistrano/)
+ [capistrano/capistrano](https://github.com/capistrano/capistrano)
+ [VAGRANT](http://www.vagrantup.com/)
