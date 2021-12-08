## OpenShift Cluster Certificate Management

### 1. 인증서를 사용하는 이유

중요한 데이터를 다루는 대부분의 웹 사이트들은 HTTPS(HTTP Secure)를 사용하고 있습니다. SSL 인증서는 HTTPS 통신을 통해 웹 사이트의 콘텐츠가 브라우저에 전송되는 동안 사이트에서 주고받는 모든 데이터를 암호화 합니다. 엄밀히 말하면 보안이 적용된 통신 채널을 제공한다고 할 수 있습니다.



네트워크 데이터가 암호화된다면, 중간에 공격자가 패킷을 열람하더라도 데이터가 유출되는 것을 막을 수 있습니다.  SSL 인증서를 통해 클라이언트(브라우저)와 서버 간 통신을 보안하고 통신 과정에서 발생할 수 있는 스니핑(데이터 엿보기), 피싱(유사 사이트), 데이터 변조 등의 해킹을 방지할 수 있습니다. SSL은 대칭키 암호화 방식과 공개키 암호화 방식을 사용합니다.



**SSL 인증서**는 CA (Certificate Authority, 인증 기관)라고도 하는 인증서 발급 회사에서 시작하는 ROOT 인증서에서 시작합니다. 일반적으로 인증서는 ROOT, Intermediate(중간 인증서), Leaf(서버 인증서)  3단계로 구성되어 있고 이를 인증서 체인(Certificate Chain)이라고 합니다. 사용자가 구입하는 **SSL 인증서**는 Leaf 인증서를 의미하며 이는 인증서 체인의 일부이지 전체가 아닙니다. 이 3개의 인증서 체인은 하위 구조의 인증서를 서명하고 상위 구조의 인증서를 참고하는 방식으로 만들어집니다.



즉, 인증서 체인은 **SSL 인증서(서버 인증서)** 에서 시작하여 ROOT 인증서로 끝나는 인증서 목록으로 구성됩니다. 서버의 인증서를 신뢰하려면 해당 서명을 ROOT CA로 다시 추적할 수 있어야 합니다. 브라우저가 웹 사이트에 접속하면 웹 사이트의 인증서를 다운로드하면서 해당 인증서를 ROOT에 다시 연결하기 시작합니다. 체인을 따라가며 신뢰할 수 있는 ROOT 인증서에 도달할 때까지 계속해서 역추적합니다.



ROOT CA 인증서는 CA에서 자체 서명한(Self-Signed) 인증서로 공개키 기반 암호화를 사용합니다. 모든 유효한 **SSL 인증서**는 업계에서 보안 리더로 알려진 신뢰할 수 있는 CA가 발행한 ROOT CA 인증서 아래 체인에 위치합니다. 



중간 인증서는 ROOT CA 인증서와 **SSL 인증서 **사이에 구분을 만들어 위험을 완화하도록 설계된 인증서입니다. 이는 ROOT 인증서가 가장 많은 권한을 갖고 보호되어야 하기 때문에 ROOT 인증서가 손상될 경우를 대비합니다.



### 2. 플랫폼의 인증서 관리

OpenShift Container Platform의 프레임워크에는 TLS 인증서를 통한 암호화를 활용하는 REST 기반 HTTPS 통신을 사용하는 여러 구성 요소가 있습니다. OpenShift Container Platform의 설치 관리자는 설치 중에 이러한 인증서를 구성합니다. 이 트래픽을 생성하는 몇 가지 기본 구성요소가 있습니다.

- Master API (API 서버 및 Controller)
- etcd
- Node
- Registry
- Router



### 3. 사용자 정의 인증서 구성

초기 설치 중 또는 인증서를 다시 배포할 때  API 서버 및 웹 콘솔의 공용 호스트 이름에 대해 사용자 정의 제공 인증서를 구성할 수 있습니다. 또한, 사용자 정의 CA도 사용할 수 있습니다.



### 3.1) OpenSSL을 통한 자체 서명 인증서 생성

**1) OpenSSL로 Root CA 생성 및 SSL 인증서 발급**

웹 서비스에 HTTPS를 적용할 경우 SSL를 VeriSign이나 Thawte, GeoTrust와 같은 공인 기관에서 인증서를 발급받아야 하지만 비용이 발생하므로 실제 운영 서버가 아닌 개발 서버에 발급 받는 경우에는 부담이 될 수 있습니다.

이러한 경우, OpenSSL을 이용하여 인증기관을 생성하고 Self Signed Certificate(자체 서명 인증서)를 생성하고 SSL 인증서를 발급할 수 있습니다. 

이렇게 발급된 SSL 인증서를 설치하여 HTTPS 서비스를 제공할 수 있습니다.



- **Self Signed Certificate(자체 서명 인증서)란?**

  인증서(digital certificate)는 개인키 소유자의 공개키(public key)에 인증기관의 개인키로 전자서명한 데이터입니다. 

  모든 인증서는 발급기관(CA)이 있어야 하나 최상위에 있는 인증기관(Root CA)은 서명해줄 상위 인증기관이 없으므로 Root CA의 개인키로 스스로의 인증서에 서명하여 최상위 인증기관의 인증서를 만듭니다.

  이렇게 스스로 서명한 Root CA 인증서를 Self Signed Certificate(SSC)라고 부릅니다.

  IE, FireFox, Chrome 등의 웹 브라우저의 제작사는 VeriSign이나 comodo와 같은 유명 Root CA들의 인증서를 신뢰하는 CA로 브라우저에 미리 탑재해 놓습니다.

  이런 기관에서 발급된 SSL 인증서를 사용해야 웹 브라우저에서는 해당 SSL 인증서를 신뢰할 수 있는데 OpenSSL로 생성한 Root CA와 SSL 인증서는 브라우저가 모르는 기관이 발급한 인증서이므로 보안 경고를 발생시키지만 테스트 용도로 사용하는 것에는 문제가 없습니다.

