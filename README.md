# ElasticBLAST on AWS 실행 가이드

## 목차
- [개요](#개요)
- [프로젝트 구성](#프로젝트-구성)
- [사전 요구사항](#사전-요구사항)
- [1단계: 환경 준비](#1단계-환경-준비)
- [2단계: 데이터 준비](#2단계-데이터-준비)
- [3단계: ElasticBLAST 설치](#3단계-elasticblast-설치)
- [4단계: 설정 파일 작성](#4단계-설정-파일-작성)
- [5단계: BLAST 실행](#5단계-blast-실행)
- [6단계: 결과 확인](#6단계-결과-확인)
- [7단계: 리소스 정리](#7단계-리소스-정리)
- [비용 예상](#비용-예상)
- [보안](#보안)
- [문제 해결](#문제-해결)

---

## 개요

이 가이드는 AWS에서 ElasticBLAST를 사용하여 대규모 BLASTN 검색을 수행하는 방법을 설명합니다.

### 프로젝트 정보
- **목적**: 500개 unmapped reads에 대한 NCBI NT 데이터베이스 검색
- **BLAST 프로그램**: BLASTN (nucleotide-nucleotide search)
- **데이터베이스**: NCBI NT (2026-09 기준 약 1,117GiB, 압축 해제 기준. 2025-12 실행 당시는 약 715GB)
- **쿼리**: 500개 시퀀스 (76KB)
- **클라우드 제공자**: AWS
- **리전**: us-east-1 (버지니아 북부)

### ElasticBLAST란?

ElasticBLAST는 클라우드 환경에서 대규모 BLAST 검색을 자동으로 분산 처리하는 도구입니다:
- 여러 EC2 인스턴스에 자동으로 작업 분산
- BLAST 데이터베이스 자동 다운로드 및 관리
- 결과를 S3에 자동 저장
- 작업 완료 후 자동 리소스 정리

---

## 프로젝트 구성

```
blast-project/
├── data/
│   └── query.fasta              # 쿼리 시퀀스 (500개)
├── elastic-blast/               # ElasticBLAST GitHub 저장소 (clone됨)
├── elasticblast-config.ini      # ElasticBLAST 설정 파일
├── .elb-venv/                   # Python 가상환경
└── README.md                    # 이 파일
```

---

## 사전 요구사항

### AWS 계정 및 권한
- AWS 계정 필요
- 다음 서비스에 대한 IAM 권한:
  - AWS Batch (작업 스케줄링)
  - EC2 (컴퓨팅 인스턴스)
  - S3 (데이터 저장)
  - CloudFormation (인프라 관리)
  - IAM (역할 생성)
  - CloudWatch Logs (로그 관리)

### 로컬 환경
- Python 3.11 이상
  - ElasticBLAST 1.5.0은 Python 3.9/3.10에서도 `pip install`은 성공하지만, `elastic-blast --version` 실행 시 `ImportError: cannot import name 'UnionType'`(3.9) 또는 `datetime.UTC` 관련 오류(3.10)로 실패합니다. 패키지의 `python_requires >= 3.7`은 갱신되지 않은 값입니다.
- AWS CLI 설치 및 구성
- 충분한 디스크 공간 (결과 다운로드용)

### AWS 리소스 제한 확인
ElasticBLAST는 다음 리소스를 사용합니다:
- EC2 인스턴스: 기본 4개 (설정 가능)
- EBS 볼륨: 인스턴스당 1개
- S3 버킷: 1개 (사전 생성 필요)

---

## 1단계: 환경 준비

### 1.1 AWS CLI 구성 확인

```bash
# AWS 계정 정보 확인
aws sts get-caller-identity

# 출력 예시:
# {
#     "UserId": "AROAXXXXX:username",
#     "Account": "123456789123",
#     "Arn": "arn:aws:sts::123456789123:assumed-role/..."
# }
```

### 1.2 작업 디렉토리 생성

```bash
mkdir -p blast-project
cd blast-project
```

### 1.3 ElasticBLAST 저장소 클론 및 janitor 역할 생성

```bash
git clone https://github.com/ncbi/elastic-blast.git

# janitor(자동 정리) IAM 역할 생성 -- 계정당 1회
bash elastic-blast/bin/aws-create-elastic-blast-janitor-role.sh

# 생성 확인
bash elastic-blast/bin/aws-describe-elastic-blast-janitor-role.sh
```

저장소를 클론하는 실제 이유는 이 스크립트입니다. ElasticBLAST는 `submit` 시 janitor 역할이 있는지 확인하고, **없으면 오류 없이(debug 로그만 남기고) 자동 정리를 비활성화**합니다. 그 상태로 `elastic-blast delete`를 잊으면 인스턴스가 계속 과금되므로 처음 사용하는 계정에서는 반드시 한 번 실행하십시오.

---

## 2단계: 데이터 준비

### 2.1 쿼리 파일 준비

쿼리 파일은 FASTA 형식이어야 합니다:

```bash
# 쿼리 파일 위치
data/query.fasta

# 파일 정보 확인
wc -l data/query.fasta        # 총 라인 수
grep -c "^>" data/query.fasta  # 시퀀스 개수
ls -lh data/query.fasta        # 파일 크기
```

**현재 프로젝트 쿼리 정보:**
- 시퀀스 개수: 500개
- 파일 크기: 76KB
- 형식: FASTA

### 2.2 S3 버킷 생성

```bash
# S3 버킷 생성 (버킷 이름은 전역적으로 고유해야 함)
aws s3 mb s3://elasticblast --region us-east-1

# 버킷 확인
aws s3 ls s3://elasticblast
```

### 2.3 쿼리 파일 S3 업로드

```bash
# 쿼리 파일을 S3에 업로드
aws s3 cp data/query.fasta s3://elasticblast/queries/query.fasta

# 업로드 확인
aws s3 ls s3://elasticblast/queries/
```

---

## 3단계: ElasticBLAST 설치

### 3.1 Python 가상환경 생성

```bash
# 가상환경 생성
python3 -m venv .elb-venv

# 가상환경 활성화
source .elb-venv/bin/activate
```

### 3.2 ElasticBLAST 설치

```bash
# wheel 패키지 설치
pip install wheel

# ElasticBLAST 설치 (버전 1.5.0)
pip install elastic-blast==1.5.0

# 설치 확인
elastic-blast --version
# 출력: elastic-blast 1.5.0
```

**중요:** 버전 1.4.0 이전은 리소스 정리 문제가 있으므로 1.5.0 이상 사용을 권장합니다.

---

## 4단계: 설정 파일 작성

### 4.1 설정 파일 생성

`elasticblast-config.ini` 파일을 생성합니다:

```ini
[cloud-provider]
aws-region = us-east-1

[cluster]
num-nodes = 4
machine-type = r5d.24xlarge
labels = owner=hyunmin

[blast]
program = blastn
db = nt
queries = s3://elasticblast/queries/query.fasta
results = s3://elasticblast/results/nt-test
options = -evalue 1.0E-3 -max_target_seqs 5 -outfmt '6 qseqid qstart qend qcovs qcovhsp qcovus sseqid stitle sstart send evalue bitscore nident pident mismatch gaps sstrand staxids sscinames'
```

### 4.2 설정 파일 설명

#### [cloud-provider] 섹션
- `aws-region`: AWS 리전 (us-east-1 권장 - NCBI 데이터베이스와 동일 리전)

#### [cluster] 섹션
- `num-nodes`: 워커 노드 수 (4개 = 병렬 처리 4개)
- `machine-type`: EC2 인스턴스 타입 (r5d.24xlarge = 768GB 메모리. 2025-12 실행 당시 nt에 맞는 최소 사양이었으나 **현재 nt는 768GB에 들어가지 않습니다** -- 아래 [인스턴스 타입 선택 가이드](#인스턴스-타입-선택-가이드) 참고)
- `labels`: 리소스 태그 (비용 추적 및 관리용)

#### [blast] 섹션
- `program`: BLAST 프로그램 (blastn, blastp, blastx 등)
- `db`: 데이터베이스 이름 (nt = NCBI Nucleotide database)
- `queries`: S3에 업로드된 쿼리 파일 경로
- `results`: 결과를 저장할 S3 경로
- `options`: BLAST 실행 옵션
  - `-evalue 1.0E-3`: E-value 임계값
  - `-max_target_seqs 5`: 최대 5개 매치 결과
  - `-outfmt '6 ...'`: 탭 구분 출력 형식 (qseqid, staxids, sscinames 등 포함)

### 4.3 NCBI 호스팅 데이터베이스

ElasticBLAST는 NCBI가 AWS S3에 호스팅하는 데이터베이스를 자동으로 사용합니다:

- **S3 버킷**: `s3://ncbi-blast-databases/`
- **리전**: us-east-1
- **NT 데이터베이스 구성**:
  - `nt_euk`: Eukaryote sequences (192 볼륨)
  - `nt_prok`: Prokaryote sequences (29 볼륨)
  - `nt_viruses`: Virus sequences (22 볼륨)
  - `nt_others`: Other sequences (1 볼륨)
- **총 크기**: `nt` 약 1,117GiB (2026-09 기준, 390 볼륨, 4.66조 염기. 압축 해제 기준이며 계속 커집니다)
  - 위 `nt_euk`/`nt_prok`/`nt_viruses`/`nt_others`는 별도 DB로, `db = nt`는 `nt.*` 볼륨만 내려받습니다. 미생물 탐지 목적이면 `nt_prok`(약 114GiB)·`nt_viruses`(약 60GiB) 지정이 비용 절감 수단입니다

**장점:**
- 자동 다운로드 및 캐싱
- 데이터 전송 비용 없음 (동일 리전)
- 최신 버전 자동 사용

---

## 5단계: BLAST 실행

### 5.1 DB 버전 기록

ElasticBLAST는 항상 `s3://ncbi-blast-databases/latest-dir`가 가리키는 최신 DB를 사용합니다. 날짜를 기록해 두지 않으면 나중에 같은 결과를 재현할 수 없으므로(다른 날 실행하면 다른 nt를 검색하게 됨), 제출 직전에 DB 버전과 BLAST+ 버전을 결과 폴더에 저장합니다.

```bash
# 결과 경로 (설정 파일의 results와 동일)
export YOUR_RESULTS_BUCKET=s3://elasticblast/results/nt-test

# DB 디렉터리(날짜)와 메타데이터의 last-updated 기록
LATEST_DIR=$(aws s3 cp s3://ncbi-blast-databases/latest-dir -)
aws s3 cp s3://ncbi-blast-databases/${LATEST_DIR}/nt-nucl-metadata.json nt-nucl-metadata.json
{
  echo "ncbi-blast-databases latest-dir: ${LATEST_DIR}"
  echo "nt last-updated: $(python3 -c "import json; print(json.load(open('nt-nucl-metadata.json'))['last-updated'])")"
  echo "elastic-blast: $(elastic-blast --version)"
  echo "BLAST+: 2.17.0 (elastic-blast 1.5.0 컨테이너 이미지에 포함)"
} > db-version.txt

aws s3 cp db-version.txt ${YOUR_RESULTS_BUCKET}/db-version.txt
aws s3 cp nt-nucl-metadata.json ${YOUR_RESULTS_BUCKET}/nt-nucl-metadata.json
```

`db = core_nt` 등 다른 DB를 쓰면 파일명을 `<db>-nucl-metadata.json`으로 바꾸십시오.

### 5.2 ElasticBLAST 제출

```bash
# 가상환경 활성화 (아직 활성화되지 않은 경우)
source .elb-venv/bin/activate

# BLAST 작업 제출
elastic-blast submit --cfg elasticblast-config.ini
```

**예상 출력:**
```
awslimitchecker 12.0.0 is AGPL-licensed free software...
WARNING: Using gp3 30GB EBS root disk because locally attached SSDs will be used
```

### 5.3 진행 상황 모니터링

```bash
# 상태 확인
elastic-blast status --cfg elasticblast-config.ini
```

**상태 변화:**
1. `SUBMITTING`: 클라우드 리소스 생성 중
2. `RUNNING`: BLAST 작업 실행 중 (배치별 진행 상황 표시)
3. `SUCCEEDED`: 모든 작업 완료

**실행 중 출력 예시:**
```
ElasticBLAST search: elasticblast-abc123
Status: RUNNING
Batches: 10 total, 7 succeeded, 3 running, 0 failed
```

### 5.4 실행 과정

ElasticBLAST는 다음 단계를 자동으로 수행합니다:

1. **AWS Batch 환경 생성** (~5분)
   - Compute Environment 생성
   - Job Queue 생성
   - Job Definition 생성

2. **EC2 인스턴스 시작** (~2-3분)
   - 4개 워커 노드 시작
   - 인스턴스 타입: 설정 파일의 `machine-type` (비우면 m5ad/c5ad/r5ad 패밀리 중 자동 선택. 2025-12에는 r5ad.24xlarge가 선택되었지만 현재 nt 크기에서는 맞는 타입이 없어 오류가 납니다. 아래 비용 절 참고)

3. **NT 데이터베이스 다운로드** (~11-13분/노드, r5d.24xlarge 기준)
   - NCBI S3에서 자동 다운로드 (노드당 1회, 로컬 NVMe에 저장)
   - 같은 노드의 다른 배치는 다운로드가 끝날 때까지 대기

4. **쿼리 배치 분할 및 실행**
   - 쿼리를 여러 배치로 분할
   - 각 배치를 병렬 실행 (노드당 5개 배치 동시, 배치당 16스레드)

5. **결과 저장**
   - 각 배치 결과를 S3에 gzip 압축하여 저장
   - 파일명: `batch_NNN-blastn-nt.out.gz`

### 5.5 예상 소요 시간

- **총 소요 시간**: 약 24분 (500개 쿼리, r5d.24xlarge x 4 실측 23.9분; 쿼리 크기에 따라 다름)
  - 인프라 생성: ~3-5분
  - 데이터베이스 다운로드: ~11-13분 (노드당, 병렬)
  - BLAST 실행: ~10-12분 (다운로드 직후 첫 배치 7-8분, 이후 배치 1-2분)

---

## 6단계: 결과 확인

### 6.1 결과 다운로드

```bash
# 결과 경로 환경 변수 설정
export YOUR_RESULTS_BUCKET=s3://elasticblast/results/nt-test

# 결과 파일 다운로드 (.out.gz 파일만)
aws s3 cp ${YOUR_RESULTS_BUCKET}/ . --exclude "*" --include "*.out.gz" --recursive

# 다운로드된 파일 확인
ls -lh batch_*.out.gz
```

### 6.2 결과 압축 해제

```bash
# 모든 결과 파일 압축 해제
gunzip batch_*.out.gz

# 압축 해제된 파일 확인
ls -lh batch_*.out
```

### 6.3 결과 파일 형식

결과는 탭으로 구분된 형식입니다 (설정 파일의 `-outfmt` 옵션에 따름):

```
qseqid  qstart  qend  qcovs  qcovhsp  qcovus  sseqid  stitle  sstart  send  evalue  bitscore  nident  pident  mismatch  gaps  sstrand  staxids  sscinames
```

**컬럼 설명:**
- `qseqid`: 쿼리 시퀀스 ID
- `qstart`, `qend`: 쿼리 정렬 시작/끝 위치
- `qcovs`: 쿼리 커버리지 (%)
- `sseqid`: 대상 시퀀스 ID
- `stitle`: 대상 시퀀스 제목
- `evalue`: E-value (통계적 유의성)
- `bitscore`: Bit score
- `pident`: 일치율 (%)
- `staxids`: 분류학적 ID
- `sscinames`: 과학적 이름

### 6.4 결과 확인 예시

```bash
# 첫 10줄 확인
head -10 batch_000-blastn-nt.out

# 특정 쿼리 결과 검색
grep "^1\t" batch_*.out

# 결과 통계
wc -l batch_*.out  # 총 매치 수
```

### 6.5 실행 요약(run-summary) 저장

`elastic-blast delete`를 실행하기 **전에** 실행 요약을 저장합니다. 노드 수·인스턴스 타입·배치별 실행 시간·DB 다운로드 시간 등이 JSON으로 기록되어 비용 분석과 재현에 쓰입니다.

```bash
elastic-blast run-summary --cfg elasticblast-config.ini -o run-summary.json

# 결과 폴더에 함께 보관
aws s3 cp run-summary.json ${YOUR_RESULTS_BUCKET}/run-summary.json
```

처음 실행하면 ElasticBLAST가 AWS Batch 잡 로그를 `s3://<results>/logs/`에 복사해 두므로, 그 뒤에는 클러스터를 삭제한 뒤에도 같은 명령으로 요약을 다시 만들 수 있습니다. 반대로 한 번도 실행하지 않고 `delete`하면 요약을 만들 때 필요한 Batch 잡 정보를 더 조회할 수 없습니다.

---

## 7단계: 리소스 정리

### 7.1 ElasticBLAST 리소스 삭제

**중요:** 작업 완료 후 반드시 실행하여 불필요한 비용 발생을 방지해야 합니다. 삭제 전에 [6.5 실행 요약 저장](#65-실행-요약run-summary-저장)을 먼저 끝내십시오.

```bash
# 가상환경 활성화
source .elb-venv/bin/activate

# 클라우드 리소스 삭제
elastic-blast delete --cfg elasticblast-config.ini
```

### 7.2 삭제 확인

```bash
# 실행 중인 EC2 인스턴스 확인
aws ec2 describe-instances \
  --filter Name=tag:billingcode,Values=elastic-blast \
  --query "Reservations[*].Instances[?State.Name=='running'].InstanceId" \
  --output text
```

출력이 없으면 모든 리소스가 정상적으로 삭제된 것입니다.

### 7.3 S3 버킷 정리 (선택사항)

```bash
# 결과 파일 삭제
aws s3 rm s3://elasticblast/results/ --recursive

# 쿼리 파일 삭제
aws s3 rm s3://elasticblast/queries/ --recursive

# 버킷 삭제 (비어있어야 함)
aws s3 rb s3://elasticblast
```

---

## 비용 예상

### 주요 비용 요소

1. **EC2 인스턴스** (가장 큰 비용)
   - 인스턴스 타입: r5d.24xlarge (수동 지정, 2025-12 실행 당시 NT DB 최소 요구사항)
   - 스펙: 96 vCPU, 768GB 메모리, 4 x 900GB NVMe SSD
   - 개수: 4개
   - 실행 시간: **약 24분** (500개 쿼리 기준, 제출~완료 벽시계. 실측 23.9분)
     - 노드당 NT DB 다운로드 11~13분 + BLAST 배치 처리 10~12분
   - 실제 비용: **$11.28** (1.63 노드-시간 x $6.912/시간, Cost Explorer 실측)

2. **EBS 볼륨**
   - 타입: gp3
   - 크기: 인스턴스당 30GB
   - 예상 비용: $0.50 미만

3. **S3 스토리지**
   - 쿼리 파일: 76KB (무시 가능)
   - 결과 파일: 수 MB ~ 수십 MB
   - 예상 비용: $0.10 미만

4. **데이터 전송**
   - NCBI S3 → EC2 (동일 리전): 무료
   - EC2 → S3 (동일 리전): 무료

**총 실제 비용: 약 $11.5** (500개 쿼리, 4개 노드, 24분 기준)

> **참고:** 이 문서의 이전 버전에 있던 "약 10-11시간, $215-240"은 집계 오류였습니다. AWS Batch 잡 25개의 실행시간 합계(약 11시간, 같은 노드의 DB 다운로드 대기 시간 포함)를 전체 소요시간으로 읽고, 거기에 노드 수를 다시 곱해 비용을 계산한 값입니다. 실제 벽시계와 청구액은 위와 같습니다.
>
> 실행 시간은 노드 수를 늘려도 거의 줄지 않습니다. 노드당 DB 다운로드(약 12분)가 고정비이고, 500개 쿼리 규모에서는 BLAST 자체가 10분 안에 끝나기 때문입니다. 6노드로 실행해도 벽시계는 23.9분으로 같았습니다.

### 인스턴스 타입 선택 가이드

#### nt 현재 크기와 메모리 요구량 (2026-09 기준)

NT DB는 계속 커집니다. 2025-12 실행 당시 nt는 2.9조 염기(디스크 715GB, 캐시 679GB)였지만, 2026-09-28 메타데이터 기준으로는 다음과 같습니다.

| 항목 | 값 |
|------|-----|
| 염기 수 | 4.66조 (9개월 사이 +61%) |
| 디스크 크기 (`bytes-total`) | 약 1,117GiB (390 볼륨) |
| 캐시 필요량 (`bytes-to-cache`) | 약 1,089GiB |
| ElasticBLAST 1.5.0이 요구하는 인스턴스 메모리 | 약 1,151GiB (= 캐시 필요량 + 여유분 60GiB + 2GiB) |

따라서 **768GB급(r5d.24xlarge, r5ad.24xlarge)으로는 nt 전체를 더 이상 메모리에 캐시할 수 없습니다.** 실행 전 아래 명령으로 현재 값을 확인하십시오.

```bash
# 최신 DB 디렉터리의 nt 메타데이터에서 캐시 필요량 확인
aws s3 cp s3://ncbi-blast-databases/$(aws s3 cp s3://ncbi-blast-databases/latest-dir -)/nt-nucl-metadata.json - \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['last-updated'], round(d['bytes-to-cache']/2**30), 'GiB to cache')"
```

**자동 선택의 한계:** `machine-type`을 비우면 ElasticBLAST는 **m5ad / c5ad / r5ad 세 패밀리만** 후보로 두고 메모리 조건을 만족하는 가장 작은 타입을 고릅니다. 세 패밀리의 최대 메모리는 768GiB(r5ad.24xlarge)이므로 현재 nt에서는 맞는 타입이 없고, `submit`이 다음 오류로 실패합니다.

```
An AWS machine type with memory 1151GB and 16 CPUs could not be found
```

`machine-type`을 직접 지정하면 이 검사는 넘어가지만, 메모리가 부족하면 DB가 RAM에 들어가지 않아 검색이 매우 느려집니다.

#### 선택지

us-east-1 Linux 온디맨드 단가 (2026-10 기준, 변동 가능):

| 선택지 | 인스턴스 타입 | 메모리 | 로컬 스토리지 | 시간당 비용 | 현재 nt 캐시 |
|--------|--------------|--------|-------------|------------|-------------|
| 2025-12 실행 구성 (참고) | r5d.24xlarge | 768 GB | 4 x 900GB NVMe | $6.912 | ✗ 약 321GiB 부족 |
| 자동 선택 결과 (참고) | r5ad.24xlarge | 768 GB | 4 x 900GB NVMe | $6.288 | ✗ 약 321GiB 부족 |
| **A. nt 유지** | x1.32xlarge | 1,952 GB | 2 x 1,920GB SSD | $13.338 | ✓ |
| **B. core_nt로 전환** | r5d.12xlarge | 384 GB | 2 x 900GB NVMe | $3.456 | ✓ (core_nt 캐시 약 253GiB, 디스크 약 281GiB) |
| 소형 DB (nt_prok 등) | r5d.4xlarge ~ r5d.12xlarge | 128 ~ 384 GB | NVMe | $1.152 ~ $3.456 | ✓ (DB에 따라) |

- **A. nt 유지 (x1.32xlarge)**: `machine-type = x1.32xlarge`를 명시합니다. 시간당 단가는 r5d.24xlarge의 약 2배이지만 nt 전체를 캐시할 수 있는 ElasticBLAST 로컬 SSD 경로 중 가장 단순한 선택입니다.
- **B. core_nt로 전환 (r5d.12xlarge)**: `db = core_nt`로 바꿉니다. core_nt는 "대부분의 진핵생물 염색체 서열을 제외한 nt"로, 전사체·유전자 서열은 동일합니다. 염색체 어셈블리에 걸리는 매치가 중요한 분석이면 소규모 쿼리로 nt와 결과를 비교한 뒤 전환하십시오.
- `r5d.12xlarge`·`r5d.16xlarge`처럼 "메모리 부족"으로 표시되던 타입은 DB를 core_nt 또는 서브셋으로 바꾸면 사용할 수 있습니다.

**r5ad(자동 선택) vs r5d(명시) 실측:** 같은 500개 쿼리 작업(2025-12, 당시 nt)에서 자동 선택된 r5ad.24xlarge는 DB 다운로드가 1.8배(21분 vs 12분), BLAST 배치가 1.4배 느려 벽시계 36분, $15.24가 걸렸습니다(r5d.24xlarge: 24분, $11.28). 단가가 9% 낮아도 총비용과 시간 모두 불리했으므로, 어떤 DB를 쓰든 `machine-type`은 자동 선택에 맡기지 말고 명시하십시오.

### 비용 절감 팁

1. **인스턴스 타입 지정**: `machine-type` 명시적 지정 (자동 선택 방지. DB 크기에 맞는 타입은 위 표 참고)
2. **노드 수 조정**: 대량 쿼리가 아니면 4개 노드로 충분
3. **즉시 정리**: 작업 완료 후 즉시 `elastic-blast delete` 실행
4. **리전 선택**: us-east-1 사용 (NCBI 데이터베이스와 동일 리전)

---

## 보안

ElasticBLAST 1.5.0의 기본 설정은 빠른 시작에 맞춰져 있어, 조직의 보안 기준에 따라 아래 항목을 조정해야 할 수 있습니다.

### 사용 통계 전송 끄기

ElasticBLAST는 기본적으로 NCBI에 사용 통계를 보냅니다. 이를 끄는 `BLAST_USAGE_REPORT`는 **설정 파일(ini) 키가 아니라 `elastic-blast` 명령을 실행하는 클라이언트의 환경변수**입니다.

```bash
export BLAST_USAGE_REPORT=false
elastic-blast submit --cfg elasticblast-config.ini
```

### EBS 루트 볼륨 암호화

ElasticBLAST가 만드는 런치 템플릿은 워커 노드의 루트 EBS 볼륨을 `Encrypted: false`로 생성하며, 설정 파일로 바꿀 수 없습니다. 암호화가 필요하면 리전 단위의 **EBS 기본 암호화**를 켜 두십시오. 켜 두면 템플릿의 값과 무관하게 새 볼륨이 암호화됩니다.

```bash
aws ec2 enable-ebs-encryption-by-default --region us-east-1
aws ec2 get-ebs-encryption-by-default --region us-east-1
```

### 프라이빗 서브넷에서 실행

기본 설정에서 ElasticBLAST는 전용 VPC를 만들고 서브넷에 퍼블릭 IP를 자동 부여합니다(`MapPublicIpOnLaunch: true`). 퍼블릭 IP가 허용되지 않는 환경이면 기존 VPC의 프라이빗 서브넷을 `[cloud-provider]` 섹션에 지정합니다.

```ini
[cloud-provider]
aws-region = us-east-1
aws-vpc = vpc-xxxxxxxx
aws-subnet = subnet-xxxxxxxx
aws-security-group = sg-xxxxxxxx
```

이때 워커 노드는 다음 두 곳에 나가는 경로가 있어야 합니다.

- `s3://ncbi-blast-databases` (DB 다운로드)와 결과 버킷: S3 게이트웨이 엔드포인트로 충분
- 컨테이너 이미지 `public.ecr.aws/ncbi-elasticblast/*`: 코드에 고정된 퍼블릭 ECR 주소이므로 **NAT 게이트웨이** 또는 ECR 퍼블릭용 인터페이스 엔드포인트가 필요

### IAM 역할 생성이 막힌 계정

ElasticBLAST는 기본적으로 CloudFormation으로 IAM 역할을 직접 만듭니다. IAM 생성 권한이 없는 계정에서는 미리 만들어 둔 역할의 ARN을 다음 키로 주입할 수 있습니다.

```ini
[cloud-provider]
aws-batch-service-role = arn:aws:iam::123456789123:role/...
aws-instance-role = arn:aws:iam::123456789123:role/...
aws-job-role = arn:aws:iam::123456789123:role/...
aws-spot-fleet-role = arn:aws:iam::123456789123:role/...
aws-auto-shutdown-role = arn:aws:iam::123456789123:role/...
aws-janitor-execution-role = arn:aws:iam::123456789123:role/...
aws-janitor-copy-zips-role = arn:aws:iam::123456789123:role/...
```

각 역할에 필요한 권한은 ElasticBLAST 저장소의 CloudFormation 템플릿(`src/elastic_blast/templates/elastic-blast-cf.yaml`)과 `bin/aws-create-elastic-blast-janitor-role.sh`를 기준으로 작성하십시오.

---

## 문제 해결

### 일반적인 문제

#### 1. "SUBMITTING" 상태에서 멈춤

**원인**: AWS Batch 환경 생성 중 또는 IAM 권한 부족

**해결 방법**:
```bash
# CloudFormation 스택 상태 확인
aws cloudformation describe-stacks \
  --query "Stacks[?contains(StackName, 'elasticblast')].{Name:StackName,Status:StackStatus}"

# CloudWatch Logs 확인
aws logs describe-log-groups --log-group-name-prefix /aws/batch
```

#### 2. 권한 오류 (AccessDenied)

**원인**: IAM 권한 부족

**해결 방법**:
- AWS Batch, EC2, S3, CloudFormation, IAM 권한 확인
- 관리자에게 필요한 권한 요청

#### 3. 인스턴스 제한 초과

**원인**: EC2 인스턴스 vCPU 제한 초과

**해결 방법**:
```bash
# 현재 제한 확인
aws service-quotas get-service-quota \
  --service-code ec2 \
  --quota-code L-1216C47A
```

#### 4. 결과 파일이 없음

**원인**: 작업 실패 또는 아직 진행 중

**해결 방법**:
```bash
# 상태 확인
elastic-blast status --cfg elasticblast-config.ini

# S3 결과 확인
aws s3 ls s3://elasticblast/results/nt-test/
```

### 로그 확인

```bash
# ElasticBLAST 로그 파일
cat elastic-blast.log

# AWS Batch 작업 로그 (AWS 콘솔)
# CloudWatch Logs → Log groups → /aws/batch/job
```

---

## 추가 정보

### ElasticBLAST 공식 문서
- 개요: https://blast.ncbi.nlm.nih.gov/doc/elastic-blast/overview.html
- AWS 퀵스타트: https://blast.ncbi.nlm.nih.gov/doc/elastic-blast/quickstart-aws.html
- GitHub: https://github.com/ncbi/elastic-blast

### NCBI BLAST 데이터베이스
- FTP 사이트: https://ftp.ncbi.nlm.nih.gov/blast/db/
- 데이터베이스 문서: https://ftp.ncbi.nlm.nih.gov/blast/documents/blastdb.html

### AWS 서비스 문서
- AWS Batch: https://docs.aws.amazon.com/batch/
- Amazon S3: https://docs.aws.amazon.com/s3/
- Amazon EC2: https://docs.aws.amazon.com/ec2/

---

## 실행 체크리스트

### 실행 전
- [ ] AWS CLI 설치 및 구성 완료
- [ ] IAM 권한 확인 (Batch, EC2, S3, CloudFormation, IAM)
- [ ] janitor 역할 생성 (`bin/aws-create-elastic-blast-janitor-role.sh`, 계정당 1회)
- [ ] S3 버킷 생성 (`s3://elasticblast`)
- [ ] 쿼리 파일 S3 업로드
- [ ] ElasticBLAST 설치 (버전 1.5.0, Python 3.11 이상)
- [ ] 설정 파일 작성 (`elasticblast-config.ini`)
- [ ] 현재 DB 크기에 맞는 `machine-type`·`db` 확인 ([인스턴스 타입 선택 가이드](#인스턴스-타입-선택-가이드))
- [ ] 필요 시 `export BLAST_USAGE_REPORT=false`, EBS 기본 암호화, 프라이빗 서브넷 설정 ([보안](#보안))

### 실행 중
- [ ] DB 버전·BLAST+ 버전 기록 (`db-version.txt`)
- [ ] `elastic-blast submit` 실행
- [ ] 상태 주기적 확인 (`elastic-blast status`)
- [ ] CloudWatch Logs 모니터링 (선택사항)

### 실행 후
- [ ] 결과 다운로드 (S3 → 로컬)
- [ ] 결과 압축 해제 및 확인
- [ ] 실행 요약 저장 (`elastic-blast run-summary -o run-summary.json`) -- delete 전
- [ ] **리소스 정리** (`elastic-blast delete`) ⚠️ 중요!
- [ ] EC2 인스턴스 삭제 확인
- [ ] S3 버킷 정리 (선택사항)

---

### 기술 지원
- ElasticBLAST 이슈: https://github.com/ncbi/elastic-blast/issues
- NCBI BLAST 지원: blast-help@ncbi.nlm.nih.gov

---

## 변경 이력

- 2026-10-09: 사전 요구사항 정정 및 절차 보완
  - Python 요구 버전 3.7 → **3.11 이상** (3.9/3.10에서는 `elastic-blast --version`이 ImportError로 실패)
  - 인스턴스 타입 선택 가이드를 현재 nt 크기(1,117GiB, 캐시 1,089GiB, 요구 메모리 1,151GiB) 기준으로 재작성. 768GB급은 캐시 불가 → x1.32xlarge 또는 core_nt. 자동 선택 후보(m5ad/c5ad/r5ad)와 미지정 시 오류 문구 수록
  - 5.1 DB 버전·BLAST+ 버전 기록 단계, 6.5 `run-summary` 저장 단계(delete 전), 1.3 janitor 역할 생성 추가
  - 보안 절 신설 (`BLAST_USAGE_REPORT`, EBS 기본 암호화, 프라이빗 서브넷, 사전 생성 IAM 역할 주입)
- 2026-10-08: 실행 시간·비용 정정
  - "약 10-11시간, $215-240" → **24분, $11.28** (Cost Explorer·AWS Batch 잡 로그 실측). 이전 수치는 잡 25개의 실행시간 합계를 전체 소요시간으로 읽고 노드 수를 다시 곱한 집계 오류
  - 인스턴스 단가표를 실제 us-east-1 온디맨드 단가로 교체(r5d.24xlarge $6.912, r5ad.24xlarge $6.288)
  - 자동 선택(r5ad.24xlarge) vs 명시(r5d.24xlarge) 실측 비교 추가: 36분/$15.24 vs 24분/$11.28
  - NT DB 크기 증가 주의 문구 추가(2026-09 기준 1,117GiB)
  - 2025-12-12 항목의 "r5d.12xlarge 지정"은 오기 → r5d.24xlarge
- 2025-12-12: 초기 문서 작성
  - AWS us-east-1 리전 사용
  - NCBI 호스팅 NT 데이터베이스 사용
  - 500개 쿼리, 4개 노드 설정
  - 인스턴스 타입 명시 (r5d.24xlarge 지정, 자동 선택 방지)
  - 비용 예상 업데이트 (실제 사용 데이터 기반)
