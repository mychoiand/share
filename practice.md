노트북 상세 분석: intro_code_execution.ipynb
이 노트북은 Google의 Gemini 2.5 Flash 모델을 사용하여 Python 코드를 자동 생성하고 실행하는 방법을 소개하는 공식 튜토리얼입니다.

1. 전체 구조 (총 41개 셀)
구분	셀 범위	내용
헤더/라이선스	cell 0~2	Apache 2.0 라이선스, Colab/GitHub 링크
개요	cell 3	목적 및 학습 목표 설명
환경 설정	cell 4~11	SDK 설치, 라이브러리 임포트, API 클라이언트 생성
기본 코드 실행	cell 12~22	단일 프롬프트로 코드 생성/실행
채팅 세션	cell 23~37	멀티턴 대화에서 코드 실행
스트리밍 세션	cell 38~39	스트리밍 모드에서 코드 실행
요약	cell 40	정리 및 다음 단계
2. 핵심 개념: Code Execution이란?
Gemini의 Code Execution은 모델이:

프롬프트를 받고 Python 코드를 자동 생성
API 서버 측에서 코드를 직접 실행
결과를 확인하고 오류가 있으면 자동 수정
최종 결과를 반환
일반적인 LLM 코드 생성과 달리, 실행까지 API가 처리한다는 점이 핵심 차별점입니다.

3. 셀별 상세 분석
(A) 환경 설정 (cell 6~11)

%pip install --upgrade --quiet google-genai   # SDK 설치

from google import genai
from google.genai.types import GenerateContentConfig, Tool, ToolCodeExecution
genai.Client: Vertex AI 또는 Google AI에 연결하는 클라이언트
Tool, ToolCodeExecution: 코드 실행 도구 정의용 타입

PROJECT_ID = "qwiklabs-gcp-01-26236439ed1e"  # GCP 프로젝트 ID
LOCATION = "global"
client = genai.Client(vertexai=True, project=PROJECT_ID, location=LOCATION)
주의: PROJECT_ID는 Qwiklabs 임시 프로젝트 ID입니다. 실제 사용 시 본인의 GCP 프로젝트 ID로 교체해야 합니다.

(B) 코드 실행 도구 정의 (cell 15)

code_execution_tool = Tool(code_execution=ToolCodeExecution())
이 한 줄로 코드 실행 도구를 생성합니다. 이후 generate_content 호출 시 tools 파라미터로 전달합니다.

(C) 기본 예제: 소수 합 계산 (cell 17~22)

PROMPT = "What is the sum of the first 50 prime numbers? Generate and run code for the calculation."

response = client.models.generate_content(
    model=MODEL_ID,
    contents=PROMPT,
    config=GenerateContentConfig(
        tools=[code_execution_tool],
        temperature=0,         # 결정론적 출력
    ),
)
응답 구조 파싱 방법:

part.executable_code → 모델이 생성한 Python 코드
part.code_execution_result → 실행 결과 (.output과 .outcome)
결과: 5117 (처음 50개 소수의 합)
(D) 채팅 세션 예제 (cell 24~37)
멀티턴 대화로 시계열 데이터 분석을 시연합니다:

턴	프롬프트	목적
1턴	"시험 용광로 온도 시계열 데이터 생성 + 노이즈 추가"	데이터 생성
2턴	"스무딩 데이터 시리즈 추가"	데이터 가공
3턴	"기술 통계량 생성 및 출력"	데이터 분석

chat = client.chats.create(
    model=MODEL_ID,
    config=GenerateContentConfig(
        tools=[code_execution_tool],
        temperature=0,
        max_output_tokens=10240,   # 채팅에서는 토큰 제한 명시
    ),
)
response = chat.send_message(PROMPT)  # send_message로 대화 이어감
핵심 포인트: chats.create로 세션을 만들면 대화 히스토리가 유지되므로, 2턴에서 1턴에서 생성한 데이터를 그대로 참조할 수 있습니다.

(E) 스트리밍 예제 (cell 39)

for chunk in client.models.generate_content_stream(
    model=MODEL_ID,
    contents=PROMPT,
    config=GenerateContentConfig(tools=[code_execution_tool], temperature=0),
):
스트리밍에서는 응답이 chunk 단위로 도착하며, 각 chunk의 parts에서:

part.text → 자연어 텍스트 스트림
part.executable_code → 코드 스트림
part.code_execution_result → 실행 결과 스트림
이렇게 세 종류를 구분하여 실시간으로 표시합니다.

