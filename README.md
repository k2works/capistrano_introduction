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
デプロイ用ユーザーとしてログインする
```bash
$ ssh-add files/default/deploy.authorized_keys
$ ssh deploy@192.168.33.10
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
role :app, %w{deploy@192.168.33.10}
role :web, %w{deploy@192.168.33.10}
role :db,  %w{deploy@192.168.33.10}
・・・
server '192.168.33.10', user: 'deploy', roles: %w{web app}, my_property: :my_value
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
$ ssh deploy@192.168.33.10
deploy@ops-berkshelf:~$ ssh git@github.com
deploy@ops-berkshelf:~$ exit
$ ssh-add
$ ssh -A deploy@192.168.33.10 'git ls-remote git@github.com:k2works/capistrano_introduction.git'
```
上記の操作は以下の操作でも実現できる
```bash
$ cap staging git:check
```
SSH接続の際`Host key verification failed.`で接続に失敗する場合は以下の操作でローカルのホストキーを更新する
```bash
$ ssh-kegen -F 192.168.33.10
$ ssh-kegen -R 192.168.33.10
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
ここでのポイントはデプロイしたいサーバーにデプロイできる権限を持つユーザーを作成することです。
デフォルトとして_/var/www/my-application_などのデプロイ先に書き込み権限のあるユーザーです。

すでに適切なパーミッションが設定されたディレクトをセットアップしているそしてチームメンバーもそこにデプロイできるようになっている。

1. リモートマシンのディレクト構造をチェックする
```bash
$ ssh deploy@192.168.33.10 'ls -lR /usr/share/nginx/www/capistrano_introduction'
/usr/share/nginx/www/capistrano_introduction:
total 8
drwxr-xr-x 2 deploy deploy 4096 Jul  2 06:18 releases
drwxr-xr-x 2 deploy deploy 4096 Jul  2 06:18 shared
/usr/share/nginx/www/capistrano_introduction/releases:
total 0
/usr/share/nginx/www/capistrano_introduction/shared:
total 0  
```
1. チェック用の最初のcapタスクを書く
_dev/lib/capistrano/tasks/access_check.rake_
```ruby
desc "Check that we can access everything"
task :check_write_permissions do
  on roles(:all) do |host|
    if test("[ -w #{fetch(:deploy_to)} ]")
      info "#{fetch(:deploy_to)} is writable on #{host}"
    else
      error "#{fetch(:deploy_to)} is not writable on #{host}"
    end
  end
end
```
タスク実行
```bash
$ cap staging check_write_permissions
DEBUG[e6e9eac6] Running /usr/bin/env [ -w /usr/share/nginx/www/capistrano_introduction ] on 192.168.33.10
DEBUG[e6e9eac6] Command: [ -w /usr/share/nginx/www/capistrano_introduction ]
DEBUG[e6e9eac6] Finished in 0.117 seconds with exit status 0 (successful).
INFO/usr/share/nginx/www/capistrano_introduction is writable on 192.168.33.10
```
Gitサーバーに接続できるか確認する
```bash
$ cap staging git:check
INFO[bc234386] Running /usr/bin/env mkdir -p /tmp/capistrano_introduction/ on 192.168.33.10
DEBUG[bc234386] Command: /usr/bin/env mkdir -p /tmp/capistrano_introduction/
INFO[bc234386] Finished in 0.089 seconds with exit status 0 (successful).
DEBUGUploading /tmp/capistrano_introduction/git-ssh.sh 0.0%
INFOUploading /tmp/capistrano_introduction/git-ssh.sh 100.0%
INFO[63f3447f] Running /usr/bin/env chmod +x /tmp/capistrano_introduction/git-ssh.sh on 192.168.33.10
DEBUG[63f3447f] Command: /usr/bin/env chmod +x /tmp/capistrano_introduction/git-ssh.sh
INFO[63f3447f] Finished in 0.006 seconds with exit status 0 (successful).
DEBUG[3b541ba8] Running /usr/bin/env git ls-remote -h git@github.com:k2works/capistrano_introduction.git on 192.168.33.10
DEBUG[3b541ba8] Command: ( GIT_ASKPASS=/bin/echo GIT_SSH=/tmp/capistrano_introduction/git-ssh.sh /usr/bin/env git ls-remote -h git@github.com:k2works/capistrano_introduction.git )
DEBUG[3b541ba8]         Warning: Permanently added 'github.com,192.30.252.128' (RSA) to the list of known hosts.
DEBUG[3b541ba8]         3193f2b4467c063c28cd7500cffa550f3496975a        refs/heads/master
DEBUG[3b541ba8]         57785e221a234033c5785cf32d0bc4a8359c2f40        refs/heads/wip
DEBUG[3b541ba8]         Warning: Permanently added 'github.com,192.30.252.128' (RSA) to the list of known hosts.
DEBUG[3b541ba8] Finished in 13.109 seconds with exit status 0 (successful).
```
SSH agent forwardingを確認するタスク  
_dev/lib/capistrano/tasks/access_check.rake_
```ruby
desc "Check if agent forwarding is working"
task :forwarding do
  on roles(:all) do |h|
    if test("env | grep SSH_AUTH_SOCK")
      info "Agent forwarding is up to #{h}"
    else
      error "Agent forwarding is NOT up to #{h}"
    end
  end
