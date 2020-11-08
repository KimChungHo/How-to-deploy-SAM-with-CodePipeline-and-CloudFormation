목표 시나리오: CodeCommite에 있는 리포의 release 브랜치에 푸쉬를 하면 자동으로 빌드 후 SAM을 통해 CloudFormation으로 Lambda Application에 배포가 되는 것이다.

---

# S3 Bucket 생성

S3에 버킷을 생성한다.

1. 버킷 만들기 버튼을 누른다.

2. 버킷 설정을 하고 생성을 한다.

- 버킷 이름: netcore-app (버킷 이름은 알아서 정할 것)
- 기본 암호화: AES-256
- 모든 퍼블릭 액세스 차단: 체크 해제

---

# IAM Role 생성

CloudFormation 역할을 생성해야한다.

1. IAM의 역할 메뉴에서 역할 만들기를 누른다.

2. 역할 설정 후 만들기를 누른다.

- 서비스: CloudFormation
- 정책: AWSLambdaExecute
- 이름: cfn-lambda-pipeline

3. 방금 만든 역할을 연다.

4. 인라인 정책을 추가한다.

- 이름: cfn

```json
{
    "Statement": [
        {
            "Action": [
                "apigateway:*",
                "codedeploy:*",
                "lambda:*",
                "cloudformation:CreateChangeSet",
                "iam:GetRole",
                "iam:CreateRole",
                "iam:DeleteRole",
                "iam:PutRolePolicy",
                "iam:AttachRolePolicy",
                "iam:DeleteRolePolicy",
                "iam:DetachRolePolicy",
                "iam:PassRole",
                "s3:GetObject",
                "s3:GetObjectVersion",
                "s3:GetBucketVersioning"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ],
    "Version": "2012-10-17"
}
```

---

# CodeCommit 설정

1. CodeCommit 에서 리포지토리 생성을 누른다.

2. 리포지토리 설정을 한 뒤에 생성을 누른다.

- 이름: NetCore

3. 프로젝트에 리포지토리를 연결한다. 리포지토리 연결하는 부분에 대해서는 이 글에선 언급하지 않겠다.

4. 연결한 후에는 푸쉬를 한번 해준다.

---

# CodePipeline 설정

1. CodePipeline에서 파이프라인 생성을 누른다.

2. 파이프라인 설정을 한다.

- 파이프라인 이름: NetCore
- 아티팩트 스토어 위치: 기본위치로 두되, 필요시 버킷을 직접 설정

3. 나머지는 자동으로 하고 다음을 누른다.

4. 소스 설정을 한다.

- 소스 공급자: CodeCommit
- 리포지토리 이름: NetCore
- 브랜치 이름: release

5. 빌드 설정을 한다.

- 빌드 공급자: AWS CodeBuild

6. 프로젝트 생성을 누른다.

- 프로젝트 이름: lambda-pipeline-build
- 운영체제: Ubuntu
- 런타임: Standard
- 이미지: aws/codebuild/standard:4.0
- 이미지 버전: 항상 최신 이미지 사용

7. 다음을 누른다.

8. 배포를 건너뛴다.

---

# IAM Role 수정

1. IAM에서 'codebuild-lambda-pipeline-build-service-role' 역할을 누른다.

2. 정책 연결을 누른다.

3. AmazonS3FullAccess를 연결한다.

---

# CodePipeline 설정 마무리

1. 편집을 누른다.

2. 하단의 스테이지 추가를 누른다.

- 스테이지 이름: Deploy

3. 작업 그룹 추가를 누른다.

- 작업 이름: Stack_Create_or_Update
- 작업 공급자: AWS CloudFormation
- 입력 아티팩트: BuildArtifact
- 작업 모드: 스택 생성 또는 업데이트
- 스택 이름: netcore-app
- 템플릿 아티팩트 이름: BuildArtifact
- 파일 이름: outputtemplate.yml
- 기능: CAPABILITY_IAM, CAPABILITY_AUTO_EXPAND
- 역할 이름: cfn-lambda-pipeline

4. 스테이지 편집을 완료 버튼을 누른 후 파이프라인 편집을 저장한다.

5. 변경 사항 릴리스를 눌러 임시로 배포를 실행한다.

6. 다시 편집을 누르고 Deploy 스테이지에서 아래에 있는 작업 그룹 추가를 누른다.

- 작업 이름: Changeset_Create_or_Replace
- 작업 공급자: AWS CloudFormation
- 입력 아티팩트: BuildArtifact
- 작업 모드: 변경 세트 생성 또는 교체
- 스택 이름: netcore-app
- 변경 세트 이름: netcore-app-changeset
- 템플릿 아티팩트 이름: BuildArtifact
- 파일 이름: outputtemplate.yml
- 기능: CAPABILITY_IAM, CAPABILITY_AUTO_EXPAND
- 역할 이름: cfn-lambda-pipeline

7. 저장 후 릴리스를 눌러 다시 한번 임시로 배포를 실행한다.

8. 마지막으로 Deploy 스테이지에 작업 그룹을 다시 추가한다.

- 작업 이름: Changeset-Run
- 작업 공급자: AWS CloudFormation
- 입력 아티팩트: BuildArtifact
- 작업 모드: 변경 세트 실행
- 스택 이름: netcore-app
- 변경 세트 이름: netcore-app-changeset

9. 다시 한번 릴리스를 실행한다.