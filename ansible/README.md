# Ansible 기반 통신사 네트워크 구성

저장소 루트의 수기 `*.ios` 구성(AT&T / KT / LG / Google, 총 19대 라우터)을
**Ansible 로 자동화한다면 어떻게 구성했을지**를 보여주는 데모입니다.

핵심 아이디어는 **"구성(Config)을 데이터(host_vars)로 모델링"** 하는 것입니다.
장비별 IP·OSPF·BGP 의도를 YAML 변수로 선언하고, Jinja2 템플릿이 이를 IOS
명령으로 렌더링한 뒤 `cisco.ios.ios_config` 가 **현재 구성과 비교(diff)하여 차이만**
장비에 적용합니다. 따라서 멱등(idempotent)하며 `--check --diff` 로 안전하게
사전 검증할 수 있습니다.

---

## 디렉터리 구조

```
ansible/
├── ansible.cfg               # 인벤토리/연결 기본 설정
├── requirements.yml          # 필요한 컬렉션 (cisco.ios, ansible.netcommon)
├── secrets.example.yml       # 자격증명 예시 (Vault 권장)
├── inventory/
│   └── hosts.yml             # 사업자(AS)별 그룹 인벤토리
├── group_vars/               # 그룹(사업자/서브AS) 공통 변수
│   ├── all.yml               #   연결정보 + OSPF/BGP 공통값
│   ├── att.yml               #   AS 123
│   ├── kt.yml                #   AS 1111
│   ├── lg.yml                #   AS 1112 (Confederation)
│   ├── lg_as65535.yml        #   Sub-AS 65535
│   ├── lg_as65536.yml        #   Sub-AS 65536
│   └── global_cp.yml         #   AS 1234 (Google)
├── host_vars/                # 장비별 구성 데이터 (19대)
│   ├── R1.yml ... R8.yml
│   ├── KT_1.yml ... KT_6.yml
│   ├── LG_1.yml ... LG_4.yml
│   └── GOOGLE.yml
├── roles/                    # 관심사별 롤 (각 롤 = 템플릿 1개 + ios_config 1태스크)
│   ├── common/               #   hostname + 공통 서비스
│   ├── interfaces/           #   L3 인터페이스
│   ├── ospf/                 #   router ospf
│   └── bgp/                  #   router bgp
├── site.yml                  # 전체 배포 (common→interfaces→ospf→bgp)
├── verify.yml                # 검증/제출자료(show) 수집 → reports/
├── backup.yml                # running-config 백업 → backups/
└── save_config.yml           # write memory
```

---

## 데이터 모델 (host_vars 예시)

`KT_1.yml` 한 장으로 인터페이스 / OSPF / BGP 의도를 모두 선언합니다.

```yaml
interfaces:
  - { name: Loopback0,   ipv4: 121.160.0.1,  mask: 255.255.255.255 }
  - { name: Ethernet0/0, ipv4: 123.45.67.41, mask: 255.255.255.248, description: "International Peering @ IXP" }
  - { name: Ethernet0/1, ipv4: 121.160.42.1, mask: 255.255.255.252, ospf_network: point-to-point }

ospf_passive_interfaces: [Ethernet0/0]
ospf_networks:
  - { prefix: 121.160.0.1,  wildcard: 0.0.0.0,   area: 0 }
  - { prefix: 121.160.42.0, wildcard: 0.0.0.255, area: 0 }

bgp_neighbors:
  - { ip: 121.160.0.6,  remote_as: 1111, update_source: Loopback0, next_hop_self: true, description: "iBGP to KT_6 (RR)" }
  - { ip: 123.45.67.45, remote_as: 123,  description: "eBGP to AT&T R4" }
```

| 변수 | 위치 | 설명 |
|------|------|------|
| `bgp_asn` | group_vars | 사업자별 AS (LG 는 서브AS 그룹에서 정의) |
| `bgp_confederation` | lg_as* | Confederation identifier / peers |
| `ospf_process_id`, `ospf_prefix_suppression` | all | 전 라우터 공통 |
| `interfaces` / `ospf_*` / `bgp_*` | host_vars | 장비 고유 의도 |

