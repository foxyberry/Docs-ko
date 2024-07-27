#### Reference 
- https://python.langchain.com/v0.2/docs/tutorials/chatbot/

## 따라하기
#### 1. 단순 버전. Model 선언 및 질문
```python
model = ChatOpenAI(model="gpt-3.5-turbo")

```
문제점 : Model은 상태의 개념이 없음, 대화를 기억할 수 없어 챗봇으로 만들기 부적합함

#### 2. 모델이 상태를 갖게 하려면, Message History 가 필요함
    1. Message History 객체를 사용해서, 모델을 랩핑하고, 상태를 두려고 함. 
    2. input/ouput을 어떤 datastore에 저장해둠 -> 이 저장된 내용을 input으로 보냄
    3. 사용시, config를 보내주는게 필요함

```python
store = {}
def get_session_history(session_id: str) -> BaseChatMessageHistory:
    if session_id not in store:
        store[session_id] = InMemoryChatMessageHistory()
    return store[session_id]


with_message_history = RunnableWithMessageHistory(model, get_session_history)
config = {"configurable": {"session_id": "abc2"}}


response = with_message_history.invoke(
    [HumanMessage(content="Hi! I'm Bob")],
    config=config,
)

response.content
```

#### 3. ChatPromptTemplate 사용하기 
ChatPromptTemplate 을 사용하면, System 메세지를 추가할 수 있다.
```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            "You are a helpful assistant. Answer all questions to the best of your ability.",
        ),
        MessagesPlaceholder(variable_name="messages"),
    ]
)

chain = prompt | model
response = chain.invoke({"messages": [HumanMessage(content="hi! I'm bob")]})

response.content
```

#### 4. ChatPromptTemplate 와 Message History를 같이 사용할 수 있다.
- ChatPromptTemplate와  Message History의 장점을 결합할 수 있음
```python
model = ChatOpenAI(model="gpt-3.5-turbo")

prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system", 
            "You are a helpful assistant. Answer all questions to the best of your ability",
        ),
        MessagesPlaceholder(variable_name="messages"),
    ]
)
chain = prompt | model


store = {}
def get_session_history(session_id: str) -> BaseChatMessageHistory:
    if session_id not in store:
        store[session_id] = InMemoryChatMessageHistory()
    return store[session_id]

with_message_history = RunnableWithMessageHistory(chain, get_session_history)
config = {"configurable": {"session_id": "abc5"}}
response = with_message_history.invoke(
    [HumanMessage(content="Hi! I'm Jim")],
    config=config,
)
response.content
```

#### 5. system에 variable을 넣고 좀 더 복잡한 상황도 처리하도록 만들기
- 좀 더 유연한 처리를 위해 variable을 추가로 정의하고, input으로 넘길 수 있다.
```Message History

dotenv.load_dotenv()
model = ChatOpenAI(model="gpt-3.5-turbo")

prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system", 
            "You are a helpful assistant. Answer all questions to the best of your ability in {language}",
        ),
        MessagesPlaceholder(variable_name="messages"),
    ]
)
chain = prompt | model

response = chain.invoke(
    {
        "messages": [HumanMessage(content="hi! I'm bob")], 
         "language": "Korean"
    }
)

response.content
```

#### 6. 5번 환경에 MessageHistory 를 추가하여 처리하기
- 5번 상황을 처리하면서, MessageHistory 이용하여 메세지 히스토리 기능을 추가할 수 있다.
```python
dotenv.load_dotenv()
model = ChatOpenAI(model="gpt-3.5-turbo")

prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system", 
            "You are a helpful assistant. Answer all questions to the best of your ability in {language}",
        ),
        MessagesPlaceholder(variable_name="messages"),
    ]
)
chain = prompt | model

store = {}
def get_sesseion(session_id: str) -> BaseChatMessageHistory: 
    if session_id not in store: 
        store[session_id] = InMemoryChatMessageHistory()
    return store[session_id]

with_message_history = RunnableWithMessageHistory(
    chain, 
    get_session_history,
    input_messages_key = "messages",
)

config = {"configurable" : {"session_id" : "a1"}}
response = with_message_history.invoke(
    {
        "messages" : [HumanMessage(content="Hi! I'm Jim")],
        "language" : "Korean"
    },
     config=config,
)

response.content
```