- **Certificate Signning Request?**

  공개키 기반(PKI)은 Private Key(개인키)와 Public Key(공개키)로 이루어져 있습니다.

  인증서라고 하는 것은 내 공개키가 맞다고 인증기관(CA)이 전자서명하여 주는 것이며 나와 보안 통신을 하려는 당사자는 내 인증서를 구해서 그 안에 있는 공개키를 이용하여 보안 통신을 할 수 있습니다.

  CSR(Certificate Signing Request)은 인증기관에 인증서 발급 요청을 하는 특별한 ASN.1 형식의 파일이며(PKCS#10 - RFC2986) 그 안에는 내 공개키 정보와 사용하는 알고리즘 정보 등이 들어 있습니다.

  개인키는 외부에 유출되면 안 되므로 저런 특별한 형식의 파일을 만들어서 인증기관에 전달하여 인증서를 발급 받습니다.

  SSL 인증서 발급시 CSR 생성은 Web Server에서 이루어지는데 Web Server 마다 방식이 다르므로 사용자들이 CSR 생성 등을 어려워하기 때문에 인증서 발급 대행 기관에서 개인키까지 생성해서 보내줍니다.



### 3.2) Root CA 인증서 생성

OpenSSL을 통해 Root CA의 개인키와 인증서를 생성해 보도록 하겠습니다.

**1) CA가 사용할 RSA Key Pair (Public, Private Key) 생성**

- 2048 bit 개인키 생성

  ```bash
  $ openssl genrsa -aes256 -out test-rootca.key 2048
  
  -- 출력 메시지 --
  Generating RSA private key, 2048 bit long modulus (2 primes)
  ..................................................................................+++++
  .........................+++++
  e is 65537 (0x010001)
  Enter pass phrase for test-rootca.key:  // 암호 입력
  Verifying - Enter pass phrase for test-rootca.key: // 암호 확인
  ```

  > 개인키 분실에 대비하여 AES 256 bit로 암호화합니다. AES이므로 암호(pass phrase)를 분실하면 개인키를 얻을 수 없으니 꼭 기억해야 합니다.

- 개인키 권한 설정 (group, other permission 제거)

  ```bash
  $ chmod 600 test-rootca.key 
  ```

  ```bash
  [hyou-redhat.com@bastion ssl-test]$ ls -rlt
  total 4
  -rw-------. 1 hyou-redhat.com users 1766 Nov 26 08:02 test-rootca.key
  ```

  > 개인키의 유출 방지를 위해 소유자만 읽을 수 있도록 group과 other의 permission을 모두 제거하는 것이 좋습니다.

- CSR (Certificate Signing Request) 생성을 위한 openssl 설정 파일을 만들고 rootca_openssl.conf (변경 가능)로 저장

  ```bash
  cat << EOF > rootca_openssl.conf
  [ req ]
  default_bits            = 2048
  default_md              = sha1
  default_keyfile         = test-rootca.key
  distinguished_name      = req_distinguished_name
  extensions             = v3_ca
  req_extensions = v3_ca
   
  [ v3_ca ]
  basicConstraints       = critical, CA:TRUE, pathlen:0
  subjectKeyIdentifier   = hash
  ##authorityKeyIdentifier = keyid:always, issuer:always
  keyUsage               = keyCertSign, cRLSign
  nsCertType             = sslCA, emailCA, objCA
  [req_distinguished_name ]
  countryName                     = Country Name (2 letter code)
  countryName_default             = KR
  countryName_min                 = 2
  countryName_max                 = 2
  
  # 회사명 입력
  organizationName              = Red Hat
  organizationName_default      = Red Hat Korea
   
  # 부서 입력
  #organizationalUnitName          = Organizational Unit Name (eg, section)
  #organizationalUnitName_default  = Condor Project
   
  # SSL 서비스할 domain 명 입력
  commonName                      = ${DOMAIN_NAME}
  commonName_default              = test's Self Signed CA
  commonName_max                  = 64 
  EOF
  ```

- Root CA용 CSR 요청 파일을 생성합니다.

- 인증서 요청 생성

  ```bash
  $ openssl req -new -key test-rootca.key -out test-rootca.csr -config rootca_openssl.conf
  
  -- 출력 메시지 ---
  Enter pass phrase for test-rootca.key: # test-rootca.key 암호 입력
  You are about to be asked to enter information that will be incorporated
  into your certificate request.
  What you are about to enter is what is called a Distinguished Name or a DN.
  There are quite a few fields but you can leave some blank
  For some fields there will be a default value,
  If you enter '.', the field will be left blank.
  -----
  Country Name (2 letter code) [KR]:
  Red Hat [Red Hat Korea]:
  *.apps.cluster-0c41.sandbox1202.opentlc.com [tests Self Signed CA]:
  ```

- 10년 동안 사용 할 수 있는 Self-Signed 인증서를 생성합니다. 인증서는 -out 옵션 뒤에 기술한 파일명 (ex: test-rootca.crt)으로 생성됩니다.

  - `-extenstions v3_ca` 옵션을 추가해야 합니다.

    ```bash
    $ openssl x509 -req -days 3650 \
    -extensions v3_ca \
    -set_serial 1 \
    -in test-rootca.csr \
    -signkey test-rootca.key \
    -out test-rootca.crt \
    -extfile rootca_openssl.conf
    
    --- 출력 메시지 ---
    Signature ok
    subject=C = KR, O = Red Hat Korea, CN = lesstifs Self Signed CA
    Getting Private key
    Enter pass phrase for test-rootca.key: # test-rootca.key 암호 입력
    ```

    > 서명에 사용할 해시 알고리즘을 변경하려면 *-sha256*, *-sha384*, *-sha512*처럼 해시를 지정하는 옵션을 전달해 줍니다. 기본값은 *-sha256*이며 **OpenSSL 1.0.2** 이상이 필요합니다. 

