# 概要と補足
- Hubsのバックエンド環境（の一部）をローカルで動かす方法です
- ローカルで管理画面を触りたいとき、管理画面を改修するときに使えます
- Reticulumの改修にも使えるはずです（デプロイは未検証）
- 一度環境を作ればコマンド一つで実行できます
- 作った環境は他の人と使いまわせます（※証明書のChromeへのインポート、Hubsクライアントへの配置は各自で必要です）
- 現状では他のユーザーと動画、音声のやりとりはできません（どこかいじれば動くようにできるかもしれません）
- Spokeもローカルで動かせるようですが未検証です
- WSL2ベースで動かす場合は、WSLのディレクトリ（`\\_wsl_$`）上に構築しないとI/Oが劇遅になります

# 必要なもの
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) （Windowsでは Hyper-V or WSL2 ベース）
- ----- or -----
- [Rancher Desktop](https://rancherdesktop.io/)（Windowsでは WSL2 ベース）


# 作業手順
1. Reticulum、Dialogをディレクトリに配置
2. 秘密鍵、公開鍵、自己署名証明書を作成
3. 秘密鍵、自己署名証明書を配置
4. Reticulum起動用のDockerfileを作成
5. docker-compose.ymlを作成
6. 環境変数ファイルを作成
7. Reticulumのソースコードを変更
8. hubs.localとhubs-proxy.localをループバックアドレスに紐づけ
9. Reticulumをビルドして起動
10. PostgRESTのSecretを指定
11. Reticulumを自動起動するように設定
12. コンテナを再起動
13. Hubsクライアントを起動
14. 作成した証明書をChromeにインポート
15. ブラウザで表示を確認
16. ユーザーを登録
17. ユーザーのadminフラグを有効化
18. 使用方法


# 構築後のディレクトリ構成例
```
hubs-backend/
  ┠reticulum/
  ┃  ┠リポジトリからcloneされたファイル
  ┃  ┠Dockerfile
  ┃  ┠priv
  ┃  ┃  ┠dev-ssl.cert
  ┃  ┃  ┗dev-ssl.key
  ┃  ┗storage/
  ┃     ┗dev/
  ┠dialog/
  ┃  ┗リポジトリからcloneされたファイル
  ┠certs/
  ┃  ┠server.key
  ┃  ┠server.pem
  ┃  ┗pub.key
  ┠tmp/
  ┃  ┗db/
  ┠env/
  ┃  ┠reticulum.env
  ┃  ┗postgrest.env
  ┗docker-compose.yml

// 任意の場所でOK
hubs/
  ┠リポジトリからcloneされたファイル
  ┠certs
  ┃  ┠cert.pem
  ┃  ┗key.pem
  ┗admin/
     ┗certs/
        ┠cert.pem
        ┗key.pem
```
## ディレクトリの補足
- reticulum/: [Reticulum](https://github.com/mozilla/reticulum) のソースコード。動画・音声以外の部分を処理するPhonenixベースのWebAPIサーバー
- reticulum/storage/: AWSのEFS相当。インポートしたシーンやアバター、ルーム内でアップロードしたファイルなどが格納されるディレクトリ
- dialog/: [Dialog](https://github.com/mozilla/dialog) のソースコード。動画・音声を処理するWebRTCサーバー
- certs/: 秘密鍵、公開鍵、自己署名証明書など
- tmp/db/: PostgreSQLのデータ置き場
- env/: ReticulumとPostgRESTの環境変数ファイル
- hubs/: hubsのクライアント側ソースコード


# 1. Reticulum、Dialogをディレクトリに配置
ディレクトリを作成しReticulum、DialogをCloneします。
```
$ mkdir hubs-backend && cd hubs-backend
$ git clone https://github.com/mozilla/reticulum.git
$ git clone https://github.com/mozilla/dialog.git
```


# 2. 秘密鍵、公開鍵、自己署名証明書を作成
certsディレクトリを作成し、san値を指定した証明書を作成します。
```
$ mkdir certs && cd certs
### 秘密鍵生成
$ openssl genrsa -out server.key 2048
### 公開鍵生成
$ openssl rsa -in server.key -pubout -out pub.key
### CSR生成 （証明書情報を聞かれるので適当に入力）
$ openssl req -new -key server.key -out server.csr
### SAN設定テキスト生成
$ echo "subjectAltName = DNS:hubs.local, DNS:hubs-proxy.local" > san.txt
### 証明書生成
$ openssl x509 -req -days 3650 -in server.csr -signkey server.key -extfile san.txt -out server.pem
### 環境変数設定用ファイル生成
$ sed -e :loop -e 'N; $!b loop' -e 's/\n/\\n/g' server.key > key.txt
```


# 3. 秘密鍵、自己署名証明書を配置
作成した証明書、秘密鍵をそれぞれ（リネームして）配置します。
- Reticulum
  - server.pem を reticulum/priv/dev-ssl.cert にリネームして配置
  - server.key を reticulum/priv/dev-ssl.key にリネームして配置
- Dialog
  - コンテナに直接マウント＆環境変数で証明書、秘密鍵、公開鍵を直で指定しているので対応不要
- Hubs（**※ここの証明書がデプロイ時にどう扱われるかわかっていないので、デプロイ時の処理は要確認**）
  - server.pem を hubs/certs/cert.pem にリネームして配置
  - server.key を hubs/certs/key.pem にリネームして配置
  - server.pem を hubs/admin/certs/cert.pem にリネームして配置
  - server.key を hubs/admin/certs/key.pem にリネームして配置


# 4. Reticulum起動用のDockerfileを作成
Reticulumコンテナ起動用の`Dockerfile`をreticulumディレクトリ内に作成します。

Reticulumは（現状）Elixir 1.8でしか動かないので、バージョン1.8のイメージを使います。
- reticulum/Dockerfile
```
FROM hexpm/elixir:1.8.2-erlang-22.3.4.23-ubuntu-focal-20210325

# ディレクトリの設定
ARG ROOT_DIR=/ret
RUN mkdir ${ROOT_DIR}
WORKDIR ${ROOT_DIR}

# ディレクトリのコピー
COPY ./ ${ROOT_DIR}

# 依存するライブラリのインストール
RUN apt-get update && apt-get install -y git inotify-tools
RUN mix local.hex --force && mix local.rebar
```


# 5. docker-compose.ymlを作成
コンテナを立ち上げるためのdocker-compose.ymlを作成します。
- docker-compose.yml
```
version: '3'
services:
  ret:
    build:
      context: ./reticulum
      dockerfile: Dockerfile
    env_file:
      - ./env/reticulum.env
    volumes:
      - ./reticulum:/ret
      - ./reticulum/storage/dev:/storage/dev
    tty: true
    ports:
      - "4000:4000"
    depends_on:
      - db
  dialog:
    build:
      context: ./dialog
      dockerfile: Dockerfile
    environment:
      - "HTTPS_CERT_FULLCHAIN=/app/certs/server.pem"
      - "HTTPS_CERT_PRIVKEY=/app/certs/server.key"
      - "AUTH_KEY=/app/certs/pub.key"
    tty: true
    volumes:
      - ./certs:/app/certs
    ports:
      - "4443:4443"
  db:
    image: postgres:11
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
        - ./tmp/db:/var/lib/postgresql/data
    ports:
      - "5432:5432"
  postgrest:
    image: postgrest/postgrest
    env_file:
      - ./env/postgrest.env
    volumes:
      - ./reticulum:/ret
    ports:
      - "3000:3000"
    depends_on:
      - db
```

## 各コンテナの説明
- ret : Reticulumを動かすためのコンテナ。先に作成したDockerfileを使用します。
- dialog : Dialogを動かすためのコンテナ。Dockerfileはリポジトリからクローンしたものをそのまま使います。
- db : PostgreSQLのコンテナ。hubs推奨のバージョン11のイメージを使います。データ保存用ディレクトリとして`tmp/db`をマウントしてデータ永続化します。
- postgrest : PostgreREST（PostgreSQLをRESTfulAPIするサーバー）起動用のコンテナ。管理画面のCRUDで必要です。


# 6. 環境変数ファイルを作成
ReticulumとPostgRESTの環境変数ファイルを作成します。

reticulum.envの`PERMS_KEY`の値には先に生成した`certs/key.txt`内の文字列を指定します。
- env/reticulum.env
```
MIX_ENV=dev
DB_HOST=db
HUBS_ADMIN_INTERNAL_HOSTNAME=host.docker.internal
HUBS_CLIENT_INTERNAL_HOSTNAME=host.docker.internal
SPOKE_INTERNAL_HOSTNAME=host.docker.internal
DIALOG_HOSTNAME=hubs.local
DIALOG_PORT=4443
POSTGREST_INTERNAL_HOSTNAME=host.docker.internal
### key.txtの中身を貼り付ける（以下例）
PERMS_KEY=-----BEGIN PRIVATE KEY-----\nMIIEvwIBADANBgkqhkiG9w0BAQEFAASCBKkwggSlAgEAAoIBAQDdzTLxivNPfE7U\nLeoWZEmdrr7TO9OezbBu/RSNoXIF4IEAnHvkaeSGtNEXvp4wAZ0Fc352wmOTK/eO\nJvwxTMdx1OgmFjENdzmE0bBIhP/RYnGfvmQU6N694i1hGc3QWCZ0pg4aEFVcvLul\nKtysIoi3FgdCigLPleMPFaQqbGw6nSSNeL8ovV7DDBojn2URzkO26a9PJk0dFXFy\nb41exish+VLIy+Kj2umzxA2Q2q7jkGaXmDLaffN+FiKKEPO/4oXCG6767s5my9HS\ndj+l0VttZjGR2hG3ZXjlN7kAiOkmL4ziLrDGi27edvdHdvLWw1F/AkH2ACIXpyEx\n17iJVC8vAgMBAAECggEARhLLMnaEsisCuJQc7aAXheKsVQ4EqJoxUI9STACihmrM\nNsY5egtecJR1rrvBPEd2NT+sx5ZYYSm110pHyMeKB1ONWGMQckGlgWnB+NdT/vHM\nFfzPn6+Gl5T6Y0AEvvrqg1XnBKL+ZQMXgXIOL6/lY3ppJsN1aeHfz2c161U/yDBA\nqEcLL/fkGwWd48zTxXtV35sQ6YjpPZNJqGi/BzuzLSM/3rd/KHQa8mF7kfFofjdc\nDCEZoKObRL0DVg6dKqcLro4g9TcDycFBcKXp/xIHFPVZuzCVKsXgcM7P9wAymyTv\nGLNM98TWt5F/ZKkhxh+YtW62oLqcJpGgAvoXvBZTQQKBgQD4LJFYQ1y0/+pq6qBg\nWqDPyDYNtyez9Yp2e8iLKYYye7TBYJ/hPG5ft4oC/XaUanWMhaRtgSmZxRpbD+b9\nw0wKX+4/Gi1atrKobEVa9pAQxojdLtTcP7P0xehFAZ+uzQ2XkYDw+mrQVu34HUmr\ncAemcjPirSce0pL9JkOFpPJOjwKBgQDky7vmKQIzNA6SEBMEPeu+zFR+tP2cpcMZ\nL2xyUGrVMOoP7gi+k4k7NQjk9W0/XCmGW9OHNbrsum04MCPpe7N/HCyHMUinbCHG\n8c2Aj05FqChPTGcEcHKOHc+o8P/VoB75u9JWwCYgrpQZqMEyyX5Rux+N91cXdVOP\n0PJeKhplYQKBgQDTpo6+S5TA+JCfZkIoaqJDEminAdDmdT4FFkROCrfcTgw174Xq\nvXMURp95NVjv8msV0NQOX91uut5jUwiY2vn6zx2My3Jqru1nHg94KFdtNyR6HfqD\nbAC5fB8+2seoeFBjN0BUQ/zceYax16iAgSbFGRiO9UUr0EJzizKCw82ymQKBgQCo\nyZRI65/v3kuRfcJJstSj4WHESAmA8tjHT7EVdDYcKQXD0rsovPbUcv+oyVZbE8ae\nlEesy/qmgbHpbbpQmS2GbGZ9JeKPgLo6ZlBQs2fvA2sZdSfaooogReXYoFJcas/v\nNJyr2v1FxwUDGPfOW8+QxBc3SG07pRaMVn028qCegQKBgQDR0a+myTm2WTP+tGL3\n7XuJ9Q13iKf9IFqwLZ5ExHeP9IAwm1KrTNMibERrs0ZHmRkW7szbrdPr1qyZemqL\nREsi/YUcBuDboVvEmF68yvcWtLaB9nCE5DmntadUtLOBD4IvZV2nXcYdTULT2VM+\nPA5w5S0p6gVPmb5ysAEXzKyicQ==\n-----END PRIVATE KEY-----

```
postgrest.envの`PGRST_JWT_SECRET`の値はReticulumを起動してから生成する必要があるのでひとまず空にしておきます。
- env/postgrest.env
```
PGRST_DB_URI=postgres://postgres:postgres@db:5432/ret_dev
PGRST_DB_SCHEMA=ret0_admin
PGRST_DB_ANON_ROLE=postgres
# ひとまず空にしておく
PGRST_JWT_SECRET=''
```


# 7. Reticulumのソースコードを変更
リクエストヘッダを指定している部分を変更します。
- reticulum/lib/ret/http_utils.ex
```
  defp retry_until_success(verb, url, body, options) do
    # 追加
    default_headers =
      if System.get_env("MIX_ENV") == "dev" do
        [{"Host", "hubs.local"}]
      else
        []
      end

    default_options = [
      # 変更
      # headers: [],
      headers: default_headers,
      cap_ms: 5_000,
      expiry_ms: 10_000,
      append_browser_user_agent: false
    ]
```


# 8. hubs.localとhubs-proxy.localをループバックアドレスに紐づけ
hostsファイルで`hubs.local`と`hubs-proxy.local`をループバックアドレスに紐づけします。
- (Windowsの場合) C:/Windows/System32/drivers/etc/hosts
- (macOSの場合) /private/etc/hosts
```
# 追加
127.0.0.1	hubs.local
127.0.0.1	hubs-proxy.local
```


# 9. Reticulumをビルドして起動
Dockerコンテナを立ち上げてからReticulumのコンテナに入り、Reticulumをビルドして起動します。
```
### hubs-backend直下で
$ docker compose up -d
$ docker compose exec ret bash

### パッケージインストール&ビルド
/ret# mix local.hex --force
/ret# mix deps.get
/ret# mix ecto.create

### 起動
/ret# iex -S mix phx.server
```


# 10. PostgRESTのSecretを指定
`iex(1)>`が表示されたら（表示されない場合はEnterを押下）続けて以下を入力してJWKを生成します。
```
iex(1)> jwk = Application.get_env(:ret, Ret.PermsToken)[:perms_key] |> JOSE.JWK.from_pem(); JOSE.JWK.to_file("reticulum-dev-jwk.json", jwk)
```
reticulumのディレクトリ直下に`reticulum-dev-jwk.json`が生成されるので、ファイルの内容をpostgrest.envの`PGRST_JWT_SECRET`の値に指定します。
- env/postgrest.env
```
PGRST_DB_URI=postgres://postgres:postgres@db:5432/ret_dev
PGRST_DB_SCHEMA=ret0_admin
PGRST_DB_ANON_ROLE=postgres
# 変更（例）
PGRST_JWT_SECRET='{"d":"DW37fiQEEy54lURDbxMA2otNV-KfAlNMV8I4MIYsJB7yVDrx7oRP43zLLYdKwjxHNbx18QQsZd8BVvdNRs_L_VTrAF-lYJjXSoaDKiBPa2uG0Rj18ntT_R34_Mjzi8JAegAqYpBkgFJA0M_wO0pJPxz8v2zP5QE0etJseYn6tx6wZ5ux83wmcGBJIHmI5sKUnQfIHsBgYpZ6TqO3NYe7aqFKR5qXHL8YItXzFP-31YLdu0xTZHZ35lQmJgr2vENaWpV7f4_NfBZwAb-p-B-wcRsJWQEH2_i3lYGKD3Ept91nB8Xr7TBW03Zsx1on-Rjg9U4ynCnGBXqZDSZU0lB4IQ","dp":"BtJwoIyrBUncns_3fs3cKJe4Y4YGbr5-sBS61tMRnMPoUhXlyHS9-JGsAwz8Qi48di76rzwQKlVbX8L5Wc7GFLQfSK8Uc8ACIjb4px9lEhSIJf2ljgS9C1KrdXnS8pzeTR6o81Ln9xhhy83tvywIRpgPvhJmdm1dsa18kJXH3aE","dq":"TzQI32svuUKI6o3bYd7kZe6ig4PaIGpzu__j2YpRoT9IpIdaM5Cqdho3w83lWWez3ZAWFvI_kwkRIfhMjBmZqU92xutrg6eJiwL6Aal5FryJu0zSz2WtayMtoYPnM8ss7K58UZKJwTGWUj-zUGFyNGxpRZSzddsgOTUgKIwxaaE","e":"AQAB","kty":"RSA","n":"-UKMzigJOM7BRJMVDLglhcZtIIzceIS5iI9B9QGSHDWJw6B0r-eGLyzH-vWCGsFkRP8t2uNR20V4kQxFLIVQfC_f50BvYPMDps9UXVfD8ynZnrAbgoVvd0LH3euUK6F55FntyGA2koGRlg8CxYeK8LgkR75o9738qrtvk21G6bTxfZKYS0tjXMa7qPGGc6FR7l68R98QFGCl7i00Y76UR2xSubkp-sOySWOPK2fPdnzgBnM2SfgaAfjwednOY2Y5wO0Doq5Hn94t3LWTCiihftcMfRMf5p-XamHcHIj5gYhn6r7YiJ_WU7pmXvoQziyhzhmfp9CNeqJuggfiaiKpEw","p":"_lIMqtrzLdpqsSjRg4vvdxl7FZyuRghc_UtGH7tUughQFwosnHjtQqRYD0oqZkwbcAvGqCeFchi2eqmp6CLuvaB4sNXvhSnr79n5UK4wj4YA8yfVbiGluXpvRmArv-aBBIIxzLiK6_R67jG8l7QyecmWTJ3uEEjT4iyYVDf71jE","q":"-ufx_EnW347G8W6_2R-Oz1UKt9AOtDoP60GIVU-IKWQbXqOF-cx_lrpICyhOGRuvSRteIcLm67bLq2CZsawftttfgoCGbFZ8kRU_N0gzetKqkM-q0WEsXBKFgLZCBpAlHwcKfoA9OY6lK6HTdBSfdDPYOpRn4ICR-L2XbtQrboM","qi":"1JgJYXFNpkmYpOXv5y4AhTT2jPC-y9BzAJzuUp8HDcUHQ1BqO8UJi6CkQSoeetQZnzHuNeina6oGia6OULBqwUDSapN_8aM0FNogM5jWVqn52ZB2P1T1ZUq6_yPYeOtYz8k-hFqUBzJxKkN1b4NSuSTwWkVr-hdTbyoAtP_CmkI"}'
```


# 11. Reticulumを自動起動するように設定
Dockerコンテナ立ち上げ時にReticulumが自動的に起動するようにdocker-compose.ymlにコマンドを追加します。
- docker-compose.yml
```
services:
  ret:
    ### 略...
    ### 追加
    command: >
      bash -c "mix local.hex --force && mix deps.get && mix ecto.create && iex -S mix phx.server"
```


# 12. コンテナを再起動
コンテナを再起動して環境変数を反映させます。
```
$ docker compose down
$ docker compose up -d
```


# 13. Hubsクライアントを起動
hubs直下とhubs/adminで `npm run local` を実行してhubsを起動します（※事前にhubs/adminでも `npm ci` の実行が必要です）


# 14. 作成した証明書をChromeにインポート
`certs/server.pem` をChromeにインポートします。
[Google Chromeへ証明書ファイルをインポートするには](https://jp.globalsign.com/support/ssl/config/cert-import-chrome.html)


# 15. ブラウザで表示を確認
https://hubs.local:4000?skipadmin をブラウザで開きトップページが表示されること、ルームに入れることを確認します。


# 16. ユーザーを登録
トップページ右上の「Sign in/Sign up」からサインインを実行します（メールは送信されないので、メールアドレスはなんでもOK）。

メール送信完了画面が出たらDockerのログにメール文面が出力されます。

以下コマンドを実行してログを確認し、ログ中にある `https://hubs.local:4000/?auth_origin=` から始まるURLをブラウザで開き認証を完了させます。
```
### ログ出力を確認
$ docker compose logs ret
```


# 17. ユーザーのadminフラグを有効化
管理画面に入るためにはadminフラグを有効にする必要があります。PostgreSQLのコンテナに入り、ユーザーの管理フラグを有効にします。
```
$ docker compose exec db bash

### dbコンテナ内
# psql -U postgres -d ret_dev
### account_idの確認
ret_dev=# SELECT * FROM accounts;

### is_adminをtrueに変更
ret_dev=# UPDATE accounts SET is_admin = true WHERE account_id = {account_id}
```
adminフラグが有効なユーザーで https://hubs.local:4000/ を開くと画面上部に「admin」が表示されます。

ここまでで構築完了です。


# 18. 使用方法
- 起動と終了
```
### 起動
$ docker compose up -d
### コンテナ破棄
$ docker compose down
```
- DBアクセス
```
### PostgreSQLのコンテナに入る
$ docker compose exec db bash
### PostgreSQLに接続
# psql -U postgres -d ret_dev
```
※A5等でも接続できます
- DB名：ret_dev
- ユーザーID：postgres
- パスワード：postgres


# 参考
- [HubsCloudをDocker環境で動かしてみる](https://synamon.hatenablog.com/entry/hubscloud-dockernize)
- [Mozilla Hubs を VirtualBox でホストしてみた](https://www.infiniteloop.co.jp/tech-blog/2022/02/mozilla-hubs-on-virtualbox/)
- [mozilla-hubs-installation-detailed](https://github.com/albirrkarim/mozilla-hubs-installation-detailed)