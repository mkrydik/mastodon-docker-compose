# Mastodon Docker Compose

- ローカルマシンの WSL2 Ubuntu 上で、Docker Compose を用いて Mastodon 環境を構築する
- Public IP を持つ VM にて nginx サーバを用意し、Let's Encrypt で SSL 証明書を取得する
- ローカルマシンから Public VM に向けて SSH ポートフォワードでトンネリングし `https://example.com/` でマストドン環境にアクセスできることを目指す

## Make Directories

```bash
$ sudo mkdir -p ./public/system/
$ sudo chown -R 991:991 ./public/
$ sudo chmod -R 755 ./public/
```

## Generate Secrets

```bash
$ source ./.env.production

$ docker-compose run --rm web bundle exec rake mastodon:webpush:generate_vapid_key
VAPID_PRIVATE_KEY=【Vapid Private Key】
VAPID_PUBLIC_KEY=【Vapid Public Key】

$ docker-compose run --rm web bundle exec bin/rails db:encryption:init
active_record_encryption:
  primary_key: 【Private Key】
  deterministic_key: 【Deterministic Key】
  key_derivation_salt: 【Key Derivation Salt】

$ docker-compose run --rm web bundle exec bin/rails secret
【Secret Key Base】

$ docker-compose run --rm web bundle exec bin/rails secret
【OTP Secret】

$ source ./.env.production
```

## Setup Database

```bash
$ docker-compose up -d db
$ docker-compose exec db bash

root:$ psql -U postgres
postgres=> CREATE USER mastodon;
CREATE ROLE
postgres=> alter role mastodon with password 'abcdefghijklmnopqrstuvwxyz';
ALTER ROLE
postgres=> create database mastodon_production with owner=mastodon;
CREATE DATABASE

$ docker compose run --rm web bundle exec rake db:migrate
```

## Setup nginx Server

```bash
# Set A Record At Public IP
$ dig example.com
;; ANSWER SECTION:
example.com.     0       IN      A       000.000.000.000

# Certbot Let's Encrypt
$ ./certbot-auto certonly --webroot -w /var/www/html -d example.com -m example@gmail.com --agree-tos -n
```

- <example.com.conf>

## Launch Docker Compose

```bash
$ docker-compose up -d
```

## SSH Port Forwards

```bash
$ ssh 【Server Name】 -R 3000:【WSL IP】:3000
$ ssh 【Server Name】 -R 4000:【WSL IP】:4000
```

- `https://example.com/`

## Create Second User

```bash
$ docker-compose run --rm web bin/tootctl account create "USERNAME" --email="example@gmail.com" --confirmed --role=Owner --approve
New password: 【User Password】
```

## References

- [mastodon-docker-compose/README.md at main · bplein/mastodon-docker-compose](https://github.com/bplein/mastodon-docker-compose/blob/main/README.md)
- [MastodonをDockerでsetupする](https://zenn.dev/pluie/articles/20230212-mastodon-setup)
- [Mastodon Docker Setup](https://gist.github.com/TrillCyborg/84939cd4013ace9960031b803a0590c4#nginx-configuration)
- [mastodon/docker-compose.yml at main · gotfirth/mastodon](https://github.com/gotfirth/mastodon/blob/main/docker-compose.yml)
- [MastodonのメールをGmailから送る #Ruby - Qiita](https://qiita.com/ymmtmdk/items/aa0d300450d370a1eca0)
- [ポートフォワードを使って一時的にローカル開発環境を外部公開する | Ｌｏｇ](https://techlog.mydns.jp/?p=154)
