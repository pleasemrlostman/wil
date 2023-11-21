# S3 Cloudfront github actions 배포 과정

## ✋ 개요

기존에 진행했던 EC2 + Ngnix 배포는 실무에서는 잘 쓰이지 않는다는 얘기를 전달 들었다. (실제로는 ECS를 더 애용)<br>
_(물론 EC2에 SSH로 접근하여 Ngnix를 이용해서 웹 서버를 구현해본 경험은 값진 경험이였다.)_<br>

프론트서버 특히 리액트 같이 정적인 파일을 제공해주는 웹서버는 EC2 보다는 S3를 이용해서 주로 배포한다는 사실을 알았고 이번에 직접 S3를 생성후 Cloundfront를 이용해서 배포 해보고 github actions를 이용해서 간단한 CI/CD를 구축한 과정을 작성해보도록 하겠다.

## 📋 과정

### 1️⃣ S3(Amazon Simple Storage Service) 버킷 생성

우선 S3를 생성해야 한다. 콘솔에서 S3를 검색한 뒤 S3 페이지로 이동해주도록 하자<br><br>
<img src="./image/s3-1.png" width="50%" title="s3"/>

S3 페이지로 이동후 버킷 만들기를 클릭하여 버킷을 만들어 주도록 하자<br><br>
<img src="./image/s3-2.png" width="50%" title="s3"/>

버킷을 만들 때 주의할 점은 해당 버킷은 전역적으로 고유해야 한다<br>
즉 중복되는 버킷 이름은 작성 할 수가 없다. 해당 배포과정을 진행 할 때 개인 계정에서만 중복되지 않으면 괜찮다고 생각했다<br> 
하지만 같이 스터디를 진행하던 인원과 버킷 이름이 동일하자 생성이 안되는 문제를 겪었다. 그러므로 버킷 이름은 꼭 고유하게 작성해주도록 하자<br>
aws리전은 가장 가까운 ap-northeast-2로 설정해주자<br><br>
<img src="./image/s3-3.png" width="50%" title="s3"/>

그 다음은 객체 소유권을 설정해줘야한다. 스터디를 진행 할 때 첫 고비였다<br>
객체 소유권이란 쉽게 말하면 버킷에 업로드된 객체 즉 업로드 된 파일을 소유한 사람과 결과적으로 해당 객체를 관리하고 액세스 할 권한이 있는 사람을 결정하는 부분이다. ACL을 활성화 하면 객체 소유권을 버킷 소유자 선호와 객체 라이터로 구분 할 수 있다.<br>

> -  버킷 소유자 선호: 해당 설정은 S3 버킷에 업로드되는 객체의 소유권을 결정한다. 해당 설정이 활성화되면 S3 버킷을 소유한 AWS 계정도 해당 버킷에 업로드 되는 모든 객체의 소유자가 된다. 즉 해당 버킷에 업로드 된 객체를 삭제하고 액세서를 제어하는 기능을 포함해 버킷 내에 저장된 객체를 완전히 제어할 수 있다. 그러나 버킷 소유자 기본 설정이 비활성화 되면 객체 소유자는 객체를 업로드한 AWS 계정에 의해 결정되며 버킷 소유자와 동일하지 않을 수 있다.
> - 객체 라이터: 객체 작성자는 객체를 S3 버킷에 업로드하는 AWS 계정 또는 IAM 사용자이다 객체 라이터는 업로드 작업을 수행 할 때 사용 되는 자격 증명에 따라 결정되며 AWS계정, IAM 사용자 또는 익명 사용자 일 수 있다. 객체 작성자는 수정 또는 삭제 기능을 포함해 객체에 대한 모든 권항을 가진다.

<br>

> 이 두 설정의 차이는 누가 버킷에 업로드 된 객체의 소유권을 가지느냐이다 전자의 경우 모든 객체의 버킷 소유권을 부여하는 반면 후자는 업로드한 계정이 해당 객체의 소유권을 가질 수 있다. ACL 설정을 하지 않으면 객체에 대한 액세스는 버킷 정책에 의해서 결정된다. ACL 설정이 활성화되면 객체 소유자는 버킷 정책을 재정의하여 특정 사용자 또는 사용자 그룹에 대한 권한을 설정할 수 있다.<br>

