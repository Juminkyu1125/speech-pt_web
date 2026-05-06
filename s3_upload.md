# S3 PDF/WAV Upload Flow

이 문서는 로컬 FastAPI 서버에서 PDF 파일과 WAV 파일을 S3에 업로드하는 과정과 원리를 정리한다.

## 전체 구조

업로드는 브라우저가 AWS 자격증명을 직접 갖지 않고, FastAPI 서버가 S3 presigned URL을 발급해주는 방식으로 동작한다.

1. 프론트엔드가 FastAPI에 업로드 URL 생성을 요청한다.
2. FastAPI가 로그인 사용자와 파일 정보를 확인하고 DB에 `pending` 업로드 레코드를 만든다.
3. FastAPI가 AWS 자격증명으로 S3 presigned `PUT` URL을 생성한다.
4. 프론트엔드가 파일 본문을 presigned URL로 직접 `PUT`한다.
5. 프론트엔드가 FastAPI에 업로드 완료를 알린다.
6. FastAPI가 DB 업로드 상태를 `uploaded`로 변경한다.

## 필요한 설정

FastAPI 서버 쪽에 AWS 자격증명이 필요하다. 프론트엔드에는 AWS 키를 넣지 않는다.

로컬 AWS CLI 설정 예시:

```bash
aws configure
```

필요한 값:

```text
AWS Access Key ID
AWS Secret Access Key
Default region name: ap-northeast-2
Default output format: json
```

프로젝트 `.env` 예시:

```env
UPLOAD_STORAGE=s3
S3_BUCKET=speechpt-upload
AWS_REGION=ap-northeast-2
```

서버는 `boto3`가 설치된 가상환경으로 실행해야 한다.

```bash
cd backend
UPLOAD_STORAGE=s3 venv/bin/python -m uvicorn app.main:app --host 127.0.0.1 --port 8000
```

## S3 CORS

브라우저가 presigned URL로 S3에 직접 `PUT`하려면 S3 버킷 CORS 설정이 필요하다.

개발 테스트용 예시:

```json
{
  "CORSRules": [
    {
      "AllowedHeaders": ["*"],
      "AllowedMethods": ["PUT", "GET", "HEAD", "POST"],
      "AllowedOrigins": ["*"],
      "ExposeHeaders": ["ETag", "x-amz-request-id", "x-amz-id-2"],
      "MaxAgeSeconds": 3000
    }
  ]
}
```

운영 환경에서는 `AllowedOrigins`를 실제 프론트엔드 도메인으로 제한해야 한다.

## 백엔드 Presign 단계

프론트엔드는 `/uploads/presign`으로 파일 정보를 보낸다.

요청 예시:

```json
{
  "note_id": "c4473457-9033-41ca-963c-69327eddad05",
  "kind": "document",
  "file_name": "example.pdf",
  "content_type": "application/pdf",
  "size_bytes": 93467
}
```

WAV 파일은 `kind`를 `audio`로 보내고 `content_type`은 보통 `audio/wav`가 된다.

FastAPI는 다음을 수행한다.

- 파일 확장자와 MIME 타입 검증
- `uploads` 테이블에 `pending` 상태 레코드 생성
- S3 object key 생성
- S3 presigned `PUT` URL 생성

S3 object key 예시:

```text
notes/{note_id}/uploads/{upload_id}/{file_name}
```

응답 예시:

```json
{
  "upload_id": "2f18325e-61f3-4178-876d-cc4620d33187",
  "method": "PUT",
  "upload_url": "https://s3.ap-northeast-2.amazonaws.com/speechpt-upload/...",
  "object_key": "notes/.../example.pdf",
  "expires_in_sec": 900
}
```

## Presigned URL 생성 원리

presigned URL은 제한된 시간 동안 특정 S3 객체에 특정 HTTP 메서드를 실행할 수 있게 서명된 URL이다.

이 프로젝트에서는 `put_object`에 대해 presigned URL을 만든다.

중요한 점:

- FastAPI 서버가 AWS 자격증명으로 URL을 서명한다.
- 프론트엔드는 AWS 키 없이 URL에 파일을 `PUT`한다.
- URL에는 만료 시간이 있다.
- `Content-Type`을 presign 생성 시 넣었다면, 실제 `PUT` 요청에서도 같은 `Content-Type`을 보내야 한다.

## 리전 엔드포인트 주의

이번 업로드 실패의 핵심 원인은 presigned URL이 글로벌 S3 엔드포인트로 생성된 것이었다.

문제가 된 형태:

```text
https://speechpt-upload.s3.amazonaws.com/...
```