- 제대로 생성이 되었는지 확인을 위해 인증서의 정보를 출력합니다.

  ```bash
  $ openssl x509 -text -in test-rootca.crt
  ```

  - 출력 정보

    ```bash
    Certificate:
        Data:
            Version: 3 (0x2)
            Serial Number: 1 (0x1)
            Signature Algorithm: sha256WithRSAEncryption
            Issuer: C = KR, O = Red Hat Korea, CN = tests Self Signed CA
            Validity
                Not Before: Nov 26 08:23:36 2021 GMT
                Not After : Nov 24 08:23:36 2031 GMT
            Subject: C = KR, O = Red Hat Korea, CN = tests Self Signed CA
            Subject Public Key Info:
                Public Key Algorithm: rsaEncryption
                    RSA Public-Key: (2048 bit)
                    Modulus:
                        00:ba:f0:96:f3:63:ba:f8:a0:b8:24:81:c3:cc:d6:
                        76:ac:ad:fd:58:18:1a:a1:b5:c3:c9:24:b7:36:8f:
                        c8:da:75:1d:3e:bf:74:7e:6f:c3:20:7a:e3:30:34:
                        8b:e6:9f:33:35:81:e4:e6:0b:e0:96:8b:4f:d5:4b:
                        62:cb:e2:f3:a2:ba:0d:6f:bd:08:d4:22:4e:ce:be:
                        8d:29:51:1c:0a:23:95:7f:ac:2c:cf:2f:e0:2e:e3:
                        8a:63:a0:b0:28:c4:71:3b:b7:f5:d5:1b:f3:55:68:
                        0d:8a:e3:0a:6d:12:d0:ef:10:8d:11:3e:eb:70:48:
                        da:67:b2:c3:1f:5a:b8:44:e4:38:39:df:fe:b9:f4:
                        86:0e:f9:32:a2:97:a0:80:d9:f1:97:8b:f7:f9:62:
                        a6:eb:1e:0f:88:94:24:dd:9a:e3:04:91:e7:71:03:
                        a2:28:b4:31:9e:65:8c:93:b0:78:7c:90:42:9c:7d:
                        2e:6a:b7:3d:0f:bd:8b:a3:d8:0c:db:0c:58:5d:18:
                        57:d8:58:51:91:a5:b7:48:21:8c:f4:bb:05:34:ce:
                        8f:e6:5c:cf:ff:3b:a6:49:63:bd:36:2d:36:1d:17:
                        41:65:7a:15:d3:ed:7c:ea:06:30:d1:05:76:24:45:
                        3e:c3:cb:bd:bf:1c:85:e2:16:e3:e4:33:82:de:d3:
                        39:bd
                    Exponent: 65537 (0x10001)
            X509v3 extensions:
                X509v3 Basic Constraints: critical
                    CA:TRUE, pathlen:0
                X509v3 Subject Key Identifier:
                    AD:A3:78:B4:EE:54:FC:C1:36:0B:E2:D1:12:28:52:AA:2F:D3:ED:AB
                X509v3 Key Usage:
                    Certificate Sign, CRL Sign
                Netscape Cert Type:
                    SSL CA, S/MIME CA, Object Signing CA
        Signature Algorithm: sha256WithRSAEncryption
             92:13:3e:1f:c5:84:b4:2f:0c:6a:aa:7d:09:e2:6a:24:92:2b:
             11:b1:89:e5:49:30:ea:93:ab:b9:cd:99:ff:86:ad:6d:b4:a0:
             3b:2f:a3:06:68:ad:c8:f4:8d:2b:1b:7b:06:2f:55:b5:bd:d7:
             33:24:16:f9:be:bf:1c:70:e6:29:22:c4:d9:b7:7f:fe:2c:20:
             13:e4:7d:77:06:92:5f:53:33:9f:04:f5:07:fc:3b:5e:51:92:
             d6:09:9b:b6:8c:9d:81:07:a4:3a:13:cb:d4:ba:87:46:31:e8:
             82:9d:13:0e:c8:77:ea:64:d6:74:79:bb:98:73:41:40:29:0c:
             33:9b:d7:50:ff:5c:02:c3:50:bb:f4:84:f6:de:ed:d9:86:42:
             57:84:44:58:73:51:a5:69:6c:23:14:c2:a5:41:0a:b5:6f:76:
             b5:42:49:f4:bf:72:af:5b:b4:14:2e:f2:34:a0:b2:db:45:e7:
             01:5a:54:c4:ed:df:55:21:25:0a:2a:99:3a:28:a0:50:57:4b:
             37:72:41:d1:ea:98:98:fb:54:91:f5:ec:b4:1f:78:9f:bc:b4:
             ea:0f:5f:14:16:bd:12:44:cf:d8:e7:c1:ab:c9:b0:1b:5f:d0:
             af:bf:21:99:6c:84:c1:e3:48:04:a0:17:d5:21:8d:bf:7b:42:
             2d:b0:0b:ba
    -----BEGIN CERTIFICATE-----
    MIIDWDCCAkCgAwIBAgIBATANBgkqhkiG9w0BAQsFADBEMQswCQYDVQQGEwJLUjEW
    MBQGA1UECgwNUmVkIEhhdCBLb3JlYTEdMBsGA1UEAwwUdGVzdHMgU2VsZiBTaWdu
    ZWQgQ0EwHhcNMjExMTI2MDgyMzM2WhcNMzExMTI0MDgyMzM2WjBEMQswCQYDVQQG
    EwJLUjEWMBQGA1UECgwNUmVkIEhhdCBLb3JlYTEdMBsGA1UEAwwUdGVzdHMgU2Vs
    ZiBTaWduZWQgQ0EwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQC68Jbz
    Y7r4oLgkgcPM1nasrf1YGBqhtcPJJLc2j8jadR0+v3R+b8MgeuMwNIvmnzM1geTm
    C+CWi0/VS2LL4vOiug1vvQjUIk7Ovo0pURwKI5V/rCzPL+Au44pjoLAoxHE7t/XV
    G/NVaA2K4wptEtDvEI0RPutwSNpnssMfWrhE5Dg53/659IYO+TKil6CA2fGXi/f5
    YqbrHg+IlCTdmuMEkedxA6IotDGeZYyTsHh8kEKcfS5qtz0PvYuj2AzbDFhdGFfY
    WFGRpbdIIYz0uwU0zo/mXM//O6ZJY702LTYdF0FlehXT7XzqBjDRBXYkRT7Dy72/
    HIXiFuPkM4Le0zm9AgMBAAGjVTBTMBIGA1UdEwEB/wQIMAYBAf8CAQAwHQYDVR0O
    BBYEFK2jeLTuVPzBNgvi0RIoUqov0+2rMAsGA1UdDwQEAwIBBjARBglghkgBhvhC
    AQEEBAMCAAcwDQYJKoZIhvcNAQELBQADggEBAJITPh/FhLQvDGqqfQniaiSSKxGx
    ieVJMOqTq7nNmf+GrW20oDsvowZorcj0jSsbewYvVbW91zMkFvm+vxxw5ikixNm3
    f/4sIBPkfXcGkl9TM58E9Qf8O15RktYJm7aMnYEHpDoTy9S6h0Yx6IKdEw7Id+pk
    1nR5u5hzQUApDDOb11D/XALDULv0hPbe7dmGQleERFhzUaVpbCMUwqVBCrVvdrVC
    SfS/cq9btBQu8jSgsttF5wFaVMTt31UhJQoqmToooFBXSzdyQdHqmJj7VJH17LQf
    eJ+8tOoPXxQWvRJEz9jnwavJsBtf0K+/IZlshMHjSASgF9Uhjb97Qi2wC7o=
    -----END CERTIFICATE-----
    ```



