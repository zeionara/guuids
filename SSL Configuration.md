# SSL Configuration

In this guide we will go through the `SSL` certificate configuration for your website. As a starting point, you will need 3 files:

1. Certificate itself - `certificate.crt`.
2. Parent certificate - `certificate_ca.crt`.
3. Private key - `certificate.key`.

On the server, create a safe directory for storing your ssl configuration files (replace `zeio.ru` with your domain):

```sh
sudo mkdir -p /etc/nginx/ssl/zeio.ru
```

Allow your no-sudo user modify this directory for now:

```sh
sudo chown kaede:kaede /etc/nginx/ssl/zeio.ru
```

Then transfer ssl files from your local machine to this folder:

```sh
scp certificate.crt kaede@shrine:/etc/nginx/ssl/zeio.ru
scp certificate.key kaede@shrine:/etc/nginx/ssl/zeio.ru
scp certificate_ca.crt kaede@shrine:/etc/nginx/ssl/zeio.ru
```

Restrict access to files in this directory:

```sh
sudo chown -R root:root /etc/nginx/ssl/zeio.ru
sudo chmod 644 /etc/nginx/ssl/zeio.ru/*
sudo chmod 600 /etc/nginx/ssl/zeio.ru/certificate.key
```

Create the full chain file by joining your certificate with authority certificates:

```sh
cat /etc/nginx/ssl/zeio.ru/certificate.crt /etc/nginx/ssl/zeio.ru/certificate_ca.crt | sudo tee > /dev/null /etc/nginx/ssl/zeio.ru/fullchain.pem
```

Then in your `openresty` config, located at `/usr/local/openresty/nginx/conf/nginx.conf` set:

```sh
http {
  server {
    ssl_certificate /etc/nginx/ssl/zeio.ru/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/zeio.ru/certificate.key;
  }
}
```

Optionally, add this parameters to the server block as well:

```sh
http {
  server {
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
  }
}
```

And add `http` to `https` redirect:

```sh
server {
  listen 80;
  server_name zeio.ru;
  return 301 https://$host$request_uri;
}
```

Test your updated configuration:

```sh
sudo /usr/local/openresty/nginx/sbin/nginx -t
```

Expected output:

```sh
nginx: the configuration file /usr/local/openresty/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /usr/local/openresty/nginx/conf/nginx.conf test is successful
```

Then reload nginx server:

```sh
sudo /usr/local/openresty/nginx/sbin/nginx -s reload
```

If `http` to `https` redirection didn't work, then some other service maybe listening port `80`, check what's the service:

```sh
sudo lsof -i :80
```

If it is `apache2`, then disable it:

```sh
sudo systemctl stop apache2
sudo systemctl disable apache2
```
