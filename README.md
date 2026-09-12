# 👴 AI Elderly Care System
### 독거노인 안심케어 AI 시스템

> 실시간 생활 소음을 감지하고,  
> 충격음 발생 시 어르신에게 음성으로 상태를 확인한 뒤  
> **STT · 키워드 가드레일 · AI 문맥 분석**을 통해 위험 상황을 판단하여  
> 보호자에게 Telegram 긴급 알림을 전송하는 AI 돌봄 시스템 프로토타입입니다.

---

## 📌 Project Overview

고령화와 독거노인 증가에 따라  
낙상이나 건강 이상과 같은 위급상황이 발생했을 때  
빠르게 상황을 인지하고 보호자에게 전달할 수 있는 시스템을 구현했습니다.

기존의 단순 임계값 기반 센서나 특정 키워드 탐지 방식에서 나아가,

- 실시간 소음 감지
- 음성 안내
- Speech-to-Text
- 위험 / 안전 키워드 필터링
- AI 기반 문맥 분석
- 보호자 긴급 알림

을 하나의 흐름으로 연결했습니다.

---

## 💡 Motivation

독거노인의 낙상 사고는 실내에서 발생할 가능성이 높고,  
사고 직후 도움을 요청하지 못하면 초기 대응이 늦어질 수 있습니다.

또한 단순 키워드 기반 시스템은

- 명확한 구조 요청이 없는 경우
- 일상적인 부정 표현이 포함된 경우
- 발화의 실제 문맥이 중요한 경우

위험 상황을 정확히 구분하기 어렵다는 한계가 있습니다.

따라서 본 프로젝트에서는

> **소리 감지 + 직접 안부 확인 + 키워드 가드레일 + AI 문맥 분석**

을 결합한 하이브리드 구조를 설계했습니다.

---

## ⚙️ System Flow

```text
실시간 마이크 모니터링
        ↓
RMS 기반 큰 소리 감지
        ↓
"어르신 괜찮으세요?" TTS 출력
        ↓
어르신 답변 녹음
        ↓
Google Speech Recognition
        ↓
한국어 텍스트 변환
        ↓
위험 키워드 확인
   ┌────┴────┐
   │         │
 발견       미발견
   │         │
즉시 경보   안전 키워드 확인
             ↓
       안전 표현이면 종료
             ↓
        그 외 문장
             ↓
       AI 문맥 분석
             ↓
   위험도 신뢰도 85% 이상
             ↓
      Telegram 긴급 알림
```

---

## ✨ Key Features

### 01. RMS 기반 실시간 소음 감지 🎤

마이크 입력을 실시간으로 모니터링하고  
RMS(Root Mean Square)를 이용하여 소리의 크기를 계산합니다.

설정된 임계값보다 큰 소리가 감지되면  
일반적인 생활소음과 다른 상황이 발생했다고 판단하여  
안부 확인 단계로 전환합니다.

```python
rms = np.sqrt(np.mean(recording**2))

if rms > threshold:
    # 큰 소리 감지 → 안부 확인 단계
```

---

### 02. TTS 기반 능동적 안부 확인 🔊

큰 소리가 발생했다고 바로 위험 상황으로 확정하지 않습니다.

시스템이 직접

> **"어르신 괜찮으세요?"**

라고 질문하여 추가적인 상황 정보를 확보합니다.

`pyttsx3`를 이용해 음성을 생성하고  
설정된 스피커 장치를 통해 출력합니다.

---

### 03. Google STT 기반 한국어 음성 인식 🗣️

어르신의 답변을 일정 시간 녹음한 뒤  
Google Speech Recognition을 이용하여 한국어 텍스트로 변환합니다.

```text
음성 입력
   ↓
WAV 임시 저장
   ↓
Google Speech Recognition
   ↓
한국어 Text
```

이후 변환된 텍스트를 위험 상황 분석에 활용합니다.

---

### 04. Hybrid Guardrail 🛡️

모든 문장을 AI 모델에 전달하지 않고  
먼저 명확한 위험 / 안전 키워드를 검사합니다.

#### Critical Keywords

```text
살려
119
쓰러
숨못
죽겠
아파
못일어
...
```

위험 키워드가 포함되어 있으면  
AI 추론 단계를 생략하고 즉시 보호자에게 알림을 전송합니다.

#### Safe Keywords

