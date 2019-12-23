---
title: Rambda addIndex
date: 2019-12-23 00:58:23
tags: ["js", "ramda", "fp"]
categories: ["JavaScript", "FP"]
comments: true
---

# What is today's function?

오랜만에 ramda 라이브러리에 대한 블로그를 쓴다.

오늘의 주제는 [addIndex](https://ramdajs.com/docs/#addIndex) 함수 되시겠다. 

# 함수 설명

JavaScript의 Array 객체에서 제공하는 `map`, `reduce`, `filter` 등의 함수의 인자로 전달되는 함수의 시그니쳐는 아래와 같다.

> callback(element[, index[, array]]

하지만 ramda에서 제공하는 `map`, `reduce`, `filter` 와 같은 함수들의 인자로 전달되는 함수는 `element` 단 하나의 파라미터만 가지고 있다.

보통은 `element` 하나의 파라미터만 받아도 왠만한 작업은 다 처리할 수 있지만

간혹 가다가 JavaScript Array 객체의 함수들 처럼 `element`의 인덱스가 필요하다던가, 아니면 전체 배열의 길이가 필요하다던가 할 수 있다.

이때를 위해 만들어진 함수가 바로 `addIndex` 이다.

우리는 개발자이니 코드로 이해를 해보자.

```JavaScript
const arr = [1, 2, 3, 4 5];
R.map(console.log, arr);
```

위 코드의 결과는 아래와 같이 배열의 `element` 만 출력한다.

```
1
2
3
4
5
[ undefined, undefined, undefined, undefined, undefined ]
```

그럼 `addIndex`를 사용해보면

```JavaScript
R.addIndex(R.map)(console.log, arr);
```

다음과 같이 `element`, `index`, `array` 가 순차적으로 출력된다.

```
1 0 [ 1, 2, 3, 4, 5 ]
2 1 [ 1, 2, 3, 4, 5 ]
3 2 [ 1, 2, 3, 4, 5 ]
4 3 [ 1, 2, 3, 4, 5 ]
5 4 [ 1, 2, 3, 4, 5 ]
[ undefined, undefined, undefined, undefined, undefined ]
```

# 코드 분석

요 함수는 [source/addIndex.js](https://github.com/ramda/ramda/blob/v0.26.1/source/addIndex.js) 에 위치한다.

전체 코드는 다음과 같다.

```JavaScript
var addIndex = _curry1(function addIndex(fn) {
  return curryN(fn.length, function() {
    var idx = 0;
    var origFn = arguments[0];
    var list = arguments[arguments.length - 1];
    var args = Array.prototype.slice.call(arguments, 0);
    args[0] = function() {
      var result = origFn.apply(this, _concat(arguments, [idx, list]));
      idx += 1;
      return result;
    };
    return fn.apply(this, args);
  });
});
```

`curryN` 함수의 콜백함수의 지역변수들을 보면 아래와 같다.

> `idx`: `element`의 `index`로 사용될 변수
> `originFn`: `fn` 으로 전달받은 함수의 `callback`으로 사용될 함수
> `list`: 마지막 파라미터로 전달된 배열
> `args`: `arguments` 변수를 복제한 변수로써 `fn` 함수의 인자로 사용
> `args[0]`: `originFn` 으로 대체되는 함수 즉, `fn`의 `callback` 함수

지역변수들은 됬고, 정말 중요한 부분은 아래 두 부분이다.

`fn`의 `callback`을 호출하는데 파라미터에 `idx`, `list` 를 같이 전달해주고 있다.

```JavaScript
args[0] = function() {
  var result = origFn.apply(this, _concat(arguments, [idx, list]));
  idx += 1;
  return result;
};
```

즉, `_concat(arguments, [idx, list])` 요 부분을 통해 우리가 위에서 봤던 `callback(element[, index[, array]]` 시그니쳐가 생성된다.

다음은 요기다. 바로 실제 `fn`을 실행시키는 부분이다.

```JavaScript
return fn.apply(this, args);
```

어때요, 참 쉽죠?


# 실제 사용 예제

DOM 리스트를 조회해서 순서가 짝수일때는 파란색을, 홀수일때는 빨강색을 배경으로 넣어야 한다고 가정해보자.

물론 실무에서 촌스럽게 아래같은 `css`를 사용하진 않겠지만

중요한건 **번갈아가며** 홀짝 홀짝 파빨파빨 색깔이 나오게 하는게 키포인트다.

```css
// css
.blue {
  background-color: blue;
}

.red {
  background-color: red;
}
```

```JavaScript
const doms = document.getElementsByClassName("row");

const callback = (el, i, arr) =>
  el.classList.add(i % 2 === 0 ? "blue" : "red");

const forEach = R.addIndex(R.forEach);

forEach(callback, doms);

/* 
vanilla JS
Array.prototype.forEach.call(doms, callback);
*/
```

Vanilla JS도 굉장히 있어보이는 기분은... 기분탓일꺼다... 😉

# 마치며...

간단하게 사용할 수 있는 함수를 구현하기 위해서는 복잡하고 어려운 개념들이 많이 필요하다.

복잡하고 어려운 개념들이 포함된 코드들을 소설책 읽듯이 물 흐르듯 이해하며 읽어갈 수 있는 실력이 생기는 그날까지 꾸준히 공부해야겠다. 🙃