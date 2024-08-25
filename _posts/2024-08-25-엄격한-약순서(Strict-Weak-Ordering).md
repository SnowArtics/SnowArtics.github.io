---
title: "[C++] 엄격한-약-순서(Strict-Weak-Ordering)"
author: seolwon
date: 2024-08-25 19:00:00 +0900
categories: [C++, Language]
tags: [C++, Language]
---

# 부제 : 비교 함수를 생성할 때, 두 값이 같으면 왜 true를 리턴해서는 안되는가?

## Strict Weak Ordering에 대해서
Strict Weak Ordering은 4개의 요소로 이루어져 있는 법칙이다.<br>
1. 반사 불가능성(Irreflexivity)<br>
	`comp(a, a)`는 항상 `false`여야 한다. 즉, 어떤 값도 자기 자신보다 크다고 간주한다.<br>
2. 비대칭성 (Asymmetry)<br>
	`comp(a, b)`가 `true`라면 `comp(b, a)`는 false여야 한다. 즉, 만약 a가 b보다 작다면 b는 a보다 작지 않다는 것이 보장된다.<br>
3. 추이성 (Transitivity)<br>
	`comp(a, b)`가 `true`이고 `comp(b, c)`가 true이면 `comp(a, c)`도 true여야 한다. 즉, a < b이고 b < c라면 a < c여야 한다.<br>
4. 등가성의 일관성 (Transitivity of Equivalence)<br>
	`!comp(a, b)`와 `!comp(b, a)`가 모두 true라면 a와 b는 논리적으로 같다고 간주할 수 있다. 이런 경우, !comp(a, c)와 !comp(b, c)도 동일한 결과를 가져야 한다.<br>
<br>

## 그렇다면 비교 함수에서 두 값을 같을 때 true를 리턴하면 무슨 문제가 생길까?

이를 위해서 C++의 `std::sort()`함수를 한 번 분해해보자.
우선 sort() 함수는 밑과 같이 구성되어 있다.
```cpp
_EXPORT_STD template <class _RanIt>
_CONSTEXPR20 void sort(const _RanIt _First, const _RanIt _Last) { // order [_First, _Last)
    _STD sort(_First, _Last, less<>{});
}
```
이 함수는 `::std::sort()` 함수로 다시 진입하는데 밑과 같다.
```cpp
_EXPORT_STD template <class _RanIt, class _Pr>
_CONSTEXPR20 void sort(const _RanIt _First, const _RanIt _Last, _Pr _Pred) { // order [_First, _Last)
    _Adl_verify_range(_First, _Last);
    const auto _UFirst = _Get_unwrapped(_First);
    const auto _ULast  = _Get_unwrapped(_Last);
    _Sort_unchecked(_UFirst, _ULast, _ULast - _UFirst, _Pass_fn(_Pred));
}
```
여기서 우리가 눈여겨 봐야할 부분은 `_Sort_unchecked()` 함수이다.
```cpp
template <class _RanIt, class _Pr>
_CONSTEXPR20 void _Sort_unchecked(_RanIt _First, _RanIt _Last, _Iter_diff_t<_RanIt> _Ideal, _Pr _Pred) {
    // order [_First, _Last)
    for (;;) {
        if (_Last - _First <= _ISORT_MAX) { // small
            _Insertion_sort_unchecked(_First, _Last, _Pred);
            return;
        }

        if (_Ideal <= 0) { // heap sort if too many divisions
            _Make_heap_unchecked(_First, _Last, _Pred);
            _Sort_heap_unchecked(_First, _Last, _Pred);
            return;
        }

        // divide and conquer by quicksort
        auto _Mid = _Partition_by_median_guess_unchecked(_First, _Last, _Pred);

        _Ideal = (_Ideal >> 1) + (_Ideal >> 2); // allow 1.5 log2(N) divisions

        if (_Mid.first - _First < _Last - _Mid.second) { // loop on second half
            _Sort_unchecked(_First, _Mid.first, _Ideal, _Pred);
            _First = _Mid.second;
        } else { // loop on first half
            _Sort_unchecked(_Mid.second, _Last, _Ideal, _Pred);
            _Last = _Mid.first;
        }
    }
}
```
이 함수에서는 특정 기준에 따라서 삽입 정렬을 사용할 것인지(`_Insertion_sort_unchecked(_First, _Last, _Pred);`)<br>
힙 정렬을 사용할 것인지(`_Sort_heap_unchecked(_First, _Last, _Pred);`)<br>
퀵 정렬을 사용할 것인지(`_Partition_by_median_guess_unchecked(_First, _Last, _Pred);`)를 정한다.<br>
<br>
대부분의 경우에서는 퀵 정렬을 사용하기 때문에 퀵 정렬 함수를 한 번 더 까보자.
해당 함수는 길이가 매우 길어서 중요한 부분만 적어 놓을테니 함수의 전체가 궁금한 사람은 `algorithm.cpp`의 `_Partition_by_median_guess_unchecked()`를 찾아보자.
```cpp
template <class _RanIt, class _Pr>
_CONSTEXPR20 pair<_RanIt, _RanIt> _Partition_by_median_guess_unchecked(_RanIt _First, _RanIt _Last, _Pr _Pred) {

	/*............................
	............................*/

    for (;;) { // partition
        for (; _Gfirst < _Last; ++_Gfirst) {
            if (_DEBUG_LT_PRED(_Pred, *_Pfirst, *_Gfirst)) {
                continue;
            } else if (_Pred(*_Gfirst, *_Pfirst)) {
                break;
            } else if (_Plast != _Gfirst) {
                _STD iter_swap(_Plast, _Gfirst);
                ++_Plast;
            } else {
                ++_Plast;
            }
        }

        for (; _First < _Glast; --_Glast) {
            if (_DEBUG_LT_PRED(_Pred, *_Prev_iter(_Glast), *_Pfirst)) {
                continue;
            } else if (_Pred(*_Pfirst, *_Prev_iter(_Glast))) {
                break;
            } else if (--_Pfirst != _Prev_iter(_Glast)) {
                _STD iter_swap(_Pfirst, _Prev_iter(_Glast));
            }
        }

        /*............................
		............................*/

    }
}
```
퀵 정렬은 하나의 피벗(여기서는 `_pFirst`)을 기준으로 큰 값들과 작은 값들을 비교해가면서 정렬을 완성한다.<br>
**자, 그러면 여기서 Strict Weak Ordering법칙을 위반하면 어떻게 될까?** <br>
만약 `sort()`함수를 돌리는 배열의 원소가 같은 값이 있다면, 두개의 반복문이 계속 수행되면서 **pivot의 생성위치가 여러개의 자리에 무한히 반복되면서 생성될 수 있다.**<br>
결국 이 경우는 엄격한 약 순서의 반사 불가능성(Irreflexivity)을 위반해서 생긴 문제이다. 이를 주의 하고 항상 비교 함수를 생성할 때는 같은 값일 때, `false`를 반환하도록 하자.

# 요약
1. Strict Weak Ordering을 지키기 위해서는 같은 값일 때는 무조건 false를 반환한다.
2. 비교 함수를 만들 때 Strict Weak Ordering을 안지킨다면 배열에 같은 값들이 있을 때, 프로그램이 죽어버린다.

