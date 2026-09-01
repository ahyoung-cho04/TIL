# VM에서 Gmail로 메일 보내기

## 환경
- Ubuntu 22.04 (VirtualBox, NAT)
- Windows PowerShell에서 SSH로 접속하여 실습

## 목표
- vm에서 'mail -s' 명령으로 내 gmail 주소에 메일 보내기.

## 직접 보내지 못하는 이유
- 가정용 회선은 스팸 발송이 악용되는 것을 막기 위해 25번 포트(SMTP)가 차단됨.
- 따라서 gmail의 발신서버(587)에 내 계정으로 로그인하여, gmail이 대신 보내게 함.
- 이 구조를 relayhost(smarthost)라고 함.

## 설정
```
# /etc/postfix/main.cf
relayhost = [smtp.gmail.com]:587
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd # 로그인 정보 파일
smtp_sasl_security_options = noanonymous # 익명 로그인 사용 안 함
smtp_tls_security_level = encrypt # 암호화 연결만 사용
```

```
# /etc/postfix/sasl_passwd
[smtp.gmail.com]:587 내gmail주소:앱비밀번호
```
```bash
sudo postmap /etc/postfix/sasl_passwd 
sudo chmod 600 /etc/postfix/sasl_passwd*
sudo systemctl restart postfix
```

## 막혔던 지점
### 1. relay=none, postfix/error
- main.cf 파일의 문제였음.
- 'relayhost'에 적은 서버 주소에 오타가 있었음.
- 설치 시 'Local only'를 골라 'default_transport = error'가 남아있었음.

### 2. 530 authentication required
- 1번에서 sasl_passwd 파일도 함께 확인하였을 때 같은 실수가 있었음. 따라서 해당 파일을 수정하였으나, postmap을 실행하지 않았음.
- postfix는 텍스트 파일이 아닌 .db 파일을 읽음.
- 따라서 수정 이전의 파일 잘못된 값이 그대로 쓰여 로그인 정보 없이 연결, gmail에서 530으로 거부함.
```
warning: database /etc/postfix/sasl_passwd.db is older than source file /etc/postfix/sasl_passwd
```

## 배운 점
- log의 relay 값으로 실패 단계를 확인할 수 있음. (none이라면 설정 문제, 서버 이름이 나온다면 인증이나 네트워크 문제)
- smtp 응답코드가 530이라면 로그인 안 함, 535라면 로그인 정보 틀린 것.
- 'postconf 항목명'으로 설정값 바로 확인 가능함.
```bash
$ postconf relayhost default_transport
relayhost = [smtp.gmail.com]:587
default_transport = smtp
```
- 파일 수정한다고 바로 적용되는 것이 아님. netplan apply처럼 적용하는 단계가 따로 있음을 유의할 것.
