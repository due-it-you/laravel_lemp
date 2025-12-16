#### Nginxの設定
##### default.conf
NginxのWebサーバーとしての基本的な設定は`/etc/nginx/conf.d/default.conf`で行う。

以下のような設定にすることで、静的ファイル以外の場合にphp-fpm側にFastCGIを用いて処理を渡してくれる。

```
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    root /var/www/html/public;
    index index.php index.html;
    #access_log  /var/log/nginx/host.access.log  main;

    # ルート配下、すなわち全URLにおけるルールの定義
    location / {
        # 静的ファイルがなければ index.phpに処理を渡す
       try_files $uri $uri/ /index.php?$query_string; 
    }

    # 上の設定で index.php が呼び込まれたときにphp-fpmに転送
    location ~ \.php$ {
        # FastCGI用のパラメータの読み込み
        include fastcgi_params;
        # PHP-FPMに処理を渡す
        fastcgi_pass app:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

##### try_files
```
        # 静的ファイルがなければ index.phpに処理を渡す
       try_files $uri $uri/ /index.php?$query_string;
```

try_filesは、左から記述されている順に存在を確認していき、最初に存在が確認されたものを返す。

今回は`$uri`ファイルや`$url`ディレクトリが確認できなければ、`/index.php`を返すような処理になっている。

`docker-compose.yml`によって、以下のように、バインドマウントを行っているため、Nginxサーバーには`index.php`がルートに存在していて、これが返されるということになる。

##### location .php$ {...}
try_filesの部分で、`index.php`に処理が渡った際に、Nginxはphpを実行できないため、`php-fpm`側に処理を渡すような処理を追加している。

```
    # 上の設定で index.php が呼び込まれたときにphp-fpmに転送
    location ~ \.php$ {
        # FastCGI用のパラメータの読み込み
        include fastcgi_params;
        # PHP-FPMに処理を渡す
        fastcgi_pass app:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
```

具体的には、`.php`ファイルの場合には全て`php-fpm`側に処理が渡される。
`php-fpm`側はポート9000を開放しているため、それを指定している。
リクエスト時のプロトコルは[[FastCGI]]を用いているため、FastCGIに関する項目が使用されている。

#### 処理の流れ
処理の流れとしては、以下のようになっている。

「**静的ファイル以外の場合に`index.php`に処理を渡らせる**」
「**`.php`の場合に`php-fpm`側のコンテナに処理を投げる**」
という２つの処理をNginxの設定ファイルに設定していることが重要。

1. `https://xxx.com/about`が打ち込まれる
2. Nginxが搭載されたWebサーバーにリクエストが送信される
3. `default.conf`の`root`を基にして、`/var/www/html/public/about`が探される
4. `try_files`によって、静的ファイル以外であるため`index.php`に処理が渡る
5. `location ~ \.php$ {...}`によって、php-fpmに処理が渡る
6. appコンテナの`index.php`が実行される
7. `routes/web.php`でルーティングを確認
8. 必要なHTMLをNginx側にレスポンス
9. Nginxがクライアント側にレスポンス
10. クライアントがHTMLをレンダリング
11. ページが表示される
