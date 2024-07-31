#### reference 
https://python.langchain.com/v0.2/docs/tutorials/rag/

## RAG
- RAG은 추가적인 데이터로 LLM 지식을 강화하기 위한 기술이다.
- LLM은 넓은 범위의 토픽에 대해서 추론할 수 있지만, LLM은 트레인 받은 시점까지의 공개 데이터에 대해서만 알고 있다. 그래서 만약 private data 에 대해서 AI가 추론할 수 있게 하고 싶다면, 모델의 지식을 강화할 필요가 있다.`적절한 데이터를 가져오고, 모델에게 입력을 넣는 것을 RAG 이라고 한다.`

## 2가지 메인 컴포넌트
1) Indexing : 데이터를 섭취하고, 인뎅싱 하는 파이프라인. 오프라인에서 일어 난다.
     - Load : 데이터를 로딩. Documents loaders 로 수행된다. 
     - Split : 모델의 한정된 컨텍스트 윈도우 안에서 큰 chunk는 검색에 어렵기 때문에, 큰 문서를 작은 chunk로 만드는 과정이 필요하다. 데이터를 인덱싱 하고 모델로 전달하기 위해 유용한 과정이다. 
     - Stroe : 나중에 검색 하기 위해서, split을 인덱스 하고 저장할 곳이 필요하고, 이 과정에 Vector store 과 Embedding 모델을 사용한다.
2) Retrieval and generation : 실제 RAG 체인, 유저 쿼리를 런타임에 실행하고, index로 부터 관련된 결과를 반환하여 모델에게 보내는 과정
    - Retrieve : retriever에 의해 storage에서 관련된 splits가 검색된다. 
    - Generate : Chat model 이나 LLM 은 질문과 관련된 데이터가 포함된 prompt를 사용하여 답을 생성한다. '

## Practice
### 라이브러리 설치 
```shell
pip install langchain langchain_community langchain_chroma
pip install -qU langchain-openai
pip install langchainhub
```

### 필요한 환경 변수 세팅 
vim .env
```shell
LANGCHAIN_TRACING_V2=true
LANGCHAIN_ENDPOINT="https://api.smith.langchain.com"
LANGCHAIN_PROJECT="abc-v1"
LANGCHAIN_API_KEY="ld9_b7e259ca96"
OPENAI_API_KEY="sk-proj-ddyIXgoAg"
USER_AGENT=default_user_agent_string                                                                                                                                  
```

### 전체 코드 

```python
from langchain_openai import ChatOpenAI
from dotenv import load_dotenv, dotenv_values
import os
import bs4
from langchain import hub
from langchain_chroma import Chroma
from langchain_community.document_loaders import WebBaseLoader
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough
from langchain_openai import OpenAIEmbeddings
from langchain_text_splitters import RecursiveCharacterTextSplitter

### 환경변수 로드
env_path = '.env'
env_vars = dotenv_values(env_path)
for key in env_vars:
    if key in os.environ:
        del os.environ[key]

# Explicitly reload the .env file
load_dotenv(env_path)

### 모델 선언
llm = ChatOpenAI(model="gpt-4o-mini")

# Load, chunk and index the contents of the blog.
loader = WebBaseLoader(
    web_paths=("https://lilianweng.github.io/posts/2023-06-23-agent/",),
    bs_kwargs=dict(
        parse_only=bs4.SoupStrainer(
            class_=("post-content", "post-title", "post-header")
        )
    ),
)
docs = loader.load()

text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
splits = text_splitter.split_documents(docs)
vectorstore = Chroma.from_documents(documents=splits, embedding=OpenAIEmbeddings())

# Retrieve and generate using the relevant snippets of the blog.
retriever = vectorstore.as_retriever()
prompt = hub.pull("rlm/rag-prompt")


def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)


rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

rag_chain.invoke("What is Task Decomposition?")

# cleanup
vectorstore.delete_collection()
```

#### 나눠서 보기 1. Indexing: Load
```python
import bs4
from langchain_community.document_loaders import WebBaseLoader

# Only keep post title, headers, and content from the full HTML.
bs4_strainer = bs4.SoupStrainer(class_=("post-title", "post-header", "post-content"))
loader = WebBaseLoader(
    web_paths=("https://lilianweng.github.io/posts/2023-06-23-agent/",),
    bs_kwargs={"parse_only": bs4_strainer},
)
docs = loader.load()

len(docs[0].page_content)
```

