# Ceph 명령어 가이드

Ceph는 분산 스토리지 시스템으로, 여러 도구와 명령어를 통해 관리됩니다. 이 문서는 Ceph 클러스터 관리를 위한 주요 명령어들을 정리합니다.

## 목차

- [기본 명령어](#기본-명령어)
- [클러스터 상태 확인](#클러스터-상태-확인)
- [풀(Pool) 관리](#풀pool-관리)
- [OSD 관리](#osd-관리)
- [Monitor 관리](#monitor-관리)
- [RADOS 명령어](#rados-명령어)
- [RBD 명령어](#rbd-명령어)
- [CephFS 명령어](#cephfs-명령어)
- [Crush Map 관리](#crush-map-관리)
- [사용자 및 권한 관리](#사용자-및-권한-관리)
- [문제 해결](#문제-해결)

## 기본 명령어

### 클러스터 상태 확인

```bash
# 클러스터 상태 확인 (간단히)
ceph status
ceph -s

# 클러스터 상태 확인 (자세히)
ceph health detail

# 클러스터 성능 상태
ceph df

# OSD 트리 구조 확인
ceph osd tree

# 클러스터 사용량
ceph df detail
```

### 클러스터 설정 관리

```bash
# 클러스터 설정 보기
ceph config ls

# 특정 설정 값 보기
ceph config get <who> <option>

# 설정 값 변경
ceph config set <who> <option> <value>

# 예: osd의 최대 backfill 설정 변경
ceph config set osd osd_max_backfills 2
```

## 풀(Pool) 관리

### 풀 생성 및 관리

```bash
# 풀 목록 보기
ceph osd lspools
ceph osd pool ls detail

# 복제 풀 생성 (기본 3 복제)
ceph osd pool create <pool-name> <pg-num> [<pgp-num>] [replicated]

# 이레이져 코딩 풀 생성
ceph osd pool create <pool-name> <pg-num> <pgp-num> erasure [erasure-code-profile]

# 풀 삭제 (먼저 풀 삭제 활성화 필요)
ceph osd pool set <pool-name> nodelete false
ceph osd pool delete <pool-name> <pool-name> --yes-i-really-really-mean-it

# 풀 복제 수준 변경
ceph osd pool set <pool-name> size <num-replicas>
ceph osd pool set <pool-name> min_size <min-num-replicas>
```

### 풀 속성 설정

```bash
# 애플리케이션 활성화 (rbd, cephfs, rgw 등)
ceph osd pool application enable <pool-name> <application-name>

# 풀 쿼터 설정
ceph osd pool set-quota <pool-name> max_objects <num-objects>
ceph osd pool set-quota <pool-name> max_bytes <num-bytes>

# PG 수 변경
ceph osd pool set <pool-name> pg_num <pg-num>
ceph osd pool set <pool-name> pgp_num <pgp-num>
```

## OSD 관리

### OSD 상태 관리

```bash
# OSD 상태 확인
ceph osd status
ceph osd stat

# 특정 OSD의 상세 정보
ceph osd dump <osd-id>

# OSD down/out 설정
ceph osd down <osd-id>
ceph osd out <osd-id>

# OSD in/up 설정
ceph osd in <osd-id>
ceph osd up <osd-id>

# 클러스터 rebalancing 일시 중지/재개
ceph osd set noout
ceph osd unset noout
ceph osd set norebalance
ceph osd unset norebalance
```

### OSD 추가 및 제거

```bash
# 신규 OSD 추가
ceph-deploy osd create --data /dev/sdX <host>

# 최신 버전에서 OSD 추가
ceph orch daemon add osd <host>:/dev/sdX

# OSD 제거
ceph osd purge <osd-id> --yes-i-really-mean-it

# OSD 가중치 조정
ceph osd crush reweight osd.<osd-id> <weight>

# OSD 위치 변경
ceph osd crush set <osd-id> <weight> root=default host=<host-name>
```

## Monitor 관리

```bash
# 모니터 상태 확인
ceph mon stat
ceph mon dump

# 모니터 맵 확인
ceph mon getmap -o /tmp/monmap
monmaptool --print /tmp/monmap

# 모니터 추가
ceph mon add <mon-id> <ip-address>:6789

# 모니터 제거
ceph mon remove <mon-id>

# 모니터 쿼럼 상태 확인
ceph quorum_status --format json-pretty
```

## RADOS 명령어

RADOS는 Ceph의 기본 객체 스토리지 계층입니다.

```bash
# 풀 내 객체 목록 보기
rados -p <pool-name> ls

# 객체 정보 확인
rados -p <pool-name> stat <object-name>

# 객체 데이터 가져오기
rados -p <pool-name> get <object-name> <output-file>

# 객체 데이터 저장
rados -p <pool-name> put <object-name> <input-file>

# 객체 삭제
rados -p <pool-name> rm <object-name>

# 풀 내 스냅샷 생성
rados -p <pool-name> mksnap <snap-name>

# 스냅샷 제거
rados -p <pool-name> rmsnap <snap-name>
```

## RBD 명령어

RBD(RADOS Block Device)는 Ceph의 블록 스토리지 인터페이스입니다.

```bash
# 이미지 목록 보기
rbd ls -p <pool-name>

# 이미지 생성
rbd create <pool-name>/<image-name> --size <size-in-MB>

# 이미지 정보 확인
rbd info <pool-name>/<image-name>

# 이미지 크기 변경
rbd resize --size <new-size-in-MB> <pool-name>/<image-name>

# 이미지 삭제
rbd rm <pool-name>/<image-name>

# 스냅샷 생성
rbd snap create <pool-name>/<image-name>@<snapshot-name>

# 스냅샷으로 롤백
rbd snap rollback <pool-name>/<image-name>@<snapshot-name>

# 스냅샷 삭제
rbd snap rm <pool-name>/<image-name>@<snapshot-name>

# 이미지 클론
rbd snap protect <pool-name>/<image-name>@<snapshot-name>
rbd clone <pool-name>/<image-name>@<snapshot-name> <pool-name>/<clone-name>
```

## CephFS 명령어

CephFS는 Ceph의 분산 파일 시스템입니다.

```bash
# 파일시스템 목록 보기
ceph fs ls

# 파일시스템 상태 확인
ceph fs status <fs-name>

# 새 파일시스템 생성
ceph osd pool create cephfs_data <pg-num>
ceph osd pool create cephfs_metadata <pg-num>
ceph fs new <fs-name> <metadata-pool> <data-pool>

# 파일시스템 삭제
ceph fs fail <fs-name>
ceph fs rm <fs-name> --yes-i-really-mean-it

# MDS 상태 확인
ceph mds stat

# MDS 맵 확인
ceph mds dump

# CephFS 마운트
mount -t ceph <mon-ip>:6789:/ /mnt/cephfs -o name=admin,secret=<admin-key>
```

## Crush Map 관리

```bash
# Crush 맵 가져오기
ceph osd getcrushmap -o /tmp/crushmap.bin
crushtool -d /tmp/crushmap.bin -o /tmp/crushmap.txt

# Crush 맵 수정 후 적용
# (텍스트 파일 수정 후)
crushtool -c /tmp/crushmap.txt -o /tmp/crushmap.new
ceph osd setcrushmap -i /tmp/crushmap.new

# Crush 룰 확인
ceph osd crush rule ls
ceph osd crush rule dump

# 호스트 추가/제거
ceph osd crush add-bucket <host-name> host
ceph osd crush remove <host-name>

# 버킷 이동
ceph osd crush move <bucket-name> root=default rack=<rack-name>
```

## 사용자 및 권한 관리

```bash
# 사용자 목록
ceph auth ls

# 새 사용자 생성
ceph auth get-or-create client.<username> mon 'allow r' osd 'allow rwx pool=<pool-name>'

# 사용자 키 확인
ceph auth get client.<username>

# 사용자 권한 변경
ceph auth caps client.<username> mon 'allow r' osd 'allow rw pool=<pool-name>'

# 사용자 삭제
ceph auth del client.<username>
```

## 문제 해결

```bash
# PG 상태 확인
ceph pg dump
ceph pg stat

# 문제가 있는 PG 확인
ceph pg dump_stuck unclean
ceph pg dump_stuck inactive
ceph pg dump_stuck stale

# 특정 PG 정보 확인
ceph pg <pg-id> query

# 로그 확인
ceph log last 100

# 클러스터 경고 및 오류 해결 (불일치 해결)
ceph health detail
ceph osd repair <pg-id>

# OSD 디버그 정보
ceph daemon osd.<osd-id> dump_ops_in_flight
ceph daemon osd.<osd-id> status
```

## 성능 모니터링

```bash
# IOPS 모니터링
ceph daemon osd.<osd-id> perf dump

# 대역폭 모니터링
ceph osd pool stats <pool-name>

# 지연 시간 모니터링
ceph osd perf

# OSD 사용량 모니터링
ceph osd df
```

## Ceph 버전별 주의사항

### Nautilus (14.x)

- `ceph-mgr` 모듈이 더 많은 관리 기능을 담당
- 대시보드 모듈 활성화: `ceph mgr module enable dashboard`

### Octopus (15.x)

- 새로운 오케스트레이터 명령어 체계: `ceph orch`
- 더 많은 기능이 `ceph-mgr` 모듈로 이동

### Pacific (16.x) 이상

- BlueStore가 유일한 스토리지 백엔드
- 새로운 `cephadm` 배포 방식

## 그 외 유용한 명령어

```bash
# 전체 설정값 덤프
ceph daemon osd.<osd-id> config show

# 클러스터 이벤트 모니터링
ceph -w

# 클러스터 디스크 사용량
ceph df

# Ceph 클러스터 통계
rados df

# 현재 실행 중인 작업 확인
ceph status -f json-pretty | grep -A 10 pgmap
```

이 문서는 Ceph 클러스터 관리를 위한 기본적인 명령어를 다루며, 더 자세한 정보는 [공식 Ceph 문서](https://docs.ceph.com/)를 참조하십시오.
