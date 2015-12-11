Role Name
=========

Role is designed for [Let's encrypt](https://letsencrypt.org/) integration. It requires propper configured web server, see [dependencies](#Dependencies).

Requirements
------------

Any pre-requisites that may not be covered by Ansible itself or the role should be mentioned here. For instance, if the role uses the EC2 module, it may be a good idea to mention in this section that the boto package is required.

Role Variables
--------------

| variable name | default value | description |
|---------------|:-------------:|-------------|
| letsencrypt_path| /opt/letsencrypt | letsencrypt clone path |
| letsencrypt_webroot| /var/www/letsencrypt| letsencrypt webroot |
| letsencrypt_reload_nginx| yes | if ```true```, adds ```service nginx reload``` task to cron|
| letsencrypt | none | dict of emails and domains, see example below |

### ```letsencrypt```variable

Domain names are grouped by email address to minimize API calls.

```
letsencrypt:
    "hostmaster@example.com": ["gitlab.example.com", "www.example.com"]
    "test@gmail.com": ["test.com", "test.net"]
```

Dependencies
------------

For successful validation your web server should respond for both http and https request at the ```/.well-known/acme-challenge``` URL with the content of ```letsencrypt_webroot``` directory. See nginx example below.

### nginx_example

##### /etc/nginx/letsencrypt.conf

```
location /.well-known/acme-challenge
{
    root /var/www/letsencrypt;
    error_page 404 =500 @500;
}

location @500 {
    return 500;
}
```

##### /etc/nginx/sites-available/mysite.conf

```
# {{ ansible_managed }}
server {
    listen 80;
    server_name -;
    server_tokens off;

    include letsencrypt.conf;

    location /
    {
        return 301 https://$host$request_uri;
    }
}
server {
        listen 443 ssl;
        server_name {{ server }};
        ssl on;
        ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

        include letsencrypt.conf;
}
```

In this example I replace error 404 with 500 one to set up Let's encrypt on multiple backends behind ha-proxy.

Example Playbook
----------------

```
- hosts: servers
  sudo: yes
  vars:
    letsencrypt:
      "hostmaster@example.com": ["gitlab.example.com", "www.example.com"]
      "test@gmail.com": ["test.com", "test.net"]

  roles:
      - { role: letsencrypt, tags: cert }

```

License
-------

BSD

TODO
-----

- Run in standalone mode if now web server is configured
- Implement individual cron times for different servers
- Implement one by one generation if run on a group of hosts
