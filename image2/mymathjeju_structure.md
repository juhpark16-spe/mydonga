# mymathjeju.py 구조 및 실행 흐름 (Mermaid Diagram)

본 문서는 LangChain AgentExecutor 기반의 **`mymathjeju.py`** 시스템 구조, Pydantic 데이터 스키마, 도구(Tool) 호출 및 데이터 저장 흐름을 Mermaid 다이어그램으로 시각화한 문서입니다.

---

## 1. 전체 시스템 아키텍처 (System Architecture)

```mermaid
flowchart TD
    subgraph Client ["💻 사용자 / CLI 인터페이스"]
        UserInput["사용자 입력 질문 (CLI)"]
        ChatHistory["대화 기록 (chat_history)"]
    end

    subgraph AgentPipeline ["🤖 LangChain Agent Pipeline"]
        Pipeline["create_agent_pipeline()"]
        LLM["ChatOpenAI (gpt-4o-mini)<br/>OpenRouter API"]
        Prompt["ChatPromptTemplate<br/>(System + History + User + Scratchpad)"]
        Agent["OpenAI Tools Agent"]
        Executor["AgentExecutor<br/>(verbose=True, return_intermediate_steps=True)"]
    end

    subgraph Tools ["🛠️ 사용 가능한 도구 (Tools)"]
        MathTool["math_tool"]
        JejuTool["jeju_tool"]
    end

    subgraph Schemas ["📋 Pydantic 스키마 및 연산 로직"]
        MathQuery["MathQuery (BaseModel)"]
        JejuQuery["JejuQuery (BaseModel)"]
        MathCalc["MathQuery.calculate()<br/>(abs, round, sqrt, pow, 사칙연산)"]
        JejuInfo["JejuQuery.get_jeju_info()<br/>(weather, tourist_spot, food, tip)"]
    end

    subgraph Storage ["💾 저장소"]
        JSONFile["data2/jejumath.json"]
    end

    UserInput --> Pipeline
    Pipeline --> LLM
    Pipeline --> Prompt
    Prompt --> Agent
    Agent --> Executor

    Executor -->|"수학 관련 질문 판단 시"| MathTool
    Executor -->|"제주도 정보 질문 판단 시"| JejuTool

    MathTool --> MathQuery
    MathQuery --> MathCalc

    JejuTool --> JejuQuery
    JejuQuery --> JejuInfo

    MathCalc --> Executor
    JejuInfo --> Executor

    Executor -->|"최종 답변 및 도구 실행 과정 반환"| ProcessQuery["process_query()"]
    ProcessQuery -->|"결과 저장"| JSONFile
    ProcessQuery -->|"히스토리 업데이트"| ChatHistory
```

---

## 2. 시퀀스 다이어그램 (Sequence Diagram)

사용자 질문 처리 시 `AgentExecutor`, `LLM`, `Tools`, `JSON 저장소` 간의 개별 상호작용 흐름입니다.

```mermaid
sequenceDiagram
    autonumber
    actor User as 사용자 (User)
    participant Main as process_query()
    participant Agent as AgentExecutor
    participant LLM as OpenRouter LLM (gpt-4o-mini)
    participant Tool as Tool (math_tool / jeju_tool)
    participant Schema as Pydantic Schema
    participant JSON as data2/jejumath.json

    User->>Main: 질문 전달 (예: "sqrt(16) 연산해줘")
    Main->>Agent: invoke({"input": question, "chat_history": history})
    Agent->>LLM: 프롬프트 + 질문 + 도구 정의 전달
    LLM-->>Agent: 도구 호출(Tool Call) 요청 (예: math_tool operation="sqrt", num1=16)
    
    alt 수학 연산 (math_tool)
        Agent->>Tool: math_tool(operation="sqrt", num1=16)
        Tool->>Schema: MathQuery 객체 생성 & calculate() 수행
        Schema-->>Tool: 연산 결과 반환 (예: 4.0)
        Tool-->>Agent: "계산 결과 (sqrt): 4.0"
    else 제주도 정보 조회 (jeju_tool)
        Agent->>Tool: jeju_tool(category="food", location="서귀포")
        Tool->>Schema: JejuQuery 객체 생성 & get_jeju_info() 수행
        Schema-->>Tool: 요약 정보 반환
        Tool-->>Agent: 제주도 가이드 텍스트
    end

    Agent->>LLM: 도구 실행 결과 (Intermediate Steps) 전달
    LLM-->>Agent: 최종 생성 응답 (Final Answer)
    Agent-->>Main: output + intermediate_steps 반환
    Main->>JSON: save_to_jejumath_json() (질문, 답변, 타임스탬프 기록)
    Main-->>User: AI 최종 답변 및 도구 실행 과정 출력
```

---

## 3. 클래스 다이어그램 (Class Diagram)

`mymathjeju.py`에 정의된 핵심 데이터 구조 및 Pydantic BaseModel 클래스 구조입니다.

```mermaid
classDiagram
    class BaseModel {
        <<Pydantic>>
    }

    class MathQuery {
        +str operation
        +float num1
        +float num2
        +calculate() float
    }

    class JejuQuery {
        +str category
        +str location
        +Optional~str~ date
        +get_jeju_info() str
    }

    class MathTool {
        <<tool>>
        +math_tool(operation, num1, num2) str
    }

    class JejuTool {
        <<tool>>
        +jeju_tool(category, location, date) str
    }

    BaseModel <|-- MathQuery
    BaseModel <|-- JejuQuery
    MathQuery <.. MathTool : args_schema
    JejuQuery <.. JejuTool : args_schema
```

---

## 4. 데이터 흐름 요약 (Data Flow Summary)

1. **입력 수신**: CLI에서 질문과 대화 기록(`chat_history`)이 `process_query()`로 전달됩니다.
2. **에이전트 자율 판단**: `AgentExecutor`가 OpenRouter LLM을 통해 어떤 툴(`math_tool` 또는 `jeju_tool`)을 호출할지 결정합니다.
3. **스키마 검증 및 실행**: Pydantic 스키마(`MathQuery`, `JejuQuery`)로 인자를 검증한 뒤 내장 함수/수학 연산 또는 제주도 가이드 응답을 생성합니다.
4. **결과 합성 및 기록**: 에이전트 도구 실행 과정(`intermediate_steps`)과 최종 답변을 결합하여 `data2/jejumath.json`에 타임스탬프와 함께 자동 기록합니다.
