# web-was-db-installation

공통 : Ubuntu 24.04 OS사용

web서버 : apache httpd
was서버 : tomcat 
db서버 : MySQL


<details>
<summary>🔍 웹서버 구축하기 (클릭하면 열립니다)</summary>
---
   
개요 : web 서버 구축을 소스 컴파일로 설치하면서 구조를 파악하고 학습한다.

---

## 1. 필수 의존성 패키지 설치
   1) sudo apt update -y : 패키지를 최신으로 업데이트한다.
   2) 컴파일 기본 도구 및 아파치 필수 의존성 패키지 설치
      ```bash
      sudo apt install -y build-essential \
        libapr1-dev \
        libaprutil1-dev \
        libpcre3-dev \
        libssl-dev \
        expat \
        libexpat1-dev
      ```
   3) 패키지 설명  
       1) install -y build-essential : 소스 컴파일할 때 필요한 `gcc`, `g++`, `make` 등을 한 번에 설치한다.
       2) libapr1-dev, libarprutil1-dev : 아파치 서버가 OS에 상관없이 잘 돌아가도록 돕는 핵심 라이브러리(APR)의 개발자 패키지
       3) libpcre3-dev : 정규표현식 처리를 위한 라이브러리
       4) libssl-dev : HTTPS(SSL/TLS) 보안 설정을 적용하기 위해 사용

## 2. 소스 파일 다운로드 및 압축해제 (관례적으로 사용하는 /usr/local/src 폴더에 다운로드)
   1) 아파치 소스 코드 다운로드 : sudo wget https://dlcdn.apache.org/httpd/httpd-2.4.68.tar.bz2
   2) 압축 해제 : sudo tar -xvf httpd-2.4.68

## 3. 아파치 컴파일 및 설치, 위치 : /usr/local/src/httpd-2.4.68
   1)  configure을 통해 설계도를 만든다.
        ```bash
           sudo ./configure --prefix=/usr/local/apache2 \
              --enable-modules=most \
              --enable-mods-shared=all \
              --enable-so \
              --with-ssl
        ```
   2) sudo make : 설계도 대로 조립을 한다.
   3) sudo make install : 조립한 것을 배치한다.

---

## 4. 실무 운영을 위한 추가 설정 

컴파일 설치 완료 후, 실제 운영 환경(Production) 수준의 안정성과 보안을 확보하기 위해 진행해야 하는 필수 설정입니다.

1) 웹 서버 전용 보안 계정 생성 (비로그인 설정)
- root 권한이 아닌 보안이 취약한 전용 유저(예: apache)를 생성하여 아파치 프로세스 구동 권한 부여
- 아파치 서버가 해킹당하더라도 공격자가 OS 시스템에 직접 로그인하지 못하도록 설정

  1) apache 시스템 그룹 생성 : sudo groupadd -r apache

  2) 로그인이 불가능한 apache 시스템 유저를 생성 (-r : 시스템 계정, -g: 그룹지정, -s : 셸 제한) : sudo useradd -r -g apache -s /usr/sbin/nologin -d /usr/local/apache2 apache
    
    1) 유저가 속한 그룹 확인 : id apache
    2) 유저의 홈 디렉토리 및 비로그인 셸 설정 확인 : grep apache /etc/passwd
    3) 그룹 생성 확인 : grep apache /etc/group
    4) 로그인 막혔는지 테스트 : sudo su - apache
  
  3) 아파치 설정 파일(httpd.conf) 수정 (/usr/local/apache2/conf/httpd.conf)
      ```bash
       User apache
       Group apache
      ```
  5) 아파치 설치 폴더 소유권 변경 : sudo chown -R apache:apache /usr/local/apache2


2) systemd 서비스 등록 (자동 시작 설정)
- 서버 재부팅 시 아파치가 자동 구동되도록 /etc/systemd/system/httpd.service 등록 및 활성화
  
  1) sudo vi /etc/systemd/system/httpd.service 파일 생성 및 아래 내용 저장
    ```ini
    [Unit]
    Description=The Apache HTTP Server
    After=network.target remote-fs.target nss-lookup.target

    [Service]
    Type=forking
    ExecStart=/usr/local/apache2/bin/apachectl start
    ExecStop=/usr/local/apache2/bin/apachectl stop
    ExecReload=/usr/local/apache2/bin/apachectl graceful
    PrivateTmp=true

    [Install]
    WantedBy=multi-user.target
    ```
    
  2) systemd 데몬 리로드 및 서비스 활성화
    sudo systemctl daemon-reload
    sudo systemctl enable httpd
    
  3) 서비스 제어 명령어
    sudo systemctl start httpd
    sudo systemctl stop httpd
    sudo systemctl status httpd
    sudo systemctl restart httpd


