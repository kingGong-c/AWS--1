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
## 📝 프로젝트 개요 (Overview)
본 프로젝트는 AWS 서버리스 아키텍처와 AI 비전 모델을 활용하여, 클라우드 저장소(S3)에 업로드되는 이미지를 실시간으로 분석하고 유해 콘텐츠를 자동으로 차단하는 **지능형 보안 검역 시스템**입니다.

## 🌟 핵심 기능 (Key Features)
* **실시간 유해물 탐지**: Amazon Rekognition을 활용하여 부적절한 이미지(폭력, 무기, 노출 등)를 업로드 즉시 식별합니다.
* **서버리스 자동 방어**: AWS Lambda를 통해 유해물 판정 시 즉시 파일을 영구 삭제하여 저장소 오염을 방지합니다.
* **지능형 보안 리포트**: AI 신뢰도에 따라 위험 등급(CRITICAL/HIGH)을 분류하고, S3 이벤트에서 업로더의 IP 주소를 추적하여 Amazon SNS로 관리자에게 긴급 알림을 전송합니다.
* **안전 파일 계층화**: 검역을 무사히 통과한 안전한 파일만 별도의 Clean 버킷(clean-zone)으로 안전하게 이동시킵니다.

## 🛠️ 기술 스택 (Tech Stack)
* **Cloud Infrastructure**: AWS S3, AWS Lambda, Amazon SNS, AWS IAM
* **AI / ML**: Amazon Rekognition
* **Language**: Python 3.12 (Boto3)

## 💡 트러블슈팅 및 보안 최적화 (Troubleshooting)
**1. 검역 신뢰도(Confidence) 민감도 튜닝**
초기 설정값(`MinConfidence=70`)에서는 배경이 복잡한 무기 사진 등이 통과되는 취약점을 발견했습니다. 보안성을 극대화하기 위해 `MinConfidence` 수치를 대폭 하향 조정하여, 시스템이 미세한 위협 요소라도 감지할 경우 즉시 격리 조치하도록 검역 기준을 강화했습니다.

**2. 관리자 가시성 확보 (IP Tracking)**
단순히 파일을 삭제하는 것을 넘어, 보안 사고 발생 시 사후 추적이 가능하도록 S3 Event 레코드 파라미터에서 `sourceIPAddress`를 추출하는 로직을 Lambda에 추가했습니다. 이를 통해 알림 메일에 공격 의심 IP를 포함시켜 실무 수준의 대응력을 갖추었습니다.
## 📸 실행 결과 (Demo)
* **AI 보안 검역 시스템 작동 및 관리자 알림 메일 수신 화면**

<img width="1565" height="855" alt="사진진" src="https://github.com/user-attachments/assets/937d8ab2-5157-4723-963a-a667532acaa9" />

## 💻 핵심 비즈니스 로직 (Lambda Code)
이 시스템의 두뇌 역할을 하는 AWS Lambda의 실제 파이썬(Python) 코드입니다. AI 비전 분석, 알림 전송, S3 객체 이동 및 삭제를 한 번에 처리합니다.

```python
import boto3
import urllib.parse

def lambda_handler(event, context):
    s3 = boto3.client('s3')
    rekognition = boto3.client('rekognition')
    sns = boto3.client('sns')
    
    dest_bucket = 'gongsumin-clean-zone' 
    sns_arn = '본인의_SNS_ARN' # 보안을 위해 실제 ARN은 마스킹 처리
    
    source_bucket = event['Records'][0]['s3']['bucket']['name']
    object_key = urllib.parse.unquote_plus(event['Records'][0]['s3']['object']['key'], encoding='utf-8')
    source_ip = event['Records'][0].get('requestParameters', {}).get('sourceIPAddress', 'Unknown IP')

    try:
        # AI 유해 콘텐츠 분석 (신뢰도 10% 이상 시 엄격 차단)
        response = rekognition.detect_moderation_labels(
            Image={'S3Object': {'Bucket': source_bucket, 'Name': object_key}},
            MinConfidence=10 
        )
        
        labels = response['ModerationLabels']
        
        if labels:
            risk_level = "🚨 CRITICAL" if labels[0]['Confidence'] >= 80 else "⚠️ HIGH"
            reasons = ", ".join([f"{l['Name']}({l['Confidence']:.1f}%)" for l in labels])
            
            message = f"[탐지 요약]\n위험 등급: {risk_level}\n탐지 사유: {reasons}\nIP 주소: {source_ip}"
            
            # 관리자 메일 발송 및 파일 영구 삭제
            sns.publish(TopicArn=sns_arn, Message=message, Subject="[긴급 보안 알림] 유해 콘텐츠 차단")
            s3.delete_object(Bucket=source_bucket, Key=object_key)
            
        else:
            # 검역 통과 파일: Clean Zone으로 이동 후 원본 삭제
            copy_source = {'Bucket': source_bucket, 'Key': object_key}
            s3.copy_object(CopySource=copy_source, Bucket=dest_bucket, Key=object_key)
            s3.delete_object(Bucket=source_bucket, Key=object_key)
            
        return {"statusCode": 200}
        
    except Exception as e:
        print(f"Error: {str(e)}")
        raise e
```

## 🏃‍♂️ 인프라 구축 과정 (How to Build)
단순히 코드를 짜는 것을 넘어, AWS 클라우드 환경에서 서비스를 직접 연결하고 권한을 제어하며 구축했습니다.

1. **IAM Role(역할) 및 최소 권한 부여**: Lambda 함수가 다른 AWS 서비스를 조작할 수 있도록 `AmazonS3FullAccess`, `AmazonRekognitionFullAccess`, `AmazonSNSFullAccess` 정책을 연결한 전용 실행 역할을 생성했습니다.
2. **S3 버킷 스토리지 분리**: 악성 파일이 무방비로 올라오는 `upload-zone`과, 검역을 통과한 무해한 파일만 저장되는 `clean-zone`으로 물리적인 저장 공간을 분리하여 보안성을 높였습니다.
3. **Lambda Event Trigger 설정**: 사용자가 파일을 올리는 즉시 검사가 시작되도록, S3 `upload-zone`의 객체 생성(ObjectCreated) 이벤트를 Lambda의 트리거로 연동했습니다.
4. **SNS 알림망 구축 및 구독**: 관리자 이메일을 엔드포인트로 하는 SNS Topic을 생성하고, 구독 승인(Confirm) 과정을 거쳐 실시간 경고 파이프라인을 완성했습니다.
