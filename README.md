    # web-was-db-installation

공통 : Ubuntu 24.04 OS사용

web서버 : apache httpd
was서버 : tomcat 
db서버 : MySQL


web서버 구축하기 (소스 컴파일 설치) 
개요 : web 서버 구축을 소스 컴파일로 설치하면서 구조를 파악하고 학습한다.

1. 필수 의존성 패키지 설치
   1.1 sudo apt update -y :  패키지를 최신으로 업데이트한다.
   1.2 컴파일 기본 도구 및 아파치 필수 의존성 패키지 설치
      ```bash
       sudo apt install -y build-essential \
        libapr1-dev \
        libaprutil1-dev \
        libpcre3-dev \
        libssl-dev \
        expat \
        libexpat1-dev
      ```
    install -y build-essential :  소스 컴파일할 때 필요한 `gcc`, `g++`, `make` 등을 한 번에 설치한다.
    libapr1-dev, libarprutil1-dev : 아파치 서버가 OS에 상관없이 잘 돌아가도록 돕는 핵심 라이블러리(APR)의 개발자 패키지
    libpcre3-dev : 정규표현식 처리를 위한 라이브러리
    libssl-dev : HTTPS(SSL/TLS) 보안 설정을 적용하기 위해 사용

2. 소스 파일 다운로드 및 압축해제 (관례적으로 사용하는 /usr/local/src 폴더에 다운로드)
   2.1 아차피 소스 코드 다운로드 : sudo wget https://dlcdn.apache.org/httpd/httpd-2.4.68.tar.bz2
   2.2 압축 해제 : sudo tar -xvf httpd-2.4.68

3. 아파치 컴파일 및 설치, 위치 : /usr/local/src/httpd-2.4.68
   3.1 configure을 통해 설계도를 만든다.
     ```bash
       sudo ./configure --prefix=/usr/local/apache2 \
       --enable-modules=most \
       --enable-mods-shared=all \
       --enable-so \
       --with-ssl
    ```
   3.2 sudo make : 설계도 대로 조립을 한다.
   3.3 sudo make install : 조립한 것을 배치한다.

---

## 🛠️ 4. 실무 운영을 위한 추가 설정 (Next Step)

컴파일 설치 완료 후, 실제 운영 환경(Production) 수준의 안정성과 보안을 확보하기 위해 진행해야 하는 필수 설정입니다.

4.1 웹 서버 전용 보안 계정 생성 (비로그인 설정)
- root 권한이 아닌 보안이 취약한 전용 유저(예: apache)를 생성하여 아파치 프로세스 구동 권한 부여
- 아파치 서버가 해킹당하더라도 공격자가 OS 시스템에 직접 로그인하지 못하도록 설정

  4.1.1 apache 시스템 그룹 생성 
    sudo groupadd -r apache

  4.1.2 로그인이 불가능한 apache 시스템 유저를 생성 (-r : 시스템 계정, -g: 그룹지정, -s : 셸 제한)
    sudo useradd -r -g apache -s /usr/sbin/nologin -d /usr/local/apache2 apache
    4.1.2.1 유저가 속한 그룹 확인
      id apache
    4.1.2.2 유저의 홈 디렉토리 및 비로그인 셸 설정 확인
      grep apache /etc/passwd
    4.1.2.3 그룹 생성 확인
      grep apache /etc/group
    4.1.2.4 로그인 막혔는지 테스트
      sudo su - apache
  
  4.1.3 아파치 설정 파일(httpd.conf) 수정 (/usr/local/apache2/conf/httpd.conf)
    User apache
    Group apache
  4.1.4 아파치 설치 폴더 소유권 변경
    sudo chown -R apache:apache /usr/local/apache2


4.2 systemd 서비스 등록 (자동 시작 설정)
- 서버 재부팅 시 아파치가 자동 구동되도록 /etc/systemd/system/httpd.service 등록 및 활성화
  4.2.1 sudo vi /etc/systemd/system/httpd.service 파일 생성 및 아래 내용 저장

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
  4.2.2 systemd 데몬 리로드 및 서비스 활성화
    sudo systemctl daemon-reload
    sudo systemctl enalbe httpd
  4.2.3 서비스 제어 명령어
    sudo systemctl start httpd
    sudo systemctl stop httpd
    sudo systemctl status httpd
    sudo systemctl restart httpd

4.3 OS 방화벽(UFW) 포트 개방
- 외부 접속을 위한 HTTP(80) 및 HTTPS(443) 포트 허용 설정
  4.3.1 방화벽 기본 상태 확인 및 활성화
    sudo ufw status
    sudo ufw enable
    sudo ufw allow 22/tcp, 80/tcp, 443/tcp
    sudo ufw relaod
    sudo ufw status varbose
    

4.4 SSL/TLS (HTTPS) 인증서 연동
- httpd-ssl.conf 활성화 및 암호화 통신 적용
- SSL 인증서 직접 생성 및 배치(테스트용 사설 인증서)
- 오픈소스 보안 도구인 openssl 을 이용하여 웹 서버 암호화 통신에 사용할 사설 인증서 키쌍(공개키, 비밀키)을 직접 생성
  4.4.1 아파치 SSL 모듈 및 설정 활성화
    LoadModule ssl_module modules/mod_ssl.so 주석 제거
    Include conf/extra/httpd-ssl.conf 작성
  4.4.2 SSL 인증서 발급 및 배치
    실무에서는 인증받은 인증서를 발급받아 사용, 나는 테스트용이므로 openssl을 이용하여 키쌍을 직접 생성한다.
    4.4.2.1 인증서 보관 디렉토리 생성 : sudo mkdir -p /usr/local/apache2/conf/ssl
    4.4.2.2 OpenSSL 명령어로 1년 유효한 인증서 및 비밀키 생성
      sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
      -keyout /usr/local/apache2/conf/ssl/server.key \  : 비밀키
      -out /usr/local/apache2/conf/ssl/server.crt       : 인증서
    4.4.2.3 httpd-ssl.conf 세부 설정
      발급받은 인증서 경로를 파일 내에 매핑해준다.
      SSLEngine on
      SSLCertificateFile "/usr/local/apache2/conf/ssl/ssl.crt"
      SSLCertificateKeyFile "/usr/local/apache2/conf/ssl/ssl.key"
    4.4.2.4 아파치 설정 검증 및 재시작
      sudo /usr/local/apache2/bin/apachectl configtest
      sudo systemctl restart httpd

  4.4.3 HTTPS 연동 확인 방법
    4.4.3.1 터미널에서 curl 명령어로 확인하기
      curl -kIv https://localhost : -k (인증서 검증 패스), -I(헤더 정보만 보기), -v(상세 통신 과정 출력)
    4.4.3.2 openssl 도구로 SSL 연결 상세 분석
      openssl s_client -connect localhost:443
    

        

4.5 가상 호스트(Virtual Host) 구성을 통한 다중 도메인 배포
- 하나의 Web 서버에서 여러 웹사이트를 격리하여 서비스하도록 설정



  💡 트러블슈팅: configtest 실행 시 socache_shmcb 에러가 나는 경우
- 에러 메시지: SSLSessionCache: 'shmcb' session cache not supported...
- 해결 방법: /usr/local/apache2/conf/httpd.conf 파일에서 아래 모듈의 주석(#)을 제거합니다.
  LoadModule socache_shmcb_module modules/mod_socache_shmcb.so



