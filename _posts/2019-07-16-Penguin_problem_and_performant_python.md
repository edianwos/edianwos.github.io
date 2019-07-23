---
layout: post
title: 펭귄 문제와 파이썬 가속
categories: Hobby
---
<script src='https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.5/MathJax.js?config=TeX-MML-AM_CHTML' async></script>

좀 시간은 흘렀지만, 펭귄 문제라는 것이 유행했다고 들었다. 핵심은 왠지 풀 수 있을 것 같으나 풀 수 없는 문제들을 내고, 당연히 못 맞추면 벌칙으로 귀여운(?) 펭귄 프사로 당분간 지내게 되는 것이다. 그 중에 내 눈길을 끌었던 것은 다음과 같은 문제다.

![펭귄문제](/assets/penguin_prob.jpeg)

왠지 익숙해 보이고 잘 하면 찾을 수 있을 것만 같은 착각에 빠져 문제에 손을 대봤는데 역시 풀 수 없었다. 자존심이 상해서 간단하게 파이썬 스크립트를 짜서 돌려봤는데 웬걸, 1000 이하의 정수 내에서는 만족하는 해가 없는 것이다. 결론부터 말하자면, 이 문제의 [답](https://www.quora.com/Is-there-a-positive-integer-solution-for-a-b-and-c-if-c-a-+-b-+-b-a-+-c-+-a-b+c-4)은 있다. 심지어 이와 관련된 [논문](http://ami.ektf.hu/uploads/papers/finalpdf/AMI_43_from29to41.pdf)도 있다.[^3] 답을 보면 알겠지만 당연히 이런 값이라면 절대로 순서대로 찾아서는 답을 구할 수 없다.
어쨌건 일을 벌인 김에, 예전부터 관심있었던 파이썬 가속 실험을 해봤다.

[^2]: [Sympy](https://www.sympy.org)를 사용하면 가능은 하다.
[^3]: 사실 이 논문은 우변의 4가 아니라 홀수일 때 만족하는 해가 없다는 내용이다.

우선 위 문제를 그대로 파이썬에 집어넣을 수 없으므로 약간 모양을 바꿔줘야 한다. 우선 파이썬은 [매스매티카](http://www.wolfram.com/mathematica/)처럼 심볼릭 연산이 안된다.[^2] 따라서 정수끼리 나눗셈을 하면 부동소수점 형식(float)의 값이 튀어나오고, 부동소수점의 오차로 인해서 계산 결과가 4라고 해도 믿을 수 없게 된다.
이 식을 그대로 쓸 수 없다면 식을 좀 변형하는 되는 법. 곱셈으로만 이루어진 식으로 약간 바꿔주면 정수형의 곱셈만으로 참/거짓을 판별할 수 있다.

$$\frac{a}{b+c} + \frac{b}{c+a} + \frac{c}{a+b} = 4$$

$$\frac{a}{b+c} + 1 + \frac{b}{c+a} + 1 + \frac{c}{a+b} + 1 = 7$$

$$\frac{a+b+c}{b+c} + \frac{a+b+c}{c+a} + \frac{a+b+c}{a+b} = 7$$

$$(a+b+c)\left[\frac{1}{b+c} + \frac{1}{c+a} + \frac{1}{a+b}\right] = 7$$

여기서 $$b+c=A$$, $$c+a=B$$, $$a+b=C$$를 대입하면,

$$(A+B+C)\left(\frac{1}{A} + \frac{1}{B} + \frac{1}{C}\right) = 14$$

$$(A+B+C)\left(AB + BC + CA\right) = 14ABC$$

이렇게 되면 나눗셈으로 인해서 정수가 실수가 되는 일 없이 식을 만족하는지 검사할 수 있다.
이제 이 식을 가지고 원하는 숫자 N까지 순서대로 훑어가면서 조건을 만족하는 정수 조합을 찾으면 된다.

# Pure Python
1 부터 100까지의 모든 a, b, c의 조합을 탐색하는 코드를 구현하면 다음과 같다.

~~~ python
def naive_pure_python(num):
    for i in range(1, num):
        for j in range(1, num):
            for k in range(1, num):
                A = i + j
                B = j + k
                C = k + i
                if (A+B+C)*(A*B+B*C+C*A) == 14*A*B*C:
                    return i, j, k
    return 0, 0, 0
~~~


N=100으로 50회 실행 결과 0.631 +- 0.126 초를 얻었다.


늘 듣는 말이지만 파이썬은 느리다. 속도를 개선할 수는 없을까.
가장 좋은 방법은 좋은 컴퓨터를 사는 것이다. 하지만 펭귄문제를 풀기 위해서 컴퓨터를 새로 살 수는 없는 노릇이다.
두 번째로 좋은 방법은 효율적인 코드를 짜는 방법이다.
인터넷 상에서는 N^3 보다 효율적인 알고리즘을 아는 분도 계시겠지만, 나는 모른다.
하지만 단순히 생각해도 속도를 6배 가량 끌어올릴 방법은 있다.
잘 보면 위 식은 순환순열(cyclic permutation)의 합이다. 이 말인 즉슨 A, B, C의 순서가 뒤집혀도 똑같은 값이 나온다는 뜻이다.
따라서 (1,2,3) 조합과 (2,1,3) 조합은 똑같은 값을 줄 것이며 중복해서 계산할 필요가 없다.
세 개의 수를 섞어서 만들 수 있는 순열은 6 가지이므로, 중복 계산만 없어도 약 6배 가량의 속도가 증가해야 한다.
다음은 중복 없는 tight loop 구현이다.

~~~ python
def tight_pure_python(num):
    for i in range(1, num):
        for j in range(1, i+1):
            for k in range(1, j+1):
                A = i + j
                B = j + k
                C = k + i
                if (A+B+C)*(A*B+B*C+C*A) == 14*A*B*C:
                    return i, j, k
    return 0, 0, 0
~~~
N=100으로 50회 실행 결과 0.159 +- 0.0485초다.
정확하게 6배 빠르지는 않지만 그래도 꽤 속도에 이득을 볼 수 있다.

6배 정도로 만족할 수 없다면, 여기서부터는 약간 수고스러운 방식으로 속도를 높일 수 있다.

# AOT compiler
인터프리터 언어로써 파이썬이 가지는 장점도 있지만 단점도 있다.
그 중 가장 악명 높은 단점이 바로 속도다.
그래서 수 많은 파이썬 사용자들은 "느리지만 빨라요" 같은 말을 입에 달고 살게 되었고, 파이썬 퍼포먼스 팁 같은 것을 구글링하게 되었다.
하지만 어찌어찌 해도 파이썬 자체만으로 극복하기 어려운 속도 문제들이 존재하기 마련이다.
이런 문제를 해결하기 위한 꽤 표준적인 접근은 C언어로 작성한 모듈을 불러와서 가장 느린 부분을 최적화 하는 방법이다.
하지만 이런 방법은 파이썬의 표현력, 보기 좋은 구문, 빠른 개발 속도 등을 부분적으로 포기해야 하며 무엇보다 C를 모르는 사람에게는 불가능한 방법이다.
다행히, 파이썬이 주는 장점들을 포기하고 C로 돌아가는 것을 싫어한 사람들이 있었다.
그래서 파이썬 자체를 컴파일해서 쓰려는 시도가 생겨났다.
당연히 제일 먼저 생각나는 방법은 파이썬 혹은 그 비슷한 코드를 미리 컴파일해 모듈로 만들어둔 뒤 메인 코드에서 불라와서 사용하는 방법이다.
이런 종류의 컴파일러는 ‘미리’ 컴파일해서 실행 시간에 불러와서 실행하므로 ‘Ahead of Time’ 컴파일러 라고 부른다.
[Cython](https://cython.org/), [Pythran](https://pythran.readthedocs.io), [Shedskin](https://shedskin.github.io/), [Nuitka](https://nuitka.net/) 등이 있는데, Shedskin은 python2.6 까지만 지원하고, Nuitka는 속도에 중점을 둔 컴파일러는 아니라서 굳이 사용해 보지는 않았다.

## Cython
Cython은 python 코드를 C/C++로 컴파일해주는 가장 대표적인 컴파일러다.
한편으로는 스스로를 파이썬 언어의 상위집합(superset) 언어라고 부르는데, 그 이유는 대부분의 파이썬 언어에 type 선언 등의 문법을 ‘추가’한 언어이기 때문이다.
(거의) 모든 파이썬 문법에 맞는 언어는 Cython 문법에도 맞다.
복잡한 내용은 제쳐두고 어떻게 쓰고 얼마나 빨라지는지 보자.
Cython 코드는 우선 확장자를 .pyx로 쓴다.
또, `setup.py` 파일을 만들어서 ‘설치’ 과정을 거쳐야 한다.
그리고, 컴파일러에게 타입을 알려주면 더 효율적인 코드를 만들어 내므로 적절하게 타입을 지정해줘야 한다.
아래는 각각 순수한 파이썬 함수와 타입을 지정한 Cython 코드다.

~~~ python
def pure_python(num):
    for i in range(1, num):
        for j in range(1, i+1):
            for k in range(1, j+1):
                A = i + j
                B = j + k
                C = k + i
                if (A+B+C)*(A*B+B*C+C*A) == 14*A*B*C:
                    return i, j, k
    return 0, 0, 0

def typed_python(num):
    cdef int i, j, k, A, B, C
    for i in range(1, num):
        for j in range(1, i+1):
            for k in range(1, j+1):
                A = i + j
                B = j + k
                C = k + i
                if (A+B+C)*(A*B+B*C+C*A) == 14*A*B*C:
                    return i, j, k
    return 0, 0, 0
~~~
`cythonized.pure_python`은 N=100으로 50회 실행 결과 0.0710 +- 0.00947초가 걸렸다.
단순히 미리 컴파일 하는 것만으로도 대략 2배의 속도 이득을 얻었다.
`cythonized.typed_python`은 N=100으로 50회 실행 결과 0.000419 +- 0.000497초가 걸렸다.
평균 값 보다 표준편차 값이 더 크다는 것은 벤치마크가 제대로 안 되고 있다는 뜻이다.
N=1000으로 10회 실행 결과 0.384 +- 0.00263초가 걸린다.
연산량이 대충 N^3로 늘어난다는 점을 고려했을 때 N=100일 때 보다 1000배 가량의 시간이 더 걸리는 셈이며, 이를 고려하면 앞선 순수 파이썬 함수에 비해 약 400배 정도 빠른 속도다.

코드를 실행하고서야 알아낸 사실인데, C 코드로 번역된 순간부터 이 코드는 1000 이상의 정수에 대해서는 제대로 된 결과를 가져다 주지 않는다.
1000은 대충 2^10 정도 되는데 이 쯤 되면 좌변이든 우변이든 계산 결과가 정수형의 2^31를 쉽게 넘어서기 때문인 것 같다.
순수 파이썬은 정수형이 기본으로 임의의 크기(arbitrary size)를 지원하므로 이런 문제를 만날 일이 없다.

## Pythran
Cython의 최대 단점은 파이썬 코드를 그대로 사용할 수 없다는 점에 있다.
Pythran은 타입 지정을 주석에 하기 때문에 잘 작동하는 파이썬 코드도 C 타입을 지정해서 활용할 수 있다.
역으로 생각하면 Pythran이 없더라도 ‘돌아는 가는’ 코드를 만들 수 있다는 뜻이기도 하다.
Cython과 마찬가지로 컴파일 하는 수고는 필요하다.

~~~ python
#pythran export tight_pythran(int)
def tight_pythran(num):
    for i in range(1, num):
        for j in range(1, i+1):
            for k in range(1, j+1):
                A = i + j
                B = j + k
                C = k + i
                if (A+B+C)*(A*B+B*C+C*A) == 14*A*B*C:
                    return i, j, k
    return 0, 0, 0
~~~

N=1000으로 10회 실행 결과 0.381 +- 0.00176초가 걸린다.
Pythran이 근소하게 앞선다는 것을 확인할 수 있다.
놀랍지는 않은 결과다. 둘 다 C/C++로 번역한 뒤 컴파일해서 실행하는 것이므로 속도는 비슷할 것이나 Pythran은 수치 계산에 조금 더 비중을 두고 있는 라이브러리라서 Cython보다 조금 빠를 수 있다.

# JIT compiler
한편으로는 인터프리터 언어이기 때문에 취할 수 있는 장점을 살린 컴파일러도 존재한다.
AOT 컴파일러는 미리 컴파일을 하고 실행하므로, 실제 실행 시간동안 어떤 일이 일어나는지 알 수 없는 부분들이 있다.
하지만 인터프리터 언어는 실행하는 와중에 어디가 가장 병목인지 어떤 함수에 어떤 인자들이 주어지는지 등의 정보를 얻을 수 있기 때문에 AOT가 최적화 할 수 있는 것보다 조금 더 최적화 된 코드를 만들어 낼 수 있다.
[Just In Time](https://en.wikipedia.org/wiki/Just-in-time_compilation) 컴파일러가 바로 이런 일을 한다.
물론 처음에 정보가 없을 때 상대적으로 최적화가 덜 되므로 조금 느리게 작동하는 것 처럼 보일 수 있다.
나는 전문가가 아니므로 자세한 설명은 다른 문서를 참고하는 것을 권한다.

## pypy 
파이썬 버전 관리에 어려움이 없는 사람에게 가장 쉽게 속도를 높일 수 있는 방법은 아마 [pypy](http://pypy.org/)를 사용하는 것일테다.
왠만하면 파이썬 코드 그대로 사용 가능하며, 아무런 작업 없이 빠른 속도를 얻을 수 있다.
몇 가지 구현에 차이가 있고, 몇 가지 특징이 있지만 나도 잘 모르니 여기서는 다루지 않겠다.
pypy3.6-v7.1.1을 다운로드 받아서 앞서 작성한 pure python 코드를 그대로 실행하면 N=100, iteration=100일 때 `naive_pure_python`은 0.00714 +- 0.00209 초, `tight_pure_python`은 0.00126 +- 0.00106 초가 걸렸다.
`tight_pure_python`이 표준편차가 크다는 점에 유의하자.
`tight_pure_python`을 N=1000으로 10회 실행 결과 9.52 +- 0.0729초가 걸린다.
처음 한 번은 그냥 실행해서 초기 오버헤드를 약간 줄인 뒤 측정한 결과다.
Cython과 Pythran에 비해서 인상적인 결과는 아니지만 순수 python 코드 그대로 얻은 성능 치고는 준수한 결과다.
다만, N^3 규칙을 따르지 않고 예상보다 훨씬 오래 걸리는 것 같은데, 이유는 알 수 없다.

## numba
[numba](https://numba.pydata.org/)는 [llvm](https://llvm.org/) 프로젝트의 JIT 기술을 활용한다.
물론 여기서도 llvm이나 jit 기술에 대해서는 내가 할 수 있는 말은 별로 없다.
numba는 `@jit` 데코레이터만 붙여주면 자동으로 모든 일을 다 해주는데, 심지어 아주 빠르기까지 하다.
단점이라면, llvm과 기타 의존성 문제가 커서 Anaconda 파이썬을 설치하지 않았다면 제대로 설치해서 활용하기 매우 어려울 수도 있다.

~~~ python
from numba import jit

@jit
def numba_jit(num):
    for i in range(1, num):
        for j in range(1, i+1):
            for k in range(1, j+1):
                A = i + j
                B = j + k
                C = k + i
                if (A+B+C)*(A*B+B*C+C*A) == 14*A*B*C:
                    return i, j, k
    return 0, 0, 0
~~~

놀라운 점은 두 번째 실행 부터는 아주 최적화가 잘 되어서 Cython보다도 빨라진다.
N=1000으로 10회 실행 결과 0.245 +- 0.00237초가 걸린다.
이번에도 처음 한 번은 함수를 그냥 실행해서 초기 오버헤드를 줄인 뒤 측정한 결과다.
오버헤드 없이는 N=1000으로 10회 실행 결과 0.276 +- 0.101초가 걸린다.
또한 Cython은 C의 int 크기에 영향을 받아서 앞서 말한대로 `N>1000`일 때 제대로 작동하지 않는 반면 numba는 제대로 작동한다.

`nopython`옵션이나 `nogil` 옵션처럼 최적화 추가옵션은 이런 단순한 예에서는 별로 의미가 없는 것 같다.
구체적으로 어떻게 작동하는지 더 알아볼 필요가 있겠다.

# 다른 언어로는?
이쯤 되면 파이썬에만 목을 메고 있을 필요는 없다.
나도 이런 가속 방법에 대해서 늘 ‘이럴 바에야 C로 짜지’ 같은 마음가짐을 가지고 있었다.
그렇다면 정말 C로 짜면 어떻게 될까.

## C
구현에 차이, 언어의 특징 같은 것 따위가 개입할 수 없을 정도로 단순한 알고리즘이라 그냥 파이썬 코드를 직역해서 집어넣으면 된다.

~~~ c
#include <stdio.h>
#include <time.h>

clock_t t1, t2;
double delta;
int n;

int i, j, k;
int a, b, c;
int num;

num = 1000;

int main() {
  n = 0;
  t1 = clock();
  for (i=1; i<num; i++) {
    for (j=1; j<i+1; j++) {
      for (k=1; k<j+1; k++) {
        A = i + j;
        B = j + k;
        C = k + i;
        if ((A+B+C)*(A*B+B*C+C*A)==14*A*B*C) {
            printf("%d, %d, %d", i, j, k);
            break;
        }
      }
    }
  }
  t2 = clock();
  deltat = (double) (t2-t1)  CLOCKS_PER_SEC;
  printf("it took %f seconds", deltat);
  return 0;
}
~~~


윈도우즈 10 Visual studio 2019로 컴파일했으며 64bit 속도 최적화 옵션일 켜고 컴파일했다.
N=1000일 때 0.324초가 걸린다.(귀찮아서 통계를 내지는 않았지만 반복 실행해도 대충 이 정도 속도가 나온다.)
놀랍게도 타입 지정한 Cython보다 아주 근소하게 빠르며, 반복실행 시 numba jit보다 느리다. (정확히는 numba jit이 더 빨라진다.)
32bit로 컴파일하면 Cython보다도 느리다.
사실 코드가 단순해서 Cython이나 numba가 최적화하는데 어렵지 않았을수는 있다.
그럼에도 불구하고 Cython, numba의 속도는 인상적이다.
사람들이 불편함을 감수하고도 이런 방법들을 활용하는 이유가 다 있구나.

## Julia
[Julia](https://julialang.org/)는 ‘C의 속도, 파이썬의 표현력, R의 통계 분석, Lisp의 메타 프로그래밍 능력 기타 등등 다른 언어의 좋은 점은 다 때려박은 언어’라고 광고하는 꽤 최근에 만들어진 오픈소스 프로그래밍 언어다.[^1]
실제로 위에 표현된 대부분의 능력을 프로그래머에게 준다고 알려져있으며 나도 관심있게 보고 있는 언어다.
지금까지는 관심만 있었지 손을 제대로 못 대고있었는데 이번 기회에 한 번 써보게 되었다.

[^1]: 이 언어가 처음 태동했을 당시에는 지금과 같은 화려한 메인페이지 대신 아주 긴 언어에 대한 소개가 쓰여있었는데, 그 곳에서 읽은 설명이다. 이제 왜인지는 모르겠지만 찾아볼 수 없다.

~~~ julia
const num = 1000

function bench(num)
    for i = 1:num
        for j = 1:i
            for k=1:j
                A = i + j
                B = j+k
                C = k+i
                if (A+B+C)*(A*B+B*C+C*A) == 14*A*B*C
                    return i, j, k
                end
            end
        end
    end
    0, 0, 0
end
~~~

N=1000으로 대략 0.23초가 걸린다.
앞서 테스트 해 본 numba보다 약간 빠른 셈이다.
Julia와 numba 모두 llvm의 jit에 기대고 있으므로, 퍼포먼스가 비슷한 것은 이상한 일이 아니다.
이쯤 되면 위에 있는 C 코드를 clang으로 컴파일했을 때 어떤 일이 일어나는지 확인하고 싶다.

# 결론

생각보다 파이썬 가속을 위한 옵션이 다양하며 쉽게 접근 가능하고 성숙하기까지 하다는 점에 놀랐다.
당연히, 빠른 수치연산이 필요하다면 제일 먼저 선택할 옵션은 [numpy](https://www.numpy.org/)겠다.
하지만 벡터화가 까다롭고 for 문에 더 적합한 계산들이라면 제일 먼저 numba를 집어들 것 같다.
나는 conda를 이용해 쉽게 설치했지만 llvm 환경 설치에 문제가 있을 수도 있다.
(문서에 따르면 `pip` 설치만 해도 wheel에 llvm lite dependency가 딸려오므로 왠만해서 이런 문제는 없을 것이다.)
아니면, 초기 컴파일 시간도 아까운 경우나 반복실행되지 않아서 JIT 컴파일이 상대적으로 손해인 경우도 있을 수 있다.
  t2 = clock();
만약 코드가 순수한 파이썬 코드로서도 동작하길 바란다면 pythran을 사용하면 될 테고, 그렇지 않으면 Cython을 사용하는 것이 나을 것이다.
Cython은 코드에서 느린(정확히는 파이썬으로 인한 오버헤드가 큰 부분) 부분을 찾아주는 기능도 있어 최적화에 좋은 가이드가 되어줄 것이다.

만약 최신 파이썬 버전에 구애받지 않고 CPython에 목멜 필요가 없다면 프로젝트를 처음부터 pypy에서 시작하는 것도 좋은 접근일 것이다.
 사실 나는 직접 실험해 보기 전까지 pypy가 빠르다는 것은 알고 있었지만 이렇게 빠르리라고는 생각하지 못했다.
 게다가 속도를 얻기 위해서 데코레이터를 붙이거나, 타입 힌트를 줄 필요도 없다.

  Julia는 매우 빠른 언어가 맞는 것 같다. 그러나 파이썬이 익숙하고 새로운 언어를 익히고 싶지 않으면 굳이 Julia를 선택할 이유는 없을 것이다.
 Numba는 llvm을 이용하고 있으므로 제대로 사용한다면 Julia에 근접한 속도를 내 줄 것 같다.
 하지만 이는 전적으로 내 추측에 불과하며, 코드가 더 복잡해 질 수록 Julia가 더 유리해질 수도 있다.
 그리고 놀랍지만, 이제 더이상 퍼포먼스를 위해서 C/C++을 고집하지 않아도 되는 시대가 온 것 같다.
 C/C++은 여전히 많은 곳에 쓰이겠지만 우리같은 잘 만들어진 코드를 가져다 쓰거나 다른 잘 최적화 된 언어를 이용하는 사람들은 굳이 C/C++ 코드를 직접 쓰지 않아도 비슷하게 빠르거나 가끔은 더 빠르기도 한 퍼포먼스를 얻을 수 있는 것이다.
 
 마지막으로, 여기 있는 모든 실행 결과는 한성 노트북 A26X ForceRecon 4257, Windows 10 운영체제 위에서 실행되었다.
 파이썬 및 기타 라이브러리는 [Anaconda python 3.7](https://www.anaconda.com/distribution/)과 딸려오는 conda package manager를 이용해 설치했다.
 벤치마크는 실행 시마다 조금씩 바뀔 수 있으며, 환경과 컴퓨터에 따라 바뀔 수 있다. 