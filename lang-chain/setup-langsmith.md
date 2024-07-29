## Setup

1. 환경 변수 세팅

.env 파일
```
LANGCHAIN_TRACING_V2=true
LANGCHAIN_ENDPOINT="https://api.smith.langchain.com"
LANGCHAIN_PROJECT="my-project-name"
LANGCHAIN_API_KEY="lsvxxxx6"
```

2. 코드 실행
```
from langchain_openai import ChatOpenAI

llm = ChatOpenAI()
llm.invoke("Hello, world!")
```

3. Projects 탭으로 가서 확인하면 새로 프로젝트가 추가됨