### 3.3) SSL 인증서 발급

**3.2** 에서 생성한 Root CA 서명키로 SSL 인증서를 발급해 보도록 하겠습니다.

**1) SSL 호스트에서 사용할 RSA Key Pair (Public, Private Key) 생성**

- 2048 bit  개인키 생성

  ```bash
  $ openssl genrsa -aes256 -out test.key 2048
  
  -- 출력 메시지 --
  Generating RSA private key, 2048 bit long modulus (2 primes)
  .........................................................................+++++
  .........+++++
  e is 65537 (0x010001)
  Enter pass phrase for test.key:
  Verifying - Enter pass phrase for test.key:
  ```

**2) Remove Passphrase from key**

> 개인키를 보호하기 위해 Key-Derived Function으로 개인키 자체가 암호화되어 있습니다. 인터넷 뱅킹 등에 사용되는 개인용 인증서는 당연히 저렇게 보호되어야 하지만 SSL에 사용하려는 키가 암호가 걸려있으면 웹 서버를 구동할 때마다 pass phrase를 입력해야 하므로 암호를 제거합니다.

- 개인키 pass phrase 제거

  ```bash
  $ cp test.key test.key.enc
  $ openssl rsa -in test.key.enc -out test.key
  -- 출력 메시지 ---
  openssl rsa -in test.key.enc -out test.key
  Enter pass phrase for test.key.enc:
  writing RSA key
  ```

- 개인키의 유출 방지를 위해 group과 other의 permission을 모두 제거합니다.

  ```bash
  $ chmod 600 test.key
  ```

### 3.4) CSR 생성

**1) CSR (Certificate Signing Request) 생성을 위한 openssl config 파일을 생성하고 host_openssl.conf(변경 가능)라는 이름으로 저장**

- **host_openssl.conf** 파일 생성

  ```bash
  cat <<EOF> host_openssl.conf
  [ req ]
  default_bits            = 2048
  default_md              = sha1
  default_keyfile         = test-rootca.key
  distinguished_name      = req_distinguished_name
  extensions             = v3_user
  ## 인증서 요청시에도 extension 이 들어가면 authorityKeyIdentifier 를 찾지 못해 에러가 나므로 막아둔다.
  ## req_extensions = v3_user
  
  [ v3_user ]
  # Extensions to add to a certificate request
  basicConstraints = CA:FALSE
  authorityKeyIdentifier = keyid,issuer
  subjectKeyIdentifier = hash
  keyUsage = nonRepudiation, digitalSignature, keyEncipherment
  ## SSL 용 확장키 필드
  extendedKeyUsage = serverAuth,clientAuth
  subjectAltName          = @alt_names
  [ alt_names]
  ## Subject AltName의 DNSName field에 SSL Host 의 도메인 이름을 적어준다.
  ## 멀티 도메인일 경우 *.lesstif.com 처럼 쓸 수 있다.
  DNS.1   = ${DOMAIN_NAME}
  
  [req_distinguished_name ]
  countryName                     = Country Name (2 letter code)
  countryName_default             = KR
  countryName_min                 = 2
  countryName_max                 = 2
  
  # 회사명 입력
  organizationName              = Red Hat
  organizationName_default      = Red Hat Korea
   
  # 부서 입력
  organizationalUnitName          = AppDev
  organizationalUnitName_default  = AppDev
   
  # SSL 서비스할 domain 명 입력
  commonName                      = quay.apps.cluster-0c41.sandbox1202.opentlc.com
  commonName_default             = lesstif.com
  commonName_max                  = 64
  EOF
  ```

