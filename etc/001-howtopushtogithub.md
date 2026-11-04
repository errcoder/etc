```
echo "# etc" >> README.md
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/errcoder/etc.git
git push -u origin main

…or push an existing repository from the command line
git remote add origin https://github.com/errcoder/etc.git
git branch -M main
git push -u origin main
```

가장 쉽고 직관적인 개인 액세스 토큰(PAT) 발급 및 등록 방법으로 해결할 수 있습니다. 

1단계: GitHub에서 토큰(Token) 발급받기

1. GitHub 홈페이지에 로그인 후, 우측 상단 프로필 사진 클릭 ➡️ Settings로 이동합니다. 
2. 왼쪽 사이드바 맨 아래에 있는 Developer settings를 클릭합니다.
3. Personal access tokens ➡️ Tokens (classic)를 차례로 선택합니다.
4. 우측 상단의 Generate new token ➡️ Generate new token (classic)를 클릭합니다.
5. 설정값을 입력합니다:
	• Note: 토큰의 용도 (예: my-mac-terminal)
	• Expiration: 토큰 유효 기간 (원하는 기간 선택)
	• Select scopes: repo 항목에 체크 (리포지토리에 push하기 위해 반드시 필요).
7. ⚠️ 중요: 생성된 토큰 값(ghp_...로 시작하는 긴 문자열)을 복사하여 메모장에 따로 저장해 두세요. 이 페이지를 벗어나면 토큰 값을 다시는 확인할 수 없습니다.

2단계: 터미널에 토큰 반영하기 (가장 확실한 방법)

터미널에서 원격 저장소 URL에 토큰을 직접 심어주면 매번 인증 정보를 입력하지 않아도 됩니다. 터미널(Git Bash 등)을 열고 아래 명령어를 입력하세요.
```bash
# 기존에 등록된 원격 저장소 주소에 토큰을 심어서 업데이트합니다.
git remote set-url origin https://<내_깃허브_아이디>:<발급받은_토큰>@github.com/<유저이름>/<저장소이름>.git
```
• 예시: 아이디가 errcoder, 토큰이 ghp_ABC123, 저장소가 etc라면:
```
git remote set-url origin https://errcoder:ghp_ABC123@github.com/errcoder/etc.git
```
설정을 마친 후 다시 push를 시도하면 정상적으로 작동합니다.
```bash
git push -u origin main
```

---

---

이미 Git에 커밋된 파일은 .gitignore에 추가하는 것만으로는 추적(Tracking)이 해제되지 않습니다. 
Git의 캐시(Index)에서 해당 파일을 제거해야 더 이상 커밋 대상에서 제외할 수 있습니다. 

[1](https://kimjingo.tistory.com/251)
[2](https://junlab.tistory.com/237), 
[3](https://shstudio.kr/git-gitignore-%EC%9E%91%EC%84%B1%EB%B2%95%EA%B3%BC-%EC%9D%B4%EB%AF%B8-%EC%BB%A4%EB%B0%8B%EB%90%9C-%ED%8C%8C%EC%9D%BC-%EC%A0%9C%EC%99%B8%ED%95%98%EB%8A%94-%EB%B0%A9%EB%B2%95/)

### 해결 방법

### 1. 특정 파일 하나만 제외할 때

1. .gitignore 파일에 무시할 파일명을 추가합니다.
2. 터미널에서 아래 명령어로 Git 캐시에서만 파일을 삭제합니다 (로컬에 있는 실제 파일은 지워지지 않습니다):
```bash
git rm --cached [파일명]
```
(예: git rm --cached config.json)

3. 변경 사항을 커밋합니다: [1](https://kimjingo.tistory.com/251)
```bash
git add .
git commit -m "Stop tracking file"
```

### 2. 전체 캐시를 초기화하고 .gitignore 규칙을 다시 적용할 때 (폴더나 여러 파일)

1. .gitignore 파일에 무시할 파일이나 폴더를 추가합니다.
2. 저장소의 Git 캐시를 전부 지운 뒤, .gitignore 규칙을 반영하여 다시 추가합니다: [1](https://kimjingo.tistory.com/251), [2](https://junlab.tistory.com/237)
```bash
git rm -r --cached .
git add .
git commit -m "Apply .gitignore and untrack files"
```
자세한 내용은 Git에서 이미 커밋된 파일 무시하기 블로그 또는 Git gitignore 작성법과 이미 커밋된 파일 제외하는 방법에서 참고하실 수 있습니다. 
[1](https://kimjingo.tistory.com/251), [2](https://shstudio.kr/git-gitignore-%EC%9E%91%EC%84%B1%EB%B2%95%EA%B3%BC-%EC%9D%B4%EB%AF%B8-%EC%BB%A4%EB%B0%8B%EB%90%9C-%ED%8C%8C%EC%9D%BC-%EC%A0%9C%EC%99%B8%ED%95%98%EB%8A%94-%EB%B0%A9%EB%B2%95/)

---

---

제외하려는 대상이 특정 폴더 전체일 때는 폴더 내의 모든 파일을 하위 디렉터리까지 재귀적으로 처리할 수 있도록 -r(recursive) 옵션을 추가해야 합니다.
아래 순서대로 터미널에 명령어를 입력하시면 됩니다.

특정 폴더 전체 추적 제외하기

1. .gitignore 파일에 폴더 추가
.gitignore 파일에 제외할 폴더명을 등록합니다. (폴더명 뒤에 /를 붙여주는 것이 좋습니다.)

(예: node_modules/ 또는 dist/)

2. Git 캐시에서 폴더 제거
로컬의 실제 폴더와 파일은 그대로 둔 채, Git이 추적하는 캐시(Index)에서만 해당 폴더를 삭제합니다.
```bash
git rm -r --cached [폴더명]
```
(예: git rm -r --cached dist)

3. 변경 사항 커밋
캐시에서 제거된 상태를 커밋하여 원격 저장소(GitHub 등)에도 반영합니다.
```bash
git add .
git commit -m "Remove [폴더명] from version control"
```
이 단계를 마치면 앞으로 해당 폴더 안의 파일이 수정되더라도 git status나 커밋 대상에 포함되지 않습니다. 
만약 이미 원격 저장소에 올라가 있던 폴더였다면, 이 커밋을 git push하는 순간 원격 저장소에서도 해당 폴더가 삭제됩니다.
---
