생성형 AI 애플리케이션을 본격적으로 개발하기 위해서는 코드를 작성하고 실행할 수 있는 안정적인 **개발 환경**이 필수적입니다. Google Cloud는 웹 브라우저만으로 간편하게 개발할 수 있는 도구부터, 전문적인 개발자를 위한 로컬 환경 지원까지 다양한 옵션을 제공하고 있습니다.

이번 챕터에서는 Vertex AI를 활용하기 위한 대표적인 세 가지 개발 환경의 특징을 비교해 보고, 현업 개발자들이 가장 선호하는 **'로컬 PC(VS Code)와 Vertex AI SDK'** 조합을 구축하는 과정을 단계별로 상세히 알아보겠습니다. 특히 기존의 API 키 방식이 아닌, 엔터프라이즈 환경에서 요구되는 보안 표준인 **IAM 기반 인증(ADC)**을 설정하는 방법을 중점적으로 다룹니다.

---

## 4-1 Cloud Shell, 로컬 환경, Workbench 비교

Vertex AI를 시작하는 방법은 프로젝트의 목적과 규모에 따라 달라집니다. 우리는 다음 세 가지 환경 중 자신의 상황에 가장 적합한 것을 선택해야 합니다.

---

### 4-1-1 Cloud Shell

**Cloud Shell**은 Google Cloud 콘솔 웹페이지 하단에서 즉시 실행할 수 있는 **리눅스 터미널 환경**입니다. 이 환경의 가장 큰 특징은 사용자가 별도의 도구를 설치할 필요가 없다는 점입니다. gcloud 커맨드 라인 도구는 물론 Python, Docker와 같은 필수 개발 도구가 이미 설치되어 있어, 언제 어디서나 브라우저만 있다면 간단한 테스트나 배포 명령어를 수행할 수 있습니다.

하지만 Cloud Shell은 일정 시간 사용하지 않으면 세션이 종료되면서 실행 중이던 프로세스가 멈출 수 있습니다. 따라서 긴 시간이 소요되는 AI 모델 학습이나 복잡한 애플리케이션 개발보다는, 간단한 API 호출 테스트나 인프라 설정을 위한 보조 도구로 활용하는 것이 좋습니다.

 > 📷 **[이미지]** 
 > Google Cloud 콘솔 하단의 Cloud Shell 아이콘을 클릭하여 터미널 창이 열린 모습입니다. 
 > gcloud list 명령어를 입력했을 때 프로젝트 목록이 출력되는 것을 확인해보세요.

---

### 4-1-2 Vertex AI Workbench

**Vertex AI Workbench**는 **주피터(Jupyter) 노트북** 기반의 완전 관리형 개발 환경입니다. 사용자가 가상 머신(VM)을 직접 관리할 필요 없이, 웹상에서 클릭 몇 번으로 고성능 GPU가 포함된 노트북 환경을 생성할 수 있습니다. 특히 BigQuery나 Google Cloud Storage와 같은 데이터 저장소와 연동이 매우 쉽기 때문에, 데이터 분석가나 머신러닝 엔지니어들이 데이터를 탐색하고 모델을 실험하는 용도로 주로 사용합니다.

그러나 Workbench는 웹 브라우저 기반이기에 Visual Studio Code와 같은 전문 IDE(통합 개발 환경)가 제공하는 강력한 디버깅 기능이나 코파일럿(Copilot) 기능을 100% 활용하기에는 다소 제약이 있습니다. 또한 인스턴스를 켜놓은 시간만큼 비용이 발생하므로 비용 관리에도 주의를 기울여야 합니다.

> 📷 **[이미지]** Vertex AI 메뉴 내의 'Workbench' 탭에서 '관리형 노트북'을 생성하고, JupyterLab 인터페이스가 실행된 웹 브라우저 화면을 캡처해 주세요.

---

### 4-1-3 로컬 환경 (Local PC + VS Code)

