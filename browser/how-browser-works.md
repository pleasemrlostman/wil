# 브라우저 작동 원리

## Browser rendering process

### Browser 구성 요소

---

브라우저에는 browser element가 있다.browser element 는 각 브라우저를 구성하는 요소들이고 이러한 구성요소들에 의하여 브라우저 렌더링이 어떻게 되는지 알아보도록 하자

### browser element

---

- 사용자 인터페이스: 브라우저 창에서 확인할 수 있는 요소들 주소창, 이전/다음 버튼, 홈 버튼, 새로고침 버튼 등 브라우저에서 사용자가 조작할 수 있는 요소들을 사용자 인터페이스라고 한다.
- 브라우저 엔진: 사용자 인터페이스에서 받은 명령을 렌더링 엔진에게 전달해주는 역할을 한다. 추가적으로 브라우저 엔진에서 자료저장소에 있는 자료를 찾아볼 수 있다.
- 렌더링 엔진: 요청한 URI를 브라우저 엔진에게 받아서 통신한다. 이후 서버로부터 받은 즉 URI에 해당하는 데이터를 (HTML, CSS, JAVASCRIPT)를 받아서 파싱한 후 자바스크립트 해석기 UI 백엔드에 전달해주는 역할을 한다.
- 통신: 렌더링 엔진으로부터 HTTP등의 요청을 받아서 네트워크 처리 후 다시 렌더링 엔진에게 전달한다.
- 자바스크립트 해석기: 전달받은 js 파일을 파싱한다.
- UI 백엔드: 최종적으로 렌더링 엔진에서 생성된 렌더 트리를 브라우저에 그리는 역할을 담당한다.

### browser rendering process

---

1. 먼저 사용자가 주소표시줄에 특정한 URI를 입력한다.
2. 작성한 URI 주소를 브라우저 엔진에게 전달하고 브라우저 엔진은 우선 URI에 해당하는 데이터를 자료저장소에서 찾아본다. 만약에 있다면 그 데이터를 그대로 렌더링 엔진에 전달하여 통신을 할 필요가 없다.
3. 만약에 자료저장소에 없다면 URI 값을 렌더링 엔진에게 전달하고 렌더링 엔진은 다시 해당 URI를 통신에게 전달하여 서버와 통신할 수 있게 한다.
4. 이후 전달받은 리소스를 다시 렌더링 엔진에게 전달하고 그 중 JS파일은 자바스크립트 해석기에게 전달한다.
5. HTML CSS 는 렌더링엔진이 파싱하고 이후 JS 해석기가 파싱한 결과가 다시 렌더링 엔진에게 전달되어 DOM Tree를 조작한다.
6. 그리고 최종적으로 만들어진 즉 js에 의해 조작이 완료된 DOM Tree는 Render Tree로 변경되는데 해당 Render Tree를 UI 백엔드에게 전달하여 브라우저 화면에 그리게된다.

### rendering engine working process

---

렌더링 엔진은 URI를 통해 요청을 받은 후 해당하는 데이터들을 렌더링하는 역할을 담당한다. 크롬과 IOS는 webkit이라는 rendering engine을 사용한다.

### 대략적 렌더링 엔진 동작과정

1. DOM Tree 구축을 위하여 HTML을 파싱한다.
   - HTML을 파싱한 후 content tree 내부에서 각각의 tag들을 DOM Node로 변환한다. 그리고 CSS 파일과 함께 모든 스타일 요소를 파싱한 후 자바스크립트의 파싱 결과물은 render tree를 생성한다. 즉 render object로 변환한다.
2. 이후 파싱한 HTML을 이용해서 Render Tree를 구축
   1. HTML과 CSS를 통해 만들어진 DOM Tree가 render tree로 바뀌는 것 색상 도는 면적 등 시각적 속성을 갖는 사각형을 포함한다. 정해진 순서대로 렌더링
3. 이후 해당 렌더트리를 배치 후
   1. 브라우저 화면에 배치를 시작한다. 각 node가 브라우저 화면에 정확한 위치에 표시되기 위해 이동
4. 최종적으로 브라우저에 그려진다.
   - 각 배치가 끝나면 UI 백엔드에서 해당 node들을 paint한다.

**여기서 1번과 2,3,4번은 병렬적으로 진행된다**

이는 즉 DOM 트리는 DOM 요소라는 자식요소로 구성되어 있고 render tree로 render object라는 자식요소로 구성되어 있다. 즉 DOM 트리는 생성되는 즉시 바로 render tree로 생성된다는 의미다. 이 과정에서 render tree는 바로바로 생성된다.

사용자 입장에서는 DOM Tree가 완성되는 와중에도 render tree가 생성되고 render tree가 생성되니까 render tree가 배치되고 render tree가 그려지는 것 이다.

따라서 사용자는 DOM Tree가 완벽하게 생성될 때 까지 기다릴 필요 없이 브라우저에 특정한 컨텐츠가 보여지는 것을 확인할 수 있다.

크롬에서의 렌더링엔진을 `webkit` 이라고 한다. 이제 `webkit` 의 동작과정을 단계별로 확인해보도록 하자.

### webkit의 동작과정

---

1. HTML을 parsing 하여 DOM tree를 생성한다.
   - HTML을 parsing하여 DOM Tree를 생성 DOM으로 변경된 HTML은 Javascript가 조작할 수 있다.
   - 브라우저 tag들은 (div, body…) parsing과 동시에 DOM tree로 생성된다
   - 또한 브라우저 tag들은parsing과 동시에 실행을 진행한다
     - 이 말은 보통 script태그가 body밑에 있는 이유를 설명할 수 있다. 그 이유는 tag가 실행되는데 script태그가 위나 중간에 있으면 실행되는 동안 다음 태그를 parsing할 수 없기에 마지막에 두는 것이다.
     - 그래서 script에 HTML5에 새로 추가된 기능으로 `defer` 와 `async` 속성이 있다.
2. CSS를 parsing 하여 스타일 규칙을 얻는다.
   - CSS도 HTML과 마찬가지로 object모델을 가지고 있다.
   - 스타일규칙이란 내가 DOM 트리에서 어떤 DOM node에 해당하는지 알아보는 것
   - 해당하는 DOM과 스타일규칙에 의하여 둘이 어태치먼트하게 되어 렌더트리가 생성된다.
3. DOM 트리를 생성하는 동시에, 이미 생성된 DOM tree와 스타일규칙(CSSOM)을 Attachment한다.
   - DOM node에는 attach라는 method를 가진다. 여기서 해당 메서드가 실행되면 DOM Node에서 render object (렌더트리의 구성요소)로 변환된다.
     - 이러한 이유로 DOM node에 CSS 규칙이 붙는다
   - DOM Tree의 DOM node의 attach가 실행되면 바로 render object를 만들면서 render tree를 생성한다.
   - 즉 DOM Tree가 다 만들어지고 render tree를 만드는게 아니라 DOM Tree 가 만들어지면서 바로바로 render tree가 만들어진다.
   - render object는 render tree의 구성요소로써 자신과 자식요소를 어떻게 배치하고 그려야할지 안다.
   - 모든 DOM node가 전부 render object로 생성되지는 않는다 (ex head tag)
   - html과 body또한 DOM node로서 render object로 구성되는데 이들은 render tree root이기 때문에 render view라고 불린다.
   - 나머지 DOM node들은 render object로 구성되어 render tree에 추가 됨
4. 구축한 render tree 를 layout 한다.
   - 배치는 html 요소에 해당하는 최상위 render object 에서 시작하여 화면 좌측상단부터 render object에 해당하는 DOM node를 그린다.