3) OS 방화벽(UFW) 포트 개방
- 외부 접속을 위한 HTTP(80) 및 HTTPS(443) 포트 허용 설정
  
  1) 방화벽 기본 상태 확인 및 활성화
    sudo ufw status
    sudo ufw enable
    sudo ufw allow 22/tcp
    sudo ufw allow 80/tcp
    sudo ufw allow 443/tcp
    sudo ufw reload
    sudo ufw status verbose
  

4) SSL/TLS (HTTPS) 인증서 연동
- httpd-ssl.conf 활성화 및 암호화 통신 적용
- SSL 인증서 직접 생성 및 배치(테스트용 사설 인증서)
- 오픈소스 보안 도구인 openssl 을 이용하여 웹 서버 암호화 통신에 사용할 사설 인증서 키쌍(공개키, 비밀키)을 직접 생성
  
  1) 아파치 SSL 모듈 및 설정 활성화 (`conf/httpd.conf`)
    LoadModule ssl_module modules/mod_ssl.so 주석 제거
    Include conf/extra/httpd-ssl.conf 주석 제거
    *(주의: httpd-ssl.conf 내부의 기본 <VirtualHost _default_:443> 블록은 아래 vhost 설정과 충돌할 수 있으므로 주석 처리하거나 제거하는 것이 좋습니다.)*
  
  2) SSL 인증서 발급 및 배치
    실무에서는 인증받은 인증서를 발급받아 사용, 나는 테스트용이므로 openssl을 이용하여 키쌍을 직접 생성한다.
    
    1) 인증서 보관 디렉토리 생성 : 
      sudo mkdir -p /usr/local/apache2/conf/ssl
      
    2) OpenSSL 명령어로 1년 유효한 인증서 및 비밀키 생성
      ```bash
      sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
        -keyout /usr/local/apache2/conf/ssl/server.key \
        -out /usr/local/apache2/conf/ssl/server.crt
      ```
      * `-keyout` : 생성될 비밀키(Private Key) 경로
      * `-out` : 생성될 인증서(Certificate) 경로
    
    3) 아파치 설정 검증 및 재시작
      sudo /usr/local/apache2/bin/apachectl configtest
      sudo systemctl restart httpd

  3) HTTPS 연동 확인 방법
     1) 터미널에서 curl 명령어로 확인하기
      curl -kIv https://localhost
      * `-k` (인증서 검증 패스), `-I` (헤더 정보만 보기), `-v` (상세 통신 과정 출력)
    
     2) openssl 도구로 SSL 연결 상세 분석
      openssl s_client -connect localhost:443
    
5) HTTP(80) -> HTTPS(443) 자동 리다이렉트 설정
- 사용자가 보안되지 않은 HTTP(80 포트)로 접속하더라도 자동으로 보안 프로토콜인 HTTPS(443 포트)로 전환되도록 가상 호스트(VirtualHost)를 구성합니다.
  
  1) 아파치 리다이렉트 필수 모듈 활성화 (`conf/httpd.conf`)
    아래 모듈들의 주석(#)이 해제되어 있는지 확인합니다.
    ```apache
    LoadModule alias_module modules/mod_alias.so
    LoadModule vhost_alias_module modules/mod_vhost_alias.so
    ```

  2) 가상 호스트 규칙 추가
    `conf/httpd.conf` 파일 맨 아랫줄에 아래 설정을 추가하여 포트별 가상 호스트를 매핑합니다.
    ```apache
    # 80 포트 요청 -> 443 포트로 자동 리다이렉트
    <VirtualHost *:80>
        ServerName localhost
        Redirect permanent / https://localhost/
    </VirtualHost>

    # 443 포트 요청 -> SSL 인증서 매핑 및 웹 서비스 제공
    <VirtualHost *:443>
        ServerName localhost
        DocumentRoot "/usr/local/apache2/htdocs"
    
        SSLEngine on
        SSLCertificateFile "/usr/local/apache2/conf/ssl/server.crt"
        SSLCertificateKeyFile "/usr/local/apache2/conf/ssl/server.key"
    </VirtualHost>
    ```

  3) 검증 및 프로세스 상태 확인
    설정 반영 후 아파치 문법 검사 및 재시작을 진행하고 리다이렉트 작동 여부와 포트를 검증합니다.
    ```bash
    sudo /usr/local/apache2/bin/apachectl configtest
    sudo systemctl restart httpd
    ```
    
    * **리다이렉트 및 헤더 결과 확인 (301 응답 및 Location 확인)**
      curl -kI http://localhost
      curl -kI https://192.168.111.133
      *(주의: 크롬 브라우저의 강력한 캐싱 기능으로 인해 리다이렉트가 반영 안 되는 것처럼 보일 수 있으니, 테스트 시 크롬 시크릿 창(Ctrl+Shift+N)이나 위 curl 명령어를 사용해 검증합니다.)*
      
    * **네트워크 포트 활성화 여부 확인 (443 포트 Listen 상태 확인)**
      sudo ss -anlp | grep 443