마지막으로 우리가 이번 실습에서 주력으로 사용할 **로컬 환경**입니다. 개발자의 PC에 Python과 VS Code를 설치하고, Google Cloud의 Vertex AI 서비스와 원격으로 통신하는 방식입니다. 이 방식은 실제 웹이나 앱 서비스를 개발할 때 가장 표준적으로 사용되는 구성입니다.

로컬 환경을 구축하면 개발자에게 익숙한 **VS Code의 강력한 편집 기능**과 **Git을 이용한 버전 관리**를 자유롭게 사용할 수 있다는 큰 장점이 있습니다. 초기 설정 과정이 다소 복잡하게 느껴질 수 있지만, 한 번 환경을 구축해 두면 추가적인 비용 없이 가장 효율적으로 애플리케이션을 개발할 수 있습니다.

> 📷 **[이미지]** 세 가지 환경(Cloud Shell, Workbench, Local)을 비교 이미지를 확인하세요.
> 각 환경의 장점과 추천 용도 등을 구분합니다.

---

## 4-2 gcloud 기본 설정과 프로젝트 컨텍스트

로컬 환경에서 Vertex AI를 사용하기 위해 가장 먼저 넘어야 할 산은 바로 **'인증(Authentication)'**입니다. 과거에는 Gemini API 키와 같은 긴 문자열을 발급받아 코드에 직접 붙여넣는 방식을 사용했습니다. 하지만 Vertex AI는 보안을 위해 **Google Cloud 계정의 권한(IAM)**을 직접 사용하여 통신합니다. 이를 가능하게 해주는 도구가 바로 **Google Cloud CLI (gcloud)**입니다.

---

### 4-2-1 Google Cloud CLI 설치

가장 먼저 여러분의 운영체제(Windows, macOS)에 맞는 **gcloud CLI 도구**를 설치해야 합니다. Google Cloud 공식 문서에서 설치 파일을 다운로드하여 설치를 진행합니다. 


### Windows 에서 설치 방법 

#### 방법 1 — winget (권장, Windows 10 이상)

```powershell
winget install Google.CloudSDK
```

설치 완료 후 **새 PowerShell 창**을 열어야 `gcloud` 명령이 인식된다.

#### 방법 2 — 공식 인스톨러

