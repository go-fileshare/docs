---
title: "상태 점검과 메트릭"
weight: 20
description: "metrics 블록의 /healthz, /readyz, /metrics 엔드포인트, 이들이 노출하는 메트릭, 그리고 알림을 설정할 대상."
tags: [관리, 메트릭]
---

```hcl
metrics { listen = "127.0.0.1:9100" }   # or unix:///run/fileshare/metrics.sock
```

[go-net-health/endpoint](https://github.com/go-net-health/endpoint)가 전용
리스너에서 제공하는 세 개의 엔드포인트입니다. **프로토콜이 사용하는 포트는
결코 쓰지 않으므로**, WebDAV에 접근할 수 있는 외부에서는 이 엔드포인트에
닿지 않습니다.

| | |
|---|---|
| `/healthz` | 프로세스가 응답합니다 |
| `/readyz` | 모든 프로토콜이 바인딩되어 있고 **또한** 한 세대가 제공 중입니다. 시작 중, [변경 적용 중]({{< relref "/administration/_index.md#a-change-is-a-new-generation" >}}) 또는 중지 중에는 `503`과 함께 본문에 그 이유가 표시됩니다 |
| `/metrics` | Prometheus 텍스트 형식 |

유닉스 소켓은 `unix:///like/this`처럼 절대 경로로 지정합니다. 그 외에는 모두
`host:port`여야 하며, 그렇지 않으면 블록이 거부됩니다.

## 측정 항목 {#what-is-measured}

| 메트릭 | 집계 대상 |
|---|---|
| `fileshare_shares{origin}` | 제공 중인 공유. `origin` = `config` 또는 `api` |
| `fileshare_shares_disabled` | 관리 API로 오프라인 처리된 공유 |
| `fileshare_generation` | 시작 이후 제공된 공유 목록의 수 |
| `fileshare_connections_accepted_total{protocol}` | 수락된 연결 |
| `fileshare_connections_open{protocol}` | 현재 열려 있는 연결 |
| `fileshare_admin_changes_total{result}` | 관리 변경. `applied` 또는 `refused` |
| `fileshare_admin_requests_total{method,code}` | 메서드와 gRPC 상태별 관리 API 호출 |
| `fileshare_directory_people` | 마지막으로 읽었을 때 디렉터리가 보유한 사용자 |
| `fileshare_directory_reloads_total{result}` | [다시 불러오기]({{< relref "/administration/reload.md" >}}): `unchanged`, `added`, `swapped`, `failed` |
| `fileshare_revocation_list_age_seconds{list}` | [폐기 목록]({{< relref "/security/revocation-lists.md" >}})의 마지막 정상 사본의 경과 시간. 한 번도 가져오지 못했으면 `-1` |
| `fileshare_revocation_list_fetch_failures_total{list}` | 해당 목록의 가져오기 실패 횟수 |
| `fileshare_ssf_last_heard_seconds` | [공유 신호]({{< relref "/security/shared-signals.md" >}}) 전송자가 마지막으로 응답한 이후의 초. 한 번도 없으면 `-1` |
| `fileshare_ssf_revoked_subjects` | 공급자가 이전 자격 증명을 폐기한 사용자(현재 보관 중인 기준) |
| `fileshare_sftp_domain_grant_total{result}` | [`ssh_domains`]({{< relref "/protocols/sftp.md#certificates-meant-for-this-host-the-domain-grant" >}})에 조회된 SFTP 인증서: `granted`, `accepted_ungranted`, `refused_not_granted`, `refused_absent`, `refused_malformed` |
| `fileshare_start_time_seconds` | 이 프로세스가 시작된 시각 |

여기에 Go 런타임과 빌드 정보가 더해집니다. 폐기 및 공유 신호 계열은 구성에
목록이나 `ssf` 블록이 있을 때만, 도메인 부여 계열은 `ssh_domains`를 설정했을
때만, 관리 요청 계열은 `admin` 블록이 있을 때만 나타납니다.

{{< callout type="error" >}}
**어떤 메트릭도 공유나 사용자의 이름을 담지 않습니다**

WebDAV는 누군가 사용할 수 없는 공유에 **404**로 응답하여 그 공유의 존재가
확인되지 않도록 하며, 메트릭 수집(scrape)도 이를 확인해 주어서는 안 됩니다.
레이블은 프로토콜, 출처, 결과, 메서드 이름이며 공유나 사용자는 절대 쓰이지
않습니다.
{{< /callout >}}

## 알림을 설정할 대상 {#what-to-alert-on}

폐기 검사는 [안전하게 차단됩니다(fail closed)]({{< relref "/security/_index.md#every-one-of-them-fails-closed" >}}).
너무 오래된 목록이나 응답이 없는 전송자가 있으면 서버는 그 목록이 관할하는
자격 증명을 **거부합니다**. 그 일이 일어난 뒤가 아니라 일어나기 전에 알림을
받으십시오.

- `fileshare_revocation_list_age_seconds`가 `ssh_krl_max_age` 또는
  `crl_max_age`(기본값 1h)에 근접할 때
- `fileshare_ssf_last_heard_seconds`가 `ssf` 블록의 `max_age`(기본값 10m)에
  근접할 때
- `fileshare_directory_reloads_total{result="failed"}`가 증가할 때: 디렉터리를
  읽을 수 없으므로 제거를 포함해 [아무것도 바뀌지 않습니다]({{< relref "/administration/reload.md" >}}).

{{< callout type="info" >}}
**`--isolate`**

`metrics` 블록은 아직 `--isolate`와 함께 쓸 수 없으며, 이 조합은 거부됩니다.
{{< /callout >}}