4.6  아파치 버전 및 OS 정보 숨기기 (보안 고도화)
4.7  로그 로테이션 (Log Rotation) 설정 (서버 장애 방지)
4.8  로그 로테이션 (Log Rotation) 설정 (서버 장애 방지)
4.9  멀티 프로세싱 모듈(MPM) 최적화 (성공적인 WAS 연동을 위한 발판)

💡 트러블슈팅: configtest 실행 시 socache_shmcb 에러가 나는 경우
- 에러 메시지: `SSLSessionCache: 'shmcb' session cache not supported...`
- 해결 방법: `/usr/local/apache2/conf/httpd.conf` 파일에서 아래 모듈의 주석(#)을 제거합니다.
  LoadModule socache_shmcb_module modules/mod_socache_shmcb.so
</details>

<details>
<summary>🌐 WAS(Tomcat) 서버 구축하기 (소스 컴파일 설치)</summary>

   1. Java 설치
     java -version
     sudo apt update
     sudo apt install openjdk-17-jdk -y

   2. Tomcat 전용 계정 생성
      sudo useradd -m -U -d /opt/tomcat -s /bin/false tomcat
      해석
         useradd : 새로운 사용자 계정을 생성
         -m : 계정을 만들 때 홈 디렉토리도 함께 생성
         -U : 생성하는 사용자명과 동일한 이름의 그룹을 자동으로 함께 생성하고 그 그룹에 포함
         -d /opt/tomcat : 생성할 사용자의 홈 디렉토리 위치를 /apt/tomcat으로 지정
         -s /bin/false : 리눅스 시스템에 로그인 불가
         -tomcat : 최종적으로 생성할 이 계정의 이름 

   3. Tomcat 다운로드 및 압축해제
      이동 : cd /tmp : 임시파일 폴더라 나중에 정리하기 편함 
      폴더 만들기 : sudo mkdir /opt/tomcat
      다운 : sudo wget https://dlcdn.apache.org/tomcat/tomcat-10/v10.1.55/bin/apache-tomcat-10.1.55.tar.gz
      해제 : sudo tar -xzvf apache-tomcat-10.1.55.tar.gz -C /opt/tomcat/ --strip-components=1
         해석
            -xzf : x는 해제, z는 .gz 형식으로 압축된 파일 처리 f는 뒤에 나올 압축 파일명을 지정하겠다 v는 풀릴때 목록 보여준다.
            -C /opt/tomcat : 압축 풀 결과물들을 현재 위치가 아닌 /opt/tomcat 디렉토리로 해라
            --strip-components=1 : 압축 파일 안에 들어있는 가장 상위 폴더는 빼라

    4. 권한 변경
       톰캣 폴더의 소유자를 tomcat으로 변경한다.
       sudo chown -R tomcat:tomcat /opt/tomcat : -R : 하위 폴더와 파일까지 전부 tomcat이 전용 계정이므로 권한을 주기위해  바꾼다.

       톰캣 실행 권한
       sudo chmod +x /opt/tomcat/bin/

    5. 수동 실행 테스트
       실행 : sudo -u tomcat /apt/tomcat/bin/startup.sh
       확인 : ss -tnlp | grep 8080 
          해석
               -t는 TCP 프로토콜을 사용하는 포트 보여준다. 
               -u는 UDP 프로토콜을 사용하는 포트 보여준다.
               -l은 연결을 요청받기 위해 대기 중인 포트만 보여준다.
               -p는 이 포트를 어떤 프로그램(프로세스)이 쓰고있는지 프로그램 이름과 PID를 보여준다.
               -n는 포트번호 보여준다.

    6. systemd 서비스 등록
          1. 파일 생성 : sudo vi /etc/systemd/system/tomcat.service
          2. 서비스 등록 : sudo systemctl daemon-reload
          3. 서비스 활성화 : sudo systemctl enable tomcat
          4. 서비스 시작 : sudo systemctl start tomcat
       

</details>

<details>
<summary>🌐 Web Was 연동하기</summary>

1) httpd.conf에서 프록시 모듈 활성화 (경로: sudo nano /usr/local/apache2/conf/httpd.conf)
      LoadModule proxy_module modules/mod_proxy.so
      LoadModule proxy_http_module modules/mod_proxy_http.so