1. [cloud.google.com/sdk/docs/install](https://cloud.google.com/sdk/docs/install) 에 접속한다.
2. **Windows 64-bit** 인스톨러(`GoogleCloudSDKInstaller.exe`)를 다운로드한다.
3. 인스톨러를 실행하고 기본값으로 설치한다.
4. 설치 완료 시 **"Start Google Cloud CLI Shell"** 체크박스가 나타난다 — 그대로 두면 설치 직후 초기 설정(`gcloud init`)이 자동 시작된다.


설치가 완료되었다면 터미널(또는 PowerShell)을 열고 아래 명령어를 입력하여 정상적으로 설치되었는지 확인합니다.

```bash
gcloud --version
```

---

### 4-2-2 구글 계정 로그인 (gcloud init)

도구 설치가 끝났다면, 이제 내 PC가 어떤 구글 계정을 사용할지, 그리고 어떤 GCP 프로젝트에 접속할지 알려주는 **초기화 과정**이 필요합니다. 아래 명령어를 실행합니다.

```bash
gcloud init
```

이 명령어를 입력하면 웹 브라우저가 자동으로 실행되면서 구글 로그인을 요청합니다. 로그인을 완료하고 터미널로 돌아오면 사용할 GCP 프로젝트를 선택하는 단계가 나옵니다. 앞서 Chapter 3에서 생성했던 프로젝트를 선택하면 기본적인 설정이 완료됩니다.

> 📷 **[이미지]** 
> 터미널에서 gcloud init 실행 후 브라우저 로그인 창이 뜬 모습과, 이후 터미널에서 프로젝트 리스트 중 하나를 번호로 선택하는 과정 확인합니다. 

---

### 4-2-3 애플리케이션 기본 자격 증명 (ADC) 설정

이 단계가 개발 환경 구성의 **핵심**입니다. 앞서 진행한 gcloud init은 여러분이 터미널에서 관리자로서 명령어를 내리기 위한 로그인 과정이었습니다. 하지만 우리가 작성할 Python 코드가 여러분의 계정 권한을 빌려 쓰기 위해서는 별도의 인증 파일이 필요합니다. 이를 **ADC(Application Default Credentials)**라고 부릅니다.

VS Code 터미널에서 다음 명령어를 입력합니다:



```bash
gcloud auth application-default login
```

이 명령어를 실행하면 다시 한번 웹 브라우저가 열리며 'Google Auth Library'가 여러분의 계정에 액세스하도록 허용할지 묻습니다. '허용'을 클릭하면 인증이 완료되고, 여러분의 PC 내 특정 폴더(예: ~/.config/gcloud/...)에 **인증 정보가 담긴 JSON 파일**이 생성됩니다.

이제부터 여러분이 작성하는 Python 코드는 자동으로 이 JSON 파일을 찾아 읽어들입니다. "아, 이 코드는 [사용자 이름]의 권한으로 실행되는구나"라고 인식하게 되는 것입니다. 덕분에 소스 코드 안에 민감한 API 키를 직접 적어 넣지 않아도 되어 **보안 사고를 원천적으로 방지할 수 있습니다.**

> 📷 **[이미지]** 
> gcloud auth application-default login 명령어 실행 후, 터미널에 "Credentials saved to file: [...]"이라는 성공 메시지가 출력된 화면을 확인합니다. 

---

## 4-3 Python 가상환경과 라이브러리 설치

인증 설정이 완료되었으므로, 이제 Python 개발 환경을 구축해 보겠습니다. 여러 프로젝트를 진행하다 보면 라이브러리 버전 간의 충돌이 발생할 수 있습니다. 이를 방지하기 위해 프로젝트마다 독립된 **가상환경(Virtual Environment)**을 사용하는 것이 좋습니다.

---

---

### 4-3-1 프로젝트 폴더 생성 및 가상환경 구성

VS Code에서 실습을 진행할 폴더를 열고, 터미널(단축키: Ctrl + `)을 실행합니다. 그리고 아래 명령어를 통해 가상환경을 생성하고 활성화합니다.

**Windows 사용자**



```bash
# 'venv'라는 이름의 가상환경을 생성합니다.
python -m venv venv
# 생성된 가상환경을 활성화합니다.
.\venv\Scripts\activate
```

**macOS / Linux 사용자**



```bash
# 'venv'라는 이름의 가상환경을 생성합니다.
python3 -m venv venv
# 생성된 가상환경을 활성화합니다.
source venv/bin/activate
```

가상환경이 정상적으로 활성화되면 터미널의 명령어 입력줄(프롬프트) 앞부분에 **(venv)**라는 초록색 표시가 나타납니다. 이는 현재 우리가 시스템 전체의 Python이 아닌, 이 프로젝트만을 위한 격리된 공간에 들어와 있음을 의미합니다.

> 📷 **[이미지]** VS Code 터미널에서 가상환경을 생성하고 활성화하여, 프롬프트 앞에 (venv)가 표시된 화면을 확인할 수 있습니다. 

---

### 4-3-2 Vertex AI SDK 설치

이제 Vertex AI의 기능을 사용할 수 있도록 Python 라이브러리를 설치할 차례입니다. 우리가 사용할 패키지의 이름은 **google-cloud-aiplatform**입니다.



```bash
pip install --upgrade google-cloud-aiplatform
```

명령어 뒤에 붙은 --upgrade 옵션은 항상 최신 버전을 설치하라는 의미입니다. Vertex AI는 매우 빠른 속도로 발전하고 있어 Gemini Pro 1.5와 같은 최신 모델이 수시로 추가되므로, 항상 라이브러리를 최신 상태로 유지하는 것이 중요합니다.

추가로 환경 변수 관리를 위한 **python-dotenv**와 데이터 처리를 위한 **pandas** 라이브러리도 함께 설치해 두면 편리합니다.

> 📷 **[이미지]** pip install google-cloud-aiplatform 명령어가 실행되면서 설치가 진행되는 것을 터미널을 통해 확인 가능합니다. 

---

## 4-4 Jupyter와 VS Code를 이용한 개발 워크플로

환경 설정의 마지막 단계로, 실제 코드가 잘 동작하는지 확인해 보겠습니다. 우리는 VS Code 내부에서 **Jupyter Notebook(.ipynb) 파일**을 생성하여 코드를 작성하고 결과를 바로 확인하는 방식을 사용할 것입니다.

---

---

### 4-4-1 VS Code에서 Jupyter Notebook 실행하기

VS Code는 별도의 설정 없이도 **.ipynb 파일**을 완벽하게 지원합니다. 탐색기에서 **test_vertex.ipynb**라는 파일을 하나 생성해 봅니다. 파일이 열리면 우측 상단에 '**커널 선택(Select Kernel)**' 버튼이 보일 것입니다. 이 버튼을 눌러 우리가 방금 만든 **venv (Python 환경)**을 선택해 줍니다.

이제 첫 번째 셀에 아래 코드를 입력하고 **실행(Shift + Enter)**해 봅니다.



```python
import vertexai
# 프로젝트 ID와 리전(Region)을 설정합니다.
# 본인의 GCP 프로젝트 ID로 변경해야 합니다.
PROJECT_ID = "your-project-id"
LOCATION = "us-central1"
# Vertex AI SDK를 초기화합니다.
vertexai.init(project=PROJECT_ID, location=LOCATION)
print("Vertex AI SDK 초기화 성공!")
```

이 코드는 Vertex AI 라이브러리를 불러오고, 내 프로젝트와 연결을 맺는 역할을 합니다. 오류 없이 **"초기화 성공!"**이라는 메시지가 출력된다면, 여러분의 PC와 Google Cloud가 정상적으로 연결된 것입니다.

> 📷 **[이미지]** VS Code에서 .ipynb 파일을 열고 위 코드를 작성
> 우측 상단 커널이 venv로 잡혀 있고, 코드 셀 아래에 "Vertex AI SDK 초기화 성공!" 출력이 나온 화면을 확인하세요.

---

### 4-4-2 첫 번째 Gemini 모델 호출

연결이 확인되었다면, 실제로 Gemini 모델을 호출하여 대화를 시도해 보겠습니다. 다음 셀에 아래 코드를 입력합니다.



```python
from vertexai.generative_models import GenerativeModel
# Gemini 1.5 Flash 모델을 로드합니다.
model = GenerativeModel("gemini-1.5-flash")
# 모델에게 콘텐츠 생성을 요청합니다.
response = model.generate_content("Vertex AI 개발자가 된 것을 축하해줘!")
# 모델의 응답 텍스트를 출력합니다.
print(response.text)
```

이 코드를 실행하면 잠시 후 Gemini가 생성한 **축하 메시지**가 출력될 것입니다. 이 과정은 여러분의 로컬 PC에서 보낸 요청이 Google 데이터센터로 전송되고, 그곳의 거대한 Gemini 모델이 추론한 결과를 다시 여러분의 PC로 보내주는 흐름으로 진행됩니다.

> 📷 **[이미지]** 로컬 PC(User)에서 인증(ADC)을 거쳐 Google Cloud(Vertex AI)로 연결되는 흐름

---

---

---

## 마무리

이것으로 Chapter 4의 모든 과정을 마쳤습니다. 이제 여러분의 PC는 단순한 코딩 도구를 넘어, 구글의 고성능 AI 모델을 자유자재로 다룰 수 있는 **강력한 개발 스테이션**으로 거듭났습니다. 다음 Chapter 5에서는 이렇게 구축된 환경 위에서 Gemini의 다양한 파라미터를 조절하고, 멀티턴 대화를 구현하는 등 더욱 깊이 있는 기능을 실습해 보겠습니다.

