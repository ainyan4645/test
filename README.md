## 開発用コンテナ設定

### docker-compose.ymlの設定メモ
#### ① nginxとphpのvolumesで /var/www/html を使う理由
歴史的な慣習：Apache や Nginx の Linux ディストリビューション（Debian, Ubuntu など）は、デフォルトでドキュメントルートを /var/www/html にしていることが多いです。

PHP-FPM のデフォルト設定：PHP-FPM のサンプル設定や多くのフレームワークの Docker テンプレートでも /var/www/html が基準になっています。

👉 つまり「必ず /var/www/html にしないと動かない」わけではないですが、周りの設定ファイルとの相性を考えると /var/www/html にしておいた方が混乱が少ない、という理由です。

もし default.conf 側で root /var/www; と書けば /var/www にしても全然OKです 👍


#### ② それぞれのイメージに container_name: を入れる・入れないの違い
入れない場合

Docker が自動で プロジェクト名_サービス名_番号 のような名前をつけます
例：myapp_php_1

複数人で開発するときにプロジェクト名が違うと、自動的に名前がユニークになるので競合しにくい

入れる場合

名前を固定できる（例：常に php という名前のコンテナになる）

docker exec -it php bash のように短く呼び出せて便利

ただし、同じ名前のコンテナは2つ立ち上げられないので、複数プロジェクトを並行で動かすと競合することがある


👉 開発環境なら「固定したい → container_name つける」でもいいし、「チーム開発で他のプロジェクトとぶつかるかも → つけない」のもアリです。


#### ③mysqlのvolumes: ./docker/mysql/data:/var/lib/mysql の懸念点
./docker/mysql/data は ホスト側のフォルダに MySQL の生データファイルを保存します

MySQL は内部でバイナリファイルを使っているため、OS や MySQL のバージョンが変わると不整合が起きやすいです

開発用なら問題ないけど、本番や長期運用では破損のリスクがある

一方で、Docker Volume を使うと：

Docker が管理する専用の領域に保存される

OS やファイルパーミッションの違いを吸収してくれる

バージョンアップにも比較的強い

👉 なので「学習用・実験用なら ./docker/mysql/data でもいい」、「安定運用や本番を想定するなら Docker Volume 推奨」です。


Docker Volume にする場合の例：
```yaml
volumes:
      - db_data:/var/lib/mysql
      - ./docker/mysql/my.cnf:/etc/mysql/conf.d/my.cnf
    ports:
      - "3306:3306"
```
Bindマウントにする場合の例：
```yaml
volumes:
    - ./docker/mysql/data:/var/lib/mysql
    - ./docker/mysql/my.cnf:/etc/mysql/conf.d/my.cnf
```

##### ③ Docker Volume 採用時の不都合について
結論から言うと 👉 学習で困ることはほぼありません。

✅ できること（Bindマウントと同じ）
phpMyAdmin から普通にテーブル・データを見られる

docker exec -it mysql bash → mysql -u laravel_user -p でログインできる

データはコンテナを再起動しても残る

⚠️ 違い（Bindマウントと比較）
ホストのフォルダに直接ファイルが見えない
→ ./docker/mysql/data に ibdata1 や *.ibd みたいなMySQLのデータファイルが見えなくなる

「ローカルのMySQLデータを直接テキストエディタで触る」みたいなことはできない（ただし普通そんなことはしない）

バックアップ取りたい時は

```bash
docker exec mysql mysqldump -u root -p laravel_db > backup.sql
```
のように mysqldump を使う

✅ 学習用におすすめの判断
「データファイルをホストで直接見たい」→ Bind マウント（./docker/mysql/data）

「シンプルに動いてくれればいい」→ Volume（db_data:/var/lib/mysql）

#### ④ 本番環境での phpmyadminの　PMA_USER / PMA_PASSWORD の扱い
本番では phpMyAdmin 自体を基本的に使いません（セキュリティリスクが高い）

もしどうしても必要なら 環境変数は .env ファイルに書き、docker-compose.yml からは参照だけにします

例：.env
```env
PMA_USER=laravel_user
PMA_PASSWORD=laravel_pass
```
例：docker-compose.yml
```yaml
phpmyadmin:
  image: phpmyadmin/phpmyadmin
  environment:
    PMA_HOST: mysql
    PMA_USER: ${PMA_USER}
    PMA_PASSWORD: ${PMA_PASSWORD}
```
👉 こうすれば GitHub に docker-compose.yml を公開しても、パスワードが漏れません。

### Dockerfileの設定メモ
#### Composer の自己アップデート
composer self-update は必ずしも必要じゃない（最新イメージの installer がすでに新しいため）

学習用なら残しておいてもOK、本番では再現性のため削る人も多いです。

#### rm -rf /var/lib/apt/lists/*
これ最後に足すと、不要なキャッシュが消えてイメージが少し軽くなる
```dockerfile
RUN apt-get update \
    && apt-get install -y default-mysql-client zlib1g-dev libzip-dev unzip \
    && docker-php-ext-install pdo_mysql zip \
    && rm -rf /var/lib/apt/lists/*
```

#### 作業ディレクトリの統一

WORKDIR が docker-compose.yml のマウントと揃っていれば問題なし。

Nginx の root が /var/www/ になっているか確認してください。

### php.iniの設定メモ
#### mbstring 設定について

PHP 7.2 以降は mbstring.internal_encoding 自体が廃止されたのでコメントアウトで正解 👍

mbstring.language = "Japanese" も実は使うことがほぼなく、Laravel 側で utf-8 を前提にするので不要です。

