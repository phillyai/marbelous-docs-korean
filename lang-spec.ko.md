# Marbelous Spec

## Overview

Marbelous 는 구슬의 움직임을 중심으로 돌아간다.

Marbelous 프로그램은 하나 이상의 직사각형 **보드**로 구성된다. 각 보드들은 두글자 **셀**들로 나눠진다.

프로그램의 전체 흐름은 **구슬**들의 움직임이나 8비트 값들로 돌아간다. 모든 구슬들은 자연히 같은 속도로 보드의 아래로 떨어진다. 프로그램은 **틱**이라는 단계로 돌아간다.

만약 구슬이 보드의 바닥으로 떨어지면 대응하는 글자(ASCII)가 STDOUT으로 출력된다. 만약 여러개의 구슬이 한 보드의 바닥으로 같은 틱에 떨어진다면, 왼쪽에서 오른쪽으로 출력이 된다. 만약 구슬이 보드의 측면으로 떨어지면 보드의 반대쪽 면으로 이동해 순환하거나 사라진다. (Undefined Behaviour)

몇몇 셀들은 다른 **장치**나 다른 보드의 호출을 포함할 수 있다. 이는 구슬들은 왼쪽이나 오른쪽으로 무조건적이나 특정 조건 아래에 움직일 수 있거나 셀을 통과하는 구슬의 값을 변경할 수 있다. (장치들에 관해서는 부록 1 참고)

모든 덧셈은 256의 모듈로로 계산된다. 음수는 0과 255사이가 될 때까지 256을 더한다.

## Section 1 - Boards

각 Marbelous 프로그램은 프로그램이 시작할때 실행되는  최소한의 하나의 보드가 있다. 이 보드는 메인 보드이고 `MB`라는 이름을 가지고 있다.

보드는 **틱**이라 불리는 타임 스텝으로 돌아간다. 한 틱동안 각 고정되지 않은 (부록 1, synchronisers 참고) 구슬은 한 셀씩 아래로 떨어진다. 만약 여러개의 구슬이 틱이 끝날때 같은 셀에 위치하게 되면 그들은 합쳐진다. (구슬들의 값의 합이 새 구슬의 합이 된다.)

```
01 .. # 16진수 리터럴 (01)
.. 02 # 16진수 리터럴 (02)
.. // # 왼쪽으로 이동

# 결과는:
틱  1      2      3
    01 ..  .. ..  .. ..
    .. 02  01 ..  .. ..
    .. //  .. 02  03 // # 틱3에서 01과 02는 합쳐진다
```

메인 보드를 포함한 각 보드들은 입력과 출력이 있다. 36개의 a-z 0-9 글자중 하나인 n에 대해 입력은 `}n`으로 표시되고 출력은 `{n`, `{v`, 또는 `{>`로 표시된다.

각 보드의 실행의 처음은 모든 입력 장치 (`}n`)가 보드의 `n`번째 입력을 값으로 가지는 구슬로 대체된다. 만약 같은 `n`을 장치가 하나 이상이면 같은 값의 구슬로 대체된다.

```
# 05, 03, 02가 각각 입력 0, 1, 2로 왔으면:
}0 }2 }1
}2 .. }1
.. .. ..

# 틱1의 시작 부분에서 보이는 결과는:
05 02 03
02 .. 03
.. .. ..
```

모든 출력 장치 (`{0`, `{D`, `{>` 등)가 구슬을 포함한다면 보드는 **종료**된다. 보드가 종료되면 각 타입의 출력 장치의 모든 구슬들은 합쳐져서 보드의 출력으로 계산된다.

```
# 1을 입력0으로 넘긴다.
}0 .. 32
{0 .. {0

# 결과는
틱  0         1
  01 .. 32  .. .. ..
  {0 .. {0  01 .. 32 # 모든 출력이 채워졌기에 보드는 종료된다.
  
출력 0: (01 + 32) = 33
```

또한 보드는 한 틱동안 아무 움직임이 없어도 종료된다.

```
24 .. 
.. ..

# 결과는
틱  0      1     2     3
    24 ..  .. .. .. .. .. ..
    .. ..  24 .. .. .. .. .. # 움직임이 없으니까 보드는 종료된다.

STDOUT: $ (값=0x24=36)
```

메인 보드의 첫 실행이 종료되면 Marbelous 프로그램은 종료된다. 반환 값은 보드에 `{0`가 있다면 그 값으로, 없다면 0이 될 것이다.