```text
덥다
춥다
짜증
귀찮
심심
시원
...
```

명확한 일상 표현은 가드레일에서 안전 상황으로 처리하여  
불필요한 AI 추론과 오경보를 줄이도록 설계했습니다.

---

## 🤖 AI Context Analysis

위험 키워드와 안전 키워드 어디에도 해당하지 않는  
애매한 문장은 AI 모델이 추가 분석합니다.

사용 모델:

```text
matthewburke/korean_sentiment
```

Hugging Face `transformers`의 sentiment-analysis pipeline을 사용했습니다.

```python
ai_engine = pipeline(
    "sentiment-analysis",
    model="matthewburke/korean_sentiment"
)
```

모델의 분석 결과가 위험 문맥으로 판단되고  
신뢰도가 **85% 이상일 경우에만** 최종 긴급 알림을 전송합니다.

이를 통해 명확한 위험 상황은 빠르게 처리하면서도  
애매한 발화는 AI를 통해 추가적으로 판단하는 구조를 구성했습니다.

---

## 🚨 Telegram Emergency Alert

최종적으로 위험 상황이 감지되면  
Telegram Bot API를 이용하여 보호자에게 알림을 전송합니다.

알림에는 다음 정보가 포함됩니다.

```text
🚨 긴급 알림

- 감지된 발화 내용
- AI 분석 신뢰도
- 위험 판단 근거
```

### Telegram 설정

공개 저장소에는 보안상 실제 인증정보를 포함하지 않습니다.

`elderly_care_system.py`에서 다음 값을  
본인의 Telegram Bot 정보로 변경해야 합니다.

```python
BOT_TOKEN = "YOUR_TELEGRAM_BOT_TOKEN"
CHAT_ID = "YOUR_TELEGRAM_CHAT_ID"
```

---

## ⏱️ Fail-Safe Monitoring

가족 방문 등의 상황에서는 `p` 키를 눌러  
모니터링을 일시적으로 중단할 수 있습니다.

```text
p → 모니터링 일시 정지
r → 수동 재가동
q → 시스템 종료
```

하지만 사용자가 시스템을 다시 켜는 것을 잊을 가능성을 고려하여  
설정된 시간이 지나면 자동으로 모니터링이 재개되는  
**Fail-Safe Timer**를 구현했습니다.

```text
일시 정지
    ↓
타이머 시작
    ↓
사용자가 직접 복귀하지 않음
    ↓
설정 시간 만료
    ↓
자동으로 모니터링 재가동
```

단순 기능 구현뿐 아니라  
사용자의 실수로 안전 기능이 장시간 비활성화되는 상황을 방지하고자 했습니다.

---

## 🛠 Tech Stack

### Language

`Python`

### AI / NLP

`Hugging Face Transformers`  
`matthewburke/korean_sentiment`

### Audio

`SoundDevice`  
`SciPy`  
`PyTTSx3`

### Speech Recognition

`SpeechRecognition`  
`Google Speech Recognition`

### Notification

`Telegram Bot API`

### Data Processing

`NumPy`

---

## 📁 Repository Structure

```text
ai-elderly-care-system/
│
├── README.md
├── .gitignore
│
├── src/
│   └── elderly_care_system.py
│
└── docs/
    └── presentation.pdf
```

---

## 🔧 Audio Device Configuration

개발 당시 사용한 마이크와 스피커 장치를 직접 지정하여 테스트했습니다.

따라서 아래 장치 번호는 **개발 환경 기준 값**이며  
다른 PC에서 실행할 경우 장치 번호를 변경해야 할 수 있습니다.

```python
sd.default.device = (1, 5)

SAMPLE_RATE = 16000
CHANNELS = 1

LAPTOP_SPEAKER_ID = 5
```

현재 사용 가능한 오디오 장치는 다음 코드로 확인할 수 있습니다.

```python
import sounddevice as sd

print(sd.query_devices())
```

---

## ▶️ Getting Started

### 1. Repository Clone

```bash
git clone <repository-url>
cd ai-elderly-care-system
```

### 2. Required Packages

```bash
pip install numpy
pip install sounddevice
pip install SpeechRecognition
pip install pyttsx3
pip install scipy
pip install transformers
pip install torch
pip install requests
```

### 3. Telegram 설정

`elderly_care_system.py`에서 Telegram 정보를 설정합니다.