- WebBaseLoader : urllib 를 사용하여 HTML를 읽는다.
- bs4 :  bs4 라이브러리는 BeautifulSoup을 포함하고 있고, BeautifulSoup은 HTML과 XML 파일을 파싱하는 데 사용되는 파이썬 라이브러리. 주로 웹 스크래핑을 위해 사용되며, 구조화되지 않은 웹 페이지 데이터를 구조화된 형태로 변환하는 데 매우 유용함.
- `bs4_strainer = bs4.SoupStrainer(class_=("post-title", "post-header", "post-content"))` :  "post-title", "post-header", "post-content" 의 내용만 읽고 싶을 경우 지정해준다.
- bs_kwargs 를 통해 parse 옵션을 설정한다.
- Document Loaders : https://python.langchain.com/v0.2/docs/integrations/document_loaders/
 
#### 나눠서 보기 2. Indexing: Split
```
from langchain_text_splitters import RecursiveCharacterTextSplitter

text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000, chunk_overlap=200, add_start_index=True
)
all_splits = text_splitter.split_documents(docs)

len(all_splits) ### 66
```


- 모델이 정보를 찾기에도 힘들고, context 윈도우 사이즈에 비해서도 너무 크기 때문에 Document를 chunk로 자르고, embedding하고 vector store에 저장해야 한다.
- 1000개의 캐릭터씩 자르고, chunk 사이에 200자가 겹치도록 설정함.
- RecursiveCharacterTextSplitter : 각각의 청크가 적절한 사이즈가 될 때까지, 뉴라인 같은 흔한 구분자를 사용하여 문서를 반복적으로 자른다. 추천되는 문자열 자르기이다. 
- add_start_index=True : 문자 인덱스가 메타데이터 속성 "start_index"로 유지되게 설정
- Document transformers : https://python.langchain.com/v0.2/docs/integrations/document_transformers/

#### 나눠서 보기 3. Indexing: Store
```
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings

vectorstore = Chroma.from_documents(documents=all_splits, embedding=OpenAIEmbeddings())
```
- runtime에서 검색하려면 66개의 청크를 인덱싱 해야 한다.
- 청크를 검색하는 흔한 방법 중 하나는 컨텐츠들을 청크로 나눠 embed하고, 그 embeding 된 결과를 벡터 디비에 넣어두고, 우리가 원하는 것을 찾기 위해서는 그 쿼리를 다시 임베딩하여 유사도 검색을 수행하는것이다.
- 가장 간단한 유사도 측정은 cosine similarity 이다. (쌍들의 코싸인 각도를 측정)
- 위의 코드는 OpenAIEmbeddings model를 이용하여 임베딩하고, 결과를 Chroma 벡터 디비에 저장한다.
- Embedding model : https://python.langchain.com/v0.2/docs/integrations/text_embedding/
- Vector stores: https://python.langchain.com/v0.2/docs/integrations/vectorstores/

#### 나눠서 보기 4. Retrieval and Generation: Retrieve
```
retriever = vectorstore.as_retriever(search_type="similarity", search_kwargs={"k": 6})

retrieved_docs = retriever.invoke("What are the approaches to Task Decomposition?")

len(retrieved_docs)

```
- VectorStoreRetriever : 흔하게 사용되는 Retriever 로써, 벡터 스토에서 유사성 검색을 수행한다.
- MultiQueryRetriever : retriever의 hit rate를 높이기 위해서, 질문의 변형을 생성함
- MultiVectorRetriever : embedding의 변형을 생성함
- Max marginal relevance 
- Retrievers : https://python.langchain.com/v0.2/docs/integrations/retrievers/
  
#### 나눠서 보기 5. Retrieval and Generation: Generate
```python
from langchain import hub
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough


prompt = hub.pull("rlm/rag-prompt")

example_messages = prompt.invoke(
    {"context": "filler context", "question": "filler question"}
).to_messages()


def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)


rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

for chunk in rag_chain.stream("What is Task Decomposition?"):
    print(chunk, end="", flush=True)
```



#### 나눠서 보기   Built-in chains
