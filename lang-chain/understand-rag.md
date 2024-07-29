#### reference 
https://python.langchain.com/v0.2/docs/tutorials/rag/

## RAG
- RAG은 추가적인 데이터로 LLM 지식을 강화하기 위한 기술이다.
- LLM은 넓은 범위의 토픽에 대해서 추론할 수 있지만, LLM은 트레인 받은 시점까지의 공개 데이터에 대해서만 알고 있다. 그래서 만약 private data 에 대해서 AI가 추론할 수 있게 하고 싶다면, 모델의 지식을 강화할 필요가 있다.`적절한 데이터를 가져오고, 모델에게 입력을 넣는 것을 RAG 이라고 한다.`

## 컨셉
1) Indexing : 데이터를 섭취하고, 인뎅싱 하는 파이프라인. 오프라인에서 일어 난다.
     - Load : 데이터를 로딩. Documents loaders 로 수행된다. 
     - Split : 모델의 한정된 컨텍스트 윈도우 안에서 큰 chunk는 검색에 어렵기 때문에, 큰 문서를 작은 chunk로 만드는 과정이 필요하다. 데이터를 인덱싱 하고 모델로 전달하기 위해 유용한 과정이다. 
     - Stroe : 나중에 검색 하기 위해서, split을 인덱스 하고 저장할 곳이 필요하고, 이 과정에 Vector store 과 Embedding 모델을 사용한다.
2) Retrieval and generation : 실제 RAG 체인, 유저 쿼리를 런타임에 실행하고, index로 부터 관련된 결과를 반환하여 모델에게 보내는 과정
    - Retrieve : retriever에 의해 storage에서 관련된 splits가 검색된다. 
    - Generate : Chat model 이나 LLM 은 질문과 관련된 데이터가 포함된 prompt를 사용하여 답을 생성한다. '

## Practice
#### 라이브러리 설치 
```shell
pip install langchain langchain_community langchain_chroma
pip install -qU langchain-openai
pip install langchainhub
```

#### 필요한 환경 변수 세팅 
vim .env
```shell
LANGCHAIN_TRACING_V2="true"
LANGCHAIN_API_KEY="ss"
OPENAI_API_KEY="ss"                                                                                      
```