4. 응답 데이터 구조 요약

response.candidates[0].content.parts[]
  ├── .text                    → 자연어 설명
  ├── .executable_code.code    → 생성된 Python 코드 (문자열)
  └── .code_execution_result
        ├── .output            → 실행 출력 (stdout)
        └── .outcome           → 실행 결과 상태 (OUTCOME_OK 등)
5. 실습에서 배울 수 있는 것
항목	설명
단발 코드 실행	한 번의 API 호출로 코드 생성 + 실행 + 결과 수신
멀티턴 코드 실행	대화 맥락 유지하며 점진적 데이터 분석
스트리밍 코드 실행	실시간으로 코드 생성/실행 결과 수신
자동 디버깅	모델이 실행 오류를 관찰하고 코드를 수정하여 재시도
6. 실제 사용 시 주의사항
PROJECT_ID를 본인의 GCP 프로젝트로 변경 필요
Vertex AI API가 활성화되어 있어야 함
코드 실행은 서버 측 샌드박스에서 실행되므로 로컬 파일 접근 불가
코드 실행은 Function Calling과 다름 — Function Calling은 클라이언트 측에서 함수를 실행하지만, Code Execution은 서버 측에서 Python을 실행
temperature=0으로 설정하여 재현 가능한 결과를 얻고 있음
intro_controlled_generation.ipynb 이걸 상세하게 제대로 분석해서 알려줘
Read d:\sw\2026_AI_teacher\Gemini_02_27\intro_controlled_generation.ipynb
Read d:\sw\2026_AI_teacher\Gemini_02_27\intro_controlled_generation.ipynb (from line 401)
노트북 상세 분석: intro_controlled_generation.ipynb
이 노트북은 Gemini API의 Controlled Generation(제어된 생성) 기능을 통해 모델 출력을 **구조화된 형식(JSON, Enum)**으로 강제하는 방법을 다루는 공식 튜토리얼입니다.

1. 전체 구조 (총 39개 셀)
구분	셀 범위	내용
헤더/라이선스	cell 0~2	Apache 2.0, Colab/GitHub 링크
개요 및 환경 설정	cell 3~14	목적 설명, SDK 설치, 클라이언트 생성
JSON 생성 3가지 방법	cell 15~23	Pydantic / OpenAPI / JSON Schema
Enum 생성	cell 24~25	열거형 값 강제 출력
실전 예제 6가지	cell 26~38	게임 캐릭터, 로그 분석, 리뷰 분석, 이미지 객체 감지, 영화 장르, 주문 상태
2. 핵심 개념: Controlled Generation이란?
일반적인 LLM 응답은 자유 형식 텍스트입니다. 하지만 실제 애플리케이션에서는 파싱 가능한 구조화된 데이터가 필요합니다. Controlled Generation은:

response_mime_type으로 출력 형식을 지정 (application/json 또는 text/x.enum)
response_schema 또는 response_json_schema로 출력 구조를 정의
모델이 해당 스키마를 엄격하게 따르도록 강제
3. JSON 생성 3가지 방법 비교
Option 1: Pydantic 모델 (cell 17~19) — 가장 Pythonic

class CountryInfo(BaseModel):
    name: str
    population: int
    capital: str
    ...

response = client.models.generate_content(
    model=MODEL_ID,
    contents="Fetch key facts for the United States.",
    config=GenerateContentConfig(
        response_mime_type="application/json",
        response_schema=CountryInfo,        # Pydantic 클래스 직접 전달
    ),
)
장점:

Python 타입 힌트로 스키마 정의 → 코드가 깔끔
response.parsed로 Pydantic 객체로 자동 역직렬화 가능

country: CountryInfo = response.parsed
print(country.name)       # "United States"
print(country.population) # 334805269
Option 2: OpenAPI Schema (cell 21) — 딕셔너리 기반

response_schema = {
    "type": "OBJECT",
    "properties": {
        "name": {"type": "STRING"},
        "population": {"type": "INTEGER"},
        ...
    },
    "required": ["name", "population", ...],
}
특징:

타입명이 대문자 (STRING, INTEGER, OBJECT 등)
지원 필드: enum, items, maxItems, nullable, properties, required
기본적으로 필드는 optional → required로 명시해야 필수
Option 3: JSON Schema (cell 23) — 가장 강력, 조건부 로직 지원

