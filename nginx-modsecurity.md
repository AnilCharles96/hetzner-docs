# WAF

### Install Dependencies

```
sudo apt update
sudo apt install -y git g++ automake autoconf libtool pkgconf build-essential zlib1g-dev \
 libpcre3 libpcre3-dev libxml2 libxml2-dev libyajl-dev \
 curl wget cmake libcurl4-openssl-dev libgeoip-dev liblmdb-dev \
 liblua5.3-dev libssl-dev ca-certificates gnupg lsb-release


```


### Build and Install modsecurity v3

```
cd /usr/local/src
git clone --depth 1 https://github.com/SpiderLabs/ModSecurity
rm -rf /usr/local/src/ModSecurity # optional
cd ModSecurity
git submodule init
git submodule update
./build.sh
./configure
make -j"$(nproc)"
sudo make install

```

### Download and build nginx + modsecurity connector

```
cd /usr/local/src
sudo git clone --depth 1 https://github.com/SpiderLabs/ModSecurity-nginx

# Get NGINX source code
NGINX_VERSION=1.28.0
wget http://nginx.org/download/nginx-$NGINX_VERSION.tar.gz
tar -xvzf nginx-$NGINX_VERSION.tar.gz
cd nginx-$NGINX_VERSION

# Configure and build the module
sudo ./configure --with-compat --add-dynamic-module=../ModSecurity-nginx
sudo make modules

# Install the module
sudo mkdir -p /etc/nginx/modules
sudo cp objs/ngx_http_modsecurity_module.so /etc/nginx/modules/
```

### Enable modsecurity module in nginx
```
sudo mkdir -p /etc/nginx/modules-enabled
echo "load_module modules/ngx_http_modsecurity_module.so;" | sudo tee /etc/nginx/modules-enabled/modsecurity.conf

```

### Setup modsecurity config and OWASP CRS

```
# Create config dir
sudo mkdir -p /etc/nginx/modsec
cd /etc/nginx/modsec

# Download main config and unicode file
sudo wget https://raw.githubusercontent.com/SpiderLabs/ModSecurity/v3/master/modsecurity.conf-recommended -O modsecurity.conf
sudo wget https://raw.githubusercontent.com/SpiderLabs/ModSecurity/v3/master/unicode.mapping

# Enable blocking mode
sudo sed -i 's/SecRuleEngine DetectionOnly/SecRuleEngine On/' modsecurity.conf

# Get OWASP CRS
sudo git clone https://github.com/coreruleset/coreruleset owasp-crs
cd owasp-crs
sudo cp crs-setup.conf.example crs-setup.conf

# Append OWASP rules to modsecurity.conf
echo -e "\nInclude /etc/nginx/modsec/owasp-crs/crs-setup.conf" | sudo tee -a /etc/nginx/modsec/modsecurity.conf
echo "Include /etc/nginx/modsec/owasp-crs/rules/*.conf" | sudo tee -a /etc/nginx/modsec/modsecurity.conf

```

### Add module to nginx.conf

```
nano /etc/nginx/nginx.conf

load_module modules/ngx_http_modsecurity_module.so;

```


### Check if it is working

```
nano /etc/nginx/conf.d/hetzner.conf

server {
    listen 80;
    server_name hetzner.carefree-energy.com;

    modsecurity on;
    modsecurity_rules '
        SecRuleEngine On
        SecRule ARGS:testparam "@streq test" "id:1234,phase:1,deny,log,status:403,msg:\'ModSecurity test rule triggered\'"
    ';

    location / {
        return 200 "ModSecurity is working!\n";
    }
}

curl -i "http://hetzner.carefree-energy.com/?testparam=test"


```


### All ruleset

```
sudo git clone https://github.com/coreruleset/coreruleset /etc/nginx/modsec/owasp-crs
sudo cp /etc/nginx/modsec/owasp-crs/crs-setup.conf.example /etc/nginx/modsec/owasp-crs/crs-setup.conf


nano /etc/nginx/modsec/modsecurity.conf

Include /etc/nginx/modsec/owasp-crs/crs-setup.conf
Include /etc/nginx/modsec/owasp-crs/rules/*.conf


nano /etc/nginx/conf.d/hetzner.conf

server {
    listen 80;
    server_name hetzner.carefree-energy.com;

    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsec/modsecurity.conf;

    location / {

        return 200 "Welcome! ModSecurity OWASP CRS enabled.\n";
    }
}

# restart nginx

nginx -t && nginx -s reload

### payload:
curl -i "http://hetzner.carefree-energy.com/?param=<script>alert(1)</script>"

```
