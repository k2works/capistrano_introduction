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
git ls-remote https://xxxx:@github.com/k2works/capistrano_introduction.git
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
1. タスク実行
```bash
$ cap staging check_write_permissions
DEBUG[e6e9eac6] Running /usr/bin/env [ -w /usr/share/nginx/www/capistrano_introduction ] on 192.168.33.10
DEBUG[e6e9eac6] Command: [ -w /usr/share/nginx/www/capistrano_introduction ]
DEBUG[e6e9eac6] Finished in 0.117 seconds with exit status 0 (successful).
INFO/usr/share/nginx/www/capistrano_introduction is writable on 192.168.33.10
```
1. Gitサーバーに接続できるか確認する
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
1. 実行結果
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
##### rails対応
_dev/Gemfile_  
_dev/Capfile_  
_dev/config/deploy/staging.rb_  
_dev/config/environments/staging.rb_  
_dev/lib/capistrano/tasks/unicorn.rake_  
_dev/config/unicorn/staging.rb_  
_dev/config/secrets.yml_
```bash
$ rails g controller wlecome index
```
_dev/config/routes.rb_
##### railsアプリケーションのデプロイ
```bash
 $ cap staging deploy
DEBUG[ce330f89] Running /usr/local/rvm/bin/rvm version on 192.168.33.10
DEBUG[ce330f89] Command: /usr/local/rvm/bin/rvm version
DEBUG[ce330f89]
DEBUG[ce330f89]         rvm 1.25.27 (stable) by Wayne E. Seguin <wayneeseguin@gmail.com>, Michal Papis <mpapis@gmail.com> [https://rvm.io/]
DEBUG[ce330f89]
DEBUG[ce330f89] Finished in 0.328 seconds with exit status 0 (successful).
rvm 1.25.27 (stable) by Wayne E. Seguin <wayneeseguin@gmail.com>, Michal Papis <mpapis@gmail.com> [https://rvm.io/]
DEBUG[ad3f0524] Running /usr/local/rvm/bin/rvm current on 192.168.33.10
DEBUG[ad3f0524] Command: /usr/local/rvm/bin/rvm current
DEBUG[ad3f0524]         ruby-2.1.0
DEBUG[ad3f0524] Finished in 0.248 seconds with exit status 0 (successful).
ruby-2.1.0
DEBUG[7b73b6cb] Running /usr/local/rvm/bin/rvm ruby-2.1.0@global do ruby --version on 192.168.33.10
DEBUG[7b73b6cb] Command: /usr/local/rvm/bin/rvm ruby-2.1.0@global do ruby --version
DEBUG[7b73b6cb]         ruby 2.1.0p0 (2013-12-25 revision 44422) [x86_64-linux]
DEBUG[7b73b6cb] Finished in 0.413 seconds with exit status 0 (successful).
ruby 2.1.0p0 (2013-12-25 revision 44422) [x86_64-linux]
INFO[3ecb3f92] Running /usr/bin/env mkdir -p /tmp/capistrano_introduction/ on 192.168.33.10
DEBUG[3ecb3f92] Command: /usr/bin/env mkdir -p /tmp/capistrano_introduction/
INFO[3ecb3f92] Finished in 0.010 seconds with exit status 0 (successful).
DEBUGUploading /tmp/capistrano_introduction/git-ssh.sh 0.0%
INFOUploading /tmp/capistrano_introduction/git-ssh.sh 100.0%
INFO[e1c2c03e] Running /usr/bin/env chmod +x /tmp/capistrano_introduction/git-ssh.sh on 192.168.33.10
DEBUG[e1c2c03e] Command: /usr/bin/env chmod +x /tmp/capistrano_introduction/git-ssh.sh
INFO[e1c2c03e] Finished in 0.007 seconds with exit status 0 (successful).
DEBUG[52c5a422] Running /usr/bin/env git ls-remote -h git@github.com:k2works/capistrano_introduction_dev.git on 192.168.33.10
DEBUG[52c5a422] Command: ( GIT_ASKPASS=/bin/echo GIT_SSH=/tmp/capistrano_introduction/git-ssh.sh /usr/bin/env git ls-remote -h git@github.com:k2works/capistrano_introduction_dev.git )
DEBUG[52c5a422]         Warning: Permanently added 'github.com,192.30.252.130' (RSA) to the list of known hosts.
DEBUG[52c5a422]         a6511ba86b213d12c21fd130687a7ef45f152170        refs/heads/master
DEBUG[52c5a422]         Warning: Permanently added 'github.com,192.30.252.130' (RSA) to the list of known hosts.
DEBUG[52c5a422] Finished in 10.840 seconds with exit status 0 (successful).
INFO[977c7be5] Running /usr/bin/env mkdir -pv /usr/share/nginx/www/capistrano_introduction/shared /usr/share/nginx/www/capistrano_introduction/releases on 192.168.33.10
DEBUG[977c7be5] Command: /usr/bin/env mkdir -pv /usr/share/nginx/www/capistrano_introduction/shared /usr/share/nginx/www/capistrano_introduction/releases
INFO[977c7be5] Finished in 0.009 seconds with exit status 0 (successful).
INFO[8d003487] Running /usr/bin/env mkdir -pv /usr/share/nginx/www/capistrano_introduction/shared/public/assets on 192.168.33.10
DEBUG[8d003487] Command: /usr/bin/env mkdir -pv /usr/share/nginx/www/capistrano_introduction/shared/public/assets
DEBUG[8d003487]         mkdir: created directory `/usr/share/nginx/www/capistrano_introduction/shared/public'
DEBUG[8d003487]         mkdir: created directory `/usr/share/nginx/www/capistrano_introduction/shared/public/assets'
INFO[8d003487] Finished in 0.009 seconds with exit status 0 (successful).
INFO[217edf7b] Running /usr/bin/env mkdir -pv /usr/share/nginx/www/capistrano_introduction/shared/config on 192.168.33.10
DEBUG[217edf7b] Command: /usr/bin/env mkdir -pv /usr/share/nginx/www/capistrano_introduction/shared/config
INFO[217edf7b] Finished in 0.007 seconds with exit status 0 (successful).
DEBUG[d4139b64] Running /usr/bin/env [ -f /usr/share/nginx/www/capistrano_introduction/shared/config/database.yml ] on 192.168.33.10
DEBUG[d4139b64] Command: [ -f /usr/share/nginx/www/capistrano_introduction/shared/config/database.yml ]
DEBUG[d4139b64] Finished in 0.006 seconds with exit status 0 (successful).
DEBUG[49e06c76] Running /usr/bin/env [ -f /usr/share/nginx/www/capistrano_introduction/current/REVISION ] on 192.168.33.10
DEBUG[49e06c76] Command: [ -f /usr/share/nginx/www/capistrano_introduction/current/REVISION ]
DEBUG[49e06c76] Finished in 0.006 seconds with exit status 1 (failed).
DEBUG[a4ca42ac] Running /usr/bin/env [ -f /usr/share/nginx/www/capistrano_introduction/repo/HEAD ] on 192.168.33.10
DEBUG[a4ca42ac] Command: [ -f /usr/share/nginx/www/capistrano_introduction/repo/HEAD ]
DEBUG[a4ca42ac] Finished in 0.005 seconds with exit status 1 (failed).
DEBUG[a394a254] Running /usr/bin/env if test ! -d /usr/share/nginx/www/capistrano_introduction; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction'" 1>&2; false; fi on 192.168.33.10
DEBUG[a394a254] Command: if test ! -d /usr/share/nginx/www/capistrano_introduction; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction'" 1>&2; false; fi
DEBUG[a394a254] Finished in 0.008 seconds with exit status 0 (successful).
INFO[21acb2e6] Running /usr/bin/env git clone --mirror git@github.com:k2works/capistrano_introduction_dev.git /usr/share/nginx/www/capistrano_introduction/repo on 192.168.33.10
DEBUG[21acb2e6] Command: cd /usr/share/nginx/www/capistrano_introduction && ( GIT_ASKPASS=/bin/echo GIT_SSH=/tmp/capistrano_introduction/git-ssh.sh /usr/bin/env git clone --mirror git@github.com:k2works/capistrano_introduction_dev.git /usr/share/nginx/www/capistrano_introduction/repo )
DEBUG[21acb2e6]         Cloning into bare repository '/usr/share/nginx/www/capistrano_introduction/repo'...
DEBUG[21acb2e6]         Cloning into bare repository '/usr/share/nginx/www/capistrano_introduction/repo'...
DEBUG[21acb2e6]         Warning: Permanently added the RSA host key for IP address '192.30.252.129' to the list of known hosts.
INFO[21acb2e6] Finished in 12.287 seconds with exit status 0 (successful).
DEBUG[bb8a1b6a] Running /usr/bin/env if test ! -d /usr/share/nginx/www/capistrano_introduction/repo; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction/repo'" 1>&2; false; fi on 192.168.33.10
DEBUG[bb8a1b6a] Command: if test ! -d /usr/share/nginx/www/capistrano_introduction/repo; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction/repo'" 1>&2; false; fi
DEBUG[bb8a1b6a] Finished in 0.004 seconds with exit status 0 (successful).
INFO[6534e822] Running /usr/bin/env git remote update on 192.168.33.10
DEBUG[6534e822] Command: cd /usr/share/nginx/www/capistrano_introduction/repo && ( GIT_ASKPASS=/bin/echo GIT_SSH=/tmp/capistrano_introduction/git-ssh.sh /usr/bin/env git remote update )
DEBUG[6534e822]         Fetching origin
INFO[6534e822] Finished in 11.155 seconds with exit status 0 (successful).
DEBUG[873b5bf4] Running /usr/bin/env if test ! -d /usr/share/nginx/www/capistrano_introduction/repo; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction/repo'" 1>&2; false; fi on 192.168.33.10
DEBUG[873b5bf4] Command: if test ! -d /usr/share/nginx/www/capistrano_introduction/repo; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction/repo'" 1>&2; false; fi
DEBUG[873b5bf4] Finished in 0.004 seconds with exit status 0 (successful).
INFO[264bbcaa] Running /usr/bin/env mkdir -p /usr/share/nginx/www/capistrano_introduction/releases/20140704110147 on 192.168.33.10
DEBUG[264bbcaa] Command: cd /usr/share/nginx/www/capistrano_introduction/repo && ( GIT_ASKPASS=/bin/echo GIT_SSH=/tmp/capistrano_introduction/git-ssh.sh /usr/bin/env mkdir -p /usr/share/nginx/www/capistrano_introduction/releases/20140704110147 )
INFO[264bbcaa] Finished in 0.009 seconds with exit status 0 (successful).
INFO[6a43c1ef] Running /usr/bin/env git archive master | tar -x -C /usr/share/nginx/www/capistrano_introduction/releases/20140704110147 on 192.168.33.10
DEBUG[6a43c1ef] Command: cd /usr/share/nginx/www/capistrano_introduction/repo && ( GIT_ASKPASS=/bin/echo GIT_SSH=/tmp/capistrano_introduction/git-ssh.sh /usr/bin/env git archive master | tar -x -C /usr/share/nginx/www/capistrano_introduction/releases/20140704110147 )
INFO[6a43c1ef] Finished in 0.014 seconds with exit status 0 (successful).
DEBUG[b5b8aa23] Running /usr/bin/env if test ! -d /usr/share/nginx/www/capistrano_introduction/repo; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction/repo'" 1>&2; false; fi on 192.168.33.10
DEBUG[b5b8aa23] Command: if test ! -d /usr/share/nginx/www/capistrano_introduction/repo; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction/repo'" 1>&2; false; fi
DEBUG[b5b8aa23] Finished in 0.005 seconds with exit status 0 (successful).
DEBUG[9845a98b] Running /usr/bin/env git rev-parse --short master on 192.168.33.10
DEBUG[9845a98b] Command: cd /usr/share/nginx/www/capistrano_introduction/repo && ( GIT_ASKPASS=/bin/echo GIT_SSH=/tmp/capistrano_introduction/git-ssh.sh /usr/bin/env git rev-parse --short master )
DEBUG[9845a98b]         a6511ba
DEBUG[9845a98b] Finished in 0.009 seconds with exit status 0 (successful).
DEBUG[4d5a2ff5] Running /usr/bin/env if test ! -d /usr/share/nginx/www/capistrano_introduction/releases/20140704110147; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction/releases/20140704110147'" 1>&2; false; fi on 192.168.33.10
DEBUG[4d5a2ff5] Command: if test ! -d /usr/share/nginx/www/capistrano_introduction/releases/20140704110147; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction/releases/20140704110147'" 1>&2; false; fi
DEBUG[4d5a2ff5] Finished in 0.006 seconds with exit status 0 (successful).
INFO[69ce238d] Running /usr/bin/env echo "a6511ba" >> REVISION on 192.168.33.10
DEBUG[69ce238d] Command: cd /usr/share/nginx/www/capistrano_introduction/releases/20140704110147 && /usr/bin/env echo "a6511ba" >> REVISION
INFO[69ce238d] Finished in 0.007 seconds with exit status 0 (successful).
INFO[301b4b4d] Running /usr/bin/env mkdir -pv /usr/share/nginx/www/capistrano_introduction/releases/20140704110147/config on 192.168.33.10
DEBUG[301b4b4d] Command: /usr/bin/env mkdir -pv /usr/share/nginx/www/capistrano_introduction/releases/20140704110147/config
INFO[301b4b4d] Finished in 0.008 seconds with exit status 0 (successful).
DEBUG[a191bcfe] Running /usr/bin/env [ -L /usr/share/nginx/www/capistrano_introduction/releases/20140704110147/config/database.yml ] on 192.168.33.10
DEBUG[a191bcfe] Command: [ -L /usr/share/nginx/www/capistrano_introduction/releases/20140704110147/config/database.yml ]
DEBUG[a191bcfe] Finished in 0.005 seconds with exit status 1 (failed).
DEBUG[1b2ebedb] Running /usr/bin/env [ -f /usr/share/nginx/www/capistrano_introduction/releases/20140704110147/config/database.yml ] on 192.168.33.10
DEBUG[1b2ebedb] Command: [ -f /usr/share/nginx/www/capistrano_introduction/releases/20140704110147/config/database.yml ]
DEBUG[1b2ebedb] Finished in 0.004 seconds with exit status 1 (failed).
INFO[688e83f6] Running /usr/bin/env ln -s /usr/share/nginx/www/capistrano_introduction/shared/config/database.yml /usr/share/nginx/www/capistrano_introduction/releases/20140704110147/config/database.yml on 192.168.33.10
DEBUG[688e83f6] Command: /usr/bin/env ln -s /usr/share/nginx/www/capistrano_introduction/shared/config/database.yml /usr/share/nginx/www/capistrano_introduction/releases/20140704110147/config/database.yml
INFO[688e83f6] Finished in 0.008 seconds with exit status 0 (successful).
INFO[e552da8a] Running /usr/bin/env mkdir -pv /usr/share/nginx/www/capistrano_introduction/releases/20140704110147/public on 192.168.33.10
DEBUG[e552da8a] Command: /usr/bin/env mkdir -pv /usr/share/nginx/www/capistrano_introduction/releases/20140704110147/public
INFO[e552da8a] Finished in 0.007 seconds with exit status 0 (successful).
DEBUG[38cb2c53] Running /usr/bin/env [ -L /usr/share/nginx/www/capistrano_introduction/releases/20140704110147/public/assets ] on 192.168.33.10
DEBUG[38cb2c53] Command: [ -L /usr/share/nginx/www/capistrano_introduction/releases/20140704110147/public/assets ]
DEBUG[38cb2c53] Finished in 0.005 seconds with exit status 1 (failed).
DEBUG[83ed0953] Running /usr/bin/env [ -d /usr/share/nginx/www/capistrano_introduction/releases/20140704110147/public/assets ] on 192.168.33.10
DEBUG[83ed0953] Command: [ -d /usr/share/nginx/www/capistrano_introduction/releases/20140704110147/public/assets ]
DEBUG[83ed0953] Finished in 0.006 seconds with exit status 1 (failed).
INFO[5d9032a5] Running /usr/bin/env ln -s /usr/share/nginx/www/capistrano_introduction/shared/public/assets /usr/share/nginx/www/capistrano_introduction/releases/20140704110147/public/assets on 192.168.33.10
DEBUG[5d9032a5] Command: /usr/bin/env ln -s /usr/share/nginx/www/capistrano_introduction/shared/public/assets /usr/share/nginx/www/capistrano_introduction/releases/20140704110147/public/assets
INFO[5d9032a5] Finished in 0.007 seconds with exit status 0 (successful).
DEBUG[bdd05e5b] Running /usr/bin/env if test ! -d /usr/share/nginx/www/capistrano_introduction/releases/20140704110147; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction/releases/20140704110147'" 1>&2; false; fi on 192.168.33.10
DEBUG[bdd05e5b] Command: if test ! -d /usr/share/nginx/www/capistrano_introduction/releases/20140704110147; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction/releases/20140704110147'" 1>&2; false; fi
DEBUG[bdd05e5b] Finished in 0.005 seconds with exit status 0 (successful).
INFO[46168fa1] Running /usr/local/rvm/bin/rvm ruby-2.1.0@global do bundle install --binstubs /usr/share/nginx/www/capistrano_introduction/shared/bin --path /usr/share/nginx/www/capistrano_introduction/shared/bundle --without development test --deployment --quiet on 192.168.33.10
DEBUG[46168fa1] Command: cd /usr/share/nginx/www/capistrano_introduction/releases/20140704110147 && /usr/local/rvm/bin/rvm ruby-2.1.0@global do bundle install --binstubs /usr/share/nginx/www/capistrano_introduction/shared/bin --path /usr/share/nginx/www/capistrano_introduction/shared/bundle --without development test --deployment --quiet
DEBUG[46168fa1]         Warning, new version of rvm available '1.25.28', you are using older version '1.25.27'.
DEBUG[46168fa1]         You can disable this warning with:    echo rvm_autoupdate_flag=0 >> ~/.rvmrc
DEBUG[46168fa1]         You can enable  auto-update  with:    echo rvm_autoupdate_flag=2 >> ~/.rvmrc
INFO[46168fa1] Finished in 1060.831 seconds with exit status 0 (successful).
DEBUG[b68270f6] Running /usr/bin/env if test ! -d /usr/share/nginx/www/capistrano_introduction/releases/20140704110147; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction/releases/20140704110147'" 1>&2; false; fi on 192.168.33.10
DEBUG[b68270f6] Command: if test ! -d /usr/share/nginx/www/capistrano_introduction/releases/20140704110147; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction/releases/20140704110147'" 1>&2; false; fi
DEBUG[b68270f6] Finished in 0.007 seconds with exit status 0 (successful).
INFO[fc9d9c38] Running /usr/local/rvm/bin/rvm ruby-2.1.0@global do bundle exec rake assets:precompile on 192.168.33.10
DEBUG[fc9d9c38] Command: cd /usr/share/nginx/www/capistrano_introduction/releases/20140704110147 && ( RAILS_ENV=staging /usr/local/rvm/bin/rvm ruby-2.1.0@global do bundle exec rake assets:precompile )
DEBUG[fc9d9c38]         I, [2014-07-04T11:19:58.372780 #15974]  INFO -- : Writing /usr/share/nginx/www/capistrano_introduction/releases/20140704110147/public/assets/application-23bdc211cee2d27c9fc052849bc3f38a.js
DEBUG[fc9d9c38]         I, [2014-07-04T11:19:58.392645 #15974]  INFO -- : Writing /usr/share/nginx/www/capistrano_introduction/releases/20140704110147/public/assets/application-2ef026b2d075e39670cea2d1960c0fa0.css
INFO[fc9d9c38] Finished in 6.231 seconds with exit status 0 (successful).
DEBUG[75987b68] Running /usr/bin/env if test ! -d /usr/share/nginx/www/capistrano_introduction/releases/20140704110147; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction/releases/20140704110147'" 1>&2; false; fi on 192.168.33.10
DEBUG[75987b68] Command: if test ! -d /usr/share/nginx/www/capistrano_introduction/releases/20140704110147; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction/releases/20140704110147'" 1>&2; false; fi
DEBUG[75987b68] Finished in 0.005 seconds with exit status 0 (successful).
INFO[dc6d98e1] Running /usr/bin/env cp /usr/share/nginx/www/capistrano_introduction/releases/20140704110147/public/assets/manifest* /usr/share/nginx/www/capistrano_introduction/releases/20140704110147/assets_manifest_backup on 192.168.33.10
DEBUG[dc6d98e1] Command: cd /usr/share/nginx/www/capistrano_introduction/releases/20140704110147 && /usr/bin/env cp /usr/share/nginx/www/capistrano_introduction/releases/20140704110147/public/assets/manifest* /usr/share/nginx/www/capistrano_introduction/releases/20140704110147/assets_manifest_backup
INFO[dc6d98e1] Finished in 0.009 seconds with exit status 0 (successful).
DEBUG[50577f6b] Running /usr/bin/env if test ! -d /usr/share/nginx/www/capistrano_introduction/releases/20140704110147; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction/releases/20140704110147'" 1>&2; false; fi on 192.168.33.10
DEBUG[50577f6b] Command: if test ! -d /usr/share/nginx/www/capistrano_introduction/releases/20140704110147; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction/releases/20140704110147'" 1>&2; false; fi
DEBUG[50577f6b] Finished in 0.004 seconds with exit status 0 (successful).
INFO[8b0d586a] Running /usr/local/rvm/bin/rvm ruby-2.1.0@global do bundle exec rake db:migrate on 192.168.33.10
DEBUG[8b0d586a] Command: cd /usr/share/nginx/www/capistrano_introduction/releases/20140704110147 && ( RAILS_ENV=staging /usr/local/rvm/bin/rvm ruby-2.1.0@global do bundle exec rake db:migrate )
INFO[8b0d586a] Finished in 2.245 seconds with exit status 0 (successful).
INFO[4949d629] Running /usr/bin/env rm -rf /usr/share/nginx/www/capistrano_introduction/current on 192.168.33.10
DEBUG[4949d629] Command: /usr/bin/env rm -rf /usr/share/nginx/www/capistrano_introduction/current
INFO[4949d629] Finished in 0.005 seconds with exit status 0 (successful).
INFO[ff638da6] Running /usr/bin/env ln -s /usr/share/nginx/www/capistrano_introduction/releases/20140704110147 /usr/share/nginx/www/capistrano_introduction/current on 192.168.33.10
DEBUG[ff638da6] Command: /usr/bin/env ln -s /usr/share/nginx/www/capistrano_introduction/releases/20140704110147 /usr/share/nginx/www/capistrano_introduction/current
INFO[ff638da6] Finished in 0.008 seconds with exit status 0 (successful).
DEBUG[98b58b50] Running /usr/bin/env ls -x /usr/share/nginx/www/capistrano_introduction/releases on 192.168.33.10
DEBUG[98b58b50] Command: /usr/bin/env ls -x /usr/share/nginx/www/capistrano_introduction/releases
DEBUG[98b58b50]         20140704110147
DEBUG[98b58b50] Finished in 0.011 seconds with exit status 0 (successful).
DEBUG[6598af60] Running /usr/bin/env if test ! -d /usr/share/nginx/www/capistrano_introduction/releases; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction/releases'" 1>&2; false; fi on 192.168.33.10
DEBUG[6598af60] Command: if test ! -d /usr/share/nginx/www/capistrano_introduction/releases; then echo "Directory does not exist '/usr/share/nginx/www/capistrano_introduction/releases'" 1>&2; false; fi
DEBUG[6598af60] Finished in 0.004 seconds with exit status 0 (successful).
INFO[23b2a19a] Running /usr/bin/env echo "Branch master (at a6511ba) deployed as release 20140704110147 by k2works" >> /usr/share/nginx/www/capistrano_introduction/revisions.log on 192.168.33.10
DEBUG[23b2a19a] Command: echo "Branch master (at a6511ba) deployed as release 20140704110147 by k2works" >> /usr/share/nginx/www/capistrano_introduction/revisions.log
INFO[23b2a19a] Finished in 0.004 seconds with exit status 0 (successful).
```
動作確認
```bash
$ ssh deploy@192.168.33.10
$ cd /usr/share/nginx/www/capistrano_introduction/current
$ /usr/local/rvm/bin/rvm ruby-2.1.0@global do bundle exec exec unicorn_rails
```
_http://192.168.33.10_が確認できればOK
#### ロールバック
```bash
$ cap staging deploy:rollback
```
## <a name="2">進んだ機能</a>
### SSH Kitを使ったリモートコマンド
Capistranoは[SSHKit](https://github.com/capistrano/sshkit)を使ってリモートサーバーでコマンドを実行する。
### リモートファイルタスク
_remote_file_タスクは予め要求されるリモートファイルの存在を許可します。
### ロールフィルタリング
ロールフィルタを使うことで特定のロールにマッチするサーバーだけにCapistranoのタスクを制限することができます。
#### ロールフィルタを指定する
ロールフィルタを特定するには３つの方法がある。
##### 環境変数
`ROLES=app,web cap production deploy`
##### 設定ファイル
`set :filter, :roles => %w{app web}`
##### コマンドライン
`cap --roles=app,web production deploy`
### ホストフィルタリング
ホストフィルタを使うことで特定のホスト名にマッチするサーバーだけにCapistranoのタスクを制限することができます。
#### ホストフィルタを指定する
ホストフィルタを特定するには３つの方法がある。
##### 環境変数
`HOStS=server1,server2 cap production deploy`
##### 設定ファイル
`set :filter, :hosts => %w{server1 server2}`
##### コマンドライン
`cap --hosts=server1,server2 production deploy`
### Rubyスクリプト内のCapistrano
configフォルダとdeploy.rbを作る代わりに単一のrubyスクリプトにプログラマブルに設定できる。
```ruby
require 'capistrano/all'
stages = "production"
set :application, 'my_app_name'
set :repo_url, 'git@github.com:capistrano/capistrano.git'
set :deploy_to, '/var/www/'
set :stage, :production
role :app, %w{}
require 'capistrano/setup'
require 'capistrano/deploy'
Dir.glob('capistrano/tasks/*.rake').each { |r| import r }
Capistrano::Application.invoke("production")
Capistrano::Application.invoke("deploy")
```
## <a name="3">フレームワーク拡張</a>
### Ruby on Rails  
<div class="github-widget" data-repo="capistrano/rails"></div>  

### Bundler
<div class="github-widget" data-repo="capistrano/bundler"></div>  

### Rbenv & RVM & Chruby
<div class="github-widget" data-repo="capistrano/rbenv"></div>  
<div class="github-widget" data-repo="capistrano/rvm"></div>  
<div class="github-widget" data-repo="capistrano/chruby"></div>  

### Plugins
<div class="github-widget" data-repo="bruno-/capistrano-postgresql"></div>  
<div class="github-widget" data-repo="bruno-/capistrano-unicorn-nginx"></div>  
<div class="github-widget" data-repo="bruno-/capistrano-rbenv-install"></div>  
<div class="github-widget" data-repo="bruno-/capistrano-safe-deploy-to"></div>  
<div class="github-widget" data-repo="scottsuch/capistrano-graphite"></div>  

# 参照
+ [Capistrano](http://capistranorb.com/documentation/overview/what-is-capistrano/)
+ [capistrano/capistrano](https://github.com/capistrano/capistrano)
+ [VAGRANT](http://www.vagrantup.com/)
+ [Chef-solo+knife-solo+Vagrantでサーバ構築を自動化してみる - その１ ユーザー追加](http://straitwalk.hatenablog.com/entry/2013/08/25/000935)
+ [Ruby - RVM の Multi-User mode について](http://babiy3104.hateblo.jp/entry/2013/08/30/103602)
+ [How To Deploy Rails Apps Using Unicorn And Nginx on CentOS 6.5](https://www.digitalocean.com/community/tutorials/how-to-deploy-rails-apps-using-unicorn-and-nginx-on-centos-6-5)
+ [Rails 3 + Nginx/Unicorn を Amazon AWS に Capistrano 3 でデプロイする](http://bekkou68.hatenablog.com/entry/2013/12/21/173641)
+ [[Ubuntu]ssh 接続しようとすると「WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!」が表示されるときは](http://d.hatena.ne.jp/PRiMENON/20090418/1240062291)
+ [Why wont bundler install json gem?](http://stackoverflow.com/questions/21095098/why-wont-bundler-install-json-gem)