- 인증서 발급 요청 (CSR) 파일 생성

  ```bash
  $ openssl req -new -key test.key -out test.csr -config host_openssl.conf
  
  -- 출력 메시지 --
  You are about to be asked to enter information that will be incorporated
  into your certificate request.
  What you are about to enter is what is called a Distinguished Name or a DN.
  There are quite a few fields but you can leave some blank
  For some fields there will be a default value,
  If you enter '.', the field will be left blank.
  -----
  Country Name (2 letter code) [KR]:
  Red Hat [Red Hat Korea]:
  AppDev [AppDev]:
  quay.apps.cluster-0c41.sandbox1202.opentlc.com [lesstif.com]
  ```

- 10년짜리 test.crt 용 SSL 인증서 발급 (서명시 Root CA 개인키로 서명)

  ```bash
  $ openssl x509 -req -days 3650 -extensions v3_user -in test.csr \
  -CA test-rootca.crt -CAcreateserial \
  -CAkey  test-rootca.key \
  -out test.crt  -extfile host_openssl.conf
  
  -- 출력 메시지 ---
  Signature ok
  subject=C = KR, O = Red Hat Korea, OU = AppDev, CN = lesstif.com
  Getting CA Private Key
  Enter pass phrase for test-rootca.key: # test-rootca.key 암호 입력
  ```