첫 보드를 제외한 모든 보드는 이름이 있어야 한다. 만약 첫 보드가 이름이 없다면 `MB`라는 이름을 가지게 된다. 보드의 이름은 `:`로 시작하는 라인에서 `:` 다음에 온다. 이 라인은 보드의 셀의 첫 라인의 바로 앞에  나타나야 한다. 보드 이름은 공백이 없는 출력가능한 ASCII 글자들로 이루어져야 한다.

보드의 이름의 길이는 `2 * max(1, N+1, M+1)` 이하여야 한다. `N`은 가장 큰 입력 번호 (`}5`는 5이고 `}A`는 10)이고 `M`은 가장 큰 출력 번호 (단, `{<`와 `{>`는 세지 않는다).

보드의 **실제 이름**은 보드에게 주어진 이름이 반복되거나 잘려서 정확히 `2 * max(1, N+1, M+1)` 의 길이를 가지게 된다.

만약 한 파일의 여러개의 보드가 같은 이름을 가진다면 (실제 이름) 그 이름의 마지막 보드가 이름을 사용하게 된다.

## Section 2 - 셀과 장치

셀은 보드의 기본적인 부분이다. 셀은 대부분 두개의 공백이 아닌 글자로 작성된다.

```
셀의 예제:
++  Ab  BA  ^0  ~~  @D  @@  >4  ..
```

셀은 공백으로 분리된다. 이는 선택적이지만 추천된다.

```
4A .. .. 3D
# 위 아래 셀은 둘 다 유효하고 같다
4A....3D
```

말했던 두 글자는 다음중 하나를 나타내야 한다:

1. 16진수 리터럴: 대문자인 두 16진수 글자로 쓰인다. 틱1에 셀의 위치에 있는 구슬을 나타낸다. 16진수 리터럴의 예시:

```
BA  30  29  54  7b # 마지막 셀은 리터럴이 아님!
```

2. 빈 셀: 보통 두 점으로 쓰이는 `..`로 빈 셀을 나타낸다.

```
..  ..  ..
```

만약 셀 사이의 공백이 사용되지 않으면 두개의 공백도 빈 셀을 나타내는데 사용될 수 있다.

```
......  .. # 5개의 빈 셀
```

이 두 스타일을 섞어 쓰는 것은 허용되지만 권장하지 않는다.

3. 장치: 장치는 통과하는 구슬의 방향이나 값을 변경한다. 모든 장치들은 부록 1을 참고하자.

```
/\ \/ -- +D ?7 # 모두 유효한 장치
```

4. 다른 보드 호출: 보드 호출에 대한 자세한 내용은 Section 3을 참고. 이 섹션의 시작 부분에 있는 셀 예제에서 다음은 다른 보드를 한다:

```
Ab @@ # Ab@@란 이름의 보드를 호출하거나
      # Ab와 @@의 보드를 호출하거나
```

만약 어떤 보드를 쓸지 모호점이 있다면 (예를 들어, `ab cd ef`는 `ab`, `abcd`, `cdef`과 `ef`란 이름의 보드들이 있을 때), 가장 긴 매칭되는 이름이 사용된다 (여기서는 `abcd`). 그리고 남은 글자들에 대해서도 가장 긴 매칭되는 이름이 사용되고 (`ef`), 계속 된다. 그러므로 `ab cd ef`는 `abcd`를 호출하고 `ef`를 호출함을 나타낸다.

만약 셀이 어떤걸 나타내는지 모호하다면 위 리스트의 아이템중 가장 첫번째로 매칭되는 규칙을 사용한다.

## Section 3 - Calling Boards

보드는 다른 보드에서 호출될 수 있다. 호출은 정확히 `max(1, M+1, N+1)` 개의 인접한 셀들이 필요하다. `M`은 보드에서 사용되는 입력중 가장 큰 입력 숫자이고 (`}D`는 `M>=14`), `N`은 사용되는 출력중 가장 큰 출력 숫자이다 (왼쪽/오른쪽 출력은 세지 않는다).

이 셀들은 공간을 채우는 데 필요한만큼 호출되거나 절단되거나 반복되는 해당 보드의 이름으로 채워진다.

예를 들어서:

```
:a # 3개의 입력 보드, }0, }1 은 사용되지 않음
}2
:bBc # 2개의 입력, 1개의 출력 보드.
}0}1}0
{0{0{0
:bC # 입력/출력 없음
..
:bD # 왼쪽 출력만 있음
32
{<
:MB
aa aa aa # a 호출
bB cb .. # bBc 호출
bc .. .. # bC 호출
bD .. .. # bD 호출
```