> **그룹 상속 활용 포인트**
> - LG 의 Confederation 서브AS(65535/65536)를 **하위 그룹**으로 나눠
>   `bgp_asn` / `bgp_confederation` 를 그룹 단위로 상속 → 장비별 중복 제거.
> - AT&T 의 P/PE 도 `att_p`(R1,R2) / `att_pe`(R3~R8) 하위 그룹으로 문서화.

---

## 사용법

```bash
# 0) 컬렉션 설치
ansible-galaxy collection install -r requirements.yml

# 1) 문법/인벤토리 점검
ansible-inventory --graph
ansible-playbook site.yml --syntax-check

# 2) 드라이런: 실제 장비 구성과의 차이만 미리 확인 (적용 안 함)
ansible-playbook site.yml --check --diff

# 3) 적용
ansible-playbook site.yml
ansible-playbook site.yml --limit kt          # KT 만
ansible-playbook site.yml --limit R4 --tags bgp # R4 의 BGP 만

# 4) 저장 / 백업 / 검증 자료 수집
ansible-playbook save_config.yml
ansible-playbook backup.yml                     # backups/<host>.cfg
ansible-playbook verify.yml                      # reports/<host>/show_outputs.txt
```

연결 자격증명은 `secrets.example.yml` 참고(Vault 권장). `inventory/hosts.yml`
의 `ansible_host` 는 실습 환경의 **관리망 주소**로 교체하세요.

---

## 기존 `*.ios` 대비 달라진 점 (의도된 개선)

1. **미사용 인터페이스 비관리** — 원본의 `no ip address` / `shutdown` 더미 포트는
   선언하지 않습니다. "관리 대상(=의도)"만 코드화하는 것이 IaC 원칙입니다.
2. **description 추가** — 피어링/CP/User 링크와 BGP 네이버에 의미 있는 설명을
   데이터로 부여(문서화 효과). 원본에는 없던 항목입니다.
3. **공통값의 단일 출처화** — `prefix-suppression`, OSPF process-id,
   `log-neighbor-changes` 등 전 장비 공통 항목을 `group_vars/all.yml` 한 곳에서 관리.

`ios_config` 는 선언한 라인만 비교/주입하므로, 위 1~3 외의 기존 라인은 건드리지
않습니다(부분 관리). 장비 전체를 강제로 일치시키려면 `cisco.ios.ios_config` 대신
구성 치환(replace) 옵션을 검토하세요.

---

## 대안: 리소스 모듈(선언형)

본 데모는 *템플릿 → `ios_config`* 방식을 택했습니다(원본 `.ios` 와 1:1 대조가 쉽고
교육적으로 투명). 운영 환경에서는 벤더 중립적이고 더 멱등적인 **리소스 모듈**도
좋은 선택입니다.

| 관심사 | 템플릿 방식(현재) | 리소스 모듈 대안 |
|--------|------------------|------------------|
| 인터페이스 | `interfaces.j2` | `cisco.ios.ios_l3_interfaces` |
| OSPF | `ospf.j2` | `cisco.ios.ios_ospfv2` |
| BGP | `bgp.j2` | `cisco.ios.ios_bgp_global` / `ios_bgp_address_family` |

> 동일한 host_vars 데이터 모델을 유지한 채 롤의 태스크만 교체하면 두 방식을
> 오갈 수 있습니다.

---

## 참고: 자동화 범위 밖

- `INTER_PEERING` 은 IXP **L2 스위치**(L3/라우팅 설정 없음)라 `fabric` 그룹에
  문서화만 하고 `site.yml`(대상: `routers`)에서는 제외했습니다.
- 토폴로지/AS/대역 설계 근거는 상위 `../README.md` 를 참고하세요.
