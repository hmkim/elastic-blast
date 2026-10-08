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
- [문제 해결](#문제-해결)

---

## 개요

이 가이드는 AWS에서 ElasticBLAST를 사용하여 대규모 BLASTN 검색을 수행하는 방법을 설명합니다.

### 프로젝트 정보
- **목적**: 500개 unmapped reads에 대한 NCBI NT 데이터베이스 검색
- **BLAST 프로그램**: BLASTN (nucleotide-nucleotide search)
- **데이터베이스**: NCBI NT (~810GB, 압축 해제 기준)
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
- Python 3.7 이상
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

### 1.3 ElasticBLAST 저장소 클론 (선택사항)

```bash
git clone https://github.com/ncbi/elastic-blast.git
```

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
- `machine-type`: EC2 인스턴스 타입 (r5d.24xlarge = 768GB 메모리, NT DB 최소 요구사항)
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
- **총 크기**: 약 810GB (압축 해제 기준)

**장점:**
- 자동 다운로드 및 캐싱
- 데이터 전송 비용 없음 (동일 리전)
- 최신 버전 자동 사용

---

## 5단계: BLAST 실행

### 5.1 ElasticBLAST 제출

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

### 5.2 진행 상황 모니터링

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

### 5.3 실행 과정

ElasticBLAST는 다음 단계를 자동으로 수행합니다:

1. **AWS Batch 환경 생성** (~5분)
   - Compute Environment 생성
   - Job Queue 생성
   - Job Definition 생성

2. **EC2 인스턴스 시작** (~2-3분)
   - 4개 워커 노드 시작
   - 인스턴스 타입: 설정 파일의 `machine-type` (비우면 자동 선택 → r5ad.24xlarge, 아래 비용 절 참고)

3. **NT 데이터베이스 다운로드** (~11-13분/노드, r5d.24xlarge 기준)
   - NCBI S3에서 자동 다운로드 (노드당 1회, 로컬 NVMe에 저장)
   - 같은 노드의 다른 배치는 다운로드가 끝날 때까지 대기

4. **쿼리 배치 분할 및 실행**
   - 쿼리를 여러 배치로 분할
   - 각 배치를 병렬 실행 (노드당 5개 배치 동시, 배치당 16스레드)

5. **결과 저장**
   - 각 배치 결과를 S3에 gzip 압축하여 저장
   - 파일명: `batch_NNN-blastn-nt.out.gz`

### 5.4 예상 소요 시간

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

---

## 7단계: 리소스 정리

### 7.1 ElasticBLAST 리소스 삭제

**중요:** 작업 완료 후 반드시 실행하여 불필요한 비용 발생을 방지해야 합니다.

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
   - 인스턴스 타입: r5d.24xlarge (수동 지정, NT DB 최소 요구사항)
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

us-east-1 Linux 온디맨드 단가 (2026-10 기준, 변동 가능):

| 인스턴스 타입 | 메모리 | 로컬 스토리지 | 시간당 비용 | NT DB 사용 가능 여부 |
|--------------|--------|-------------|------------|---------------------|
| r5d.12xlarge | 384 GB | 2 x 900GB NVMe | $3.456 | ✗ 메모리 부족 (681.6GB 필요) |
| r5d.16xlarge | 512 GB | 4 x 600GB NVMe | $4.608 | ✗ 메모리 부족 |
| r5d.24xlarge | 768 GB | 4 x 900GB NVMe | $6.912 | ✓ 사용 가능 (권장) |
| r5ad.24xlarge | 768 GB | 4 x 900GB NVMe | $6.288 | ✓ 사용 가능 (단가는 낮지만 느림, 아래 참고) |

**중요:** NT 데이터베이스는 최소 681.6GB 메모리가 필요합니다.
r5d.24xlarge 또는 r5ad.24xlarge만 사용 가능합니다.

> **주의 (2026-10):** NT DB는 계속 커집니다. 2025-12 실행 당시 nt는 2.9조 염기(디스크 715GB, 캐시 679GB)였지만, 2026-09 기준 4.66조 염기(약 1,117GiB)로 768GB 메모리에 더 이상 전부 들어가지 않습니다. 실행 전 `aws s3 cp s3://ncbi-blast-databases/$(aws s3 cp s3://ncbi-blast-databases/latest-dir -)/nt-nucl-metadata.json -` 로 `bytes-to-cache`를 확인하고, 768GB를 넘으면 `core_nt`(진핵생물 염색체 서열 제외, 약 250GiB) 또는 메모리 1.2TiB 이상 인스턴스를 사용하십시오.

**`machine-type`을 비우면 ElasticBLAST가 r5ad.24xlarge(AMD)를 자동 선택합니다.** 같은 500개 쿼리 작업에서 r5ad.24xlarge는 DB 다운로드가 1.8배(21분 vs 12분), BLAST 배치가 1.4배 느려 벽시계 36분, $15.24가 걸렸습니다(r5d.24xlarge: 24분, $11.28). 단가가 9% 낮아도 총비용과 시간 모두 불리하므로 `machine-type = r5d.24xlarge`를 명시하십시오.

### 비용 절감 팁

1. **인스턴스 타입 지정**: `machine-type = r5d.24xlarge` 명시적 지정 (자동 선택 방지)
2. **노드 수 조정**: 대량 쿼리가 아니면 4개 노드로 충분
3. **즉시 정리**: 작업 완료 후 즉시 `elastic-blast delete` 실행
4. **리전 선택**: us-east-1 사용 (NCBI 데이터베이스와 동일 리전)

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
- [ ] S3 버킷 생성 (`s3://elasticblast`)
- [ ] 쿼리 파일 S3 업로드
- [ ] ElasticBLAST 설치 (버전 1.5.0)
- [ ] 설정 파일 작성 (`elasticblast-config.ini`)

### 실행 중
- [ ] `elastic-blast submit` 실행
- [ ] 상태 주기적 확인 (`elastic-blast status`)
- [ ] CloudWatch Logs 모니터링 (선택사항)

### 실행 후
- [ ] 결과 다운로드 (S3 → 로컬)
- [ ] 결과 압축 해제 및 확인
- [ ] **리소스 정리** (`elastic-blast delete`) ⚠️ 중요!
- [ ] EC2 인스턴스 삭제 확인
- [ ] S3 버킷 정리 (선택사항)

---

### 기술 지원
- ElasticBLAST 이슈: https://github.com/ncbi/elastic-blast/issues
- NCBI BLAST 지원: blast-help@ncbi.nlm.nih.gov

---

## 변경 이력

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
