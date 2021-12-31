# IPv6-Proxy
### Step 1. Install the Nginx
If you don’t have the Nginx installed, use the [following steps](https://docs.nginx.com/nginx/admin-guide/installing-nginx/installing-nginx-open-source/) to install Nginx. Otherwise, continue to step 2.
### Step 2. Check module in Nginx
```sh
sudo nginx -V
```
We need to enable IPv6 and Stream on Nginx (***--with-ipv6 --with-stream=dynamic***)

```sh
configure arguments: --prefix=/usr/share/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib64/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --http-client-body-temp-path=/var/lib/nginx/tmp/client_body --http-proxy-temp-path=/var/lib/nginx/tmp/proxy --http-fastcgi-temp-path=/var/lib/nginx/tmp/fastcgi --http-uwsgi-temp-path=/var/lib/nginx/tmp/uwsgi --http-scgi-temp-path=/var/lib/nginx/tmp/scgi --pid-path=/var/run/nginx.pid --lock-path=/var/lock/subsys/nginx --user=nginx --group=nginx --with-file-aio --with-ipv6 --with-http_ssl_module --with-http_v2_module --with-http_realip_module --with-http_addition_module --with-http_xslt_module=dynamic --with-http_image_filter_module=dynamic --with-http_geoip_module=dynamic --with-http_sub_module --with-http_dav_module --with-http_flv_module --with-http_mp4_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_random_index_module --with-http_secure_link_module --with-http_degradation_module --with-http_slice_module --with-http_stub_status_module --with-http_perl_module=dynamic --with-mail=dynamic --with-mail_ssl_module --with-pcre --with-pcre-jit --with-stream=dynamic --with-stream_ssl_module --with-debug --with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector --param=ssp-buffer-size=4 -m64 -mtune=generic' --with-ld-opt=' -Wl,-E'
```
### Step 3. Configuring TCP Proxy on Nginx
First, you will need to configure reverse proxy so that Nginx can forward TCP connections or UDP datagrams from clients to an upstream group or a proxied server.
Open the Nginx configuration file and perform the following steps:
1. Create a top‑level ```stream {}``` block:
    ```sh
    stream {
        # ...
    }
    ```
2. Define one or more ```server {}``` configuration blocks for each virtual server in the top‑level ```stream {}``` context
3. Within the ```server {}``` configuration block for each server, include the ```listen``` directive to define the IP address and/or port on which the server listens.
For UDP traffic, also include the udp parameter. As TCP is the default protocol for the stream context, there is no tcp parameter to the listen directive:
    ```sh
    stream {
        server {
            # Listen only on IPv6:
            listen [::]:12345 ipv6only=on;
            # ...
        }
        server {
            # Listen only on IPv6:
            listen [::]:53 udp;
            # ...
        }
        # ...
    }
    ```
4. Include the ```proxy_pass``` directive to define the proxied server or an upstream group to which the server forwards traffic:
    ```sh
    stream {
        server {
            # Listen only on IPv6:
            listen [::]:12345 ipv6only=on;
            #TCP traffic will be forwarded to the "stream_backend" upstream group
            proxy_pass stream_backend;
        }
        server {
            # Listen only on IPv6:
            listen [::]:12346 ipv6only=on;
            #TCP traffic will be forwarded to the specified server
            proxy_pass backend.example.com:12346;
        }
        server {
            # Listen only on IPv6:
            listen [::]:53 udp;
            #UDP traffic will be forwarded to the "dns_servers" upstream group
            proxy_pass dns_servers;
        }
        # ...
    }
    ```
5. If the proxy server has several network interfaces, you can optionally configure NGINX to use a particular source IP address when connecting to an upstream server. This may be useful if a proxied server behind NGINX is configured to accept connections from particular IP networks or IP address ranges.
Include the ```proxy_bind``` directive and the IP address of the appropriate network interface:
    ```sh
    stream {
        # ...
        server {
            listen     [2001:df2:d900:1:103:102:129:165]:12345;
            proxy_pass backend.example.com:12345;
            proxy_bind [2001:df2:d900:1:103:102:129:165]:12345;
        }
    }
    ```
6. Optionally, you can tune the size of two in‑memory buffers where NGINX can put data from both the client and upstream connections. If there is a small volume of data, the buffers can be reduced which may save memory resources. If there is a large volume of data, the buffer size can be increased to reduce the number of socket read/write operations. As soon as data is received on one connection, NGINX reads it and forwards it over the other connection. The buffers are controlled with the ```proxy_buffer_size``` directive:
    ```sh
    stream {
        # ...
        server {
            listen            [2001:df2:d900:1:103:102:129:165]:12345;
            proxy_pass        backend.example.com:12345;
            proxy_buffer_size 16k;
        }
    }
    ```
### Step 4. Allow Port on Ip6tables
Type the following command to see current ipv6 firewall configuration:
```sh
# ip6tables -nL --line-numbers
```
If no rules appear, activate IPv6 firewall and ensure that it starts at boot by typing the following command:
```
# chkconfig ip6tables on
```
Edit ```/etc/sysconfig/ip6tables```, enter:
```sh
# vi /etc/sysconfig/ip6tables
```
You will see default rules as follows:
```sh
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p ipv6-icmp -j ACCEPT
-A INPUT -i lo -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
-A INPUT -d fe80::/64 -p udp -m udp --dport 546 -m state --state NEW -j ACCEPT
-A INPUT -j REJECT --reject-with icmp6-adm-prohibited
-A FORWARD -j REJECT --reject-with icmp6-adm-prohibited
COMMIT
```
To open port 12345 (Game Service) add the following before COMMIT line:
```sh
-A INPUT -p tcp -m state --state NEW -m tcp --dport 12345 -j ACCEPT
```
Save and close the file. Restart ip6tables firewall:
```sh
# service ip6tables restart
# ip6tables -vnL --line-numbers
```
### Step 5. Check Port Nginx Listen
First you need to reload the nginx configuration after configuring the proxy
```sh#
 service nginx reload
```
Check port listen
```sh
netstat -antup| grep 12345
```
Sample Outputs:
```sh
tcp        0      0 :::12345                    :::*                        LISTEN      31332/nginx
```
