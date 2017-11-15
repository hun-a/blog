---
title: Euclidean_algorithm
comments: true
date: 2017-11-15 12:20:56
tags: ['Algorithms', 'Euclidean']
categories: ['Algorithms']
---

# 잡담
원래 [github pages](https://pages.github.com/)와 [jekyll](https://jekyllrb-ko.github.io/)을 이용해서 블로그를 운영하다가 잠시 [tistory](http://seunghunchan.tistory.com/)로 옮겼다가 이번에 [hexo](https://hexo.io/ko/)를 이용해서 다시 [github pages](https://pages.github.com/)로 돌아왔다. ~~이게 뭐하는거?~~

이 글은 예전에 [tistory](http://seunghunchan.tistory.com/16)에 포스팅 했던건데 이리로 옮겨온 것이다.

그럼 본론으로 들어가 보자.


# 소개
최대 공약수를 구할 수 있는 파워풀한 알고리즘은 [유클리드 호제법](https://namu.wiki/w/%EC%9C%A0%ED%81%B4%EB%A6%AC%EB%93%9C%20%ED%98%B8%EC%A0%9C%EB%B2%95) 에 대해 알아보도록 하겠다.

이리 좋은걸 왜 우리는 학교에서 안배운걸까? 안타깝다. 내딸은 아직 6개월뿐이 안됬지만 나중에 내가 가르쳐야겠다 ㅋㅋ

# 정의
유클리드 호제법의 정의는 다음과 같다.

> 두 양의 정수 a, b (b > a) 에 대해 b = aq + r (0 <= r < q) 라 하면
> gcd(a, b) = gcd(a, r)이 성립한다.

즉, a, b의 최대 공약수는 a, r의 최대 공약수와 같다는 말이다. _(왜?)_

자 그럼 왜 그런지 증명을 해보자.

> gcd(a, b) = G 라고 하고, 서로소 A, B에 대해 다음과 같은 식이 성립된다.
> a = GA, b = GB

> 유클리드 호제법 정의의 b = aq + r 에 위의 수식을 대입하면
> GB = GAq + r 이고 이를 이항 후 G로 묶으면
> r = G(B - Aq) 가 성립한다.
> 유클리드 호제법의 정의에 따르면 gcd(a, b) = gcd(a, r) = G 가 된다.
> a = GA이고, r = G(B - Aq) 이므로
> A와 B - Aq가 서로소이면 증명이 끝난다.

> gcd(A, B - Aq) = m 이라고 가정하고 이를 만족하는 적당한 서로소를 k, l 이라고 해보자.
> 그럼 A = mk, B - Aq = ml 이 성립한다.
> B = Aq + ml 이고, A = mk 이므로 B = mkq + ml = m(kq + l) 이 성립한다.
> 즉, B = m(kq + l) 이므로 m은 A와 B의 공약수이다.

> 하지만 맨 위에서 A와 B는 서로소라고 했으니 m = 1 이되므로
> A와 B - Aq 역시 서로소가 된다.

> 따라서 gcd(a, b) = gcd(a, r) 이 성립된다.

> 증명 끝!

굉장했다.

증명은 [나무위키](https://namu.wiki/w/%EC%9C%A0%ED%81%B4%EB%A6%AC%EB%93%9C%20%ED%98%B8%EC%A0%9C%EB%B2%95)에서 보고 참고했다.

실제 알고리즘은 위에 증명에 비하면 너무너무너무 간단하다.

`b = aq + r (0 <= r < q)` 만 이용하면 된다.


<br>

48, 60 을 가지고 구해보자.

> b = aq + r 을 이용하여
> 60 = 48 x 1 + 12
> 48 = 12 x 4 + 0


음?! 너무 간단하게 나왔다.

`12가 최대 공약수`임을 알았다.


<br>

다시, 좀 더 복잡한 95, 250 을 가지고 해보자.

> 250 = 95 x 2 + 60
> 95 = 60 x 1 + 35
> 60 = 35 x 1 + 25
> 35 = 25 x 1 + 10
> 25 = 10 x 2 + 5
> 10 = 5 x 2 + 0


`최대 공약수는 5`가 되겠다.

<br>

이제 위 과정을 일반화를 통해 알고리즘을 구현해 보자.

<br>

gcd(95, 250)에서 a = 95, b = 250 이다.

250 % 95 -> 60 이므로 r = 60 이 된다.

즉, gcd(95, 250) = gcd(95, 60) 과 같다.

근데, (60, 95) 에 대한 최대 공약수나, (95, 60) 에 대한 최대 공약수나 똑같다.

gcd(95, 60) 이나 gcd(60, 95)나 똑같다는 의미이다.

그럼, 다음 식을 보자.

gcd(60, 95) 에서 a = 60, b = 95 이다.

95 % 60 -> 35 이므로 r = 35 이다.

즉, gcd(60, 95) = gcd(60, 35) = gcd(35, 60) 이다.

다음 식은

60 % 35 -> 25 이므로

gcd(35, 60) = gcd(35, 25) =  gcd(25, 35) 이고,

35 % 25 -> 10 이므로

gcd(25, 35) = gcd(25, 10) = gcd(10, 25) 이고...

25 % 10 -> 5 이므로

gcd(10, 25) = gcd(10, 5) = gcd(5, 10) 이고

10 % 5 -> 0 이므로

최대 공약수는 5 가 된다.

<br>


요약하면 gcd(95, 250) = gcd(5, 10) 과 같은 것이다.

<br>

그럼, 위 과정을 일반화 해서 보면

gcd(a, b) 는 다음 단계에서 gcd(b %a, a) 가 된다는 걸 알 수 있다.

하지만 b % a 가 0일 경우, 우리는 최대 공약수를 찾았으므로 a 를 반환하면 된다.



구현해보자!

### JavaScript
```JavaScript
const eu = function(a, b) {
  return b % a ? eu(b % a, a) : a;
}
```

### Java
```Java
public int eu(int a, int b) {
  return b % a != 0 ? eu(b % a, a) : a;
}
```

참고로 최소 공배수는 유클리드 호제법을 안다면 굉장히 간단하게 구할 수 있다.

lcd(a, b) = a * b / gcd(a, b) 이다.

<br>

직접 구현해보기 바란다.
