# 클라우드플레어의 tunnel로 인해 테라폼의 destroy 작업이 원활히 이루어지지 않을 때

문제: 클라우드플레어의 tunnel을 사용 중인 EC2 서버는 테라폼의 destroy 명령어로 한번에 종료되지 않음

## 실패 케이스

**systemd**

EC2 종료 시, tunnel cleanup 작업을 수행하는 스크립트 작성, systemd 서비스로 등록

```shell
#!/bin/sh
touch cleanup_tunnel.service
cat > cleanup_tunnel.service << EOF
[Unit]
Description=Run cleanup_tunnel script at shutdown
DefaultDependencies=no
Before=shutdown.target reboot.target halt.target

[Service]
Type=oneshot
ExecStart=sudo systemctl stop cloudflared
TimeoutStartSec=0

[Install]
WantedBy=halt.target reboot.target shutdown.target
EOF

sudo mv cleanup_tunnel.service /etc/systemd/system/cleanup_tunnel.service
sudo systemctl enable cleanup_tunnel.service
```

실패 원인: 스크립트가 실행되기 전에, 테라폼의 tunnel 리소스 제거 작업이 먼저 실행되어 tunnel에 활성화된 연결이 있다는 오류 발생. 실행 순서가 원인으로 보임(추정)

**클라우드플레어 API 활용**

tunnel의 연결을 제거하는 클라우드플레어 API 활용.

```shell
#!/bin/sh

if [ ! -z "${TUNNEL_ID}" ]; then
  curl -X DELETE https://api.cloudflare.com/client/v4/accounts/"${CF_ACCOUNT_ID}"/cfd_tunnel/"${TUNNEL_ID}"/connections \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer ${CF_TOKEN}"

  echo "Tunnel ${TUNNEL_ID} deleted."
fi

if [ -z "${TUNNEL_ID}" ]; then
  echo "Tunnel ID not found."
fi
```

위 스크립트를 테라폼의 destroy 라이프사이클을 사용해서 실행시킴

```terraform
resource "null_resource" "cleanup_tunnel" {
  triggers = {
    CF_ACCOUNT_ID = local.envs["CF_ACCOUNT_ID"]
    CF_EMAIL      = local.envs["CF_EMAIL"]
    CF_TOKEN      = local.envs["CF_TOKEN"]
    TUNNEL_ID     = cloudflare_tunnel.staging.id
  }

  provisioner "local-exec" {
    when = destroy

    command = "chmod +x ../common-scripts/cleanup-tunnel.sh; sh ../common-scripts/cleanup-tunnel.sh"
    environment = {
      CF_ACCOUNT_ID = self.triggers["CF_ACCOUNT_ID"]
      CF_EMAIL      = self.triggers["CF_EMAIL"]
      CF_TOKEN      = self.triggers["CF_TOKEN"]
      TUNNEL_ID     = self.triggers["TUNNEL_ID"]
    }
    working_dir = path.module
    interpreter = ["/bin/sh", "-c"]
  }
}
```

실패 원인: Actions에서 triggers로 지정된 변수에 값을 제대로 저장하지 못하고 가져오지 못하는 것으로 보임(추정). 이로 인해 스크립트 실행이 실패하고, state lock(DynamoDB)이 계속 활성화 되는 등 심각한 문제 발생.

## 성공 케이스

가장 단순하면서도 확실함. 사이드 이펙트가 없음.

**destroy 명령어 2번 이상 실행**

`terraform destroy` 실행 도중 오류가 발생하면, 한번 더 실행하도록 스크립트 작성. 아래 코드에선 `max_retries`를 3으로 여유있게 설정. 대부분 2번의 시도 내에서 destroy 작업이 완전하게 수행됨.

```yaml
  - name: Terraform Destroy Staging
    id: destroy-staging
    run: |
      retry_count=0
      max_retries=3

      while [[ "$retry_count" -lt "$max_retries" ]]; do
        terraform -chdir=./infra/tf/staging/applications/${{ matrix.workdir }} destroy -auto-approve && break
        retry_count=$((retry_count + 1))
        echo "Terraform Destroy failed. Retrying ($retry_count/$max_retries)..."
        sleep 3
      done

      if [[ "$retry_count" -eq "$max_retries" ]]; then
        echo "Terraform Destroy failed after $max_retries attempts. Exiting."
        exit 1
      fi
```