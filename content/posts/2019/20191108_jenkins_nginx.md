---
title: "Jenkins導入とNginxでリバースプロキシ設定"
date: 2019-11-08T10:20:00+09:00
draft: false
toc: false
images:
tags: 
  - dev
---

業務でCIツールとしてJenkinsを入れることになったので、その時の記録。
jenkinsの説明は、[Jenkinsとはなんぞや？](https://qiita.com/ko8@github/items/e6e058976d48d3fc3424)​が簡潔でわかりやすかった。

## バージョン
- nginx 1.16.1
- centOS 7.6.1810
- Jenkins 2.190.3

# Jenkins導入方法

**(1) install Java**
Jenkinsはjavaで実装されており、javaの実行環境が必要があるためない場合はインストールする。

```
# openJDK
yum install java-1.8.0-openjdk
```

**(2) install Jenkins**

[公式](http://pkg.jenkins-ci.org/redhat-stable/)の手順。

```
# jenkinsのyumリポジトリを取得
wget -O /etc/yum.repos.d/jenkins.repo http://pkg.jenkins-ci.org/redhat/jenkins.repo

# 公開鍵追加
rpm --import https://jenkins-ci.org/redhat/jenkins-ci.org.key

# インストール
yum install jenkins
```

**(3) 各種設定**
設定ファイル： `/etc/sysconfig/jenkins`
各種設定はこのファイルで変更することができる。以下は、デフォルトの値。

```
# ポート
JENKINS_PORT=“8080"
​
# 実行ユーザ
JENKINS_USER="jenkins"
```

実行ユーザーの変更を行った場合以下のディレクトリ/ファイルの権限も同様に変更する必要がある。

- `/var/lib/jenkins` 
- `/var/log/jenkins` 
- `/var/cache/jenkins`

**(4) 起動**

```
# 起動
systemctl start jenkins

# 再起動
systemctl restart jenkins

# 終了
systemctl stop jenkins

# 確認
systemctl status jenkins
```

起動して、statusがrunningになったら、`http://{IP}:{PORT}`で接続できる。外部からの接続が遮断されている場合は、解放することを忘れずに。

[firewall設定 counfigure firewall 参照](https://wiki.jenkins.io/display/JENKINS/Installing+Jenkins+on+Red+Hat+distributions)

# Nginxでリバースプロキシ設定
リバースプロキシとは受け取ったリクエストを転送する機能で、ロードバランスや、リクエストの書き換え、アクセス制限などに使われてる。
​
今回、毎回ポート番号指定する代わりに、`http://{IP}/jenkins`で接続できるように以下の設定を行った。
​
![jenkins_nginx](/jenkins_nginx.png)

**(1) jenkins側の設定**

設定ファイル(`/etc/sysconfig/jenkin`）を以下のように書き換える。

```
JENKINS_ARGS="--prefix=/jenkins"
```

**(2) Nginx側の設定**

- 共通設定：`/etc/nginx/nginx.conf`
- serverブロックごとの設定：`/etc/nginx/conf.d/`
- デフォルトのlistenポート80番の設定：`/etc/nginx/conf.d/default.conf`
​

`/etc/nginx/conf.d/default.conf`で80番ポートをlistenしているserverディレクティブ内に以下を追記。nginxを経由することで、リクエスト情報が変わるので、headerをここでセットしてあげる必要がある。

```
location ~ /jenkins {
    proxy_redirect     off;
    proxy_set_header   Host $host;
    proxy_set_header   X-Forwarded-Proto $scheme;
    proxy_set_header   X-Real-IP         $remote_addr;
    proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_pass         http://jenkins;
}
```


- 変数 （$がつくものはnginxの組み込み変数。）
    - `$scheme`：リクエストされたスキーマ（http、https）
    - `$http_host`：ポート番号付きホスト
    - `$remote_addr`：アクセス元のIPアドレス
    - `$proxy_add_x_forwarded_for`：ユーザーが経由したアドレス
    - `$host`：マッチした、サーバ名/Hostヘッダの値。
- ディレクティブ
    - `proxy_set_header ヘッダーフィールド名　値`：リクエストをプロキシする際に特定のヘッダ情報を付与する。
    - `proxy_pass 転送先`：転送先URL。
​
nginxで使える変数や、ディレクティブの一覧は[ここ](http://nginx.org/en/docs/http/ngx_http_core_module.html)や[ここ](http://www2.matsue-ct.ac.jp/home/kanayama/text/nginx/all.html)が参考になった。
​

`/etc/nginx/conf.d/jenkins.conf`を作成し、以下を記載。`127.0.0.1` は、ループバック・アドレス（自分自身を指すIPアドレス）のこと。default.confの`proxy_pass         http://jenkins;`のjenkinsをここで定義。

``` 
upstream jenkins {
　　server 127.0.0.1:8080 fail_timeout=0;
}
```

**(3) 再起動**

```
# jenkins再起動
systemctl restart jenkins

# nginx再起動
systemctl restart nginx
```

`http://{IP}/jenkins`にアクセスして、jenkinsが無事に表示されたら完了。

jenkinsで`"リバースプロキシの設定がおかしいようです"`のエラーが出てたが、jenkinsの管理→システムの設定→JenkinsのURLを変更すると消えた。

# 参考 
- nginx実践入門
- https://christina04.hatenablog.com/entry/2016/10/25/190000
- https://wiki.jenkins.io/display/JENKINS/Jenkins+behind+an+NGinX+reverse+proxy