- 제대로 생성되었는지 확인을 위해 인증서의 정보를 출력합니다.

  ```bash
  $ openssl x509 -text -in test.crt
  ```

  - 출력 정보

    ```bash
    Certificate:
        Data:
            Version: 3 (0x2)
            Serial Number:
                23:c3:ff:54:4a:13:11:c9:26:51:85:0d:c2:d2:b7:f9:5e:1a:10:f9
            Signature Algorithm: sha256WithRSAEncryption
            Issuer: C = KR, O = Red Hat Korea, CN = tests Self Signed CA
            Validity
                Not Before: Nov 26 08:39:58 2021 GMT
                Not After : Nov 24 08:39:58 2031 GMT
            Subject: C = KR, O = Red Hat Korea, OU = AppDev, CN = lesstif.com
            Subject Public Key Info:
                Public Key Algorithm: rsaEncryption
                    RSA Public-Key: (2048 bit)
                    Modulus:
                        00:d0:1c:0f:e6:aa:ec:c4:28:f6:d7:3b:3a:d0:63:
                        9e:e1:2f:16:4e:49:05:c1:aa:1d:8b:b3:02:ee:0a:
                        8e:2a:77:a9:00:f9:cf:b0:26:0c:4b:2d:04:a6:f5:
                        3d:31:36:ac:a4:97:d5:77:88:91:0c:71:3e:d6:b6:
                        04:42:54:9b:8b:4f:63:e2:cb:68:b7:ff:27:80:46:
                        e7:97:ca:05:8f:44:bb:08:e5:f6:a3:4c:a8:84:97:
                        08:21:49:6a:de:0a:71:27:1f:2c:47:7e:a1:90:8b:
                        1f:5e:7f:e4:b4:0b:a9:70:97:69:b4:60:94:a2:38:
                        bc:55:3d:6c:49:84:4f:af:7f:95:90:cf:08:b6:3f:
                        be:b5:6e:e2:2b:83:53:bc:aa:f5:26:00:66:9a:05:
                        97:aa:08:7f:77:93:ce:4b:86:62:5a:1e:7c:cf:68:
                        ba:fc:81:00:8b:e3:59:ea:1e:50:8b:3c:c8:2a:6b:
                        b8:ac:1b:79:82:32:f3:9d:ef:e4:40:ab:ed:6b:16:
                        24:48:33:03:5c:ea:3c:f1:9d:0b:d6:58:d1:f6:78:
                        fb:2a:45:2d:b5:32:7f:49:09:ff:98:50:6e:ee:63:
                        44:62:81:7f:bc:96:f6:63:a3:65:e3:71:f5:7c:6d:
                        96:72:d4:2d:d9:8d:02:f4:a8:0a:83:bd:1d:35:c0:
                        49:f1
                    Exponent: 65537 (0x10001)
            X509v3 extensions:
                X509v3 Basic Constraints:
                    CA:FALSE
                X509v3 Authority Key Identifier:
                    keyid:AD:A3:78:B4:EE:54:FC:C1:36:0B:E2:D1:12:28:52:AA:2F:D3:ED:AB
    
                X509v3 Subject Key Identifier:
                    8D:D1:EE:0E:01:A3:F4:54:0A:1E:F5:2A:C7:EF:E0:87:3F:FB:37:03
                X509v3 Key Usage:
                    Digital Signature, Non Repudiation, Key Encipherment
                X509v3 Extended Key Usage:
                    TLS Web Server Authentication, TLS Web Client Authentication
                X509v3 Subject Alternative Name:
                    DNS:quay.apps.cluster-0c41.sandbox1202.opentlc.com
        Signature Algorithm: sha256WithRSAEncryption
             6e:cb:c4:95:b2:51:e1:80:65:0e:cb:2c:42:0e:10:7a:d2:44:
             dc:b8:6c:bd:7d:69:74:b8:a7:60:4b:cb:aa:7a:3d:c8:74:a8:
             7d:24:e6:53:f6:dc:64:17:ad:1c:ba:dd:5a:cc:0f:99:b5:77:
             a5:6f:58:a5:d4:bc:28:9f:03:59:d6:a1:81:25:ad:61:15:30:
             c5:c1:e4:36:e0:3a:d4:c7:87:af:97:fb:29:7e:b2:d3:87:ff:
             cd:15:de:8e:78:ff:71:6f:cb:ab:f3:91:8d:37:28:34:e6:47:
             74:4e:3b:42:62:e2:92:ea:26:e3:0a:5c:20:cc:b8:66:9a:8f:
             31:1b:c8:4f:9e:86:1e:c4:ce:cf:ec:06:c1:ea:a5:8b:1b:1b:
             e0:97:ca:90:6f:3f:cb:d7:50:7d:70:58:dd:c2:84:64:59:62:
             34:05:bb:0b:13:cb:a8:b0:7a:d8:36:b5:4a:68:2d:c2:4c:99:
             8b:41:28:73:35:b7:73:03:7d:56:a7:c4:49:dd:d2:6d:89:6e:
             5c:3f:79:3e:c3:62:3d:3c:e3:b7:94:fb:ff:e3:55:da:b6:67:
             ee:cf:2b:80:d9:94:4f:51:26:7f:d4:0b:42:2f:9a:9e:cf:82:
             53:90:a6:b9:a3:d4:30:8d:50:8b:ce:dc:1a:6f:1c:81:78:d8:
             01:29:f8:d1
    -----BEGIN CERTIFICATE-----
    MIID1DCCArygAwIBAgIUI8P/VEoTEckmUYUNwtK3+V4aEPkwDQYJKoZIhvcNAQEL
    BQAwRDELMAkGA1UEBhMCS1IxFjAUBgNVBAoMDVJlZCBIYXQgS29yZWExHTAbBgNV
    BAMMFHRlc3RzIFNlbGYgU2lnbmVkIENBMB4XDTIxMTEyNjA4Mzk1OFoXDTMxMTEy
    NDA4Mzk1OFowTDELMAkGA1UEBhMCS1IxFjAUBgNVBAoMDVJlZCBIYXQgS29yZWEx
    DzANBgNVBAsMBkFwcERldjEUMBIGA1UEAwwLbGVzc3RpZi5jb20wggEiMA0GCSqG
    SIb3DQEBAQUAA4IBDwAwggEKAoIBAQDQHA/mquzEKPbXOzrQY57hLxZOSQXBqh2L
    swLuCo4qd6kA+c+wJgxLLQSm9T0xNqykl9V3iJEMcT7WtgRCVJuLT2Piy2i3/yeA
    RueXygWPRLsI5fajTKiElwghSWreCnEnHyxHfqGQix9ef+S0C6lwl2m0YJSiOLxV
    PWxJhE+vf5WQzwi2P761buIrg1O8qvUmAGaaBZeqCH93k85LhmJaHnzPaLr8gQCL
    41nqHlCLPMgqa7isG3mCMvOd7+RAq+1rFiRIMwNc6jzxnQvWWNH2ePsqRS21Mn9J
    Cf+YUG7uY0RigX+8lvZjo2XjcfV8bZZy1C3ZjQL0qAqDvR01wEnxAgMBAAGjgbUw
    gbIwCQYDVR0TBAIwADAfBgNVHSMEGDAWgBSto3i07lT8wTYL4tESKFKqL9PtqzAd
    BgNVHQ4EFgQUjdHuDgGj9FQKHvUqx+/ghz/7NwMwCwYDVR0PBAQDAgXgMB0GA1Ud
    JQQWMBQGCCsGAQUFBwMBBggrBgEFBQcDAjA5BgNVHREEMjAwgi5xdWF5LmFwcHMu
    Y2x1c3Rlci0wYzQxLnNhbmRib3gxMjAyLm9wZW50bGMuY29tMA0GCSqGSIb3DQEB
    CwUAA4IBAQBuy8SVslHhgGUOyyxCDhB60kTcuGy9fWl0uKdgS8uqej3IdKh9JOZT
    9txkF60cut1azA+ZtXelb1il1LwonwNZ1qGBJa1hFTDFweQ24DrUx4evl/spfrLT
    h//NFd6OeP9xb8ur85GNNyg05kd0TjtCYuKS6ibjClwgzLhmmo8xG8hPnoYexM7P
    7AbB6qWLGxvgl8qQbz/L11B9cFjdwoRkWWI0BbsLE8uosHrYNrVKaC3CTJmLQShz
    NbdzA31Wp8RJ3dJtiW5cP3k+w2I9POO3lPv/41XatmfuzyuA2ZRPUSZ/1AtCL5qe
    z4JTkKa5o9QwjVCLztwabxyBeNgBKfjR
    -----END CERTIFICATE-----
    ```

  > 검증이 완료된 이후에는 해당 인증서를 등록하여 사용 할 수 있습니다.



### 4. OpenShift Ingress CA 인증서를 통한 SSL 인증서 발급

OpenShift의 Ingress CA 인증서를 이용하여 SSL 인증서를 생성할 수 있습니다. 

### 4.1) OpenShift RootCA 인증서 정보 확인

- OpemShift RootCA 인증서 정보 확인

   - `openshift-ingress-operator` 프로젝트 > Secret > router-ca 

    - 인증서 추출

      ```bash
      $ oc -n openshift-ingress-operator get secret router-ca -o jsonpath="{ .data.tls\.crt }" | base64 -d -i > ca.crt
      $ oc -n openshift-ingress-operator get secret router-ca -o jsonpath="{ .data.tls\.key }" | base64 -d -i > ca.key
      ```