#### 7.  Conversation History 를 관리하기 
- 대화가 많아지면 LLM 이 처리할 수 없을 만큼 input이 많아지기 때문에 관리가 필요함
- 메세지의 리스트를 관리하는 빌트인 헬퍼들이 있다 -> https://python.langchain.com/v0.2/docs/how_to/#messages
- 그 중에서 trim_messages 를 사용하는 모델에 보내는 수를 줄이는 방법은 다음과 같다. 
```python

dotenv.load_dotenv()
model = ChatOpenAI(model="gpt-3.5-turbo")

prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system", 
            "You are a helpful assistant. Answer all questions to the best of your ability in {language}",
        ),
        MessagesPlaceholder(variable_name="messages"),
    ]
)

trimmer = trim_messages(
    max_tokens=65,
    strategy="last",
    token_counter=model,
    include_system=True,
    allow_partial=False,
    start_on="human",
)

messages = [
    SystemMessage(content="you're a good assistant"),
    HumanMessage(content="hi! I'm bob"),
    AIMessage(content="hi!"),
    HumanMessage(content="I like vanilla ice cream"),
    AIMessage(content="nice"),
    HumanMessage(content="whats 2 + 2"),
    AIMessage(content="4"),
    HumanMessage(content="thanks"),
    AIMessage(content="no problem!"),
    HumanMessage(content="having fun?"),
    AIMessage(content="yes!"),
]

trimmer.invoke(messages)

chain = ( 
    RunnablePassthrough.assign(messages=itemgetter("messages") | trimmer) 
         | prompt 
         | model 
)

response = chain.invoke(
    {
        "messages": messages + [HumanMessage(content="what's my name?")],
        "language": "English",
    }
)
response.content

```


#### 8. 7번 상황에서 Message History를 추가하기
- 7번 상황을 처리하면서, Message History 를 사용하여 장점을 이용할 수 있다.

```python
from langchain_openai import ChatOpenAI
from langchain_core.messages import (
HumanMessage, AIMessage)
from langchain_core.chat_history import (
    BaseChatMessageHistory,
    InMemoryChatMessageHistory,
)
from langchain_core.runnables.history import RunnableWithMessageHistory
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.messages import SystemMessage, trim_messages
from langchain_core.runnables import RunnablePassthrough
import dotenv
from operator import itemgetter



dotenv.load_dotenv()
model = ChatOpenAI(model="gpt-3.5-turbo")

prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system", 
            "You are a helpful assistant. Answer all questions to the best of your ability in {language}",
        ),
        MessagesPlaceholder(variable_name="messages"),
    ]
)

trimmer = trim_messages(
    max_tokens=65,
    strategy="last",
    token_counter=model,
    include_system=True,
    allow_partial=False,
    start_on="human",
)

messages = [
    SystemMessage(content="you're a good assistant"),
    HumanMessage(content="hi! I'm bob"),
    AIMessage(content="hi!"),
    HumanMessage(content="I like vanilla ice cream"),
    AIMessage(content="nice"),
    HumanMessage(content="whats 2 + 2"),
    AIMessage(content="4"),
    HumanMessage(content="thanks"),
    AIMessage(content="no problem!"),
    HumanMessage(content="having fun?"),
    AIMessage(content="yes!"),
]



store = {}
def get_session_history(session_id: str) -> BaseChatMessageHistory: 
    if session_id not in store: 
        store[session_id] = InMemoryChatMessageHistory()
    return store[session_id]

chain = (
    RunnablePassthrough.assign(messages=itemgetter("messages") | trimmer)
    | prompt
    | model
)



with_message_history = RunnableWithMessageHistory(
    chain,
    get_session_history,
    input_messages_key="messages",
)


config = {"configurable" : {"session_id" : "a7"}}
response = with_message_history.invoke(
    {
        "messages":  messages + [HumanMessage(content="whats my name?")],
        "language": "Enlish",
    },
    config=config,
)

response.content

```
