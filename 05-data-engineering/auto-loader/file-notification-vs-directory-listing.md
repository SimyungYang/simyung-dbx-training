# 파일 감지 모드: Directory Listing vs File Notification

## 왜 파일 감지 모드를 이해해야 하는가?

Auto Loader가 클라우드 스토리지에서 새 파일을 발견하는 방식에는 **Directory Listing**과 **File Notification** 두 가지가 있습니다. 이 선택에 따라 **지연 시간, 비용, 확장성**이 크게 달라지므로, 프로덕션 환경에서는 워크로드 특성에 맞는 모드를 선택하는 것이 매우 중요합니다.

> 💡 **비유**: Directory Listing은 "5분마다 우편함을 직접 열어보는 것"이고, File Notification은 "우편이 도착하면 알림이 오는 스마트 우편함"입니다.

---

## Directory Listing 모드 (기본값)

### 동작 원리

Directory Listing 모드는 Auto Loader가 지정된 경로의 **파일 목록을 주기적으로 스캔**하여 새 파일을 감지합니다. 이전 스캔 결과와 비교하여 새로 추가된 파일만 처리합니다.

```python
# Directory Listing 모드 (기본값이므로 별도 설정 불필요)
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.useNotifications", "false")  # 기본값
    .option("cloudFiles.schemaLocation", "s3://bucket/schema/")
    .load("s3://bucket/data/")
)
```

### 장점과 단점

| 장점 | 단점 |
|------|------|
| 추가 인프라 설정이 불필요합니다 | 대규모 디렉토리에서 LIST API 호출 비용이 증가합니다 |
| IAM 권한이 간단합니다 (읽기 권한만 필요) | 파일 수가 많으면 스캔 시간이 길어집니다 |
| 즉시 시작할 수 있습니다 | 실시간에 가까운 처리가 어렵습니다 |

### Incremental Listing (증분 리스팅)

대규모 디렉토리에서 Directory Listing의 성능을 크게 개선하는 기능입니다. 매번 전체 디렉토리를 스캔하는 대신, **이전 스캔 이후 변경된 부분만** 확인합니다.

```python
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "parquet")
    .option("cloudFiles.useIncrementalListing", "true")
    .option("cloudFiles.schemaLocation", "s3://bucket/schema/")
    .load("s3://bucket/data/year=*/month=*/day=*/")
)
```

> ⚠️ **Incremental Listing의 전제 조건**: 파일이 **시간순으로 정렬된 디렉토리 구조**(예: `/year=2025/month=03/day=15/`)에 저장되어야 효과적입니다. 파일 이름이나 경로에 시간 정보가 없으면 효과가 제한적입니다.

| `useIncrementalListing` 값 | 동작 |
|----------------------------|------|
| `auto` (기본값) | Auto Loader가 디렉토리 구조를 분석하여 자동으로 결정합니다 |
| `true` | 증분 리스팅을 강제 활성화합니다 |
| `false` | 매번 전체 디렉토리를 스캔합니다 |

---

## File Notification 모드

### 동작 원리

File Notification 모드는 클라우드 스토리지의 **이벤트 알림 서비스**를 통해 새 파일 도착을 감지합니다. 파일이 업로드되면 이벤트가 발생하고, Auto Loader가 이를 수신하여 즉시 처리합니다.

