# 마크다운의 코드블록 영역은 어떻게 색상이 칠해질까?

## 계기

어제 어느 지인께서 `Prism.js` 라는 라이브러리의 사용법을 질문해주셨다. 나보다 훨씬 더 뛰어난 개발자이신데 JS 영역은 경험해본적이 없으셔서 아마도 나에게 질문하신 것 같다. Prism은 결론적으로 내가 작성한 코드들의 가독성을 높여주기위해서 자체적으로 CSS 스타일링을 해주는 라이브러리라고 정의할 수 있겠다. 또한 굉장히 다양한 언어들을 지원해주며 추가할 수 있는 많은 플러그인 또한 제공된다.

사실 이전까지는 노션이나 웹에서 작성되는 마크다운의 코드블록의 코드 하이라이팅 기능을 당연하게 주어지는 기능이라고 생각했다. 그러나 이 또한 누군가의 개발이 적용된 부분임을 인지하니 해당 코드는 언어별로 어떻게 구현된건지 궁금해지기 시작했다.

매우 deep하게 공부해볼 생각은 없기 때문에 Prism.js의 작동원리를 간략하게 파악해보고 다른 사이트들에서는 어떤 라이브러리를 사용하는지 확인해보도록 하자

## 코드 까보기

### component

`prism.js`의 깃허브를 살펴보면 최상단에 `component` 라는 폴더를 확인해 볼 수 있다. 해당 폴더에는 수많은 js파일들이 있는데 해당 파일들은 각 언어별로 생성 되어있다. 우리는 가장 익숙한 자바스크립트의 규칙을 확인하기 위해 `prism-javascript.js` 라는 파일을 확인 해보도록 하자

```js
Prism.languages.javascript = Prism.languages.extend('clike', {....})
```

우선 가장 상단의 코드를 확인 해보자 `Prism`이란 객체는 우선 공식 도큐에 따르면 `prism-core.js, line 19` 에서 자동으로 생성된다고 한다 (편의상 생성이라는 말을 사용하겠다) 아무튼 아주 깊게 공부할 생각은 없으므로 간단하게 정리하면 `Prism` 객체를 생성하고 해당 객체의 `languages` 프로퍼티에서 `javascript` 프로퍼티를 생성해서 `languages` 객체가 제공해주는 `extend` 메서드를 이용해서 분류된 토큰 즉 소스크드들의 최소 단위에게 `id`(아마 본인 생각은 `class`가 아닌가 생각된다) 값을 부여해준다.

다시 정리하면 Prism의 extend 메서드는 기존 언어의 정의를 확장하고, 새로운 언어 정의를 만드는 데 사용된다. 첫 번째 매개변수는 확장할 기존 언어의 이름이다.

여기서 의아한 점이 엥? 나는 `javascript`를 정의중인데 `clike`는 뭐냐? 이럴 수 있는데 말 그대로 c언어와 비슷한 언어라는 소리이다. 본인은 자바스크립트가 c언어와 비슷하다는 소리를 한번도 들어본적이 없지만(이는 아마도 본인의 무지에서 발생한 문제일 것이다) 아무튼 해당 라이브러의 개발자들은 해당 토큰을 정의할 때 C언어의 규칙을 따르고 일부 수정하는 방법을 선택했다.

그리고 `Prism`의 `extend` 메서드의 두 번째 매개변수는 새로운 언어 정의를 추가하거나 기존 언어 정의를 수정하기 위한 설정을 포함하는 객체이다. 이 객체에는 확장하려는 언어에 대한 새로운 속성이나 수정할 기존 속성을 작성할 수 있다.

여기서 본인이 놀랐던 점은 우리가 너무도 쉽게 사용했던 코드블록의 하이라이팅 같은 기능들이 굉장히 복잡한 정규식 로직에 의해 구현 됐다는 부분이였다. 아래 코드를 살펴보자

```js
'class-name': [
		Prism.languages.clike['class-name'],
		{
			pattern: /(^|[^$\w\xA0-\uFFFF])(?!\s)[_$A-Z\xA0-\uFFFF](?:(?!\s)[$\w\xA0-\uFFFF])*(?=\.(?:constructor|prototype))/,
			lookbehind: true
		}
	],
```

결론적으로 말하면 `class-name` 프로퍼티 명은 실제로 `token`들에 부여될 `class` 명이다 그리고 해당 프로퍼티의 값들은 그래서 어떤 토큰들인데? 라고 정의해주는 부분이고 해당 부분은 정규식이 들어갈 수도 있고 객체 그리고 배열 값도 적용 가능하다 (해당 라이브러리가 타입스크립트로 만들지 않았기 때문에 조금 헷갈리는 부분이 있었다)

우선 해당 코드를 살펴보면 기본적으로 `clike`의 `class-name` 프로퍼티의 규칙을 가져온다 이후 일부 변경해주고 싶은 패턴을 객체 혹은 정규식을 그대로 배열값에 추가하면 된다.

