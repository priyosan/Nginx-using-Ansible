# Nginx-using-Ansible
Install dan Config Nginx service
=============================================================

## @priyosan : I don't have time to manage anymore this role. 
## Don't hesitate to fork and made your own version.


This role installs and configures the nginx web server. The user can specify
any http configuration parameters they wish to apply their site. Any number of
sites can be added with configurations of your choice.

[![Build Status](https://travis-ci.org/jdauphant/ansible-role-nginx.svg?branch=master)](https://travis-ci.org/jdauphant/ansible-role-nginx)
[![Ansible Galaxy](https://img.shields.io/ansible/role/466.svg)](https://galaxy.ansible.com/jdauphant/nginx/)

Requirements
------------

This role requires Ansible 2.4 or higher and platform requirements are listed
in the metadata file. (Some older version of the role support Ansible 1.4)
For FreeBSD a working pkgng setup is required (see: https://www.freebsd.org/doc/handbook/pkgng-intro.html )
Installation of Nginx Amplify agent is only supported on CentOS, RedHat, Amazon, Debian and Ubuntu distributions.

Install
-------

```sh
ansible-galaxy install jdauphant.nginx
```

Role Variables
--------------

The variables that can be passed to this role and a brief description about
them are as follows. (For all variables, take a look at [defaults/main.yml](defaults/main.yml))

```yaml
# The user to run nginx
nginx_user: "www-data"

# A list of directives for the events section.
nginx_events_params:
 - worker_connections 512
 - debug_connection 127.0.0.1
 - use epoll
 - multi_accept on

# A list of hashes that define the servers for nginx,
# as with http parameters. Any valid server parameters
# can be defined here.
nginx_sites:
 default:
     - listen 80
     - server_name _
     - root "/usr/share/nginx/html"
     - index index.html
 foo:
     - listen 8080
     - server_name localhost
     - root "/tmp/site1"
     - location / { try_files $uri $uri/ /index.html; }
     - location /images/ { try_files $uri $uri/ /index.html; }
 bar:
     - listen 9090
     - server_name ansible
     - root "/tmp/site2"
     - location / { try_files $uri $uri/ /index.html; }
     - location /images/ {
         try_files $uri $uri/ /index.html;
         allow 127.0.0.1;
         deny all;
       }

# A list of hashes that define additional configuration
nginx_configs:
  proxy:
      - proxy_set_header X-Real-IP  $remote_addr
      - proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for
  upstream:
      - upstream foo { server 127.0.0.1:8080 weight=10; }
  geo:
      - geo $local {
          default 0;
          127.0.0.1 1;
        }
  gzip:
      - gzip on
      - gzip_disable msie6

# A list of hashes that define configuration snippets
nginx_snippets:
  error_pages:
    - error_page 500 /http_errors/500.html
    - error_page 502 /http_errors/502.html
    - error_page 503 /http_errors/503.html
    - error_page 504 /http_errors/504.html

# A list of hashes that define user/password files
nginx_auth_basic_files:
   demo:
     - foo:$apr1$mEJqnFmy$zioG2q1iDWvRxbHuNepIh0 # foo:demo , generated by : htpasswd -nb foo demo
     - bar:$apr1$H2GihkSo$PwBeV8cVWFFQlnAJtvVCQ. # bar:demo , generated by : htpasswd -nb bar demo

# Enable Real IP for CloudFlare requests
nginx_set_real_ip_from_cloudflare: True

# Enable Nginx Amplify
nginx_amplify: true
nginx_amplify_api_key: "your_api_key_goes_here"
nginx_amplify_update_agent: true

# Define modules to enable in configuration
#
# Nginx installed via EPEL and APT repos will also install some modules automatically.
# For official Nginx repo use you will need to install module packages manually.
#
# When using with EPEL and APT repos, specify this section as a list of configuration
# file names, minus the .conf file name extension.

# When using the official Nginx repo, specify this section as list of module file
# names, minus the .so file name extension.
#
# Available module config files in EPEL and APT repos:
# (APT actually has several more, see https://wiki.debian.org/Nginx/)
# - mod-http-geoip
# - mod-http-image-filter
# - mod-http-perl
# - mod-http-xslt-filter
# - mod-mail
# - mod-stream
#
# Available module filenames in Official NGINX repo:
# - ngx_http_geoip_module
# - ngx_http_image_filter_module
# - ngx_http_perl_module
# - ngx_http_xslt_filter_module
# - ngx_http_js_module
#
# Custom compiled modules are ok too if the .so file exists in same location as a packaged module would be:
# - ngx_http_modsecurity_module
#
nginx_module_configs:
  - mod-http-geoip
```

Examples
========

## 1) Install nginx with HTTP directives of choice, but with no sites configured and no additional configuration:

```yaml
- hosts: all
  roles:
  - {role: nginx,
     nginx_http_params: ["sendfile on", "access_log /var/log/nginx/access.log"]
                          }
```

## 2) Install nginx with different HTTP directives than in the previous example, but no
sites configured and no additional configuration.

```yaml
- hosts: all
  roles:
  - {role: nginx,
     nginx_http_params: ["tcp_nodelay on", "error_log /var/log/nginx/error.log"]}
```

Note: Please make sure the HTTP directives passed are valid, as this role
won't check for the validity of the directives. See the nginx documentation
for details.

## 3) Install nginx and add a site to the configuration.

```yaml
- hosts: all

  roles:
  - role: nginx
    nginx_http_params:
      - sendfile "on"
      - access_log "/var/log/nginx/access.log"
    nginx_sites:
      bar:
        - listen 8080
        - location / { try_files $uri $uri/ /index.html; }
        - location /images/ { try_files $uri $uri/ /index.html; }
    nginx_configs:
      proxy:
        - proxy_set_header X-Real-IP  $remote_addr
        - proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for
```

## 4) Install nginx and add extra variables to default config

```yaml
-hosts: all
  vars:
    - my_extra_params:
      - client_max_body_size 200M
# retain defaults and add additional `client_max_body_size` param
  roles:
    - role: jdauphant.nginx
      nginx_http_params: "{{ nginx_http_default_params + my_extra_params }}"
```

Note: Each site added is represented by a list of hashes, and the configurations
generated are populated in /etc/nginx/site-available/ and linked from /etc/nginx/site-enable/ to /etc/nginx/site-available.

The file name for the specific site configuration is specified in the hash
with the key "file_name", any valid server directives can be added to the hash.
Additional configurations are created in /etc/nginx/conf.d/

## 5) Install Nginx, add 2 sites (different method) and add additional configuration

```yaml
---
- hosts: all
  roles:
    - role: nginx
      nginx_http_params:
        - sendfile on
        - access_log /var/log/nginx/access.log
      nginx_sites:
         foo:
           - listen 8080
           - server_name localhost
           - root /tmp/site1
           - location / { try_files $uri $uri/ /index.html; }
           - location /images/ { try_files $uri $uri/ /index.html; }
         bar:
           - listen 9090
           - server_name ansible
           - root /tmp/site2
           - location / { try_files $uri $uri/ /index.html; }
           - location /images/ { try_files $uri $uri/ /index.html; }
      nginx_configs:
         proxy:
            - proxy_set_header X-Real-IP  $remote_addr
            - proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for
```

## 6) Install Nginx, add 2 sites, add additional configuration and an upstream configuration block

```yaml
---
- hosts: all
  roles:
    - role: nginx
      nginx_error_log_level: info
      nginx_http_params:
        - sendfile on
        - access_log /var/log/nginx/access.log
      nginx_sites:
        foo:
           - listen 8080
           - server_name localhost
           - root /tmp/site1
           - location / { try_files $uri $uri/ /index.html; }
           - location /images/ { try_files $uri $uri/ /index.html; }
        bar:
           - listen 9090
           - server_name ansible
           - root /tmp/site2
           - if ( $host = example.com ) { rewrite ^(.*)$ http://www.example.com$1 permanent; }
           - location / {
             try_files $uri $uri/ /index.html;
             auth_basic            "Restricted";
             auth_basic_user_file  auth_basic/demo;
           }
           - location /images/ { try_files $uri $uri/ /index.html; }
      nginx_configs:
        proxy:
            - proxy_set_header X-Real-IP  $remote_addr
            - proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for
        upstream:
            # Results in:
            # upstream foo_backend {
            #   server 127.0.0.1:8080 weight=10;
            # }
            - upstream foo_backend { server 127.0.0.1:8080 weight=10; }
      nginx_auth_basic_files:
        demo:
           - foo:$apr1$mEJqnFmy$zioG2q1iDWvRxbHuNepIh0 # foo:demo , generated by : htpasswd -nb foo demo
           - bar:$apr1$H2GihkSo$PwBeV8cVWFFQlnAJtvVCQ. # bar:demo , generated by : htpasswd -nb bar demo
```

## 7) Install Nginx, add a site and use special yaml syntax to make the location blocks multiline for clarity

```yaml
---
- hosts: all
  roles:
    - role: nginx
      nginx_http_params:
        - sendfile on
        - access_log /var/log/nginx/access.log
      nginx_sites:
        foo:
           - listen 443 ssl
           - server_name foo.example.com
           - set $myhost foo.example.com
           - |
             location / {
               proxy_set_header Host foo.example.com;
             }
           - |
             location ~ /v2/users/.+?/organizations {
               if ($request_method = PUT) {
                 set $myhost bar.example.com;
               }
               if ($request_method = DELETE) {
                 set $myhost bar.example.com;
               }
               proxy_set_header Host $myhost;
             }
```
## 8) Example to use this role with my ssl-certs role to generate or copy ssl certificate ( https://galaxy.ansible.com/jdauphant/ssl-certs )
```yaml
 - hosts: all
   roles:
     - jdauphant.ssl-certs
     - role: jdauphant.nginx
       nginx_configs:
          ssl:
               - ssl_certificate_key {{ssl_certs_privkey_path}}
               - ssl_certificate     {{ssl_certs_cert_path}}
       nginx_sites:
          default:
               - listen 443 ssl
               - server_name _
               - root "/usr/share/nginx/html"
               - index index.html
```
## 9) Site configuration using a custom template.
Instead of defining a site config file using a list of attributes,
you may use a hash/dictionary that includes the filename of an alternate template.
Additional values are accessible within the template via the `item.value` variable.
```yaml
- hosts: all

  roles:
  - role: nginx
    nginx_sites:
      custom_bar:
        template: custom_bar.conf.j2
        server_name: custom_bar.example.com
```
Custom template: custom_bar.conf.j2:
```handlebars
# {{ ansible_managed }}
upstream backend {
  server 10.0.0.101;
}
server {
  server_name {{ item.value.server_name }};
  location / {
    proxy_pass http://backend;
  }
}
```
Using a custom template allows for unlimited flexibility in configuring the site config file.
This example demonstrates the common practice of configuring a site server block
in the same file as its complementary upstream block.
If you use this option:
* _The hash **must** include a `template:` value, or the configuration task will fail._
* _This role cannot check tha validity of your custom template.
If you use this method, the conf file formatting provided by this role is unavailable,
and it is up to you to provide a template with valid content and formatting for NGINX._

## 10) Install Nginx, add 2 sites, use snippets to configure access controls
```yaml
---
- hosts: all
  roles:
    - role: nginx
      nginx_http_params:
        - sendfile on
        - access_log /var/log/nginx/access.log
      nginx_snippets:
        accesslist_devel:
          - allow 192.168.0.0/24
          - deny all
      nginx_sites:
        foo:
           - listen 8080
           - server_name localhost
           - root /tmp/site1
           - include snippets/accesslist_devel.conf
           - location / { try_files $uri $uri/ /index.html; }
           - location /images/ { try_files $uri $uri/ /index.html; }
        bar:
           - listen 9090
           - server_name ansible
           - root /tmp/site2
           - location / { try_files $uri $uri/ /index.html; }
           - location /images/ { try_files $uri $uri/ /index.html; }
```

Dependencies
------------

None

License
-------
BSD

Author Information
------------------

- Original : Github
- Modified copyright by : DAUPHANT Julien
