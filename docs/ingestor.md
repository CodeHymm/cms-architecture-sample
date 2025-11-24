# End-to-End Data Flow: Binary → Netty → Avro → MQ → Publish

의료 장비에서 생성된 RAW 바이너리 패킷은 아래 단계를 거쳐 CMS2 전체 시스템으로 전달됩니다.

---

## 1. Binary Packet (RAW Data)
장비는 ECG, SpO₂, NIBP, Alarm 등의 데이터를 **바이너리 형태(Byte Stream)**로 전송합니다.  
각 장비는 다음 구조를 가짐:

- Start Flag  
- Packet Length  
- Payload (ECG, Vital, Alarm 등)  
- CRC / Checksum  

이 RAW 데이터는 250Hz(ECG) 등 실시간 속도로 지속적으로 전송됨.

---

## 2. Netty TCP Server
CMS2 Ingestor는 **Netty 기반 Non-blocking TCP 서버**로 바이너리 패킷을 수신합니다.

처리 과정:

1. **ChannelInitializer** 설정  
2. **IdleStateHandler** 로 장비 연결 유지 관리  
3. **Packet Decoder** 로 Fragment 재조립  
4. **DevicePacketHandler** 가 완성된 패킷 전달

Netty는 ByteBuf 기반이므로 고속 처리가 가능하고 100대 이상의 장비 연결도 안정적으로 유지됨.

---

## 3. Packet Parsing → Domain Model
Netty로 완성된 패킷이 전달되면:

1. Header / Length / Checksum 검증  
2. Payload 해석  
3. ECG, Vital, Alarm 등의 Domain Model로 변환  
4. 단일 표준화된 구조로 매핑

예:  
```
EcgRawPacket → EcgParsedData
```

---

## 4. Avro Serialization & Schema Registry
파싱된 Domain Model은 CMS2의 **Avro Schema Registry** 기준으로 직렬화됩니다.

### 이 과정에서 이루어지는 작업:
- Schema Version 부여  
- Payload Normalization  
- Envelope 생성  
- Binary Avro Record 직렬화  

Envelope 구조 예:

```
{
  "schemaVersion": 3,
  "timestamp": 1711251235,
  "deviceId": "D700-001",
  "patientId": "P-xxxx",
  "payload": { ... }
}
```

Avro 사용 이유:
- 바이너리 직렬화로 성능 우수  
- Backward Compatible 스키마 제공  
- 장비 펌웨어 변경 시 안정적  

---

## 5. Publish to ActiveMQ Artemis (MQ)
Ingestor는 직렬화된 Avro 메시지를 MQ Topic으로 발행합니다.

Topic 예:
- `ecg.raw.avro`
- `spo2.raw.avro`
- `alarm.event.avro`
- `trend.1sec.avro`

전송 방식:
- Async Publish(비동기)  
- Failover 지원  
- Producer Acknowledgement  

MQ를 사용함으로써 Ingestor와 Transformer 간의 **분리(Decoupling)**가 이루어지고,  
소비자 장애가 있어도 데이터 손실 없이 운영 가능함.

---

## 6. Transformer Consumption (후속 단계)
MQ로 전달된 Avro 메시지는 Transformer에서 Consume하여:

- Avro → Domain Model 역직렬화  
- Trend Aggregation  
- OLAP 저장  
- 실시간 Event 처리  

이 단계 이후는 Transformer 문서에서 상세히 기술됨.

---

# Summary Diagram
```
[Device Binary] 
      ↓ TCP
[Netty (Packet Decoder)]
      ↓
[Domain Model Parsing]
      ↓
[Avro Serialization + Schema Registry]
      ↓
[ActiveMQ Artemis]
      ↓
[Publish to Topic]
```