잘리거나 반복으로 일어나는 충돌의 경우는 같은 파일의 가장 마지막으로 정의된 보드 (혹은 현재 파일에 없다면 가장 마지막에 포함된 파일)가 사용된다.

이 보드들은 입력이 가득 찰 때까지 호출되지 않으며,  호출이 이뤄질 때 까지 `synchroniser` 처럼 작동한다 (부록 1, synchroniser 참고). 첫번째 셀은 0번째 입력이 되고 호출이 끝나면 0번째 출력으로 대체될 것이다. 두번째 칸은 1번째 입력/출력을 차지하고 나머지도 같다.

```
24 .. 24
.. .. ..
.. 32 .. 
Bo ar ..
.. .. ..

:Boar
}1 }0
{0 {0 # 두 입력의 합

# 결과는
보드/틱      MB/1      MB/2      MB/3      MB/4   Boar/1    MB/4      MB/5
           29 .. 24  .. .. ..  .. .. ..  .. .. ..  32 29  .. .. ..  .. .. ..
           .. .. ..  29 .. 24  .. .. ..  .. .. ..  {0 {0  .. .. ..  .. .. ..
           .. 32 ..  .. .. ..  29 .. 24  .. .. .. Boar/2  .. .. ..  .. .. ..
           Bo ar ..  Bo 32 ..  Bo 32 ..  29 32 24  .. ..  5B ar 24  Bo ar ..
           .. .. ..  .. .. ..  .. .. ..  .. .. ..  32 29  .. .. ..  5B .. 24

STDOUT: [$ (0x5B=[, then 0x24=36=$)
```

보드의 실행은 불려진 보드의 한 틱에서 시작하고 끝난다. 여러개의 보드가 실행될 때 같은 틱으로 작동될지는 정해지지 않았다 (만약 두 보드가 STDIN에서 읽고 STDOUT으로 출력하지 않으면 문제가 되지 않는다). (아니 이 사람은 왜이렇게 undefined behaviour를 좋아해???)

왼쪽/오른쪽 출력도 비슷하고 작동하지만 보드가 실행된 뒤 틱에서 구슬들은 호출된 블럭의 아래가 아닌 왼쪽/오른쪽에 위치된다.

## Section 4 - 포함과 주석

샾 표시 (`#`)는 주석이나 포함문의 시작을 나타낸다.

만약 `#` 이 라인의 공백을 제외한 첫 글자이고 바로 `include`라는 단어가 붙으면, 이는 포함문이다. 그렇지 않으면 이는 주석이고 새줄이 시작되면 끝난다.

포함문은 다음처럼 생겨먹었다:

```
#include file_name.mbl
```


따옴표가 사용되지 않음을 주의하자. 포함문은 표시된 파일의 모든 보드들을 불러올 것이다.

만약 보드의 이름이 포함된 파일에서 충돌나는 경우에는 다음의 경우에는 현재 파일을 사용한다:

1. 그 이름의 보드가 없는 파일이 현재 파일을 포함하는 파일의 경우
2. 현재 파일 내에서

다음의 경우에는 포함된 파일의 보드를 사용한다:

1. 포함된 파일의 보드 내에서

보드가 어떤 파일을 포함하는 파일을 포함한다고 해서 그 파일을 사용할 수 있는 것은 아니다. 다시 말해, 파일 A가 파일 B를 포함하고, 파일 B가 파일 C를 포함하고 파일 C는 `Test`라는 보드를 정의할 때 파일 A에서는 파일 C의 `Test` 보드를 파일 C를 직접적으로 포함하지 않는 한 사용하지 못한다.

각 파일은 각자 `MB` 보드를 가지고 있어야 한다. 이 `MB` 보드는 포함되는 경우에는 실행되지 않고, 오직 직접 파일이 실행되는 경우에만 실행된다. 이것은 포함된 파일에서 유닛 테스트를 돌릴 있게 해준다.

## 부록 1 - 장치 리스트

기호|장치 이름|설명
---|-----------|----------
 |**흐름 조정**|