2) 아파치 설정 파일 문법 검사 : sudo /usr/local/apache2/bin/apachectl configtest

3) 가상 호스트 및 리버스 프록시 설정 : sudo nano /usr/local/apache2/conf/extra/httpd-vhosts.conf

[httpd-vhosts.conf 내부 설정 내용]

    # 80 포트 요청 -> 443 포트로 자동 리다이렉트
    <VirtualHost *:80>
        ServerName localhost
        Redirect permanent / https://localhost/
    </VirtualHost>

    # 443 포트 요청 -> SSL 인증서 매핑 및 웹 서비스 제공
    <VirtualHost *:443>
        ServerName localhost
        DocumentRoot "/usr/local/apache2/htdocs"

        SSLEngine on
        SSLCertificateFile "/usr/local/apache2/conf/ssl/server.crt"
        SSLCertificateKeyFile "/usr/local/apache2/conf/ssl/server.key"

        # 리버스 프록시 설정 추가
        ProxyPreserveHost On
        ProxyPass / http://localhost:8080/
        ProxyPassReverse / http://localhost:8080/
    </VirtualHost>

</details>


<details>
<summary>🌐 DB 설치하고 연동하기 (MySQL)</summary>

Web과 WAS는 서비스 아키텍처 이해를 위해 소스 컴파일로 설치해 보았고, DB는 데이터 저장소의 안정적인 운영과 백업/복구 매커니즘 학습에 집중하기 위해 안정적인 패키지 설치로 진행한다.

---

### 1. MySQL 설치

1) apt 업데이트 :
sudo apt update

2) mysql 설치 :
sudo apt install -y mysql-server

3) 버전 확인 :
mysql --version

4) 서비스 확인 :
sudo systemctl status mysql

5) 부팅 시 자동 시작 :
sudo systemctl enable mysql

---

### 2. 보안 설정

보안에 취약한 기본 설정들을 실무 가이드에 맞게 강화해 주는 필수 보안 초기화 스크립트이다.
sudo mysql_secure_installation

[보안 초기화 스크립트 주요 단계]
1) VALIDATE PASSWORD COMPONENT (비밀번호 복잡도 검사 기능 활성화)
2) Change the password for root? (root 비밀번호 변경)
3) Remove anonymous users? (익명 사용자 삭제)
4) Disallow root login remotely? (root 계정의 원격 접속 차단)
5) Remove test database and access to it? (test 데이터베이스 삭제)
6) Reload privilege tables now? (권한 테이블 즉시 적용)

---

### 3. 서비스 DB 생성

MySQL 접속 후 다음 명령어를 실행한다.
1) DB 생성 :
CREATE DATABASE shopmall;

2) DB 확인 :
SHOW DATABASES;

---

### 4. 서비스 계정 생성

* 주의: 실무에서는 보안을 위해 root 계정을 절대 직접 사용하지 않는다.

1) 사용자 생성 :
CREATE USER 'shopadmin'@'localhost' IDENTIFIED BY 'Password123123!';

2) 권한 부여 :
GRANT ALL PRIVILEGES ON shopmall.* TO 'shopadmin'@'localhost';

3) 권한 반영 :
FLUSH PRIVILEGES;

---

### 5. 덤프 파일을 이용한 테이블 생성

1) mysql 덤프 파일을 DB 서버에 옮긴 후 mysql에 적용한다 :
mysql -u shopadmin -p shopmall < ./Dump.sql

2) mysql에 접속 :
mysql -u shopadmin -p

3) DB 사용 :
USE shopmall;

4) Table 조회 :
SHOW TABLES;

5) 데이터 조회 (검증용 상위 5개) :
SELECT * FROM 테이블명 LIMIT 5;

---

### 6. 스프링 프로젝트 연결하기

1) ROOT.war 파일을 DB서버에 옮긴다.

2) ROOT.war 파일을 설치한 톰캣 경로에 복사한다 :
sudo cp ./ROOT.war /opt/tomcat/webapps/

3) webapps 폴더 내부의 소유권을 tomcat 계정으로 변경한다 :
sudo chown -R tomcat:tomcat /opt/tomcat/webapps/

4) 톰캣 재실행 :
sudo systemctl restart tomcat

* 배포 메커니즘 참고:
tomcat의 server.xml 설정을 보면 <Host name="localhost" appBase="webapps" unpackWARs="true" autoDeploy="true"> 라는 코드가 있어서 webapps 폴더 밑으로 war파일을 복사하면 자동으로 압축이 풀리며 배포된다.

5) 연동된것 확인

<img width="1491" height="873" alt="image" src="https://github.com/user-attachments/assets/c4ed7139-6f24-4879-9bc0-1077f5502fcb" />

</details>



</details>