![Auto Loader File Notification 아키텍처](https://docs.databricks.com/aws/en/_images/auto-loader-file-notification.png)

> 이미지 출처: [Databricks: File notification mode](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/file-detection-modes.html)

### 클라우드별 이벤트 알림 서비스

| 클라우드 | 이벤트 서비스 | 큐 서비스 | 자동 프로비저닝 |
|---------|-------------|----------|-------------|
| **AWS** | S3 Event Notifications → SNS | SQS | 지원 (IAM 권한 필요) |
| **Azure** | Azure Event Grid | Azure Queue Storage | 지원 (Contributor 역할 필요) |
| **GCP** | Google Cloud Pub/Sub | Pub/Sub Subscription | 지원 (IAM 권한 필요) |

### AWS에서의 설정 예제

```python
# File Notification 모드 — 자동 프로비저닝
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.useNotifications", "true")
    .option("cloudFiles.region", "ap-northeast-2")
    .option("cloudFiles.schemaLocation", "s3://bucket/schema/")
    .load("s3://bucket/data/")
)
```

자동 프로비저닝 시 Auto Loader가 생성하는 AWS 리소스는 다음과 같습니다.

| 리소스 | 설명 |
|--------|------|
| **SNS Topic** | S3 이벤트를 수신하는 토픽입니다 |
| **SQS Queue** | SNS에서 전달받은 이벤트를 저장하는 큐입니다 |
| **S3 Event Configuration** | S3 버킷에 이벤트 알림 규칙을 추가합니다 |

### 기존 SQS 큐 사용 (수동 설정)

이미 구성된 이벤트 인프라가 있다면 직접 지정할 수 있습니다.

```python
df = (spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.useNotifications", "true")
    .option("cloudFiles.queueUrl", "https://sqs.ap-northeast-2.amazonaws.com/123456789/my-queue")
    .option("cloudFiles.schemaLocation", "s3://bucket/schema/")
    .load("s3://bucket/data/")
)
```

### 필요한 IAM 권한

자동 프로비저닝을 사용하려면 다음 IAM 권한이 필요합니다.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetBucketNotification",
                "s3:PutBucketNotification",
                "sns:CreateTopic",
                "sns:DeleteTopic",
                "sns:GetTopicAttributes",
                "sns:Subscribe",
                "sqs:CreateQueue",
                "sqs:DeleteQueue",
                "sqs:GetQueueAttributes",
                "sqs:GetQueueUrl",
                "sqs:ReceiveMessage",
                "sqs:DeleteMessage",
                "sqs:SetQueueAttributes"
            ],
            "Resource": "*"
        }
    ]
}
```

---

## 모드 비교 총정리

| 비교 항목 | Directory Listing | File Notification |
|-----------|-------------------|-------------------|
| **감지 방식** | 주기적 디렉토리 스캔 | 이벤트 기반 수신 |
| **지연 시간** | 스캔 주기에 따라 다름 (분 단위) | 거의 실시간 (수 초) |
| **파일 수 확장성** | 수십만 파일부터 성능 저하 | 파일 수에 무관 |
| **비용** | LIST API 호출 비용 | 이벤트 서비스 비용 (매우 저렴) |
| **설정 복잡도** | 간단 (기본값) | 중간 (IAM 권한 필요) |
| **클라우드 권한** | 읽기(GET/LIST) 만 필요 | 이벤트 서비스 관리 권한 필요 |
| **리소스 정리** | 별도 정리 불필요 | 스트림 종료 시 리소스 정리 필요 |

---

## 선택 기준 가이드

| 시나리오 | 권장 모드 | 이유 |
|----------|----------|------|
| 파일 수 10만 개 미만, 빠른 시작 | **Directory Listing** | 설정 없이 즉시 사용 가능합니다 |
| 파일 수 10만 개 이상 | **File Notification** | LIST API 비용과 스캔 시간을 절감합니다 |
| 실시간에 가까운 처리 필요 | **File Notification** | 수 초 이내 감지가 가능합니다 |
| IAM 권한이 제한된 환경 | **Directory Listing** | 이벤트 서비스 권한 불필요 |
| 파티션 구조가 잘 정리된 디렉토리 | **Directory Listing + Incremental** | 증분 리스팅으로 성능이 충분합니다 |
| 여러 소스에서 동시에 파일 수집 | **File Notification** | 각 소스에서 이벤트를 독립적으로 수신합니다 |

---

## 리소스 정리 (File Notification 모드)

File Notification 모드에서 자동 프로비저닝된 리소스는 스트림 종료 시 **자동으로 정리되지 않습니다**. 더 이상 사용하지 않는 경우 수동으로 정리해야 합니다.

```python
# 자동 프로비저닝된 리소스 정리
result = spark.read.format("cloudFiles")  \
    .option("cloudFiles.format", "json")  \
    .option("cloudFiles.useNotifications", "true")  \
    .option("cloudFiles.resourceGroup", "auto-loader-rg")  \
    .load("s3://bucket/data/")

# cloudFiles.teardown 명령으로 정리
dbutils.fs.rm("s3://bucket/schema/_notification_resources", True)
```

> ⚠️ **정리하지 않으면 불필요한 클라우드 비용이 계속 발생할 수 있습니다.** 특히 SQS 큐에 메시지가 쌓이면서 비용이 누적됩니다.

---

## 정리

| 핵심 포인트 | 설명 |
|------------|------|
| **Directory Listing** | 기본 모드로, 설정 없이 즉시 사용할 수 있지만 대규모 디렉토리에서 성능이 저하됩니다 |
| **Incremental Listing** | Directory Listing의 성능을 개선하는 옵션으로, 파티션 구조가 있는 디렉토리에서 효과적입니다 |
| **File Notification** | 이벤트 기반으로 거의 실시간 감지가 가능하며, 대규모 디렉토리에 적합합니다 |
| **자동 프로비저닝** | File Notification 모드에서 클라우드 이벤트 인프라를 자동으로 생성합니다 |
| **리소스 정리** | File Notification의 자동 생성된 리소스는 수동 정리가 필요합니다 |

---

## 참고 링크

- [Databricks: File detection modes](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/file-detection-modes.html)
- [Databricks: File notification mode](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/file-notification-mode.html)
- [Databricks: Auto Loader options](https://docs.databricks.com/aws/en/ingestion/cloud-object-storage/auto-loader/options.html)
- [Azure Databricks: Auto Loader](https://learn.microsoft.com/en-us/azure/databricks/ingestion/cloud-object-storage/auto-loader/)