`//`|왼쪽 전향 장치|지나는 구슬을 셀의 왼쪽으로 이동시킨다. 구슬이 이 기호에 닿은 다음 틱에서 왼쪽으로 움직여지고 구슬은 그대로 아래로 떨어진다.
`\\`|오른쪽 전향 장치|지나는 구슬을 셀의 오른쪽으로 이동시킨다. 구슬이 이 기호에 닿은 다음 틱에서 오른쪽으로 움직여지고 구슬은 그대로 아래로 떨어진다.
`@n`|포탈|포탈에 들어간 구슬은 같은 `n`값을 가지는 보드의 포탈로 나온다. 만약 포탈이 여러개 있다면 출구는 랜덤으로 정해진다. 만약 같은 `n`값을 가지는 포탈이 없다면 구슬은 그대로 아래로 떨어진다. 구슬이 포탈에 닿은 같은 틱에 구슬은 포탈로 이동한다. 
`&n`|동기화 장치|구슬이 같은 `n`값을 가지는 모든 동기화 장치들에 채워질 때 까지 구슬을 잡고 있는다. 만약 구슬이 동기화 장치에 잡혀 있는 동안 다른 구슬이 동기화 장치에 들어온다면 두 구슬은 합쳐진다.
 |**조건문**|
`=n`|같냐|만약 지나는 구슬이 `n`값과 같다면 이 장치는 빈 셀처럼 작동한다. 그렇지 않다면 오른쪽 전향 장치처럼 작동한다.
`>n`|크냐|만약 지나는 구슬이 `n`보다 크다면 이 장치는 빈 셀처럼 작동한다. 그렇지 않다면 오른쪽 전향 장치처럼 작동한다.
`<n`|작냐|만약 지나는 구슬이 `n`보다 작다면 이 장치는 빈 셀처럼 작동한다. 그렇지 않다면 오른쪽 전향 장치처럼 작동한다. `n=0`에 대해 구슬들은 전부 오른쪽으로 움직여진다.
 |**산술**|
`+n`|덧셈|지나는 구슬에 `n`만큼 더한다.
`-n`|뺄셈|지나는 구슬에 `n`만큼 뺀다.
`++`|증가|지나는 구슬에 1을 더한다. `+1`와 같음.
`--`|감소|지나는 구슬에 1을 뺀다. `-1`와 같음.
 |**비트연산**|
`^n`|비트 체커|비트 체커는 지나는 구슬의 `n`번째 비트를 0또는 1로 반환한다. 0번째 비트는 작은 쪽을 말한다. `n`은 `0`과 `7` 사이이거나 둘 중 하나여야 한다.
`<<`|왼쪽 비트 쉬프터|지나는 구슬을 왼쪽으로 1만큼 쉬프트 시킨다.
`>>`|오른쪽 비트 쉬프터|지나는 구슬을 오른쪽으로 1만큼 쉬프트시킨다.
`~~`|반전|지나는 구슬의 비트를 반전시킨다.
 |**입출력**|
`]]`|STDIN|STDIN에서 한 바이트를 읽어온다. 만약 실패하면 지나는 구슬을 오른쪽으로 옮긴다. 그렇지 않으면 구슬은 읽어진 바이트 값을 가지고 아래로 떨어진다.
`}n`|입력|보드로 보내진 `n`번째 입력값으로 보드가 호출될 때마다 대체된다.
`{n`|출력|모든 출력이 채워지면 보드가 종료되는 것만 빼면 동기화 장치와 비슷하게 작동한다. 같은 `n`값을 가지는 출력 장치들은 보드마다 하나 이상 있을 수 있다. 섹션 1에 더 자세히 나와있음.
`{<`|왼쪽 출력|출력 장치와 비슷하다. 대신 출력된 구슬이 보드의 아래가 아닌 왼쪽으로 나오는게 다르다. 섹션 1에 더 자세히 나와있음.
`{>`|오른쪽 출력|출력 장치와 비슷하다. 대신 출력된 구슬이 보드의 아래가 아닌 오른쪽으로 나오는게 다르다. 섹션 1에 더 자세히 나와있음.
 |**기타**|
`\/`|시공의 폭풍|시공의 폭풍을 지나는 구슬은 보드에서 사라진다. (원본: 쓰레기통)
`/\`|복사기|복사기는 지나는 구슬을 복사해 셀의 왼쪽과 오른쪽 칸에 붙여진다. 원래 구슬은 사라진다.
`!!`|종료|구슬이 종료 장치에 닿은 틱의 끝에 보드는 종료될 것이다. 이 순간에 채워진 출력은 그대로 사용된다.
`?n`|랜덤|지나는 구슬의 값을 `0`과 `n` 혹은 둘 중 하나의 사이의 랜덤 값으로 변경한다.
`??`|랜덤|지나는 구슬의 값을 0과 구슬의 값 혹은 둘 중 하나의 사이의 랜덤 값으로 변경한다.
