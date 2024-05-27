---
layout: post
title: Babel asynchronously 에러 해결 - ECMAScript module configuration file
category: React Native
published: true
excerpt: "처음에 내가 생각한 원인은 다음과 같았다. nodeJS 의 기본 모듈 시스템은 commonJS 다. 그런데 현재 ECMAScript 가 사용된 부분이 감지되었다. 그래서 Babel 을 이용해 컴파일 해줘야 하는데, Babel 정상적으로 동작하지 않는 것 같다.일단 대충 이렇게 감을 잡았다. 그런데 에러 메세지에서 몇 가지 궁금한 점들이 생겼다. 이러한 차이 때문에 Babel이 ES6 모듈을 CommonJS로 변환하기 위해서는 비동기적으로 실행되어야 한다고 하는 것이다. Babel이 비동기적으로 실행된다는 것은, ES6 를 지원하기 위해 파일을 읽고 변환하는 과정이 비동기적으로 처리된다는 의미이다."
---

## 에러 발생 상황
expo 를 통해 React Native 프로젝트 환경을 준비하고 있었다. 

ios 시뮬레이터까지 세팅하고 npx expo start 로 시작하자마자 에러를 만났다. 
![](https://i.imgur.com/wb6QN84.png)

>"You appear to be using a native ECMAScript module configuration file, which is only supported when running Babel asynchronously"

그대로 번역하면 "Babel을 비동기적으로 실행할 때만 지원되는 기본 ECMAScript 모듈 구성 파일을 사용하고 있는 것 같다" 라고 한다. 

일단 문제의 원인을 정확하게 파악하기 위해 각 구성요소에 대해 정확히 알아야 했다
## 🤔 Babel 이란?
>Babel is a Javascript compiler 

[docs - What is Babel?](https://babeljs.io/docs/)

바벨은 자바스크립트 컴파일러다. 컴파일러는 프로그래밍 언어로 작성된 코드를 기계어로 바꿔주는 역할을 한다. 

하지만 자바스크립트는 인터프리터 언어인데, 왜 컴파일러가 필요할까?

그 이유는 모든 브라우저가 최신 문법과 기능(ES6 이상)을 지원하지 않기 때문에 최신 자바스크립트 코드(ES6 이상)를 이전 버전(ES5)으로 변환하는 작업이 필요하기 때문이다.

즉, 이 맥락에서 자바스크립트는 인터프리터 언어이지만, 모든 브라우저와 호환되기 위해 Babel이라는 컴파일러를 사용한다고 이해할 수 있다. Babel은 최신 문법을 구형 브라우저에서도 동작할 수 있도록 변환해준다.
## CommonJS 와 ECMAScript
CommonJS 모듈 시스템이다. NodeJS 의 기본 모듈 시스템으로 채택되었다. 
{% highlight JS%}
// add.js
module.exports.add = (x, y) => x + y;

// main.js
const { add } = require('./add');

add(1, 2);
{% endhighlight %}

ECMAScript Modules (ESM) 는 자바스크립트 모듈 시스템의 최신 표준으로, ES6 (ECMAScript 2015)에서 도입되었다. 
{% highlight JS%}
// add.js
export function add(x, y) {
  return x + y
}

// main.js
import { add } from './add.js';

add(1, 2);
{% endhighlight %}

CommonJS 는 `require` / `module.exports` 를 사용하고, ESM은 `import` / `export` 을 사용한다. 또한 CommonJS 는 동기적으로, ESM 은 비동기적으로 동작한다는 차이가 있다. 
## 원인?
이제 원인을 분석해보겠다. 에러를 다시 살펴보자. 

>"You appear to be using a native ECMAScript module configuration file, which is only supported when running Babel asynchronously"

처음에 내가 생각한 원인은 다음과 같았다. 

"nodeJS 의 기본 모듈 시스템은 commonJS 다. 그런데 현재 ECMAScript 가 사용된 부분이 감지되었다. 그래서 Babel 을 이용해 컴파일 해줘야 하는데, Babel 정상적으로 동작하지 않는 것 같다."

일단 대충 이렇게 감을 잡았다. 그런데 에러 메세지에서 몇 가지 궁금한 점들이 생겼다. 

바로 "Babel 이 비동기적으로 실행되어야 한다" 라는 부분이다.Babel이 비동기적으로 실행되어야 한다는 게 무슨 의미일까?

사실, 이 말은 ES6와 CommonJS의 차이에서 비롯된다. CommonJS 문법을 보면 `require()` 함수를 통해 모듈을 불러옵니다.
{% highlight JS%}
const { add } = require('./add');
{% endhighlight %}
이 경우 코드는 동기적으로 실행된다. 즉, 코드가 순차적으로 실행되는 것이다. 

하지만 ES6 의 `import` 는 네트워크를 통해 모듈을 로드하기 때문에 비동기적으로 모듈을 불러온다. 
{% highlight JS%}
import { add } from './add.js';
{% endhighlight %}

이러한 차이 때문에 Babel이 ES6 모듈을 CommonJS로 변환하기 위해서는 "비동기적"으로 실행되어야 한다고 하는 것이다. Babel이 비동기적으로 실행된다는 것은, ES6 를 지원하기 위해 파일을 읽고 변환하는 과정이 비동기적으로 처리된다는 의미이다.
# 해결 방법
nodeJS 가 ES6 을 인식하기 위해선 Babel 을 설정해줘야 한다.

`babel.config.js`가 CommonJS로 작성되어 있어 파일 확장자를 수정해야 한다.
Babel.config.js -> Babel.config.cjs
![](https://i.imgur.com/bpmzAib.png)

그리고 babel.config.cjs 의 옵션을 다음과 같이 수정한다. 
{% highlight JS%}
module.exports = {
  presets: [
    [
      '@babel/preset-env',
      {
        targets: {
          node: 'current'
        }
      }
    ]
  ]
};
{% endhighlight %}
여기서 `@babel/preset-env` 는 가장 많이 사용되는 Babel 프리셋 중 하나로, 최신 JavaScript 기능을 호환 가능한 버전으로 변환한다.

또한 `targets` 옵션을 통해 현재 Node.js 버전에 맞춰 코드를 변환하도록 설정한다.

마지막으로 package.json 에 `"type": "module"` 을 추가한다.
![](https://i.imgur.com/A9WWRPA.png)

#### Reference
- [CommonJS와 ESM에 모두 대응하는 라이브러리 개발하기: exports field](https://toss.tech/article/commonjs-esm-exports-field)