### 4.2) 서버 키 생성

```bash
$ openssl genrsa -out ssl.key 2048
```

### 4.3) Quay 인증서 생성을 위한 CSR 생성 

인증서 생성시 호스트명이 길면 제한에 걸려 생성되지 않으므로, 짧은 이름으로 route를 하나 추가해주고 해당 이름으로 인증서를 생성합니다.

```bash
$ openssl req -new -key ssl.key -out ssl.csr

-- 출력 메시지 ---
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:KR
State or Province Name (full name) []:Gagnam-gu
Locality Name (eg, city) [Default City]:Seoul
Organization Name (eg, company) [Default Company Ltd]:Red Hat
Organizational Unit Name (eg, section) []:AppDev
Common Name (eg, your name or your server's hostname) []:quay.apps.cluster-7234.sandbox941.opentlc.com
Email Address []:
  
Please enter the following 'extra' attributes
  to be sent with your certificate request
A challenge password []:
An optional company name []:
```

### 4.4) SAN(Subject Alternative Name) 확장을 정의하는데 필요한 구성 파일 생성

```bash
cat <<EOF> quay.ext
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
  subjectAltName = @alt_names

[alt_names]
DNS.1 = quay.apps.cluster-7234.sandbox941.opentlc.com
EOF
```

### 4.5) Quay 인증서 생성

```bash
 $ openssl x509 -req -in ssl.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out ssl.crt -days 3650 -sha256 -extfile quay.ext
-- 출력 메시지 --
  Signature ok
subject=C = KR, ST = Gangnam-gu, L = Seoul, O = Red Hat, OU = AppDev, CN = quay.apps.cluster-7234.sandbox941.opentlc.com
Getting CA Private Key
```

### 4.6) 생성된 인증서 목록 보기

```bash
$ ls -rlt
total 28
  -rw-r--r--. 1 hyou-redhat.com users 1119 Nov 22 06:36 ca.crt
-rw-r--r--. 1 hyou-redhat.com users 1679 Nov 22 06:37 ca.key
  -rw-------. 1 hyou-redhat.com users 1679 Nov 22 06:37 ssl.key
-rw-r--r--. 1 hyou-redhat.com users 1058 Nov 22 06:37 ssl.csr
-rw-r--r--. 1 hyou-redhat.com users  236 Nov 22 06:37 quay.ext
  -rw-r--r--. 1 hyou-redhat.com users   41 Nov 22 06:38 ca.srl
  -rw-r--r--. 1 hyou-redhat.com users 1350 Nov 22 06:38 ssl.crt
```

### 4.7) root certificate 인증서에 새로운 클라이언트 인증서 추가

```bash
$ cat ca.crt >> ssl.crt
```

### 4.8) 추가된 인증서 보기

