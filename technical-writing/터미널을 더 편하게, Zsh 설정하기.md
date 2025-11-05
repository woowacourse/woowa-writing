## 들어가며

개발을 할 때면 IDE를 주로 사용하지만 터미널을 써야 하는 경우가 꼭 있다. Git 명령어를 실행하거나 패키지를 설치하거나, 서버를 실행할 때처럼 말이다. 그런데 기본 터미널은 솔직히 좀 불편하다. 자동완성도 별로고, 보기에도 심심하다.

그래서 이번에는 터미널 환경을 훨씬 쓸 만하게 만들어주는 Zsh와 그 설정 파일인 .zshrc를 다뤄보려고 한다. 한 번 제대로 설정해두면 개발 생산성이 확실히 올라가는 걸 느낄 수 있을 것이다.

## 셸이 뭔데?

이번에 살펴볼 Zsh는 셸이다. 먼저 셸이 무엇인지, 그리고 어떤 종류가 있는지 알아보자.

셸은 사용자와 운영체제 사이의 중간 다리 역할을 하는 프로그램이다. 쉽게 말해 사용자가 컴퓨터와 소통할 수 있게 해주는 명령어 해석기다. 운영체제의 핵심인 커널을 감싸고 있는 껍데기와 같다고 해서 셸이라는 이름이 붙었다. 셸 덕분에 사용자가 직접 커널에 접근하지 않고, 셸을 통해 간접적으로 접근할 수 있는 것이다.

동작 방식을 간단히 표현하면 이렇다: 사용자 → 셸 → 운영체제(커널) → 하드웨어

### 주요 셸 종류들

1. Bash(Bourne Again Shell) : Linux 배포판의 기본 셸로, 가장 널리 사용되고 호환성이 좋다. 웬만한 Linux 서버에서는 Bash를 쓴다고 보면 된다.
2. Zsh(Z Shell) : Bash와 호환되면서 더 많은 기능을 제공한다. macOS Catalina부터 기본 셸로 채택될 정도로 요즘 대세이다. 뛰어난 자동완성, 강력한 글로빙 패턴, 그리고 테마와 플러그인 생태계(Oh My Zsh)가 큰 장점이다.
3. Fish(Friendly Interactive Shell) : 이름 그대로 사용자 친화적인 설계가 특징이다. 실시간 자동완성, 문법 강조, 직관적인 구문, 웹 기반 설정 인터페이스를 제공한다. 다만 Bash와 문법이 달라서 기존 스크립트 호환성은 떨어진다.
4. PowerShell : Microsoft에서 개발한 셸로, .NET 기반으로 객체 지향적이다. Windows 환경에서 강력하지만, Unix 계열과는 사뭇 다른 접근 방식을 취한다.

이 중에서 이번 글에서는 Zsh를 기준으로 다룰 것이다.

## .zshrc 파일로 내 터미널 꾸미기

### .zshrc가 뭐길래?

.zshrc 파일은 Zsh의 설정 파일이다. Zsh 셸이 시작될 때마다 자동으로 실행되는 스크립트 파일로, 셸 환경을 사용자가 원하는 대로 설정하고 커스터마이징할 수 있다.

이 파일은 홈 디렉토리(`~`)에 위치하며, 파일명 앞의 점(`.`)은 숨김 파일임을 의미한다. 그래서 Finder에서는 기본적으로 보이지 않는다. 터미널에서는 `ls -a` 명령어를 입력해 숨김 파일까지 모두 볼 수 있고, Finder에서는 `Command + Shift + .`을 눌러 모두 볼 수 있다.

### zsh 설정파일 열기

.zshrc 파일을 수정하려면 먼저 파일을 열어야 한다. 여러 가지 방법이 있는데, 자신이 편한 에디터를 사용하면 된다.

1. nano 에디터로 열기

   ```bash
   nano ~/.zshrc
   ```

   nano는 가벼운 터미널 기반 텍스트 에디터다. 간단한 수정에는 이게 가장 편하다. 저장은 `Control + O`, 종료는 `Control + X`이다.

2. vim 에디터로 열기

   ```bash
   vim ~/.zshrc
   ```

   vim에 익숙하다면 이 방법도 좋다. 다만 vim은 학습 곡선이 있어 처음 사용하는 사람들에게는 조금 어려울 수 있다.

