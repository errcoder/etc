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