```bash
$ openssl x509 -in ssl.crt --text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            67:2a:ba:7a:33:b9:46:5d:1f:90:7d:03:c2:dd:9d:73:2c:e3:02:e7
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = ingress-operator@1637542903
        Validity
            Not Before: Nov 22 06:38:18 2021 GMT
            Not After : Nov 20 06:38:18 2031 GMT
        Subject: C = KR, ST = Gangnam-gu, L = Seoul, O = Red Hat, OU = AppDev, CN = quay.apps.cluster-7234.sandbox941.opentlc.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (2048 bit)
                Modulus:
                    00:c9:14:ab:5a:7d:52:6c:88:4b:c0:99:f6:af:98:
                    4a:78:e6:f7:d2:cf:d8:55:bf:48:38:f1:24:cf:b0:
                    02:7c:58:29:dd:15:6e:5e:9b:1e:e8:a8:61:93:b2:
                    82:b6:69:13:9a:6e:e2:25:a9:30:1b:59:21:c9:0c:
                    81:9e:e3:1d:b0:ad:db:fb:65:e8:53:a9:05:b5:78:
                    02:8f:32:eb:8b:c1:a5:9c:ca:a9:76:75:2d:dd:ff:
                    93:4c:ae:7c:02:f1:9b:56:ab:02:c4:c4:f4:b7:d5:
                    02:02:ee:9d:ed:c0:91:f1:bd:b1:4b:a0:d2:4e:41:
                    8f:2e:c4:af:a2:f2:78:00:db:46:30:57:c4:86:ec:
                    44:68:56:97:63:e1:21:35:de:92:33:c5:ce:b1:59:
                    e9:ad:de:ea:bb:a7:a9:c0:ac:27:e3:bf:d7:d6:28:
                    40:78:98:6a:9c:57:74:57:97:53:c7:05:b7:69:4a:
                    2a:7a:17:17:53:ce:68:36:2d:55:48:3b:d9:74:1e:
                    7f:79:a2:56:ad:6c:9d:09:20:93:50:42:f5:f6:90:
                    7b:79:f4:4a:49:50:6a:10:5f:2f:5e:f3:ba:d6:81:
                    6f:0a:cd:57:86:db:0d:f6:a0:ee:da:fa:7a:5d:f1:
                    09:41:96:03:0b:f6:41:ab:91:c9:0e:62:d9:27:a2:
                    c3:8f
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Authority Key Identifier:
                keyid:9B:A9:70:73:AD:7E:05:C7:1E:84:05:D0:4A:E3:24:62:33:16:91:73

            X509v3 Basic Constraints:
                CA:FALSE
            X509v3 Key Usage:
                Digital Signature, Non Repudiation, Key Encipherment, Data Encipherment
            X509v3 Subject Alternative Name:
                DNS:quay.apps.cluster-7234.sandbox941.opentlc.com
    Signature Algorithm: sha256WithRSAEncryption
         90:ee:d0:2d:bf:2d:01:dd:96:e0:2e:6b:7a:a9:e9:9f:85:96:
         9d:68:82:db:85:d7:87:f5:a5:90:a4:c2:d9:a8:02:96:51:76:
         fe:d1:7f:cd:41:98:86:99:cd:cf:e9:a2:e8:28:63:d6:66:4e:
         3b:b2:21:99:5e:61:97:f1:4e:61:bf:d1:36:18:94:b3:31:4d:
         c9:fa:5a:f2:46:82:a5:e8:96:1d:60:2f:8e:34:7e:08:99:6d:
         3e:ef:f9:5d:e2:e4:3e:c4:76:40:bf:ce:20:9c:b2:c9:46:8a:
         7e:31:7f:ef:15:8e:56:0e:27:43:17:61:82:a3:71:b4:cf:2d:
         26:5e:7c:bc:ef:99:90:19:0b:1f:ca:36:a0:38:ab:4e:df:1c:
         0c:51:d3:af:6e:16:e2:5c:8a:88:31:c2:91:46:63:67:e5:cc:
         03:05:6b:fb:99:40:de:86:3a:92:1a:ff:3e:f3:f4:13:70:26:
         5d:f1:9d:b3:47:b7:e1:5a:19:97:90:18:7f:ee:82:1a:f6:da:
         c8:dc:58:2c:42:21:e7:6c:15:06:f5:21:97:48:0a:80:f1:04:
         55:d9:19:21:08:50:e8:53:6a:75:33:01:c4:99:dd:e4:9c:8a:
         ae:96:e5:65:b7:3e:a3:3d:c8:eb:91:ee:26:3c:f5:1f:2b:46:
         69:a0:00:83
-----BEGIN CERTIFICATE-----
MIIDtzCCAp+gAwIBAgIUZyq6ejO5Rl0fkH0Dwt2dcyzjAucwDQYJKoZIhvcNAQEL
BQAwJjEkMCIGA1UEAwwbaW5ncmVzcy1vcGVyYXRvckAxNjM3NTQyOTAzMB4XDTIx
MTEyMjA2MzgxOFoXDTMxMTEyMDA2MzgxOFowgY0xCzAJBgNVBAYTAktSMRMwEQYD
VQQIDApHYW5nbmFtLWd1MQ4wDAYDVQQHDAVTZW91bDEQMA4GA1UECgwHUmVkIEhh
dDEPMA0GA1UECwwGQXBwRGV2MTYwNAYDVQQDDC1xdWF5LmFwcHMuY2x1c3Rlci03
MjM0LnNhbmRib3g5NDEub3BlbnRsYy5jb20wggEiMA0GCSqGSIb3DQEBAQUAA4IB
DwAwggEKAoIBAQDJFKtafVJsiEvAmfavmEp45vfSz9hVv0g48STPsAJ8WCndFW5e
mx7oqGGTsoK2aROabuIlqTAbWSHJDIGe4x2wrdv7ZehTqQW1eAKPMuuLwaWcyql2
dS3d/5NMrnwC8ZtWqwLExPS31QIC7p3twJHxvbFLoNJOQY8uxK+i8ngA20YwV8SG
7ERoVpdj4SE13pIzxc6xWemt3uq7p6nArCfjv9fWKEB4mGqcV3RXl1PHBbdpSip6
FxdTzmg2LVVIO9l0Hn95olatbJ0JIJNQQvX2kHt59EpJUGoQXy9e87rWgW8KzVeG
2w32oO7a+npd8QlBlgML9kGrkckOYtknosOPAgMBAAGjdTBzMB8GA1UdIwQYMBaA
FJupcHOtfgXHHoQF0ErjJGIzFpFzMAkGA1UdEwQCMAAwCwYDVR0PBAQDAgTwMDgG
A1UdEQQxMC+CLXF1YXkuYXBwcy5jbHVzdGVyLTcyMzQuc2FuZGJveDk0MS5vcGVu
dGxjLmNvbTANBgkqhkiG9w0BAQsFAAOCAQEAkO7QLb8tAd2W4C5reqnpn4WWnWiC
  24XXh/WlkKTC2agCllF2/tF/zUGYhpnNz+mi6Chj1mZOO7IhmV5hl/FOYb/RNhiU
szFNyfpa8kaCpeiWHWAvjjR+CJltPu/5XeLkPsR2QL/OIJyyyUaKfjF/7xWOVg4n
  QxdhgqNxtM8tJl58vO+ZkBkLH8o2oDirTt8cDFHTr24W4lyKiDHCkUZjZ+XMAwVr
+5lA3oY6khr/PvP0E3AmXfGds0e34VoZl5AYf+6CGvbayNxYLEIh52wVBvUhl0gK
gPEEVdkZIQhQ6FNqdTMBxJnd5JyKrpblZbc+oz3I65HuJjz1HytGaaAAgw==
-----END CERTIFICATE-----
```

### 4.9) 인증서 확인

```bash
$ openssl verify -CAfile ca.crt ssl.crt
ssl.crt OK
```



### 5. 참고

참고 URL) 

https://access.redhat.com/documentation/ko-kr/openshift_container_platform/4.6/html/security_and_compliance/configuring-certificates (OpenShift 인증서 구성)

https://access.redhat.com/solutions/5606051 (How to create local CA for Openshift 4)

https://docs.openshift.com/container-platform/4.9/security/index.html (Security and compliance)

