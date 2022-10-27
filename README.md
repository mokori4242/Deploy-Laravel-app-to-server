## ConohaVpsでLaravel環境を立ち上げた後の流れ
LAMP環境を1から構築しようと思ったが、phpインストール後下記エラーが表示して断念、解決できず...。

PHP Warning: PHP Startup: ^(text/|application/xhtml\+xml) (offset=0): unrecognised compile-time option bit(s) in Unknown on line 0

下記ページなど参考にするが解決できず

https://github.com/oerdnj/deb.sury.org/issues/1682

## 一般ユーザーを作成する
下記ページを参考に一般ユーザーを作成しアクセス出来るようにする。
https://cgpipeliner.info/2022/03/28/conoha-vps/

## UbuntuにLet's EncryptのSSL/TSLサーバ証明書を取得する
- rootユーザーに変更
[@.com ~]$ su

- certbotのインストール
subo apt install certbot

- ポート開放する:

sudo ufw allow 22

sudo ufw allow 80

sudo ufw allow 443

sudo ufw allow 5000

sudo ufw reload

sudo ufw status

- confファイルを下記に書き換え
cd /etc/apache2/sites-available/

vi 000-default.conf 

vi default-ssl.conf

ServerName 取得したドメイン名

DocumentRoot /var/www/html※希望のドキュメントルート

ServerAdmin 管理したいメールアドレス

a2ensite 000-default //有効化する

service apache2 restart //再起動する

- SSL証明書取得:
certbot certonly --webroot -w ドキュメントルート -d ドメイン名

メールアドレス登録後、規約に同意してそのほかの質問に答える。

- SSL証明書取得時の作成ファイル

/etc/letsencrypt/live/ドメイン名/cert.pem

/etc/letsencrypt/live/ドメイン名/privkey.pem

/etc/letsencrypt/live/ドメイン名/chain.pem

- SSLに関するモジュールを有効化する:
a2enmod ssl

- 設定ファイルを編集　以下修正箇所
vi default-ssl.conf

ルートディレクトリとして公開するディレクトリのパスへ修正:

DocumentRoot = /var/www/html/アプリのディレクトリ名/public

取得したサーバ証明書と公開鍵のパスに変更:

SSLCertificateFile      /etc/letsencrypt/live/ドメイン名/cert.pem

取得した秘密鍵のパスに変更:

SSLCertificateKeyFile   /etc/letsencrypt/live/ドメイン名/privkey.pem

コメント解除して取得した中間証明書のパスに変更:

SSLCertificateChainFile /etc/letsencrypt/live/ドメイン名/chain.pem

Esc→:wqで保存して終了

- サイト設定を有効化する
a2ensite default-ssl

- Apache2の再起動
systemctl restart apache2

- この時、通常のHTTP接続に対してHTTPSへのリダイレクトをかけることで常にSSL/TLS通信を行うことも可能となる。以下、設定ファイルに追記。

vi /etc/apache2/sites-available/000-default.conf

↑設定ファイルを編集。通常のHTTP用サイト設定に追記します。(HTTPでアクセスした際にhttpsに変換する)

<VirtualHost *:80>

　　～

　　～

RewriteEngine On

RewriteCond %{HTTPS} off

RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [R=301,L]

↑3行をVirtualhost内最下部にでも追記

</VirtualHost>

- 証明書をcronで自動更新
vi /etc/crontab

末行に以下を追記:

0 0 * * * root /usr/bin/cerbot renew

## ローカルで作成したLaravelアプリをサーバーへデプロイ
- ローカルで作成したLaravelアプリをgitにプッシュする
npm run prodを実行後にgitへpushする。

- git clone SSHかHTTPSでクローン
httpsの場合はアクセストークン作成の必要あり。

求められるパスワードはアクセストークンを入力(一般用でよさそうclassic)

- envファイルを作成:
下記コマンドでenvファイルをコピーして作成。

cp .env.example .env

- envファイルに本番環境用の設定を入力
DBなどの設定を変更する。

- サーバー上のMysqlにアプリで使用するDB・ユーザーを作成
下記SQL文を実行。

CREATE DATABASE データベース名 CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;

create user laravel@localhost identified by '任意のパスワード';

- mod_rewrite を有効にする
a2enmod rewrite

- seedコマンドでデータベースにデータをmigrate
本番環境に必要な情報にseederを書き換える

php artisan migrate:refresh --seed

を実行

## エラー対応
- The stream or file "" could not be opened in append mode: Failed to open stream: Permission denied The exception occurred while attempting to log: The stream or file

- 実行フォルダに移動して権限を与える
chmod 777 -R storage/