```python
BOT_TOKEN = "YOUR_TELEGRAM_BOT_TOKEN"
CHAT_ID = "YOUR_TELEGRAM_CHAT_ID"
```

### 4. Audio Device 설정

사용 중인 마이크와 스피커의 장치 번호를 확인한 뒤  
환경에 맞게 수정합니다.

```python
sd.default.device = (MIC_DEVICE_ID, SPEAKER_DEVICE_ID)
LAPTOP_SPEAKER_ID = SPEAKER_DEVICE_ID
```

### 5. Run

```bash
python src/elderly_care_system.py
```

---

## 🔎 Example

### 명확한 위험 발화

```text
"살려줘, 못 일어나겠어"
```

```text
충격음 감지
→ STT
→ 위험 키워드 매칭
→ AI 분석 생략
→ Telegram 즉시 알림
```

### 일상적인 표현

```text
"오늘 너무 덥다"
```

```text
충격음 감지
→ STT
→ Safe Keyword 확인
→ 일상 상황으로 판단
→ 알림 미전송
```

### 문맥 판단이 필요한 표현

```text
"기운이 하나도 없고 이상해"
```

```text
충격음 감지
→ STT
→ 명확한 위험 / 안전 키워드 없음
→ AI 문맥 분석
→ 위험도 및 신뢰도 확인
→ 조건 충족 시 Telegram 알림
```

---

## ⚠️ Limitations

### 1. Speech Recognition 의존성

음성을 텍스트로 변환하는 과정에서  
Google Speech Recognition의 성능에 영향을 받습니다.

고령자의 부정확한 발음, 방언, 주변 소음 등에 따라  
STT 결과의 정확도가 저하될 수 있습니다.

### 2. 발화 불능 상황

현재 시스템은 충격음 발생 이후  
어르신의 답변을 분석하여 최종 상황을 판단합니다.

따라서 낙상 이후 의식을 잃거나  
대답할 수 없는 상태가 되면 STT 결과가 생성되지 않아  
위험 상황을 놓칠 가능성이 있습니다.

### 3. RMS 기반 소리 감지

현재는 소리의 크기를 기반으로 이벤트를 감지하기 때문에  
큰 생활소음과 실제 낙상 충격음을 완벽하게 구분하지는 못합니다.

### 4. Hardware Dependency

마이크와 스피커 장치 번호를 개발 환경에 맞게 직접 지정하였기 때문에  
PC 환경이 바뀌면 장치 설정을 수정해야 할 수 있습니다.

---

## 🚀 Future Improvements

### 침묵 상황 위험 처리

충격음 이후 안부 확인에 대한 STT 결과가 공백으로 반환될 경우,

```text
응답 없음
→ 단순 유예
```

가 아니라,

```text
응답 없음
→ 의식 불명 가능성
→ 추가 확인 또는 보호자 알림
```

으로 처리하도록 개선할 수 있습니다.

### Client-Server Architecture

모바일 환경에서 AI 모델을 직접 실행할 경우  
배터리와 연산 자원에 부담이 발생할 수 있습니다.

향후에는

```text
Smartphone
   ↓
Audio Input / Output
   ↓
Server
   ↓
STT + AI Analysis
   ↓
Emergency Notification
```

형태의 Client-Server 구조로 확장하여  
스마트폰의 연산 부담을 줄일 수 있습니다.

### Mobile Background Service

스마트폰에서도 지속적으로 모니터링할 수 있도록  
백그라운드 서비스 기반의 상시 실행 구조로 확장할 수 있습니다.

---

## 🎯 Project Point

이 프로젝트에서 중요하게 생각한 것은  
단순히 AI 모델을 호출하는 것이 아니라,

> **센서 → 사용자 확인 → STT → 규칙 기반 판단 → AI 판단 → 보호자 알림**

까지 하나의 실제 서비스 흐름으로 연결하는 것이었습니다.

또한 명확한 위험 상황에는 AI 추론을 생략해 빠르게 대응하고,  
애매한 발화에서만 모델을 사용하도록 하여  
**Rule-based Guardrail과 AI를 결합한 하이브리드 구조**를 구현했습니다.

---

## 📄 Presentation

프로젝트의 문제 정의, 시스템 설계, 알고리즘 흐름,  
실험 결과 및 한계점은 `docs/presentation.pdf`에서 확인할 수 있습니다.
