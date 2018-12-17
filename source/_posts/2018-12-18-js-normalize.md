---
title: JS - String.normalize()
date: 2018-12-18 01:00:00
comments: true
tags: ["JS", "normalize"]
categories: ["JavaScript", "Basic"]
---

# 먼저 들어가기 앞서...

본인은 앱등이라 Mac OS X를 사용한다.

그런데 메일로 첨부파일을 보내면 받는 쪽 (아마도 Windows 사용자?) 에서 파일명이 왜 이렇게 오냐고 뭐라고 한 적이 있었다.

대충 아래와 같은 느낌? 🤬

```
ㅈㅏㅂㅏㅅㅡㅋㅡㄹㅣㅂㅌㅡㅉㅏㅇ.doc
```

그때는 그냥 OS끼리 호환이 안되나 보다 하고 별거 아닌듯 넘겼었는데

오늘 이것 때문에 엄청난 시간을 허비했다...

## 재현

5개의 `depth`를 가진 카테고리 값을 가진 속성이 있다.

하위 카테고리로 내려갈수록 카테고리 분류가 더욱 상세히 바뀐다. 따라서 상위 카테고리보다 하위 카테고리의 갯수가 앞도적으로 많다.

아직 초기 개발단계라 상위 카테고리에 해당되는 이미지들만 DB에 저장하려고 했다. 한 10개 정도?

카테고리 명은 당연히 한글이고, DB에 저장할 이미지들도 일단은 한글 카테고리명에 맞게 저장을 하였다.

아마도... 이런 느낌?

```javascript
import fs from "fs";
import path from "path";
import db from "../db";

const loadImages = dirpath => fs.readdirSync(dirpath).map(parse);

const parse = v => {
  const filename = path.basename(v);
  return {
    key: filename.replace(/\..{3}$/, ""),
    img: filename
  };
};

const update = async dirpath =>
  await Promisee.all(
    loadImages(dirpath).map(
      async v =>
        await db.Model.update({ img: v.img }, { where: { key: v.key } })
    )
  );
```

`db`는 `sequelize`를 사용했다. 대충 그런 느낌?

아무튼 *point*는, 저런 방식으로 `key`에 매칭되는 로우의 `img` 값을 업데이트 하려고 하였다.

하지만... 몇시간 동안... 로우는... `img`는... 꿈쩍도 하지 않았다...

# 멘붕

카테고리명을 `가전제품` 이라고 예를 들면, 일반적으로 `"가전제품".length` 의 값은 `4` 로 평가된다.

즉, 한글 문자 1개를 1로 평가한다.

근데 `fs.readdir` 함수로 읽은 파일명은 화면에서는 `가전제품` 이라고 출력되면서 `length`가 10 으로 평가되었다.

이게 뭐지? 혼란스럽다.

# 조언

인코딩이 문제인줄 알았다.

내장 메소드인 `encodeURIComponent()`, `decodeURIComponent()`도 같이 써보고,

내 Mac이 혹시 엉뚱한 문자셋을 사용하는지 확인도 해보고

`MySQL`의 스키마와 테이블이 문자셋이 일치하는지도 확인 해보고

`img` 문자셋도 이리 바꿔보고 저리 바꿔보고...

다 부질 없어서 팀장님께 도움을 요청했고,

흘리듯이 _100% Mac이라서 문제인거 같은데..._ 라고 하셨다.

# 유레카!

가장 처음 말했던 메일 첨부파일명이 거지같이 나오던 문제가 떠올랐다.

바로 구굴님께 여쭤보았다.

그러다가 `NFD (Normalization Form Canonical Decomposition)`, `NFC (Normalization Form Canonical Composition)` 이라는 생소한 단어를 알겍 되었다.

전자의 경우는 Mac OS X에서 사용되는 유니코드 정규화 방식이며, 후자는 Windows 에서 사용되는 방식이다.

자세한 내용은 [여기](https://blogs.technet.microsoft.com/spsofficesupportko/2017/01/06/%ED%8C%8C%EC%9D%BC%EB%AA%85%EC%9D%98-%ED%95%9C%EA%B8%80%EC%9E%90%EB%AA%A8%EA%B0%80-%EB%B6%84%ED%95%B4%EB%90%98%EC%96%B4-%EB%B3%B4%EC%97%AC%EC%A7%80%EB%8A%94-%ED%98%84%EC%83%81-unicode-nfd/) 를 참고하면 참 좋겠다.

근데 둘 다 표준이라 사실 뭘 쓰던 상관은 없는 셈이다.

영어권 국가에서 태어났다면 이런 고민도 없었을 텐데... 코드도 좀 더 읽기 쉽게 작성할 수 있었을 텐데... 영어공부나 해야겠다.

아무튼 OS가 보여주고 싶은 맘데로 보여주는거라 사실상 터미널에서 보이던 `가전제품` 이라는 문자열은 이미 `ㄱㅏㅈㅓㄴㅈㅔㅍㅜㅁ` 으로 되어 있는 것이었다.

~~(Mac OS X 이 과하게 친절한 녀석... 😭)~~

또한, [깜장토끼님의 블로그](https://m.blog.naver.com/PostView.nhn?blogId=kiros33&logNo=220671385630&proxyReferer=https%3A%2F%2Fwww.google.co.kr%2F) 에서 [String.normalize()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/normalize) 라는 엄청난 녀석을 발견했다.

# 해결

굉장히 간단했다.

`"가전제품".normalize('NFC')` 라고 `normalize()` 메소드에 `NFC` 라는 값을 넣어주면 해결!

# 결론

영어 공부를 하자. 😜
