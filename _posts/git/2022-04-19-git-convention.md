---
title: "[Git] Git 메시지 컨벤션이란?"

categories: git

toc: true
toc_sticky: git

date: 2022-04-19
last_modified_at: 2022-04-19
---

# Git - Commit Message Convention

커밋 메시지를 작성할 때는 원칙을 정하고 일관성있게 정해야 협업간에 쉬운 의사소통이 가능하다. 이번에 기회가 되면 정리하고자 하던 깃 메세지 컨벤션을 정리해 보았다.  
아직 깃커밋 메시지가 익숙하지 않아 내가 참고하려고 정리를 다시 해보았다.  
이런 약속은 사소하지만 정말 중요한것 같다.

## 1. 깃 메시지 구조

깃메시지는 아래와 같은 구조로 이루어져 있다.

```text
[타입(type): 제목(title, subject)]

본문내용(body)

꼬리말(footer)
```

## 2. [타입(type): 제목(title, subject)]

### 2-1 Type

타입에 들어가는 형태는 다음과 같다.

|       Type        | 설명                         |
|:-----------------:|----------------------------|
|       Feat        | 새로운 기능을 추가                 |
|        Fix        | 버그 수정                      |
|       Docs        | 문서 수정                      |
|       Style       | 포맷, 세미클론, 코드 수정이 없는 경우     |
|     Refactor      | 코드 리팩토링                    |
|       Test        | 테스트 코드 작성, 리팩토링 테스트 코드 추가  |
|       Chore       | 빌드 업무 수정, 패키지 매니저 수정       |
|      Design       | CSS등 사용자 UI 변경             |
| !BREAKING CHANGE  | 커다란 API의 변경                |
|      !HOTFIX      | 급하게 치명적인 버그 고쳐야 하는 경우      |
|      Comment      | 주석 추가 변경                   |
|      Rename       | 파일 또는 폴더명 수정하거나 옮기는 경우     |
|      Remove       | 파일을 삭제하는 작업만 수행한 경우        |

### 2-2 Title, Subject

- 제목은 50자를 넘기지 않는다.
- 대문자로 작성하며 마침표를 붙이지 않는다.
- 과거시제보다 현재시제를 사용하며, 명령어로 작성한다.

> Fix: callback error  
> Refactor: UserController refactoring  
> Feat : add user login   

위와 같은 형식으로 사용하면 된다.

## 3. 본문내용(body)

- 커밋 본문 내용은 선택사항이기 때문에 모든 커밋에 본문 내용을 작성할 필요는 없다.
- 바디 내용은 어떻게 보다 '무엇을' 그리고 '왜'에 대한 내용을 설명한다.
- 72자를 넘기지 않고 제목과 구분하기 위해 한 칸을 띄어 작성한다.

## 4. 꼬리말(footer)

- Footer는 선택사항이기 때문에 모든 커밋에 꼬리말을 작성할 필요는 없다.
- Issue tracker ID를 작성할 때 사용한다.