end
```
実行結果
```bash
$ cap staging forwarding
DEBUG[bbbaf0f3] Running /usr/bin/env env | grep SSH_AUTH_SOCK on 192.168.33.10
DEBUG[bbbaf0f3] Command: env | grep SSH_AUTH_SOCK
DEBUG[bbbaf0f3]         SSH_AUTH_SOCK=/tmp/ssh-UAdJzw9090/agent.9090
DEBUG[bbbaf0f3] Finished in 0.091 seconds with exit status 0 (successful).
INFOAgent forwarding is up to 192.168.33.10
```
### フロー
#### デプロイ
```bash
$ cap staging deploy
INFO[f6ef4009] Running /usr/bin/env mkdir -p /tmp/capistrano_introduction/ on 192.168.33.10
DEBUG[f6ef4009] Command: /usr/bin/env mkdir -p /tmp/capistrano_introduction/
INFO[f6ef4009] Finished in 0.091 seconds with exit status 0 (successful).
DEBUGUploading /tmp/capistrano_introduction/git-ssh.sh 0.0%
INFOUploading /tmp/capistrano_introduction/git-ssh.sh 100.0%
INFO[da59e4dd] Running /usr/bin/env chmod +x /tmp/capistrano_introduction/git-ssh.sh on 192.168.33.10
DEBUG[da59e4dd] Command: /usr/bin/env chmod +x /tmp/capistrano_introduction/git-ssh.sh
INFO[da59e4dd] Finished in 0.008 seconds with exit status 0 (successful).
DEBUG[ae690cfa] Running /usr/bin/env git ls-remote -h git@github.com:k2works/capistrano_introduction.git on 192.168.33.10
DEBUG[ae690cfa] Command: ( GIT_ASKPASS=/bin/echo GIT_SSH=/tmp/capistrano_introduction/git-ssh.sh /usr/bin/env git ls-remote -h git@github.com:k2works/capistrano_introduction.git )
DEBUG[ae690cfa]         Warning: Permanently added 'github.com,192.30.252.129' (RSA) to the list of known hosts.
DEBUG[ae690cfa]         3193f2b4467c063c28cd7500cffa550f3496975a        refs/heads/master
DEBUG[ae690cfa]         57785e221a234033c5785cf32d0bc4a8359c2f40        refs/heads/wip
DEBUG[ae690cfa]         Warning: Permanently added 'github.com,192.30.252.129' (RSA) to the list of known hosts.
DEBUG[ae690cfa] Finished in 11.045 seconds with exit status 0 (successful).
INFO[0fc6e47c] Running /usr/bin/env mkdir -pv /usr/share/nginx/www/capistrano_introduction/shared /usr/share/nginx/www/capistrano_introduction/releases on 192.168.33.10
DEBUG[0fc6e47c] Command: /usr/bin/env mkdir -pv /usr/share/nginx/www/capistrano_introduction/shared /usr/share/nginx/www/capistrano_introduction/releases
INFO[0fc6e47c] Finished in 0.006 seconds with exit status 0 (successful).
DEBUG[4900bf6b] Running /usr/bin/env [ -f /usr/share/nginx/www/capistrano_introduction/current/REVISION ] on 192.168.33.10
DEBUG[4900bf6b] Command: [ -f /usr/share/nginx/www/capistrano_introduction/current/REVISION ]
DEBUG[4900bf6b] Finished in 0.004 seconds with exit status 1 (failed).
DEBUG[8285e25b] Running /usr/bin/env [ -f /usr/share/nginx/www/capistrano_introduction/repo/HEAD ] on 192.168.33.10
DEBUG[8285e25b] Command: [ -f /usr/share/nginx/www/capistrano_introduction/repo/HEAD ]
DEBUG[8285e25b] Finished in 0.005 seconds with exit status 1 (failed).
DEBUG[8fb9cd95] Running /usr/bin/env if test ! -d /usr/share/nginx/www/capistrano_introduction; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction'" 1>&2; false; fi on 192.168.33.10
DEBUG[8fb9cd95] Command: if test ! -d /usr/share/nginx/www/capistrano_introduction; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction'" 1>&2; false; fi
DEBUG[8fb9cd95] Finished in 0.004 seconds with exit status 0 (successful).
INFO[cb9a37c3] Running /usr/bin/env git clone --mirror git@github.com:k2works/capistrano_introduction.git /usr/share/nginx/www/capistrano_introduction/repo on 192.168.33.10
DEBUG[cb9a37c3] Command: cd /usr/share/nginx/www/capistrano_introduction && ( GIT_ASKPASS=/bin/echo GIT_SSH=/tmp/capistrano_introduction/git-ssh.sh /usr/bin/env git clone --mirror git@github.com:k2works/capistrano_introduction.git /usr/share/nginx/www/capistrano_introduction/repo )
DEBUG[cb9a37c3]         Cloning into bare repository '/usr/share/nginx/www/capistrano_introduction/repo'...
DEBUG[cb9a37c3]         Cloning into bare repository '/usr/share/nginx/www/capistrano_introduction/repo'...
DEBUG[cb9a37c3]         Warning: Permanently added the RSA host key for IP address '192.30.252.131' to the list of known hosts.
INFO[cb9a37c3] Finished in 11.038 seconds with exit status 0 (successful).
DEBUG[de63e2cc] Running /usr/bin/env if test ! -d /usr/share/nginx/www/capistrano_introduction/repo; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction/repo'" 1>&2; false; fi on 192.168.33.10
DEBUG[de63e2cc] Command: if test ! -d /usr/share/nginx/www/capistrano_introduction/repo; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction/repo'" 1>&2; false; fi
DEBUG[de63e2cc] Finished in 0.004 seconds with exit status 0 (successful).
INFO[78438f62] Running /usr/bin/env git remote update on 192.168.33.10
DEBUG[78438f62] Command: cd /usr/share/nginx/www/capistrano_introduction/repo && ( GIT_ASKPASS=/bin/echo GIT_SSH=/tmp/capistrano_introduction/git-ssh.sh /usr/bin/env git remote update )
DEBUG[78438f62]         Fetching origin
DEBUG[78438f62]         Fetching origin
DEBUG[78438f62]         Warning: Permanently added the RSA host key for IP address '192.30.252.130' to the list of known hosts.
INFO[78438f62] Finished in 12.342 seconds with exit status 0 (successful).
DEBUG[ddef9ba5] Running /usr/bin/env if test ! -d /usr/share/nginx/www/capistrano_introduction/repo; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction/repo'" 1>&2; false; fi on 192.168.33.10
DEBUG[ddef9ba5] Command: if test ! -d /usr/share/nginx/www/capistrano_introduction/repo; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction/repo'" 1>&2; false; fi
DEBUG[ddef9ba5] Finished in 0.005 seconds with exit status 0 (successful).
INFO[2c628fac] Running /usr/bin/env mkdir -p /usr/share/nginx/www/capistrano_introduction/releases/20140704051705 on 192.168.33.10
DEBUG[2c628fac] Command: cd /usr/share/nginx/www/capistrano_introduction/repo && ( GIT_ASKPASS=/bin/echo GIT_SSH=/tmp/capistrano_introduction/git-ssh.sh /usr/bin/env mkdir -p /usr/share/nginx/www/capistrano_introduction/releases/20140704051705 )
INFO[2c628fac] Finished in 0.005 seconds with exit status 0 (successful).
INFO[c698e7dd] Running /usr/bin/env git archive master | tar -x -C /usr/share/nginx/www/capistrano_introduction/releases/20140704051705 on 192.168.33.10
DEBUG[c698e7dd] Command: cd /usr/share/nginx/www/capistrano_introduction/repo && ( GIT_ASKPASS=/bin/echo GIT_SSH=/tmp/capistrano_introduction/git-ssh.sh /usr/bin/env git archive master | tar -x -C /usr/share/nginx/www/capistrano_introduction/releases/20140704051705 )
INFO[c698e7dd] Finished in 0.011 seconds with exit status 0 (successful).
DEBUG[3a92a55f] Running /usr/bin/env if test ! -d /usr/share/nginx/www/capistrano_introduction/repo; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction/repo'" 1>&2; false; fi on 192.168.33.10
DEBUG[3a92a55f] Command: if test ! -d /usr/share/nginx/www/capistrano_introduction/repo; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction/repo'" 1>&2; false; fi
DEBUG[3a92a55f] Finished in 0.005 seconds with exit status 0 (successful).
DEBUG[411b927b] Running /usr/bin/env git rev-parse --short master on 192.168.33.10
DEBUG[411b927b] Command: cd /usr/share/nginx/www/capistrano_introduction/repo && ( GIT_ASKPASS=/bin/echo GIT_SSH=/tmp/capistrano_introduction/git-ssh.sh /usr/bin/env git rev-parse --short master )
DEBUG[411b927b]         3193f2b
DEBUG[411b927b] Finished in 0.007 seconds with exit status 0 (successful).
DEBUG[3698b024] Running /usr/bin/env if test ! -d /usr/share/nginx/www/capistrano_introduction/releases/20140704051705; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction/releases/20140704051705'" 1>&2; false; fi on 192.168.33.10
DEBUG[3698b024] Command: if test ! -d /usr/share/nginx/www/capistrano_introduction/releases/20140704051705; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction/releases/20140704051705'" 1>&2; false; fi
DEBUG[3698b024] Finished in 0.005 seconds with exit status 0 (successful).
INFO[07d90721] Running /usr/bin/env echo "3193f2b" >> REVISION on 192.168.33.10
DEBUG[07d90721] Command: cd /usr/share/nginx/www/capistrano_introduction/releases/20140704051705 && /usr/bin/env echo "3193f2b" >> REVISION
INFO[07d90721] Finished in 0.009 seconds with exit status 0 (successful).
INFO[74fed8a1] Running /usr/bin/env rm -rf /usr/share/nginx/www/capistrano_introduction/current on 192.168.33.10
DEBUG[74fed8a1] Command: /usr/bin/env rm -rf /usr/share/nginx/www/capistrano_introduction/current
INFO[74fed8a1] Finished in 0.008 seconds with exit status 0 (successful).
INFO[3323c4e6] Running /usr/bin/env ln -s /usr/share/nginx/www/capistrano_introduction/releases/20140704051705 /usr/share/nginx/www/capistrano_introduction/current on 192.168.33.10
DEBUG[3323c4e6] Command: /usr/bin/env ln -s /usr/share/nginx/www/capistrano_introduction/releases/20140704051705 /usr/share/nginx/www/capistrano_introduction/current
INFO[3323c4e6] Finished in 0.006 seconds with exit status 0 (successful).
DEBUG[9fca0e00] Running /usr/bin/env ls -x /usr/share/nginx/www/capistrano_introduction/releases on 192.168.33.10
DEBUG[9fca0e00] Command: /usr/bin/env ls -x /usr/share/nginx/www/capistrano_introduction/releases
DEBUG[9fca0e00]         20140704051705
DEBUG[9fca0e00] Finished in 0.009 seconds with exit status 0 (successful).
DEBUG[1ef9a30a] Running /usr/bin/env if test ! -d /usr/share/nginx/www/capistrano_introduction/releases; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction/releases'" 1>&2; false; fi on 192.168.33.10
DEBUG[1ef9a30a] Command: if test ! -d /usr/share/nginx/www/capistrano_introduction/releases; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction/releases'" 1>&2; false; fi
DEBUG[1ef9a30a] Finished in 0.008 seconds with exit status 0 (successful).
INFO[dffd3f57] Running /usr/bin/env echo "Branch master (at 3193f2b) deployed as release 20140704051705 by k2works" >> /usr/share/nginx/www/capistrano_introduction/revisions.log on 192.168.33.10
DEBUG[dffd3f57] Command: echo "Branch master (at 3193f2b) deployed as release 20140704051705 by k2works" >> /usr/share/nginx/www/capistrano_introduction/revisions.log
INFO[dffd3f57] Finished in 0.006 seconds with exit status 0 (successful).
```
#### ロールバック

## <a name="2">進んだ機能</a>
## <a name="3">フレームワーク拡張</a>


# 参照
+ [Capistrano](http://capistranorb.com/documentation/overview/what-is-capistrano/)
+ [capistrano/capistrano](https://github.com/capistrano/capistrano)
+ [VAGRANT](http://www.vagrantup.com/)
+ [Chef-solo+knife-solo+Vagrantでサーバ構築を自動化してみる - その１ ユーザー追加](http://straitwalk.hatenablog.com/entry/2013/08/25/000935)
+ [Ruby - RVM の Multi-User mode について](http://babiy3104.hateblo.jp/entry/2013/08/30/103602)
+ [How To Deploy Rails Apps Using Unicorn And Nginx on CentOS 6.5](https://www.digitalocean.com/community/tutorials/how-to-deploy-rails-apps-using-unicorn-and-nginx-on-centos-6-5)
+ [Rails 3 + Nginx/Unicorn を Amazon AWS に Capistrano 3 でデプロイする](http://bekkou68.hatenablog.com/entry/2013/12/21/173641)
+ [[Ubuntu]ssh 接続しようとすると「WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!」が表示されるときは](http://d.hatena.ne.jp/PRiMENON/20090418/1240062291)