→ 今のままでも問題なし。シンプルにしたければ消してOK。

### default.confの設定メモ
#### include fastcgi_params; を include fastcgi.conf; に変更

fastcgi_params だと SCRIPT_FILENAME が2重定義になることがあります

PHP公式イメージの例でも fastcgi.conf を使うのが一般的

```nginx
include fastcgi.conf;
fastcgi_param PATH_INFO $fastcgi_path_info;
```

#### セキュリティ面

.htaccess や .env ファイルをブラウザから見られないようにブロックしておくのが本番用ではベストプラクティスです

例:

一番下に追加
```nginx
location ~ /\. {
    deny all;
}
```

### my.cnf設定メモ
本番では bind-address や skip-networking の設定を入れて、外部アクセスを制限することがあります

## 動作確認手順
1. Docker Compose ビルド
```bash
docker-compose build
```
php コンテナに必要な PHP 拡張や Composer がインストールされているか確認

1と2を同時に実行するなら
```bash
docker-compose up -d --build
```

2. コンテナ起動

```bash
docker-compose up -d
```
-d でバックグラウンド起動<br>
起動後、コンテナの状態を確認

```bash
docker-compose ps
```
nginx, php, mysql, phpmyadmin が Up になっていればOK

3. Laravel プロジェクト作成
10系の最新安定版
```bash
docker-compose exec php composer create-project laravel/laravel:^10.0 .
```
./src に Laravel プロジェクトが作成される<br>
（PHP-FPM コンテナ内で Composer を実行しているので依存関係も正しくインストールされる）

8系固定なら
```php
composer create-project "laravel/laravel=8.*" . --prefer-dist
```
Laravel はバージョンごとにルーティング、コントローラの書き方、認証周りなど結構変わります。

例えば Laravel 9 以降は ルート定義で use 文が不要になったり、PHP の要求バージョンも上がったり します。

教材どおりに進めるなら 8系固定のほうが無難
（採点システムや動作検証も「8系前提」になっている可能性があります）

新しい環境に慣れたいなら 10系
（ただし教材のコードと違う部分が出てくる）

4. Nginx / PHP-FPM 接続確認

ブラウザで確認
```arduino
http://localhost
```
Laravel の初期画面が出ればOK<br>
出ない場合は nginx のログを確認
```bash
docker-compose logs nginx
```

5. MySQL 接続確認

phpMyAdmin から
```arduino
http://localhost:8080
```
ユーザー名：laravel_user<br>
パスワード：laravel_pass<br>
ホスト名：mysql<br>
データベース：laravel_db<br>
正しくログインできればOK

bash から直接
```bash
docker-compose exec mysql bash
mysql -u laravel_user -p
# パスワード: laravel_pass
```
SHOW DATABASES; で laravel_db が見えればOK

6. Laravel DB 接続確認

.env を編集（DB設定を Docker に合わせる）
```env
DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=laravel_db
DB_USERNAME=laravel_user
DB_PASSWORD=laravel_pass
```
マイグレーションを実行して確認
```bash
docker-compose exec php php artisan migrate
```
エラーが出ずにテーブルが作成されればOK

7. 停止・削除（必要に応じて）

```bash
docker-compose down
```
コンテナ停止＋削除

ネットワーク削除

ただし イメージやボリュームは残る

→ MySQL データ (./docker/mysql/data) は Bind マウントなので残る

→ php ビルドイメージはキャッシュが残る

👉 開発中は基本これで十分

```bash
docker-compose down --rmi all --volumes --remove-orphans
```
イメージ削除（--rmi all）

ボリューム削除（--volumes）

孤立したコンテナ削除（--remove-orphans）

👉 「完全初期化」したい場合はこちらを実行すると、毎回まっさらな状態でテストできます。
ただし MySQL のデータも消えます（Bind マウントならホスト側に残るけど）。

```bash
docker system prune -a
```
未使用のイメージ / コンテナ / ネットワーク / キャッシュを全削除

強力すぎるので基本は使わない

ディスク容量が膨らんできたときの大掃除用

### error対処
`docker-compose up -d --build` で
```vbnet
! phpmyadmin The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8) and no specific platform was requested
```
あなたの MacBook Air (M1/M2 → ARM64) と、公式 phpMyAdmin イメージ（phpmyadmin/phpmyadmin → amd64優先）のアーキテクチャが違うために出る警告。

ただし `docker-compose ps` で

phpMyAdmin コンテナは 正常に起動してポート8080を公開できればOK。<br>
→ 実際にブラウザで http://localhost:8080 を開けるなら 動作上は問題なし。

開けなければ、<br>
「docker-compose.yml」の「phpmyadmin」サービスに以下を追加：
```yaml
phpmyadmin:
  image: phpmyadmin/phpmyadmin
  platform: linux/arm64/v8   # ← ここを追加
  ports:
    - "8080:80"
```
👉 これで警告は消えるはず。再ビルドしてみる。

`docker-compose up -d --build`で最初の方に出る警告
```vbnet
WARN[0000] /Users/ainyan4869/coachtech/test/docker-compose.yml: the attribute version is obsolete, it will be ignored, please remove it to avoid potential confusion
```
Docker Compose の v2 以降 では、docker-compose.yml の version: '3.9' は 不要 になっています。

書いてあっても無視されるため、警告が出ています。



### 実際のテスト用md
```
docker-compose up -d --build

docker-compose exec php bash

composer -v

composer create-project "laravel/laravel=8.*" . --prefer-dist

ls
```
