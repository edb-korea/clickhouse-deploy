# EDB Postgres AI for ClickHouse 배포 (Ansible)

`clickhouse.yml` playbook과 `edb-clickhouse` role로 EDB ClickHouse(`edb-clickhouse-*` RPM)를 배포합니다. 지원하는 구성은 세 가지입니다.

| 구성 | `ch_topology` | 노드 | 설명 |
|---|---|---|---|
| 단일 노드 | `single` | 1 | Keeper와 클러스터 정의 없이 서버 1대 |
| 3노드 순환 | `ring` | 3 (N ≥ 3) | 샤드 3개 × replica 2개. Keeper는 3대에 같이 배치 |
| Keeper 분리 | `sharded` | Keeper 3 + 서버 N | Keeper 전용 3대와 서버 노드를 따로 둠. 샤드/replica 수는 inventory로 정함 |

지원 OS는 RHEL 8/9/10, Rocky Linux 9/10입니다(`edb-repository` role과 같은 제약). 패키지명과 Keeper 설정은 [EDB 문서](https://www.enterprisedb.com/docs/clickhouse/latest/install_guide/rhel_compatible)를 기준으로 했습니다.

> **검증 상태**: 대상 서버가 없어 playbook을 실제 노드에 실행하지 못했습니다. 문법 검사, `ansible-lint`, 템플릿 렌더링(XML 파싱 포함)만 확인했으니, 먼저 테스트 환경에서 `ch_run_smoke_test: true`로 실행해 보세요.

## 파일 구성

```
clickhouse.yml                              playbook
ansible.cfg                                 Ansible 설정
roles/edb-clickhouse/                       role (설치, 설정, 기동, 검증)
roles/edb-repository/                       EDB 저장소 등록 role
inventories/clickhouse/single.yml           예제: 단일 노드
inventories/clickhouse/ring3.yml            예제: 3노드 순환 (3 shard x 2 replica)
inventories/clickhouse/dedicated-keeper.yml 예제: Keeper 3대 분리 + 서버 4대 (2 shard x 2 replica)
```

## 사용법

1. EDB 구독 토큰을 `~/.edb-token`에 저장합니다. `clickhouse` 플랜 접근 권한이 필요합니다. 다른 경로의 파일을 쓰려면 `-e ch_edb_token_file=/path/to/token`을 지정하세요.
2. 예제 inventory를 복사해서 호스트와 IP를 바꿉니다.
3. 실행합니다.

```bash
ansible-playbook -i inventories/clickhouse/ring3.yml clickhouse.yml
```

```bash
# 배포 후 샤딩/복제까지 확인하는 smoke test
ansible-playbook -i inventories/clickhouse/ring3.yml clickhouse.yml -e ch_run_smoke_test=true
```

playbook은 다음 순서로 진행합니다.

1. 모든 호스트에 EDB 저장소, 패키지, 설정 파일을 배포합니다. 이 단계에서는 서비스를 건드리지 않습니다.
2. Keeper를 **한 번에 한 노드씩** 기동합니다. 설정이 바뀌어 재시작할 때도 quorum이 깨지지 않습니다.
3. Keeper quorum을 확인합니다(`ruok`, `mntr`).
4. 서버를 기동합니다. 기본은 한 번에 1대씩이며, 최초 배포를 빠르게 하려면 `-e ch_serial=4`처럼 올립니다.
5. `system.clusters`, `system.zookeeper`로 클러스터와 Keeper 연결을 확인합니다.

설정이 바뀌지 않으면 서비스를 재시작하지 않으므로 다시 실행해도 안전합니다.

## inventory 변수

클러스터는 `clickhouse_clusters` 그룹 아래 `cluster_<cluster_name>` 자식 그룹으로 정의합니다.

| 변수 | 위치 | 설명 |
|---|---|---|
| `cluster_name` | 그룹 | 클러스터 이름. ClickHouse 클러스터 이름(`{cluster}` 매크로)으로도 쓰입니다 |
| `ch_topology` | 그룹 | `single`, `ring`, `sharded` |
| `ch_roles` | 호스트 | `[server]`, `[keeper]`, `[server, keeper]` |
| `ch_keeper_id` | 호스트 | Keeper 노드의 고유 정수 ID (`keeper` 역할에 필수) |
| `ch_shard` | 호스트 | 샤드 번호 1..N, 빈 번호 없이 (`ring`, `sharded`에 필수) |
| `ch_replica` | 호스트 | 샤드 안의 replica 번호 (`sharded`에 필수) |
| `ch_address` | 호스트 | 노드 간 통신에 쓸 주소. 기본은 `ansible_host`, 없으면 inventory 이름 |

전체 옵션(포트, 데이터 디렉터리, 패키지 목록, 비밀번호 등)과 기본값은 [roles/edb-clickhouse/defaults/main.yml](roles/edb-clickhouse/defaults/main.yml)에 있습니다. 주요 항목은 다음과 같습니다.

| 변수 | 기본값 | 설명 |
|---|---|---|
| `ch_edb_repository` | `clickhouse` | EDB 저장소(플랜) 이름 |
| `ch_upgrade` | `install-but-no-automatic-upgrades` | `always-upgrade-to-latest-minor`로 바꾸면 패키지를 최신으로 올립니다 |
| `ch_data_directory` | `/var/lib/clickhouse` | 서버 데이터 디렉터리 |
| `ch_default_password` | 빈 값 | `default` 사용자 비밀번호. EDB 패키지 기본값은 비밀번호 없음입니다 |
| `ch_cluster_secret` | 빈 값 | 멀티 노드에서 비밀번호를 쓸 때 필수. Distributed 쿼리의 노드 간 인증에 사용됩니다 |
| `ch_interserver_password` | 빈 값 | replica 간 파트 전송 인증 |
| `ch_run_smoke_test` | `false` | 배포 후 테스트 테이블로 샤딩과 복제를 확인 |

비밀번호와 secret은 Ansible Vault에 저장하세요.

## 구성별 설명

### 단일 노드 (`single`)

서버 1대만 설치합니다. Keeper와 `<remote_servers>`는 만들지 않습니다.

### 3노드 순환 (`ring`)

3대로 샤드 3개, 샤드마다 replica 2개를 만들려면 노드 한 대가 샤드 두 개를 맡아야 합니다. 그래서 **호스트마다 ClickHouse 인스턴스를 2개** 실행합니다.

| 호스트 | 인스턴스 1 (포트 9000) | 인스턴스 2 (포트 9001) |
|---|---|---|
| ch-ring-1 | 샤드 1 | 샤드 3 |
| ch-ring-2 | 샤드 2 | 샤드 1 |
| ch-ring-3 | 샤드 3 | 샤드 2 |

각 샤드의 replica는 자기 호스트(인스턴스 1)와 다음 호스트(인스턴스 2)에 있으므로, 호스트 1대가 죽어도 모든 샤드의 데이터가 남습니다. 마지막 샤드는 첫 번째 호스트로 돌아가는 순환 구조입니다. `ch_shard`를 1..N으로 주면 N대(N ≥ 3)까지 같은 방식으로 늘릴 수 있습니다.

- 인스턴스 1은 패키지의 `clickhouse-server` 서비스와 `/etc/clickhouse-server`를 그대로 씁니다.
- 인스턴스 2는 패키지 설정(`config.xml`, `users.xml`)을 `/etc/clickhouse-server-2`로 복사하고, 별도 포트·데이터 디렉터리(`/var/lib/clickhouse-2`)·로그 디렉터리·systemd 서비스(`clickhouse-server-2`)로 실행합니다. systemd unit은 role이 만들며 `ch_server_binary`(기본 `/usr/bin/clickhouse-server`)를 실행합니다.
- 두 인스턴스 모두 일반 replica이므로 `ON CLUSTER` DDL을 그대로 쓸 수 있습니다. 샤드/replica 매크로가 인스턴스마다 설정되어 있습니다.
- Keeper는 3대에 함께 배치합니다(`clickhouse-keeper` 서비스, 포트 9181/9234).
- 방화벽에서 노드 간 포트 9000, 9001, 9009, 9010, 9181, 9234를 열어야 합니다. 클라이언트 접속에는 8123/8124(HTTP)나 9000/9001(native)을 씁니다.
- 한 호스트의 메모리와 디스크를 두 인스턴스가 나눠 쓰므로 용량을 그에 맞춰 잡으세요.

```sql
-- 어느 인스턴스에서든 실행
CREATE TABLE events ON CLUSTER ch_ring (
    event_id UInt32, user_id UInt32, event_type String, created_at DateTime
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/events', '{replica}')
ORDER BY event_id;

CREATE TABLE events_dist ON CLUSTER ch_ring AS events
ENGINE = Distributed(ch_ring, default, events, rand());
```

### Keeper 분리 (`sharded`)

Keeper 전용 노드 3대(`ch_roles: [keeper]`, `edb-clickhouse-keeper` 패키지)와 서버 노드(`ch_roles: [server]`)를 따로 둡니다. 서버 노드의 `ch_shard`, `ch_replica`로 모양을 정하므로, 예제의 2샤드 × 2replica 외에 3샤드 × 2replica 같은 구성도 호스트만 추가하면 됩니다. 샤드마다 replica가 1개뿐이면 Keeper 없이도 구성할 수 있습니다.

서버 노드에 `[server, keeper]`를 같이 주면 Keeper를 서버와 한 노드에 둘 수 있습니다.

## 확인

```bash
clickhouse-client -h <server-ip> --query "SELECT cluster, shard_num, replica_num, host_name, port FROM system.clusters WHERE cluster = 'ch_ring'"
clickhouse-keeper-client -h 127.0.0.1 -p 9181 -q mntr   # Keeper 노드에서
```

## 제한 사항

- `ring`에서 `sharded`로 바꾸는 식의 토폴로지 변경이나 샤드 추가(resharding)는 다루지 않습니다. 데이터 재분배는 ClickHouse 쪽 작업입니다.
- `--check`(dry run)는 첫 번째 play(저장소, 패키지, 설정 배포)까지만 의미가 있습니다. 아직 설치되지 않은 호스트에서는 저장소 등록과 서비스 기동이 실제로 일어나지 않아 이후 단계가 실패합니다. 문법만 볼 때는 `--syntax-check`를 쓰세요.
- 방화벽, SELinux, OS 커널/파일 제한 튜닝은 이 role의 범위가 아닙니다.
- Keeper 노드를 추가하거나 제거하는 멤버십 변경은 한 노드씩 직접 진행해야 합니다.