country_info_schema = {
    "type": "object",         # 소문자 (표준 JSON Schema)
    "properties": { ... },
    "required": ["country_name", "capital", "is_g7_member", "government_type"],
    "allOf": [
        {
            "if": {"properties": {"government_type": {"const": "Republic"}}},
            "then": {"required": ["head_of_state"]},
        },
        {
            "if": {"properties": {"government_type": {"const": "Constitutional Monarchy"}}},
            "then": {"required": ["monarch"]},
        },
    ],
}
핵심 차이점: allOf + if/then 으로 조건부 필수 필드 설정 가능. 예를 들어:

공화국이면 → head_of_state 필수
입헌군주제면 → monarch 필수
G7 회원이면 → g7_join_year 필수
파라미터도 다릅니다: response_schema 대신 response_json_schema 사용

4. 3가지 방법 요약 비교표
항목	Pydantic	OpenAPI Schema	JSON Schema
파라미터	response_schema	response_schema	response_json_schema
타입 표기	Python 타입 (str, int)	대문자 (STRING, INTEGER)	소문자 (string, integer)
조건부 로직	불가	불가	allOf + if/then 지원
자동 파싱	response.parsed → Pydantic 객체	response.parsed → dict	response.parsed → dict
추천 용도	단순 구조, Python 프로젝트	REST API 통합	복잡한 조건부 스키마
5. Enum 생성 (cell 25)

class InstrumentEnum(Enum):
    PERCUSSION = "Percussion"
    STRING = "String"
    WOODWIND = "Woodwind"
    BRASS = "Brass"
    KEYBOARD = "Keyboard"

config=GenerateContentConfig(
    response_mime_type="text/x.enum",      # JSON이 아닌 Enum 모드
    response_schema=InstrumentEnum,
)
response_mime_type="text/x.enum" 설정 시, 모델은 enum 값 중 하나만 텍스트로 반환
결과: Keyboard (다중 선택이 아닌 단일 분류)
감정 분류, 카테고리 분류 등에 적합
6. 실전 예제 6가지 상세
(1) 게임 캐릭터 프로필 (cell 28)
중첩 스키마: 캐릭터 내에 children 배열 (이름, 나이)
Safety Settings 함께 사용: SafetySetting으로 유해 콘텐츠 차단 레벨 설정
결과: 이름, 나이, 직업, 배경, 조작 가능 여부, 자녀 3명 정보
(2) 로그 데이터 에러 추출 (cell 30)
nullable: True 활용: ERROR가 아닌 INFO 로그에는 error_code가 None으로 반환
비정형 로그 텍스트 → 구조화된 JSON 배열로 변환
(3) 제품 리뷰 감정 분석 (cell 32)
스키마 내 enum 필드: sentiment를 ["POSITIVE", "NEGATIVE", "NEUTRAL"]로 제한
모델이 감정 분류 + 설명을 함께 생성
(4) 이미지 객체 감지 (cell 34)
멀티모달 + Controlled Generation 결합
Part.from_uri()로 GCS 버킷의 이미지 2장 전달
각 이미지별 감지된 객체 목록을 2차원 배열로 반환
(5) 영화 장르 분류 (cell 36)
OpenAPI 스키마 + text/x.enum 조합
Python Enum 클래스 없이 딕셔너리로도 가능: {"type": "STRING", "enum": ["drama", "comedy", "documentary"]}
결과: documentary
(6) 주문 상태 조건부 스키마 (cell 38)
JSON Schema의 allOf + if/then 활용
SHIPPED → tracking_number 필수, DELIVERED → delivery_date 필수, PENDING → 추가 필드 없음
3건의 주문을 한 번에 처리하여 조건에 맞는 다른 필드를 포함한 배열 반환
7. 핵심 설정 파라미터 정리
파라미터	값	용도
response_mime_type	"application/json"	JSON 형식 출력 강제
response_mime_type	"text/x.enum"	열거형 단일 값 출력 강제
response_schema	Pydantic 클래스 또는 OpenAPI dict	출력 구조 정의
response_json_schema	JSON Schema dict	조건부 로직 포함 출력 구조 정의
safety_settings	SafetySetting 리스트	유해 콘텐츠 차단 수준 설정
8. 이 노트북의 핵심 교훈
LLM 출력을 프로그래밍적으로 신뢰할 수 있게 만드는 핵심 기법
단순 구조는 Pydantic, 복잡한 조건부 구조는 JSON Schema 사용
분류 문제는 text/x.enum으로 깔끔하게 해결
**멀티모달(이미지)**과도 결합 가능
response.parsed로 문자열 파싱 없이 바로 Python 객체/딕셔너리로 사용 가능
