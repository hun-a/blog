---
title: Encoding the querystring in express
date: 2019-03-11 23:46:15
tags: ["node.js", "nodejs", "express", "querystring", "encode"]
categories: ["JavaScript", "Node.js"]
comments: true
---

# 버그의 시작

맘에 안들지만 임시방편으로 굉장히 기괴한 API를 하나 만들었다.

검색어에 대한 정보를 뿌려줘야 하는데, 새로 개발한 프로젝트가 플랫폼 이슈로 인해 사용할 수 없었고,

기존에 있던 프로젝트의 API를 호출해서 검색어에 대한 정보를 뿌려주도록 구현을 했다. 

정말 맘에 안든다.

그렇게... 버그가 태어나기 시작했다.


# 버그의 출현

검색어는 당연 한글이 들어온다. 이건 전혀 문제가 되지 않는다.

그런데 괴상한 `Exception`이 자꾸 로그에 찍히고 있었다.

```
Uncaught URIError: URI malformed
라인 번호는 여기가 어떻고
이쪽 스택은 또 어쩌고 저쩌고
저쪽 스택도 어쩌고 저쩌고
블라블라...
```

문제가 된 라인을 확인하니 `decodeURI(req.query.keyword)` 가 있었다.

# 해결?!

이것도 해보고 저것도 해보고 스택over플로우님께도 여쭤봤다.

unicode sequence가 어쩌고 저쩌고... 영알못은 오늘도 너무 괴로웠다.

그런데 이것 저것 해보다 보니...

`decodeURI('%')` 와 같이 `%` 가 인코딩 된 애들과 다른 형태 즉,

혼자 따로 노는 `%` 가 있을 경우 에러를 뱉어냈다.

그럼 `%` 를 escape 처리하면 되겠네?

그래서 웹에서 전역으로 사용 가능한 `escape()` 함수를 사용했다.

하지만 전~ 혀 코딱지 만큼도 해결되지 않고 똑같은 `Uncaught URIError: URI malformed` 를 연신 뱉어냈다.

심지어 `escape()` 이친구는 strictly deprecated 된 친구였다. [(여기 참조)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/escape)


# 근데 이건 뭐??

이제부터가 이 포스팅을 시작한 진짜 이유다.

후후. 이것도 버그가 아닐까 생각이 들지만...

**한글 + %** 가 파라미터로 들어오면 멋쟁이 express가 이걸 자동으로 encoding 해버린다.

물론 **영어 + %** 는 원래대로 들어온다. ~~영어최고!~~

못 믿겠다고? 그럼 아래 코드를 실행하고, `localhost:3000?q=한글%` 라고 웹 브라우저에서 실행해 보면 된다.

```javascript
const express = require('express');
const app = express();
const port = 3000;

app.get('/', (req, res) => {
  res.json({ query: req.query, url: req.url });
});

app.listen(port, () => console.log(`Example app listening on port ${port}!`));
```

결과는 아래와 같다.

![wtf?](/images/wtf_encoding_20190312.png)

우리의 멋쟁이 express 가 너무나도 과도한 친절을 베풀고 있는게 아닐까 의심이 된다.

아, 참고로 인코딩을 왜 하는지 모르는 분들이 있을 수도 있다.

나도 잘 몰랐는데 검색하면 굉장히 잘 나온다. 그래도 한번 써본다.

일단은, `URL`로 넘어갈 수 있는 놈은 `ASCII` 문자 뿐이다. ~~구시대의 유물 이랄까...?~~

`UTF-8`, `EUC-KR` 이런애들이 아닌 오로지 `ASCII` 이다. ~~EUC-KR도 사실 극혐...~~

따라서 `UTF-8` 문자를 `encodeURI()`, `encodeURIComponent()` 와 같은 함수로 `%16진수16진수` 형태로 인코딩 해서 보내는 것이다.

`ASCII`는 고작 1바이트로 사용하는데 `%16진수16진수`로 인코딩 하면 더 풍부한 표현력이 생기지 않는가? ~~겁나 똑독해! 누구 아이디어지?~~

아무튼 잡담은 이제 그만 하고,

`NodeJS`의 멋진 API [querystring.unescape()](https://nodejs.org/dist/latest-v10.x/docs/api/querystring.html#querystring_querystring_unescape_str) 를 사용하면 express의 과도한 친절을 사양할 수 있게 된다.

아래의 코드로 수정하고 다시 `localhost:3000?q=한글%`를 웹 브라우저 또는 `curl` 로 실행 해보자.

```javascript
const qs = require('querystring');
const express = require('express');
const app = express();
const port = 3000;

app.get('/', (req, res) => {
  res.json({ query: req.query.q, original: qs.unescape(req.query.q), url: req.url });
});

app.listen(port, () => console.log(`Example app listening on port ${port}!`));
```

결과가 어떤가? 야근을 안해도 되겠는가?

![요시! 집으로 이쿠조!](/images/good_job_20190312.png)


# 이건 또 뭐지?

집에 가기 전에 몇 가지 테스트를 좀 더 해봐야겠다.

하지만 이내 괜히 테스트를 더 한게 아닌가 하는 좌절감에 빠졌다...

웹 브라우저 또는 `curl`을 이용해서 `localhost:3000?q=100%한글` 을 실행해 보자.

![WTF 2!!](/images/wtf2_20190312.png)

세종대왕님이 창조하신 한글은 위대하지만... 컴퓨터 사이언스에서 한글은... 

아무튼, 문제는 `%` 가 한글 앞에 붙으면 한글이 깨져버린다는 사실!

위에서 말한 내용 중 `%16진수16진수` 부분과 연관이 있는 부분이다.

`%한` 이라는 문자열을 [querystring.unescape()](https://nodejs.org/dist/latest-v10.x/docs/api/querystring.html#querystring_querystring_unescape_str) 이친구가 멋대로 디코딩 해 버린 것이다!

어쩌겠는가... `% + 2byte` 를 디코딩 하도록 작성된 코드일 뿐인것을...


# 해결 방안은?

파라미터에서 `%` 가 이상한 패턴으로 나오면 `%`를 escape 해버리면 어떨까? 

함수를 하나 만들어 보자.

```javascript
const parseParam = param => {
  const result = { encoded: false, param };
  if (/%(?=[a-f0-9]{2})/gi.test(param)) {
    result.original = qs.unescape(result.param.replace(/%(?=%)/g, '%25'));
    result.encoded = true;
  }

  return result;
};
```

파라미터를 받아서 파라미터가 인코딩 되었는지 확인 후 `qs.unescape()` 로 디코딩 한다.

`replace` 는 `%`가 그냥 문자로 사용될 경우 `한글 30%증가` 와 같은 파라미터가 들어오면 여전히 한글이 깨지기 때문에 escape 처리 한 것이다.

`qs` 는 `querystring` 이다. 전체 코드는 요 바로 아래에 있다.

```javascript
const qs = require('querystring');
const express = require('express');
const app = express();
const port = 3000;

const parseParam = param => {
  const result = { encoded: false, param };
  if (/%(?=[a-f0-9]{2})/gi.test(param)) {
    result.original = qs.unescape(result.param.replace(/%(?=%)/g, '%25'));
    result.encoded = true;
  }

  return result;
};

app.get('/', (req, res) => {
  res.json(parseParam(req.query.q));
});

app.listen(port, () => console.log(`Example app listening on port ${port}!`))
```

그럼 이제 정말 집에 갈 수 있을까?

![그렇다 퇴근이다!](/images/go_to_home_20190312.png)