NGINX Plus, NGINX Plus App Protect WAF, NGINX App Protect DoS 설치 가이드


# NGINX Plus, NGINX App Protect (WAF, DOS)


NGINX Plus, 
NGINX Plus App Protect (WAF)는 NGINX Plus r22 버전부터 지원합니다.
NGINX App Protect (DoS)는 NGINX Plus r24 버전부터 지원합니다.






## NGINX Plus


NGINX Plus 설치는 크게 License 업로드, Repository 등록, Installation 과정으로 나뉩니다.


License는 `nginx-repo.crt`, `nginx-repo.key`를 말하며,  `/var/tmp` 위치에 업로드되어 있다고 가정합니다.






#### yum(Centos, RHEL)


##### License 업로드


```bash
sudo mkdir -p /etc/ssl/nginx && cp /var/tmp/nginx-repo.* /etc/ssl/nginx/
```


##### Repository 등록


```bash
sudo curl -sL https://cs.nginx.com/static/files/nginx-plus-7.repo \
    -o /etc/yum.repos.d/nginx-plus-7.repo
sudo yum makecache fast
```


##### Installation


```bash
sudo yum install -y nginx-plus ca-certificates
sudo systemctl enable --now nginx
```






#### apt-get(Ubuntu, Debian)


##### License 업로드


```bash
sudo mkdir -p /etc/ssl/nginx && cp /var/tmp/nginx-repo.* /etc/ssl/nginx/
```


##### Repository 등록


```bash
sudo wget https://cs.nginx.com/static/keys/nginx_signing.key
sudo apt-key add nginx_signing.key


sudo apt-get install apt-transport-https lsb-release ca-certificates


printf "deb https://pkgs.nginx.com/plus/ubuntu `lsb_release -cs` nginx-plus\n" | sudo tee /etc/apt/sources.list.d/nginx-plus.list
sudo wget -P /etc/apt/apt.conf.d https://cs.nginx.com/static/files/90pkgs-nginx
sudo apt-get update
```


##### Installation


```bash
sudo apt-get install -y nginx-plus
sudo systemctl enable --now nginx
```












## NGINX Plus + App Protect (WAF)


NGINX Plus 설치는 크게 License 업로드, Repository 등록, Installation 과정으로 나뉩니다.


License는 `nginx-repo.crt`, `nginx-repo.key`를 말하며,  `/var/tmp` 위치에 업로드되어 있다고 가정합니다.






#### yum(Centos 7.4+, RHEL 7.4+)


##### License 업로드


```bash
sudo mkdir -p /etc/ssl/nginx && cp /var/tmp/nginx-repo.* /etc/ssl/nginx/
```


##### Repository 등록


```bash
sudo yum install wget epel-release


sudo curl -sL https://cs.nginx.com/static/files/nginx-plus-7.repo -o /etc/yum.repos.d/nginx-plus-7.repo


sudo wget -P /etc/yum.repos.d https://cs.nginx.com/static/files/app-protect-7.repo
sudo yum makecache fast
```


##### Installation


```bash
sudo yum install -y nginx-plus ca-certificates
sudo yum install -y app-protect app-protect-attack-signatures
sudo systemctl enable --now nginx
```










#### apt-get(Ubuntu, Debian)


##### License 업로드


```bash
sudo mkdir -p /etc/ssl/nginx && cp /var/tmp/nginx-repo.* /etc/ssl/nginx/
```


##### Repository 등록


```bash
sudo wget https://cs.nginx.com/static/keys/nginx_signing.key
sudo apt-key add nginx_signing.key


sudo wget https://cs.nginx.com/static/keys/app-protect-security-updates.key
sudo apt-key add app-protect-security-updates.key


sudo apt-get install apt-transport-https lsb-release ca-certificates


printf "deb https://pkgs.nginx.com/plus/ubuntu `lsb_release -cs` nginx-plus\n" | sudo tee /etc/apt/sources.list.d/nginx-plus.list


printf "deb https://pkgs.nginx.com/app-protect/ubuntu `lsb_release -cs` nginx-plus\n" | sudo tee /etc/apt/sources.list.d/nginx-app-protect.list
printf "deb https://pkgs.nginx.com/app-protect-security-updates/ubuntu `lsb_release -cs` nginx-plus\n" | sudo tee -a /etc/apt/sources.list.d/nginx-app-protect.list


sudo wget -P /etc/apt/apt.conf.d https://cs.nginx.com/static/files/90pkgs-nginx
sudo apt-get update
```


##### Installation


```bash
sudo apt-get install -y nginx-plus
sudo apt-get install app-protect app-protect-attack-signatures


sudo systemctl enable --now nginx
```






### NGINX Plus + App-Protect 적용


NGINX Config File에서 적용합니다. 아래는 `nginx.conf`에서 NGINX 전체에 App Protect를 적용하는 예시입니다.