3. VSCode 에디터로 열기

   ```bash
   code ~/.zshrc
   ```

   개인적으로는 이 방법을 제일 선호한다. 익숙한 에디터에서 편하게 수정할 수 있고, 문법 강조도 되고, 자동완성도 된다.

   만약 `code` 명령어로 파일이 열리지 않는다면? VSCode에서 `Command + Shift + P`를 누른 후 "Shell Command: install 'code' command in PATH"를 검색해서 실행해주자. 그럼 터미널에서 code 명령어를 사용할 수 있게 된다.

   ![image.png](%ED%84%B0%EB%AF%B8%EB%84%90%EC%9D%84%20%EB%8D%94%20%ED%8E%B8%ED%95%98%EA%B2%8C,%20Zsh%20%EC%84%A4%EC%A0%95%ED%95%98%EA%B8%B0%2025189de02fde80df9adec2dc0d6a6c10/image.png)

   VSCode 기반의 Cursor로도 물론 실행이 가능하다. Cursor를 터미널에서 실행하기 위해서는 [`Cursor CLI`](https://cursor.com/cli)를 설치해주어야 한다.

   ```bash
   curl https://cursor.com/install -fsS | bash
   ```

### 설정 파일 적용하기

.zshrc 파일은 수정한 후에는 변경사항을 적용해야 한다. 터미널을 껐다 켜도 되지만, 더 간단한 방법이 있다.

```bash
source ~/.zshrc
```

source는 현재 셸 세션에서 설정 파일을 다시 읽어들인다. 수정한 내용이 바로 반영되는 걸 확인할 수 있다. 터미널을 여러 개 띄어놨다면 각 터미널마다 이 명령어를 실행해줘야 한다는 점만 기억하자.

## 실전 활용: 생산성 높이는 설정들

### 단축키 지정하기 (alias)

자주 사용하는 명령어를 매번 길게 타이핑하는 건 여간 귀찮은 일이 아니다. `alias`를 이용하면 긴 명령어를 짧은 단축키로 만들 수 있다.

```bash
# .zshrc
alias cc="claude --dangerously-skip-permissions"
alias ls="eza --icons --tree --level=1"
alias vi="nvim"
alias gs="git status"
alias gp="git push"
alias gc="git commit"
alias ll="ls -la"
```

특히 요즘 자주 사용하는 명령어는 위의 `cc`로, `claude --dangerously-skip-permissions`를 간편하게 실행할 수 있다. Claude Code를 쓸 때 매번 긴 옵션을 입력하지 않아도 되니 정말 편하다.

![image.png](%ED%84%B0%EB%AF%B8%EB%84%90%EC%9D%84%20%EB%8D%94%20%ED%8E%B8%ED%95%98%EA%B2%8C,%20Zsh%20%EC%84%A4%EC%A0%95%ED%95%98%EA%B8%B0%2025189de02fde80df9adec2dc0d6a6c10/image%201.png)

`ls` 명령어를 `eza --icons`로 대체한 것도 실용적이다. [eza](https://github.com/eza-community/eza)는 ls의 모던한 대안으로, 색상과 아이콘을 지원해서 파일 목록을 한눈에 보기 쉽다. 물론 eza를 먼저 [설치](https://github.com/eza-community/eza/blob/main/INSTALL.md)해야 한다. [다양한 옵션](https://github.com/eza-community/eza?tab=readme-ov-file#command-line-options)을 설정해서 파일을 원하는 구조로 볼 수도 있다.

![image.png](%ED%84%B0%EB%AF%B8%EB%84%90%EC%9D%84%20%EB%8D%94%20%ED%8E%B8%ED%95%98%EA%B2%8C,%20Zsh%20%EC%84%A4%EC%A0%95%ED%95%98%EA%B8%B0%2025189de02fde80df9adec2dc0d6a6c10/image%202.png)

### 커스텀 함수 만들기

alias보다 더 복잡한 작업이 필요할 때는 함수를 만들 수 있다. 예를 들어 특정 포트를 사용 중인 프로세스를 죽이는 작업은 개발하면서 자주 겪는 상황이다.

3000, 8080 포트에서 '해당 포트에서 이미 실행 중이어서 실행할 수 없습니다.'라는 에러 메시지를 본 적이 있을 것이다. 그러한 경우 .zshrc에 함수를 미리 설정해두면 간편하게 해당 포트의 서버를 종료할 수 있다.

```bash
# .zshrc
kp() {
    if [ -z "$1" ]; then
        echo "사용법: kp <포트번호>"
        return 1
    fi

    local pids=$(lsof -ti:$1)
    if [ -n "$pids" ]; then
        echo "포트 $1에서 실행 중인 프로세스들을 종료합니다..."
        echo $pids | xargs kill -9
        echo "완료!"
    else
        echo "포트 $1에서 실행 중인 프로세스가 없습니다."
    fi
}

```

이제 `kp 3000`만 입력하면 3000 포트를 사용 중인 프로세스를 바로 종료할 수 있다. 일일이 프로세스 ID를 찾아서 kill 명령어를 실행하는 번거로움이 사라진다.

비슷한 방식으로 자주 사용하는 작업 흐름을 함수로 만들어둘 수 있다. 예를 들면 이런 것들이다:

```bash
# Git 브랜치 생성하고 바로 전환
gcb() {
    git checkout -b $1
}

# 프로젝트 디렉토리로 빠르게 이동
proj() {
    cd ~/Projects/$1
}

# Docker 컨테이너 일괄 정리
dclean() {
    echo "사용하지 않는 Docker 리소스를 정리합니다..."
    docker system prune -af
    echo "완료!"
}

```

## 더 나아가기: Oh My Zsh와 플러그인

Zsh를 더 강력하게 만들어주는 도구가 바로 **Oh My Zsh**다. 테마와 플러그인을 쉽게 관리할 수 있게 해주는 프레임워크인데, 많은 개발자들이 사용하고 있다.

### [Oh My Zsh](https://ohmyz.sh/) 설치

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

```

설치하면 .zshrc 파일에 Oh My Zsh 관련 설정이 자동으로 추가된다.

### 유용한 플러그인들

Oh My Zsh의 진가는 [플러그인](https://github.com/ohmyzsh/ohmyzsh/wiki/plugins)에 있다. .zshrc 파일에서 `plugins=()` 부분을 찾아서 원하는 플러그인을 추가하면 된다.

```bash
plugins=(
  zsh-autosuggestions
  zsh-syntax-highlighting
  git
  docker
  npm
)

```

1. [**zsh-autosuggestions**](https://github.com/zsh-users/zsh-autosuggestions)는 정말 강력하다. 이전에 입력했던 명령어를 회색 글씨로 자동 제안해준다. 방향키 오른쪽을 누르면 바로 채워지므로 반복적인 명령어 입력이 훨씬 빨라진다.

   ![image.png](%ED%84%B0%EB%AF%B8%EB%84%90%EC%9D%84%20%EB%8D%94%20%ED%8E%B8%ED%95%98%EA%B2%8C,%20Zsh%20%EC%84%A4%EC%A0%95%ED%95%98%EA%B8%B0%2025189de02fde80df9adec2dc0d6a6c10/image%203.png)

2. [**zsh-syntax-highlighting**](https://github.com/zsh-users/zsh-syntax-highlighting)은 명령어 문법을 실시간으로 강조 표시해준다. 올바른 명령어는 초록색, 잘못된 명령어는 빨간색으로 표시되니까 엔터 누르기 전에 오타를 발견할 수 있다.

   ![image.png](%ED%84%B0%EB%AF%B8%EB%84%90%EC%9D%84%20%EB%8D%94%20%ED%8E%B8%ED%95%98%EA%B2%8C,%20Zsh%20%EC%84%A4%EC%A0%95%ED%95%98%EA%B8%B0%2025189de02fde80df9adec2dc0d6a6c10/image%204.png)

3. [git](https://github.com/ohmyzsh/ohmyzsh/tree/master/plugins/git), [docker](https://github.com/ohmyzsh/ohmyzsh/tree/master/plugins/docker), [npm](https://github.com/ohmyzsh/ohmyzsh/tree/master/plugins/npm)은 모두 별칭과 여러 가지 함수들을 제공해주는 플러그인이다.

### 테마 적용하기

Oh My Zsh는 다양한 테마도 제공한다. .zshrc 파일에서 `ZSH_THEME="robbyrussell"` 부분을 찾아서 원하는 테마로 변경하면 된다.

```bash
ZSH_THEME="agnoster"  # 인기 있는 테마
# 또는
ZSH_THEME="powerlevel10k/powerlevel10k"  # 많은 커스터마이징 옵션 제공

```

개인적으로는 powerlevel10k 테마를 추천한다. 처음 설정할 때 대화형 마법사가 나와서 취향대로 쉽게 꾸밀 수 있고, Git 정보나 실행 시간 같은 유용한 정보를 프롬프트에 표시해준다.

### 그 외 다양한 명령어

그 외 명령어는 [zsh 명령어](https://www.notion.so/zsh-26e89de02fde807b8eb3df33dab59ecc?pvs=21) 페이지를 참고 바란다.

## 실수하지 않기 위한 팁들

### 백업은 필수

.zshrc 파일을 수정하기 전에 백업 복사본을 만들어두는 게 좋다. 실수로 문법 오류가 생기면 터미널이 제대로 작동하지 않을 수 있다.

```bash
cp ~/.zshrc ~/.zshrc.backup
```

문제가 생기면 백업 파일로 복구하면 된다:

```bash
cp ~/.zshrc.backup ~/.zshrc
source ~/.zshrc
```

### 설정 파일 테스트하기

.zshrc 파일에 큰 변경을 가할 때는 새 터미널 창을 열어서 테스트해보자. 기존 터미널 창은 그대로 두고 말이다. 만약 문제가 생기더라도 기존 터미널에서 수정할 수 있다.

### 주석 활용하기

시간이 지나면 자신이 왜 특정 설정을 추가했는지 잊어버리기 쉽다. 주석을 달아두면 나중에 관리하기 편하다.

```bash

# Android 경로 설정
export ANDROID_HOME=/Users/home/Library/Android/sdk
export PATH=$PATH:$ANDROID_HOME/emulator
export PATH=$PATH:$ANDROID_HOME/tools
export PATH=$PATH:$ANDROID_HOME/tools/bin
export PATH=$PATH:$ANDROID_HOME/platform-tools
```

## 맺으며

처음 터미널을 접했을 때는 검은 화면에 하얀 글씨만 있는 게 왠지 어렵게 느껴졌다. VSCode 같은 IDE나 GitHub Desktop 같은 GUI 도구들이 훨씬 친근하고 편해 보였고, 실제로도 그랬다. 그래서 한동안은 정말 필요한 경우가 아니면 터미널을 피하곤 했다.

하지만 개발을 하다 보니 어느 순간부터 터미널을 자연스럽게 쓰고 있는 나를 발견했다. Git 커밋을 하거나, 패키지를 설치하거나, 서버를 실행하는 것처럼 반복적인 작업들은 터미널에서 명령어 몇 줄로 처리하는 게 오히려 더 빠르고 정확하다는 걸 알게 됐다. 특히 여러 프로젝트를 동시에 다루거나, 배포 스크립트를 실행하거나, 로그를 확인할 때는 터미널만 한 게 없었다.

그렇게 터미널을 쓰다 보니 조금씩 욕심이 생겼다. '이왕 쓰는 거 좀 더 편하게 쓸 수는 없을까?' 하는 생각이 들었고, 그게 바로 이번 글에서 다룬 .zshrc 설정의 시작이었다. 자주 쓰는 명령어에 alias를 걸어두고, 반복되는 작업은 함수로 만들어두면서 조금씩 내 터미널을 내 손에 맞게 다듬어갔다.

지금은 터미널을 열면 자동으로 내가 원하는 환경이 세팅돼 있고, `cc` 한 번으로 Claude Code가 실행되고, `kp 3000`으로 귀찮은 포트 충돌을 해결한다. 예전 같았으면 구글링하며 시간을 허비했을 작업들이 이제는 손가락 몇 번 움직임으로 끝난다. 이런 작은 최적화들이 모여서 하루 개발 시간이 생각보다 많이 단축됐다.

개발자로서 성장한다는 건 결국 이런 작은 것들의 축적이 아닐까. 더 나은 도구를 찾고, 더 효율적인 방법을 익히고, 조금씩 내 환경을 개선해나가는 것. .zshrc 설정은 그런 여정의 좋은 시작점이 될 수 있다.