해당 내용을 공부하기 전 그러니까 현재 설정된 값은 ACL을 활성화 해두었는데 특별한 사유가 없으니 ACL을 비활성화로 변경해야겠다.
<br><br>
<img src="./image/s3-4.png" width="50%" title="s3"/>

해당 S3는 웹서버처럼 이용하기 위해 외부의 접근을 허용해야한다. 그러므로 모든 퍼블릭 엑세스를 허용하고 아래 생성되는 경고창을 체크해주도록하자
<br><br>
<img src="./image/s3-5.png" width="50%" title="s3"/>

이후 버킷 버전관리, 태그는 이번 과정에선 필요가 없으니 패스해주고 암호화는 Amazon S3 관리형 키를 그대로 선택해주고 고급설정은 사용할 일이 없으니 비활성화하고 버킷을 만들어 주도록 하자
<br><br>
<img src="./image/s3-6.png" width="50%" title="s3"/>

---

### 2️⃣ S3(Amazon Simple Storage Service) 버킷 설정 변경 및 정책 설정

버킷이 잘 생성됐다. 이제 해당 버킷을 선택해서 액세서를 퍼블릭으로 설정해주도록 하자 (현재 이미지의 버킷은 이미 퍼블릭으로 수정해둔 상태이다.)
<br><br>
<img src="./image/s3-7.png" width="50%" title="s3"/>

버킷을 클릭 후 속성탭으로 이동해서 가장아래의 정적 웹 사이트 호스팅 섹션으로 이동하여 편집을 눌러주도록 하자
<br><br>
<img src="./image/s3-8.png" width="50%" title="s3"/>

이후 정적 웹 사이트 호스팅 활성화를 체크해주고 호스팅 유형도 정적 웹 사이트 호스팅으로 체크해주도록 하자 또한 인덱스 문서와 오류 문서를 index.html 으로 작성해주도록 하자 (이번 스터디에서는 React를 배포할 예정이므로 두 개 전부 React를 빌드할 때 나오는 html 파일인 index.html로 작성해 주도록 하자)
<br><br>
<img src="./image/s3-9.png" width="50%" title="s3"/>

그 다음 권한탭으로 이동해서 버킷 정책을 설정해주도록 하자 정책은 JSON 타입처럼 작성해주는데 내가 직접 작성하는게 아니라 정책생성기에서 정책을 고른 뒤 생성하면 JSON 파일 처럼 작성된다. 이번 배포에서 정책 생성은 아래 처럼 선택해주면 된다.<br><br>
<img src="./image/s3-10.png" width="50%" title="s3"/>
<img src="./image/s3-11.png" width="50%" title="s3"/>

- Select Type of Policy: S3 Bucket Policy
- Effect: Allow 
- Principal: * 
- Actions: GetObject 선택 (버킷 접근 권한)
- Amazon Resource Name(ARN): '버킷 정책 편집 페이지'의 버킷 ARN을 복사한 후, 뒷부분에 /*을 붙여서 입력.

<img src="./image/s3-12.png" width="50%" title="s3"/>

---

### 2️⃣ S3(Amazon Simple Storage Service)에 파일 업로드 하기

S3 세팅은 일단락 했다. 이제 실제 빌드된 파일을 업로드해야 한다.<br>
물론 AWS GUI를 이용해서 업로드 하는 방법도 있지만 조금 더 개발자스럽게 AWS CLI를 이용하기로 했다.<br>
AWS CLI 설치 방법은 AWS에서 잘 설명해주고 있다 [링크](https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/getting-started-install.html)<br>
본인은 mac을 이용해서 스터디를 진행했기 때문에 
```shell
$ curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
$ sudo installer -pkg AWSCLIV2.pkg -target /
```
해당 명령어를 통해 설치 후
```shell
$ which aws
/usr/local/bin/aws 
$ aws --version
aws-cli/2.10.0 Python/3.11.2 Darwin/18.7.0 botocore/2.4.5
```
해당 명령어를 통해 잘 설치 됐는지 확인했다. (설치가 어려우면 상단 링크를 참조하자)

### 3️⃣ IAM 계정 설정

AWS CLI를 이용해서 업로드 하기 위해서는 설치 후 내 컴퓨터에 AWS configure 설정을 해줘야 한다.<br>
해당 설정을 위해서는 IAM 계정을 만들어야 하므로 콘솔에 IAM을 검색하고 IAM 페이지로 이동하도록 하자<br><br>
<img src="./image/s3-13.png" width="50%" title="s3"/>