이 URL은 `ap-northeast-2` 버킷에 대해 `307 Temporary Redirect`를 일으킬 수 있다. 브라우저의 presigned `PUT`은 이 리다이렉트 과정에서 실패할 수 있다.

수정된 형태:

```text
https://s3.ap-northeast-2.amazonaws.com/speechpt-upload/...
```

백엔드에서는 S3 클라이언트를 만들 때 리전 엔드포인트와 SigV4를 명시한다.

```python
from botocore.config import Config

s3_client = boto3.session.Session().client(
    "s3",
    region_name=S3_REGION,
    endpoint_url=f"https://s3.{S3_REGION}.amazonaws.com",
    config=Config(signature_version="s3v4"),
)
```

## 프론트엔드 PUT 단계

프론트엔드는 presign 응답의 `upload_url`에 파일을 직접 업로드한다.

```javascript
const uploadResponse = await fetch(uploadUrl, {
  method: "PUT",
  headers: {
    "Content-Type": file.type || "application/octet-stream",
  },
  body: file,
});
```

S3 presigned URL에는 FastAPI 로그인 토큰을 붙이지 않는다. `Authorization: Bearer ...` 헤더가 S3로 가면 서명 검증과 충돌할 수 있다.

반대로 로컬 FastAPI 업로드 fallback URL에는 로그인 토큰이 필요하므로 `authFetch`를 사용한다.

## Complete 단계

S3 `PUT`이 성공하면 프론트엔드는 `/uploads/complete`를 호출한다.

요청 예시:

```json
{
  "upload_id": "2f18325e-61f3-4178-876d-cc4620d33187",
  "checksum": null
}
```

FastAPI는 해당 업로드 레코드를 찾아 `status`를 `uploaded`로 변경한다.

응답 예시:

```json
{
  "upload_id": "2f18325e-61f3-4178-876d-cc4620d33187",
  "status": "uploaded"
}
```

PDF와 WAV 파일이 모두 `uploaded` 상태가 되면 분석 생성 요청으로 이어진다.

## 검증 방법

서버 상태 확인:

```bash
curl -s http://127.0.0.1:8000/healthz
```

AWS CLI 자격증명 확인:

```bash
aws sts get-caller-identity
```

버킷 접근 확인:

```bash
aws s3api head-bucket --bucket speechpt-upload
```

S3 객체 업로드 여부 확인:

```bash
aws s3api head-object --bucket speechpt-upload --key "{object_key}"
```

성공하면 `ContentLength`, `ContentType`, `ETag`, `LastModified` 등이 출력된다.

## 대표 실패 원인

### `/uploads/presign`이 500

가능한 원인:

- 서버를 `boto3`가 설치되지 않은 Python으로 실행함
- AWS 자격증명이 없음
- `S3_BUCKET`, `AWS_REGION` 설정이 잘못됨
- IAM 사용자에게 `s3:PutObject` 권한이 없음

확인:

```bash
backend/venv/bin/python -c "import boto3, botocore; print('ok')"
aws sts get-caller-identity
```

### S3 PUT이 브라우저에서 실패

가능한 원인:

- S3 CORS 미설정
- 프론트 origin이 CORS `AllowedOrigins`에 없음
- presigned URL이 글로벌 엔드포인트로 생성되어 `307 Temporary Redirect` 발생
- presign 생성 시의 `Content-Type`과 실제 `PUT` 요청의 `Content-Type`이 다름
- S3 요청에 불필요한 `Authorization` 헤더를 붙임

### DB에는 레코드가 있는데 파일이 없음

가능한 원인:

- `/uploads/presign` 후 S3 `PUT` 실패
- `/uploads/complete`가 S3 PUT 성공 여부와 무관하게 잘못 호출됨
- 프론트가 실패를 숨기고 일반 오류 메시지만 표시함

현재 프론트는 S3 전송 실패 시 실제 에러 메시지를 표시하도록 개선되어 있다.

## 현재 확인된 성공 흐름

실제 테스트에서 다음 흐름이 성공했다.

- PDF `/uploads/presign`: `200 OK`
- PDF S3 `PUT`: 성공
- PDF `/uploads/complete`: `200 OK`
- WAV `/uploads/presign`: `200 OK`
- WAV S3 `PUT`: 성공
- WAV `/uploads/complete`: `200 OK`
- 분석 생성: `201 Created`

S3 `head-object`로 확인된 파일:

- PDF: `ContentType application/pdf`, `ContentLength 93467`
- WAV: `ContentType audio/wav`, `ContentLength 1064526`
