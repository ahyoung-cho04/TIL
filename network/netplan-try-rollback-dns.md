# netplan try 롤백 후 DNS만 정상 작동하지 않았음

## 환경
- Ubuntu 22.04 (VirtualBox, NAT)
- Windows PowerShell에서 SSH로 접속하여 실습
- netplan + systemd-networkd + systemd-resolved
- netplan yaml: dhcp 설정을 끄고 addresses, routes, nameservers 직접 지정

## 현상
netplan try 실행 후에 엔터를 누르지 않아 롤백되었음.
그 뒤 IP 통신은 되지만 도메인 이름 해석만 실패함.

```bash
$ ping -c 3 8.8.8.8
3 packets transmitted, 3 received, 0% packet loss

$ ping -c 3 google.com
ping: google.com: Temporary failure in name resolution
```

## 처음 의심한 원인
**/etc/hosts**
직전 실습에서 /etc/hosts에 잘못된 IP로 google.com을 등록하여 일부러 오류를 냈었기에 의심함.
그러나 확인해보니 오류를 낸 부분은 이미 지워놓은 상태였음.

## 진단
**1. dig로 외부 DNS 직접 확인**
```bash
$ dig google.com @8.8.8.8
->>HEADER<<- opcode: QUERY, status: NOERROR, id: 37750
flags: qr rd ra; QUERY: 1, ANSWER: 6, ...
```
dig를 통해 다음과 같은 결과를 얻었고, 
에러없이 외부 DNS 통신이 정상적으로 되는 걸 확인할 수 있었음.

**2. resolvectl로 로컬 상태 확인**
```bash
$ resolvectl status
Link 2 (enp0s3)
Current Scopes: none
```
resolvectl을 통해 다음과 같은 결과를 얻었고, 
해당 링크에 등록된 DNS 서버가 하나도 없는 것을 확인할 수 있었음.

## 해결
yaml 파일을 수정하지 않고 netplan try 후 엔터로 확정했더니 복구됨.
따라서 문제의 원인이 파일이 아니었음을 추측할 수 있었음.
```bash
$ resolvectl status
Link 2 (enp0s3)
Current Scopes: DNS
```

## 원인
netplan try의 롤백은 yaml 파일을 복원하지만, 복원이 제대로 되지 않는 경우가 있는 모양임.
나의 경우 ip 통신은 제대로 됐었던 것으로 보아, system-resolved의 링크별 DNS 설정까지는 복구되지 않았던 것으로 추측함.

## 배운 점
- ping으로 ip 통신은 잘 되는데 이름만 안 된다면 DNS 계층의 문제임.
- dig @서버 는 로컬 resolver를 우회하므로 내부, 외부 문제를 가리는 데 씀.
- 롤백이 완전한 원상복구는 아닐 수 있음.
- netplan try는 반드시 엔터로 확정하고 직후 resolvectl status로 확인할 것.
