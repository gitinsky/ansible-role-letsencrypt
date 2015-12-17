letsencrypt
=========

Role is designed for [Let's encrypt](https://letsencrypt.org/) integration. It installs requested certificates and sets up automatic validation task to cron. Propper configured web server is required, see [requirements](#requirements).

There's a ```dontgen``` tag that dosn't generate certificates but just shows you the generation commands and puts cron tasks.

Requirements
------------

For successful validation your web server should respond for both http and https request at the ```/.well-known/acme-challenge``` URL with the content of ```letsencrypt_webroot``` directory. See nginx example below. Built in web server will be used for the first time generation if no services listen to 80 and 443 ports.

Do _not_ run multiple letsencrypt roles on the same host as cron tasks will be overwritten.

Certificate update tasks are placed to cron with 5 minutes interval starting from 4:04 in the 14th day of each month.

### nginx_example

##### /etc/nginx/letsencrypt.conf

```
# {{ ansible_managed }}
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

Role Variables
--------------

| variable name | default value | description |
|---------------|:-------------:|-------------|
| letsencrypt_path| /opt/letsencrypt | letsencrypt clone path |
| letsencrypt_webroot| /var/www/letsencrypt| letsencrypt webroot |
| letsencrypt_reload_nginx| yes | if ```true```, adds ```service nginx reload``` task to cron|
| letsencrypt | none | list of dictionaries, see example below |

### ```letsencrypt```variable

Variable is a list of dictionaries, each item should produce a key and a cron task.
```email``` is an emails address assigned to the list of domains in ```domains``` array.
```domains``` is a list of domains included in a key. Last item in this list will be used in a path to certificates.

```
letsencrypt:
    - { email: "hostmaster@example.com", domains: ["example.com", "www.example.com"] }
    - { email: "hostmaster@example.com", domains: ["gitlab.example.com"] }
    - { email: "test@gmail.com",         domains: ["test.com", "test.net"] }
```

Here you are going to get the following certificate paths:

- /etc/letsencrypt/live/www.example.com/fullchain.pem
- /etc/letsencrypt/live/gitlab.example.com/fullchain.pem
- /etc/letsencrypt/live/test.net/fullchain.pem

Example Playbook
----------------

```
- hosts: servers
  sudo: yes
  vars:
      letsencrypt:
          - { email: "hostmaster@example.com", domains: ["example.com", "www.example.com"] }
          - { email: "hostmaster@example.com", domains: ["gitlab.example.com"] }
          - { email: "test@gmail.com",         domains: ["test.com", "test.net"] }

  roles:
      - { role: letsencrypt, tags: cert }

```

License
-------

BSD

Known issues
------------

Multiple domains renew seems to be broken. More test are required to check if they could be updated with a single command.

TODO
-----

- Implement individual cron times for different servers
- Implement one by one generation if run on a group of hosts
