### Install nginx on Ubuntu
It just installs nginx on your Ubuntu jammy/focal (22.04/20.04).<br />
### Variables
```
# nginx gpg URL
install_nginx_repo_key_url: "https://nginx.org/keys/nginx_signing.key"

# path to save the repo key
install_nginx_repo_key_file_path: "/usr/share/keyrings/nginx.gpg"

# nginx repository file content (edit only if you have changed install_nginx_repo_key_file_path)
install_nginx_repo_string: "deb [arch=amd64 signed-by=/usr/share/keyrings/nginx.gpg] http://nginx.org/packages/ubuntu {{ ansible_facts['distribution_release'] }} nginx"

# name of the repo file in /etc/apt/sources.d
install_nginx_repo_file: nginx

# nginx version to install, if this variable is not defined,
# the latest version will be installed
install_nginx_version: "1.24.0-1~jammy"
```