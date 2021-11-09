# ELB Custom SSL

These .ebextensions will install some configuration files and hooks that deploy a letsencrypt certificate using the webroot method with a running nginx proxy in an ELB.

## Installation

### Submodule
You can install this as git submodule with
```sh
  git submodule add "$clone_method_uri" .ebextensions/elb-custom-ssl
```

### Copy
For our deployment submodules is a subject I won't get into for now. Thus we can use a method of just copying a certain commit or the HEAD of the project into the right place.
```sh
  git clone "$clone_method_uri" .ebextensions/elb-custom-ssl
  rm -rf .ebextensions/elb-custom-ssl/{.git,README.md}
```

### Deployment
After deploying to ELB with these ebextensions included SSL will be applied/rechecked after configuration deploy. Only when the `active_condition.sh` exits with 0 will SSL be applied, otherwise it will be teared down. SSL needs some minimal configuration:
* `CUSTOM_SSL_DOMAINS`: comma-separated list of domains
* `CUSTOM_SSL_EMAIL`: email registered with certificate

By default, `active_condition.sh` will check that these Environment Variables are not empty.

## Components

### SSL Traffic
The file `allow_ssl.config` opens port 443 for inbound traffic.

### Certbot installation
The file `install_certbot.config` applies 2 hooks that both check if certbot is installed and install it if necessary. This happens right after initial package installations or before new configuration deployment.

### NGINX Configuration
The file `nginx_ssl.config` installs 3 files to configure NGINX:
* `dhparam`
  * as recommended by [Mozilla's SSL Configuration Generator](https://ssl-config.mozilla.org/#server=nginx&version=1.19.8&config=intermediate&openssl=1.1.1d&hsts=false&guideline=5.6)
* `https.conf`
  * dynamically added to NGINX conf.d
  * as recommended by [Mozilla's SSL Configuration Generator](https://ssl-config.mozilla.org/#server=nginx&version=1.19.8&config=intermediate&openssl=1.1.1d&hsts=false&guideline=5.6)
* `http-01.conf`
  * added in default proxy above `location / {`
  * enables ACME webroot responses

### Apply SSL
The file `apply_ssl.config` installs 4 hooks to dynamically apply the SSL configuration:
* `active_condition.sh`
  * checks to see if SSL should be active
  * edit this file to change when SSL is activated
* `process_new_certs.sh`
  * links a new live certificate to the ebcert location
* `39inject_ssl_configuration.sh`
  * before new configuration deployment
  * enables/disables NGINX configuration to respond ACME http-01 challenge
* `01certbot_initiate.sh`
  * runs after configuration deployment
  * initiates certbot when active
  * enables/disables cron to run `certbot renew`
  * enables/disables NGINX `https.conf`
  * reloads nginx to apply new configuration
