# Quick Start Guide

Before we begin be sure to [download and install confd](installation.md). 

## Select a backend

confd supports the following backends:

* etcd
* consul
* environment variables

### Add keys

This guide assumes you have a working [etcd](https://github.com/coreos/etcd#getting-started), or [consul](http://www.consul.io/intro/getting-started/install.html) server up and running and the ability to add new keys.

#### etcd

```
etcdctl set /myapp/database/url db.example.com
etcdctl set /myapp/database/user rob
```

#### consul

```
curl -X PUT -d 'db.example.com' http://localhost:8500/v1/kv/myapp/database/url
curl -X PUT -d 'rob' http://localhost:8500/v1/kv/myapp/database/user
```

#### environment variables

```
export MYAPP_DATABASE_URL=db.example.com
export MYAPP_DATABASE_USER=rob
```

### Create the confdir

The confdir is where template resource configs and source templates are stored.

```
sudo mkdir -p /etc/confd/{conf.d,templates}
```

### Create a template resource config

Template resources are defined in [TOML](https://github.com/mojombo/toml) config files under the `confdir`.

/etc/confd/conf.d/myconfig.toml
```
[template]
src = "myconfig.conf.tmpl"
dest = "/tmp/myconfig.conf"
keys = [
    "/myapp/database/url",
    "/myapp/database/user",
]
```

### Create the source template

Source templates are [Golang text templates](http://golang.org/pkg/text/template/#pkg-overview).

/etc/confd/templates/myconfig.conf.tmpl
```
[myconfig]
database_url = {{getv "/myapp/database/url"}}
database_user = {{getv "/myapp/database/user"}}
```

### Process the template

confd supports two modes of operation daemon and onetime. In daemon mode confd polls a backend for changes and updates destination configuration files if necessary.

#### etcd

```
confd -onetime -backend etcd -node 127.0.0.1:4001
```

#### consul

```
confd -onetime -backend consul -node 127.0.0.1:8500
```

#### env

```
confd -onetime -backend env
```

Output:
```
2014-07-08T20:38:36-07:00 confd[16252]: WARNING Skipping confd config file.
2014-07-08T20:38:36-07:00 confd[16252]: INFO Target config /tmp/myconfig.conf out of sync
2014-07-08T20:38:36-07:00 confd[16252]: INFO Target config /tmp/myconfig.conf has been updated
```

The `dest` configuration file should now be in sync.

```
cat /tmp/myconfig.conf
```

Output:
``` 
# This a comment
[myconfig]
database_url = db.example.com
database_user = rob
```

## Advanced Example

In this example we will use confd to manage two nginx config files using a single template. 

### Add keys

#### etcd

```
etcdctl set /myapp/subdomain myapp
etcdctl set /myapp/upstream/app2 "10.0.1.100:80"
etcdctl set /myapp/upstream/app1 "10.0.1.101:80"
etcdctl set /yourapp/subdomain yourapp
etcdctl set /yourapp/upstream/app2 "10.0.1.102:80"
etcdctl set /yourapp/upstream/app1 "10.0.1.103:80"
```

#### consul

```
curl -X PUT -d 'myapp' http://localhost:8500/v1/kv/myapp/subdomain
curl -X PUT -d '10.0.1.100:80' http://localhost:8500/v1/kv/myapp/upstream/app1
curl -X PUT -d '10.0.1.101:80' http://localhost:8500/v1/kv/myapp/upstream/app2
curl -X PUT -d 'yourapp' http://localhost:8500/v1/kv/yourapp/subdomain
curl -X PUT -d '10.0.1.102:80' http://localhost:8500/v1/kv/yourapp/upstream/app1
curl -X PUT -d '10.0.1.103:80' http://localhost:8500/v1/kv/yourapp/upstream/app2
```

### Create template resources

/etc/confd/conf.d/myapp-nginx.toml

```
[template]
prefix = "/myapp"
src = "nginx.tmpl"
dest = "/tmp/myapp.conf"
owner = "nginx"
mode = "0644"
keys = [
  "/subdomain",
  "/upstream",
]
check_cmd = "/usr/sbin/nginx -t -c {{.src}}"
reload_cmd = "/usr/sbin/service nginx reload"
```

/etc/confd/conf.d/yourapp-nginx.toml

```
[template]
prefix = "/yourapp"
src = "nginx.tmpl"
dest = "/tmp/yourapp.conf"
owner = "nginx"
mode = "0644"
keys = [
  "/subdomain",
  "/upstream",
]
check_cmd = "/usr/sbin/nginx -t -c {{.src}}"
reload_cmd = "/usr/sbin/service nginx reload"
```

### Create the source template

/etc/confd/templates/nginx.tmpl
```
upstream {{getv "/subdomain"}} {
{{range getvs "/upstream/*"}}
    server {{.}};
{{end}}
}

server {
    server_name  {{getv "/subdomain"}}.example.com;
    location / {
        proxy_pass        http://{{getv "/subdomain"}};
        proxy_redirect    off;
        proxy_set_header  Host             $host;
        proxy_set_header  X-Real-IP        $remote_addr;
        proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
   }
}
```
