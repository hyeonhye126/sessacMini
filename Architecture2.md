# Architecture2. WordPress 2-Tier 아키텍쳐
![아키텍처 Q2](https://github.com/user-attachments/assets/6a9b795d-f849-4fcf-a43e-82a65c5ef22d)

이 프로젝트는 WordPress 웹 서버와 MySQL 데이터베이스 서버를 분리하여
**2-Tier**로 구성한 Vagrant 환경 설정 예시입니다. 
<br><br>
## Vagrantfile 내용
```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :
vm_image = "nobreak-labs/rocky-9"
vm_subnet = "192.168."

Vagrant.configure("2") do |config|
  config.vm.synced_folder ".", "/vagrant", disabled: true

  config.vm.define "wp2" do |node|
    node.vm.box = vm_image
    node.vm.provider "virtualbox" do |vb|
      vb.name = "wp2"
      vb.cpus = 2
      vb.memory = 2048
    end

    node.vm.network "private_network", ip: vm_subnet + "56.11", nic_type: "virtio"
    node.vm.hostname = "wp2"
  end
  
  # DB Server
  config.vm.define "db2" do |node|
    node.vm.box = vm_image
    node.vm.provider "virtualbox" do |vb|
      vb.name = "db2"
      vb.cpus = 1
      vb.memory = 1024
    end

    node.vm.network "private_network", ip: vm_subnet + "56.12", nic_type: "virtio"
    node.vm.network "private_network", ip: vm_subnet + "57.12", nic_type: "virtio"
    node.vm.hostname = "db2"
  end
end
```
<br><br>

# Web 서버 구축


## 1. HTTP

### 1.1 HTTP 설치

```bash
sudo dnf install -y httpd
```
### 1.2 HTTP 시작 및 재부팅 시 자동 시작
```bash
sudo systemctl start httpd
sudo systemctl enable httpd
```
### 1.3 HTTP 방화벽 열기
```bash
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --reload
```
<br>

## 2. PHP

### 2.1 PHP 설치
```bash
sudo dnf install -y php php-mysqlnd
```

wordpress는 php로 실행되므로, 이를 위해 php와 mysql 데이터베이스 연결을 위한 php-mysqld 패키지를 설치한다.

<br>


## 3. WordPress

### 3.1 WordPress 압출 파일 가져오기
```bash
curl -o wordpress.tar.gz https://wordpress.org/lastest.tar.gz
```
### 3.2 WordPress 파일 추출
```bash
sudo tar xvf wordpress.tar.gz -C /var/www/html
```
### 3.3 WordPress config 파일 수정
```bash
sudo cp /var/www/html/wordpress/wp-config-sample.php /var/www/html/wordpress/wp-config.php
sudo vi /var/www/html/wordpress/wp-config.php
```
```php
/** The name of the database for WordPress */
define( 'DB_NAME', 'database 이름' );

/** Database username */
define( 'DB_USER', 'mysql user 이름' );

/** Database password */
define( 'DB_PASSWORD', 'mysql user 비밀번호' );

/** Database hostname */
define( 'DB_HOST', 'DB 서버 ip 주소' );
```
WordPress config 파일을 작성하기 위해 기존에 있는 sample 파일을 복사 후 config 파일에 db에 대한 내용을 작성한다.
```php
/** The name of the database for WordPress */
define( 'DB_NAME', 'wp2' );

/** Database username */
define( 'DB_USER', 'wp2-user' );

/** Database password */
define( 'DB_PASSWORD', 'qwer2' );

/** Database hostname */
define( 'DB_HOST', '192.168.56.12' );
```
### 3.4 Apache 서버의 WordPress.conf 파일 수정
```bash
sudo vi /etc/httpd/conf.d/wordpress.conf
```
```apache
<VirtualHost *:80>
        ServerName example.com
        DocumentRoot /var/www/html/wordpress
        <Directory "/var/www/html/wordpress">
                AllowOverride All
        </Directory>
</VirtualHost>
```
```bash
sudo systemctl restart httpd
```
<br>

## 4. SELinux
### 4.1 SELinux 설정
```sudo
sudo setsebool -P httpd_can_network_connect_db 1
```
웹서버가 외부 DB와 연결될 수 있도록 SELinux의 보안 설정을 변경한다.

<br>


## 5. MySQL
### 5.1 MySQL 설치
```bash
sudo dnf install -y mysql
```
### 5.2 MySQL 접속
```bash
mysql -h "DB 서버 ip 주소" -u "mysql user 이름" -p
```
```bash
mysql -h 192.168.56.12 -u wp2-user -p
```

<br><br>


# DB 서버 구축
## 1. MySQL 
### 1.1 MySQL server 설치
```bash
sudo dnf install -y mysql mysql-server
```
### 1.2 MySQL 시작 및 재부팅 시 자동 시작
```bash
sudo systemctl start mysqld
sudo systemctl enable mysqld
```
### 1.3 MySQL 서비스 방화벽 열기
```bash
sudo firewall-cmd --add-service=mysql --permanent
sudo firewall-cmd --reload
```
### 1.4 MySQL 접속
```bash
sudo mysql -u root
```
### 1.5 MySQL DB 생성 및 USER 생성, 권한 부여
```mysql
mysql> create database "database 이름";

mysql> create user 'mysql user 이름'@'DB 서버의 게이트웨이' identified by 'mysql user 비밀번호';

mysql>  grant all privileges on "database 이름".* to 'mysql user 이름'@'DB 서버의 게이트웨이';
```
```mysql
mysql> create database wp2;
Query OK, 1 row affected (0.02 sec)

mysql> CREATE USER 'wp2-user'@'192.168.56.11' IDENTIFIED BY 'qwer2';
Query OK, 0 rows affected (0.01 sec)

mysql> GRANT ALL PRIVILEGES ON wp2.* TO 'wp2-user'@'192.168.56.11';
Query OK, 0 rows affected (0.01 sec)
```
#### 1.5.1 (추가) bind-address로 접속 허용
```
sudo vi /etc/my.cnf
```
