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
デプロイ用ユーザーを追加する  
_ops/recipes/default.rb_
```ruby
group 'deploy' do
  group_name 'deploy'
  gid 999
  action :create
end

user 'deploy' do
  comment 'deploy user'
  group 'deploy'
  home '/home/deploy'
  shell '/bin/bash'
  supports :manage_home => true
  action :create
end
```

#### 認証
確認なしで認証をする必要のある場所が２箇所あります。

1. ワークステーション／ノートブック／その他からサーバー。**key agent**を使って生成した**SSH keys**を使います。
1. サーバーからレポジトリホスト。アプリケーションのコードをGitHubなどのサービスからチェックアウトできるようにします。これは通常**SSH agent forwading**,HTTP認証,またはデプロイキーを使って実行されます。

#### ワークステーションのSSH keysからサーバー
```bash
$ cd ops
$ ssh-keygen  -t rsa -C 'deploy@capistrano_introduction'
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/k2works/.ssh/id_rsa): files/default/deploy.authorized_keys
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in files/default/deploy.authorized_keys.
Your public key has been saved in files/default/deploy.authorized_keys.pub.
The key fingerprint is:
c4:b6:63:16:fe:39:e4:2b:19:58:ae:e3:5c:3c:77:06 deploy@capistrano_introduction
The key's randomart image is:
+--[ RSA 2048]----+
|                 |
|       .         |
|        =        |
|       +.o       |
|       +S E      |
|      .+o= o     |
|       .+o* o    |
|     .o.oo =     |
|     .o. ..      |
+-----------------+
```
_ops/recipes/default.rb_
```ruby
# 公開鍵の登録
directory "/home/deploy/.ssh/" do
  owner 'deploy'
  group 'deploy'
  mode 0755
end

cookbook_file "/home/#{params[:name]}/.ssh/authorized_keys" do
  owner params[:name]
  mode 0600
  source "#{params[:name]}.authorized_keys.pub"
end
```
接続の確認
```bash
$ ssh -i files/default/deploy.authorized_keys deploy@192.168.33.10
Welcome to Ubuntu 12.04.2 LTS (GNU/Linux 3.5.0-23-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

deploy@ops-berkshelf:~$
```

#### サーバーからレポジトリホストへ
ワークステーションからサーバーへの接続はできるようになった。次のデプロイユーザーが自動的にコードレポジトリアクセスできるようにするには以下のいずれかの方法を選択する。

1. SSH Agent Forwarding
```bash
$ ssh-add ops/files/default/deploy.authorized_keys
Identity added: ops/files/default/deploy.authorized_keys (ops/files/default/deploy.authorized_keys)
$ ssh-add -l
$ ssh deploy@192.168.33.10
deploy@ops-berkshelf:~$ ssh git@github.com
The authenticity of host 'github.com (192.30.252.129)' can't be established.
RSA key fingerprint is 16:27:ac:a5:76:28:2d:36:63:1b:56:4d:eb:df:a6:48.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'github.com,192.30.252.129' (RSA) to the list of known hosts.
PTY allocation request failed on channel 0
Hi k2works! You've successfully authenticated, but GitHub does not provide shell access.
Connection to github.com closed.
deploy@ops-berkshelf:~$ exit
$ ssh -A deploy@192.168.33.10 'git ls-remote git@github.com:k2works/capistrano_introduction.git'
Warning: Permanently added the RSA host key for IP address '192.30.252.131' to the list of known hosts.
3193f2b4467c063c28cd7500cffa550f3496975a        HEAD
3193f2b4467c063c28cd7500cffa550f3496975a        refs/heads/master
7847245623da46bd2de8cc3df758367003e13a81        refs/heads/wip
7847245623da46bd2de8cc3df758367003e13a81        refs/pull/1/head
```
1. HTTP 認証
1. GitHubのユーザー名・パスワードを使う方法
```bash
git ls-remote https://ユーザー名:パスワード@github.com/k2works/capistrano_introduction.git
```
1. OAuthパーソナルAPIトークンを使う
```bash
git ls-remote https://xxxx:github.com/k2works/capistrano_introduction.git
```
xxxxがパーソナルAPIトークン
1. デプロイキー
Githubに公開鍵を登録する  
[Managing Deploy Keys](https://developer.github.com/guides/managing-deploy-keys/)

#### 委任
次のトピックとしてデプロイユーザーがサーバーのデプロイメントディレクトにアクセスできるようにする必要がある。

_ops/attributes/default.rb_  
```ruby
・・・
default['app']['deploy_to'] = '/usr/share/nginx/www/capistrano_introduction'
・・・
```

_ops/recipes/default.rb_
```ruby
directory "#{node['app']['deploy_to']}" do
  owner 'deploy'
  group 'deploy'
end

bash "SetupForDeployDirectory" do
  user 'deploy'
  cwd "#{node['app']['deploy_to']}"
  command "umask 0002"
  command "chmod g+s #{node['app']['deploy_to']}"
  command "mkdir #{node['app']['deploy_to']}/{releases,shared}"
end
```
### コールドスタート
### フロー
### ロールバック

## <a name="2">進んだ機能</a>
## <a name="3">フレームワーク拡張</a>


# 参照
+ [Capistrano](http://capistranorb.com/documentation/overview/what-is-capistrano/)
+ [capistrano/capistrano](https://github.com/capistrano/capistrano)
+ [VAGRANT](http://www.vagrantup.com/)
+ [Chef-solo+knife-solo+Vagrantでサーバ構築を自動化してみる - その１ ユーザー追加](http://straitwalk.hatenablog.com/entry/2013/08/25/000935)
+ [Ruby - RVM の Multi-User mode について](http://babiy3104.hateblo.jp/entry/2013/08/30/103602)
+ [How To Deploy Rails Apps Using Unicorn And Nginx on CentOS 6.5](https://www.digitalocean.com/community/tutorials/how-to-deploy-rails-apps-using-unicorn-and-nginx-on-centos-6-5)
