### Jupyterlab에 Python 커널 추가하기

1. 가상 환경 만들기
```
   python3 -m venv langchain-env 
   source langchain-env/bin/activate
```
2. JupyterLab과 ipykernel 설치:
```
pip install jupyterlab ipykernel
```

3. 가상 환경을 Jupyter 커널로 추가:
```
python -m ipykernel install --user --name=langchain-env --display-name "Python (langchain-env)"
```

4. Jupyterlab 을 다시 수행하기


### 디렉토리별 환경 설정을 자동으로 로드하게 설정

```
brew install direnv
```

```
echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc
source ~/.zshrc
```

```
cd path/to/your/project
echo 'source langchain-env/bin/activate' > .envrc
direnv allow
```

