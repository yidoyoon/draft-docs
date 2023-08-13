# 테라폼에서 AWS EC2에 SSH로 접근하기 위한 클라우드플레어 Tunnel 설정

클라우드플레어에서 도메인 네임 설정 시, Proxy 모드를 사용하면 도메인 네임을 SSH 접근에 활용할 수 없음

테라폼, GitHub Actions 등으로 자동화를 구성할 때, 도메인 네임을 활용하지 못하는 것은 큰 불편

이것을 해결하기위해 클라우드플레어의 Tunnel 기능을 활용

## 용어

로컬: 로컬 호스트. 클라이언트

원격지: SSH 접근 대상. EC2 서버

## `cloudflared` 설치

터널을 활용해 SSH로 접근하려면 로컬과 원격지에 `cloudflared`를 설치해야함

로컬 `cloudflared`: 터널에 연결
원격지 `cloudflared`: 터널 서버 구성

### 로컬(WSL2)

수동 설치 진행

`cloudflared` 설치 링크: https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/

### 원격지(Ubuntu)

1. 테라폼에서 터널 생성 + 터널 CNAME 레코드 생성
2. EC2서버에 터널 서버 설치, 설정, 실행

**테라폼**

클라우드플레어 프로바이더의 터널 관련 리소스 활용

```terraform
# 클라우드플레어 토큰(사용자 인증정보)
provider "cloudflare" {
  api_token = local.envs["CF_TOKEN"]
}

# 테라폼을 활용한 터널 구성에 사용하는 시크릿
resource "random_password" "ssh_tunnel" {
  length  = 32
  special = false
}

# 터널 구성
resource "cloudflare_tunnel" "ssh" {
  account_id = local.envs["CF_ACCOUNT_ID"]
  name       = "ssh-${element(split(".", local.envs["HOST_URL"]), 0)}"
  secret     = random_password.ssh_tunnel.result
}

# 터널 설정. 내부 SSH 서버와 연결
resource "cloudflare_tunnel_config" "ssh" {
  account_id = local.envs["CF_ACCOUNT_ID"]
  tunnel_id  = cloudflare_tunnel.ssh.id

  config {
    warp_routing {
      enabled = false
    }
    ingress_rule {
      service = "ssh://localhost:22"
    }
  }
}

# 터널 CNAME 레코드 생성
resource "cloudflare_record" "ssh_tunnel" {
  zone_id = local.envs["CF_ZONE_ID"]
  name    = "ssh-${local.envs["HOST_URL"]}"
  value   = cloudflare_tunnel.ssh.cname
  type    = "CNAME"
  proxied = "true"
}
```

**스크립트**

EC2 서버에 `cloudflared`를 설치하고 실행하는 스크립트

${account}: 클라우드플레어 어카운트 ID

${secret}: `random_password` 리소스로 생성한 시크릿

```shell
#!/bin/bash

mkdir /etc/cloudflared
touch /etc/cloudflared/cert.json
touch /etc/cloudflared/config.yml

cat > /etc/cloudflared/cert.json << EOF
{
    "AccountTag"   : "${account}",
    "TunnelID"     : "${tunnel_id}",
    "TunnelName"   : "${tunnel_name}",
    "TunnelSecret" : "${secret}"
}
EOF

cat > /etc/cloudflared/config.yml << EOF
tunnel: ${tunnel_id}
credentials-file: /etc/cloudflared/cert.json
logfile: /var/log/cloudflared.log
url: ssh://localhost:22
loglevel: info
EOF

wget -q https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb

sudo cloudflared --config /etc/cloudflared/config.yml service install
sudo cloudflared start &
```

이후 Tunnel 대시보드에서 Tunnel 상태 확인

### 접속

**One-liner**

`SSH_TUNNEL_CNAME`에 SSH 접근용 CNAME으로 설정한 주소를 넣는다.

```shell
ssh -o "ProxyCommand=/usr/local/bin/cloudflared access ssh" -o "StrictHostKeyChecking no" -i ~/.ssh/ssh ubuntu@<SSH_TUNNEL_URL>
```

**config**

`~/.ssh/ssh`의 내용을 아래와 같이 작성한다.

```shell
Host staging
        StrictHostKeyChecking no
        IdentityFile ~/.ssh/ssh
        HostName <SSH_TUNNEL_URL>
        ProxyCommand /usr/local/bin/cloudflared access ssh --hostname %h
        User <USER_NAME>
```

이후 아래 명령어로 접속한다.

```shell
ssh staging
```

## 관련 오류

아래와 같은 오류가 발생하면 클라우드플레어 토큰의 권한 설정이 필요함(DNS, Tunnel 관련 권한 등)

```shell
│ Error: error creating Access Application for zones "56999d098ed5fec2b0cca446026a9897": error from makeRequest: Authentication error (10000)
```
