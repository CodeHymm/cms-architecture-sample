# Vector Search Overview  
본 문서는 CMS2에서 ECG Waveform 기반 패턴 탐색을 지원하기 위해 적용하는  
**ClickHouse Vector Index 구조 및 활용 전략**을 설명합니다.

Vector Search는 ECG 패턴의 Morphology(형태) 유사도를 비교하고  
부정맥 패턴 탐색을 지원하기 위해 설계되었습니다.

---

# 1. Overview

CMS2의 Trend ECG 데이터는 시간 구간별(1초 단위)로 분리되며,  
각 구간마다 Waveform 정보를 벡터(Embedding)로 변환하여 저장할 수 있습니다.

Vector Index를 사용하면 다음 기능들을 효율적으로 수행할 수 있습니다:

- 비정상 ECG 패턴의 유사 구간 탐색  
- 부정맥(Ectopy, PVC 등) Morphology 유사도 기반 검색  
- 특정 Lead(II, III) 형태 비교  
- 환자 간 ECG 패턴 유사도 기반 클러스터링  
- 긴 Recording 중 특정 형태와 가장 유사한 지점 자동 탐색  

---

# 2. Architecture

ECG Waveform → Embedding → Vector Index → Nearest Neighbor Search  
구조는 아래와 같습니다.

```
[ECG Waveform 250Hz] 
       ↓ Embedding Model (256 dims)
[Vector Embedding 256 floats]
       ↓
[ClickHouse Table (ecg_trend)]
       ↓
[Vector Index: HNSW]
       ↓
[cosineDistance 기반 유사 패턴 조회]
```

CMS2는 **On-Prem 병원 환경**에서도 운영 가능해야 하므로,  
ClickHouse의 내장 Vector Index(HNSW)를 활용합니다.

---

# 3. Data Model

아래는 ECG Vector Search에 사용되는 저장 모델 예시입니다.

```sql
CREATE TABLE ecg_trend
(
    patient_id String,
    timestamp  DateTime,
    trend_values Array(Float32),  -- ECG Trend (1-second)
    vector_embedding Array(Float32),  -- 256-dim embedding
)
ENGINE = MergeTree
ORDER BY (patient_id, timestamp);
```

Vector Embedding은 다음 원칙을 따릅니다:

- ECG 1초 구간을 입력으로 Embedding Model 적용  
- 256~512 차원의 float 벡터  
- Lead II, III 등 채널별 Embedding 가능  
- Float32 기반 저장으로 메모리 효율 최적화  

---

# 4. Vector Index (HNSW)

CMS2는 ClickHouse의 **HNSW (Hierarchical Navigable Small World)** 기반  
Vector Index를 사용합니다.

장점:
- 고속 Approximate Nearest Neighbor (ANN)
- Cosine, Euclidean Distance 지원
- CPU 기반에서도 빠른 검색
- GPU 필요 없음 (온프렘 병원 환경 최적화)
- 장기간 ECG 데이터에서도 빠른 Similarity Search 가능

Index 생성 예:

```sql
ALTER TABLE ecg_trend
ADD INDEX idx_vector vector_embedding TYPE vector_cosine(256) GRANULARITY 1;
```

---

# 5. Vector Search Query

임의의 ECG 벡터와 가장 유사한 구간 Top-10을 조회하는 예시입니다.

```sql
SELECT
    patient_id,
    timestamp,
    cosineDistance(vector_embedding, toFloat32(array( ... ))) AS dist
FROM ecg_trend
ORDER BY dist ASC
LIMIT 10;
```

검색 방식:
- cosineDistance 기반 유사도 계산  
- dist가 0 → 완전히 동일한 벡터  
- dist가 작을수록 ECG 형태가 유사함  

---

# 6. 활용 시나리오

## 6.1 부정맥 형태 유사도 탐색
PVC, PAC 등 특정 형태의 Beat Morphology를 벡터로 변환 후  
과거 기록에서 유사 패턴 검색 가능.

## 6.2 환자 간 ECG 형태 비교
Embedding 기반으로 각 환자의 ECG 패턴을 군집화하여  
비슷한 Morphology를 가진 환자 그룹을 탐색할 수 있음.

## 6.3 Historical ECG 중 특정 구간 자동 탐색
긴 모니터링 ECG에서 특정 패턴과 가장 유사한 지점을  
AI 없이 Vector Search로 빠르게 찾을 수 있음.

## 6.4 초고속 Trend 분석
OLAP 기반 Vector Search로 대규모 Trend 데이터도 빠르게 탐색 가능.

---

# 7. System Integration

Vector Search는 CMS2의 기존 OLAP 파이프라인과 자연스럽게 통합됩니다.

```
[Ingestor] 
      ↓ Avro
[Transformer]
      ↓
[Trend 1sec 생성]
      ↓
[ECG → Embedding 벡터 변환]
      ↓
[ClickHouse 저장 + Vector Index 업데이트]
      ↓
[API 서버 Vector Query 제공]
```

- Transformer 단계에서 Embedding 생성 가능  
- 또는 API 호출 시 실시간 생성도 가능(비추천)  
- 장비 데이터량이 많아도 OLAP 구조라 검색 속도 유지됨  

---

# 8. Advantages in Medical Systems

Vector Index 기반 ECG 검색은 기존 의료 시스템에서 거의 제공하지 않는 기능으로,  
CMS2 Platform의 차별화된 강점이 될 수 있습니다.

장점 요약:

- 실시간 또는 저장 데이터 기반 ECG 패턴 검색  
- 부정맥 진단 보조 기능 제공  
- 대규모 시계열 데이터에서도 고성능  
- Without Deep Learning Inference (비용 ↓, 속도 ↑)
- 온프레미스 환경에서도 충분히 운영 가능  

---

# 9. Related Documents

- [OLAP TTL Strategy](./olap-ttl-strategy.md)
- [OLAP Storage Overview](../storage/olap-overview.md)
- [Architecture Overview](../architecture-overview.md)

---

# Version
```
v1.0 – Initial Vector Search Overview Document
```