그런데 정규식이야 그렇다쳐도 `lookbehind` 무엇인가? 이 부분은 부정형 전방 탐색((?=...))의 반대로, 매칭된 부분을 실제로 반환하기 전에 문자열의 일부를 되돌아보는 것을 의미한다. 따라서 매칭된 부분이 반환되지 않고, 해당 부분 이전의 문자열이 반환된다고 보면된다.

**_해당부분의 정확한 의미파악을 본인도 100% 하고있지 못해서 이 부분은 공부가 더 필요해 보인다....._**

그 다음으로는 `insertBefore` 라는 메서드를 확인해볼 필요가 있다. 아래 코드를 살펴보자

```js
Prism.languages.insertBefore('javascript', 'keyword', {
    'custom-keyword': 정규식;
}
```

위 예제에서는 `javaScript` 언어 정의에 새로운 키워드 패턴을 추가해 확장했다. `insertBefore` 메서드를 사용하여 `keyword` 토큰 이전에 `custom-keyword` 토큰을 삽입했다. 이렇게 하면 사용자 정의 키워드가 기존 키워드보다 우선하여 클래스 값이 부여된다.

이는 어제 지인과 공유한 이슈에서 생긴 문제를 해결할 수 있었는데 지인은 이미 정의된 토큰중 일부만 따로 정규식으로 분리하여 새로은 `class` 를 부여해주고 싶었다. 하지만 해당 기능이 잘 작동하지 않았는데 해당 이슈는 `insertBefore` 메서드를 이용해 해결가능할 것으로 보인다.

그러면 마지막으로 `insertBefore`의 몇가지 프로퍼티의 속성값을 확인해보도록 하자 아래 코드를 살펴보자

```js
Prism.languages.insertBefore('c', 'string', {
	'macro': {
		// allow for multiline macro definitions
		// spaces after the # character compile fine with gcc
		pattern: /(^[\t ]*)#\s*[a-z](?:[^\r\n\\/]|\/(?!\*)|\/\*(?:[^*]|\*(?!\/))*\*\/|\\(?:\r\n|[\s\S]))*/im,
		lookbehind: true,
		greedy: true,
		alias: 'property',
		inside: {
			'string': [
				{
					// highlight the path of the include statement as a string
					pattern: /^(#\s*include\s*)<[^>]+>/,
					lookbehind: true
				},
				Prism.languages.c['string']
			],
			'char': Prism.languages.c['char'],
			'comment': Prism.languages.c['comment'],
			'macro-name': [
				{
					pattern: /(^#\s*define\s+)\w+\b(?!\()/i,
					lookbehind: true
				},
				{
					pattern: /(^#\s*define\s+)\w+\b(?=\()/i,
					lookbehind: true,
					alias: 'function'
				}
			],
			// highlight macro directives as keywords
			'directive': {
				pattern: /^(#\s*)[a-z]+/,
				lookbehind: true,
				alias: 'keyword'
			},
			'directive-hash': /^#/,
			'punctuation': /##|\\(?=[\r\n])/,
			'expression': {
				pattern: /\S[\s\S]*/,
				inside: Prism.languages.c
			}
		}
	}
```

- greedy: true: 이 속성은 정규 표현식이 가능한 많은 텍스트를 보도록 한다. 이것은 매크로 정의가 여러 줄에 걸쳐 있을 때 사용된다.
- alias: 'property': 이 토큰은 'property' 토큰의 별칭으로 정의된다. 이것은 매크로가 속성과 비슷한 역할을 한다는 것을 의미.
- inside: 이 속성은 'macro' 토큰 내부의 다른 토큰들에 대한 정의를 포함하는 객체. 이 객체 내부에는 여러 토큰 및 해당 정규 표현식 패턴이 포함된다. 이들은 각각 매크로의 내부 구성요소를 강조하기 위해 사용된다.

## 결론

딥하게 공부할 예정이 아니기 때문에 내가 작성한 코드들이 어떻게 분류되어 각각 다른 `class` 이름이 부여되는지 정도만 확인해봤다. 아마 이후에는 해당 객체를 이용하여 내가 작성한 마크다운 부분을 따로 인자로 받아 `<pre>` 라는 태그를 만들어서 작성한 모든 문자열 값을 각각의 문자로 쪼개서 각각 <span>태그로 감싸주고 해당 정규식 표현에 맞는 문자들에 알맞는 `class` 가 부여되는 로직일텐데,,,, 해당 부분 로직은 다음에 살펴보도록 하겠다.

또한 `Prism`말고 해당 기능을 제공해주는 다양한 라이브러리들이 존재한다 `highlight` 라는 라이브러리도 존재하며 (티스토리가 사용중으로 보임) 깃허브는 아마 자체적으로 만든 라이브러리를 사용하지 않나? 라는 생각이들기도한다. 추가적으로 velog는 `prism`을 사용하고있다. 다른 모든 라이브러리를 면밀하게 살펴본건 아니지만 `highlight` 라이브러리도 내부적으로 각 언어들마다 정규식으로 각각 구분하고있는걸로 보아 전반적으로 해당 기능을 제공해주는 라이브러리는 비슷한 구조로 제작됐을 것으로 추측된다.
