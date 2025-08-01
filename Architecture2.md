# Archtecture2. WordPress 2-Tier 아키텍쳐
이 프로젝트는 WordPress 웹 서버와 MySQL 데이터베이스 서버를 분리하여
**2-Tier**로 구성한 Vagrant 환경 설정 예시입니다.

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
```
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --reload
```
## 2. PHP

### 2.1 PHP 설치
```
sudo dnf install -y php php-mysqlnd
```
wordpress는 php로 실행되므로, 이를 위해 php와 mysql 데이터베이스 연결을 위한 php-mysqld 패키지를 설치한다.

## 3. WordPress

### 3.1 WordPress 압출 파일 가져오기
```
curl -o wordpress.tar.gz https://wordpress.org/lastest.tar.gz
```
### 3.2 WordPress 파일 추출
```
sudo tar xvf wordpress.tar.gz -C /var/www/html
```
### 3.3 WordPress config 파일 수정
```
sudo cp /var/www/html/wordpress/wp-config-sample.php /var/www/html/wordpress/wp-config.php
sudo vi /var/www/html/wordpress/wp-config.php

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
```
/** The name of the database for WordPress */
define( 'DB_NAME', 'wp2' );

/** Database username */
define( 'DB_USER', 'wp2-user' );

/** Database password */
define( 'DB_PASSWORD', 'qwer2' );

/** Database hostname */
define( 'DB_HOST', '192.168.56.12' );
```