###### Load module


```nginx
user  nginx;
worker_processes  auto;


error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;


load_module modules/ngx_http_app_protect_module.so;


events {
    worker_connections  1024;
}


http {
    app_protect_enable on;  ## App Protect Enable ##
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
……[후략]
```


###### Nginx Reload And App Protect Apply


```bash
sudo nginx -t && sudo nginx -s reload
```


###### App Protect 적용 여부 확인


아래 명령어로 NGINX Master/Worker 프로세스와 App Protect가 사용하는 `bd_agent`, `bd-socket-plugin` 프로세스가 동작 중인지 확인합니다.


```bash
sudo ps aux | grep -e "bd_agent" -e "bd-socket-plugin" -e "nginx"
```










## NGINX Plus + App Protect (DoS)


NGINX Plus 설치는 크게 License 업로드, Repository 등록, Installation 과정으로 나뉩니다.


License는 `nginx-repo.crt`, `nginx-repo.key`를 말하며,  `/var/tmp` 위치에 업로드되어 있다고 가정합니다.






#### yum(Centos 7.4+, RHEL 7.4+)


##### License 업로드


```bash
sudo mkdir -p /etc/ssl/nginx && cp /var/tmp/nginx-repo.* /etc/ssl/nginx/
```


##### Repository 등록


```bash
sudo yum install wget epel-release


sudo curl -sL https://cs.nginx.com/static/files/nginx-plus-7.repo -o /etc/yum.repos.d/nginx-plus-7.repo


sudo wget -P /etc/yum.repos.d https://cs.nginx.com/static/files/app-protect-dos-7.repo
sudo yum makecache fast
```


##### Installation


```bash
sudo yum install -y nginx-plus ca-certificates
sudo yum install -y app-protect-dos
sudo systemctl enable --now nginx
```










#### apt-get(Ubuntu 18.04)


##### License 업로드


```bash
sudo mkdir -p /etc/ssl/nginx && cp /var/tmp/nginx-repo.* /etc/ssl/nginx/
```


##### Repository 등록


```bash
sudo wget https://cs.nginx.com/static/keys/nginx_signing.key
sudo apt-key add nginx_signing.key


sudo wget https://cs.nginx.com/static/keys/app-protect-security-updates.key
sudo apt-key add app-protect-security-updates.key


sudo apt-get install apt-transport-https lsb-release ca-certificates


printf "deb https://pkgs.nginx.com/plus/ubuntu `lsb_release -cs` nginx-plus\n" | sudo tee /etc/apt/sources.list.d/nginx-plus.list


printf "deb https://pkgs.nginx.com/app-protect/ubuntu `lsb_release -cs` nginx-plus\n" | sudo tee /etc/apt/sources.list.d/nginx-app-protect.list
printf "deb https://pkgs.nginx.com/app-protect-dos/ubuntu `lsb_release -cs` nginx-plus\n" | sudo tee /etc/apt/sources.list.d/nginx-app-protect-dos.list


sudo wget -P /etc/apt/apt.conf.d https://cs.nginx.com/static/files/90pkgs-nginx
sudo apt-get update
```


##### Installation


```bash
sudo apt-get install -y nginx-plus
sudo apt-get install app-protect-dos


sudo systemctl enable --now nginx
```


##### 설치 확인


```shell
sudo admd -v
```


### NGINX Plus + App-Protect DoS 적용


NGINX Config File에서 적용합니다. 아래는 `nginx.conf`에서 NGINX 전체에 App Protect를 적용하는 예시입니다.


###### Load module


```nginx
user  nginx;
worker_processes  auto;


error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;


load_module modules/ngx_http_app_protect_dos_module.so;


events {
    worker_connections  1024;
}


http {
    ### App Protect DoS 적용부 ###
    app_protect_dos_enable on;
    app_protect_dos_name "vs-example";
    app_protect_dos_policy_file "/etc/app_protect_dos/BADOSDefaultPolicy.json";
    app_protect_dos_monitor "example.com/";
    ### App Protect DoS 적용부 ###
    
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
……[후략]
```


###### Nginx Reload And App Protect Apply


```bash
sudo nginx -t && sudo nginx -s reload
```






###### App Protect DoS Selinux Policy 수정


```bash
sudo vi nginx.te
```
```bash
module nginx 1.0;
require {
    type unconfined_t;
    type httpd_t;
    type unconfined_service_t;
    class shm { associate read unix_read unix_write write };
}
allow httpd_t unconfined_service_t:shm { associate read unix_read unix_write write };
allow httpd_t unconfined_t:shm { unix_read unix_write };
```






```bash
checkmodule -M -m -o nginx.mod nginx.te
semodule_package -o nginx.pp -m nginx.mod
semodule -i nginx.pp


setsebool -P httpd_can_network_connect 1
```
