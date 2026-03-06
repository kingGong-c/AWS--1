# AWS--1
하나하나 차근차근 aws 정복
```mermaid
flowchart LR
    User((👤 사용자)) -->|1. 이미지 업로드| S3_Upload[(S3 버킷\nupload-zone)]
    
    S3_Upload -->|2. S3 Event Trigger| Lambda[⚡ AWS Lambda\n콘텐츠 검역 엔진]
    
    Lambda <-->|3. AI 분석 요청 및 신뢰도 반환| Rekognition{🧠 Amazon Rekognition\nAI 비전 모델}
    
    Lambda -->|4-A. 유해 판정\nCRITICAL / HIGH| SNS{{✉️ Amazon SNS\n보안 알림 통로}}
    Lambda -->|4-B. 안전 판정\nClean| S3_Clean[(S3 버킷\nclean-zone)]
    
    SNS -->|5. IP 및 위험등급 보고| Admin((👨‍💻 보안 관리자))
    Lambda -->|6. 즉시 삭제 조치| Trash([🗑️ 원본 영구 삭제])

    classDef aws fill:#FF9900,stroke:#232F3E,stroke-width:2px,color:black;
    classDef ai fill:#00A4A6,stroke:#232F3E,stroke-width:2px,color:white;
    classDef storage fill:#3F8624,stroke:#232F3E,stroke-width:2px,color:white;
    
    class S3_Upload,S3_Clean storage;
    class Lambda aws;
    class Rekognition ai;
    class SNS aws;
```
