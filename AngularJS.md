# Drama & Company AngularJS Style Guide
드라마앤컴퍼니에서 사용하는 AngularJS의 스타일 가이드입니다.
이 스타일 가이드는 기본적으로
[[John Papa의 스타일 가이드](https://github.com/johnpapa/angular-styleguide)]
를 참조하여 만들어졌습니다.

이 글에서는 John Papa에서 제시하는 스타일 가이드 외의 내용에 대하여만 명시하고 있습니다. 이 글에 나와있지 않은 부분들은 모두 [[John Papa의 스타일 가이드](https://github.com/johnpapa/angular-styleguide)]를 참고하시면 됩니다.

## 목차

  1. [Feature File Names](#feature-file-names)
  1. [Folders-by-Feature Structure](#folders-by-feature-structure)
  1. [Performance](#performance)
  1. [Events](#events)
  1. [Common directive modules](#common-directive-modules)
  1. [Constants](#constants)
  1. [Bower](#bower)
  1. [그 외 참고한 스타일 가이드들](#그-외-참고한-스타일-가이드들)

## Feature File Names
[[Style Y121](https://github.com/johnpapa/angular-styleguide#style-y121)]

가이드에 나와있는 대로 `{feature}.{type}.{format}`와 같은 형식을 따르지만, type의 길이가 너무 길어지게 되므로, type을 줄여서 사용합니다.

### type의 종류
* ctrl: Controller
* drtv: Directive
* fltr: Filter
* svc: Service, Factory 등
* model: Model
* const: Constant
* cfg: Config

### format의 종류
* css
* js
* html

## Folders-by-Feature Structure
[[Style Y152](https://github.com/johnpapa/angular-styleguide#style-y152)]

이 가이드의 directive 부분에 나와있는 대로 같은 directive에 대한 HTML, CSS, JS 파일들을 모두 한 폴더 안에 넣는 것이 관리와 접근성에 있어서 좋은 방법이나, Ruby on Rails의 기본 `assets pipeline`과 `views/../*.erb`을 그대로 사용하는데 있어서는 좋지 않습니다. 따라서 이 경우에는 `javascripts` 폴더의 controller나 directive에서 사용하는 구조 그대로 `stylesheets`, `views`에 적용합니다. 예는 다음과 같습니다.

```
.
└── app
    ├── assets
    │   ├── javascripts
    │   │   ├── app
    │   │   │   └── card
    │   │   │       ├── controllers
    │   │   │       │   └── cardInfo.ctrl.js
    │   │   │       ├── directives
    │   │   │       │   └── card.drtv.js
    │   │   │       └── services
    │   │   │           └── cardData.svc.js
    │   │   └── application.js
    │   └── stylesheets
    │       ├── app
    │       │   └── card
    │       │       ├── controllers
    │       │       │   └── cardInfo.ctrl.scss
    │       │       ├── directives
    │       │       │   └── card.drtv.scss
    │       │       └── services
    │       │           └── cardData.svc.scss
    │       └── application.css
    └── views
        └── card
            ├── controllers
            │   └── cardInfo.ctrl.html.erb
            └── directives
                └── card.drtv.erb
```

## Performance
mgechev angularjs-in-patterns
[[영문](https://github.com/mgechev/angularjs-style-guide#performance)]
[[한글](https://github.com/mgechev/angularjs-style-guide/blob/master/README-ko-kr.md)]

## Events
Custom 이벤트는 다음과 같은 두 경우를 제외하고는 최대한 지양합니다. Event(특히 $rootScope로 모두 보내버리는)는 관리가 매우매우 힘들어집니다.
* 어플리케이션 전체에 걸친 event (토큰 만료 등)
* 1-depth length로만 이루어진 directive와 controller간의 통신

Controller끼리 통신하기 위해서는 service를 이용합니다.

## Common directive modules
꼭 이 프로젝트 뿐만이 아니라, 모든 AngularJS 프로젝트에서 사용할 수 있는 컴포넌트들은 git에 open source로 올려서 Bower에 등록하는 것을 권장합니다. 이렇게 할 경우, 다른 프로젝트들에서 같은 코드를 가져와 사용할 수 있으며, 완벽히 의존성이 없는 모듈성 코드를 작성할 수 있습니다.

## Constants
Client key와 같은 constant 값들은 보통 Rails의 config 파일에 보관합니다. 그리고 constant 파일들만 담아두는 javascript파일을 erb 파일로 만들어서 사용합니다.

```javascript
// constants.const.js.erb
angular
  .module('myApp')
  .constant('FB_CLIENT_ID', '<%= Rails.application.config.facebook_client_id %>')
  .constant('GA_CLIENT_ID', '<%= Rails.application.config.ga_client_id %>');
```

## Bower
모든 javscript/css 등의 라이브러리, 패키지 들은 [Bower](http://bower.io)를 이용하여 사용합니다. AngularJS를 비롯한 모든 패키지들은 버전을 항상 명시하여 사용합니다.

## 그 외 참고한 스타일 가이드들
* Todd Motto의 angularjs-styleguide
[[Link](https://github.com/toddmotto/angularjs-styleguide)]

* mgechev의 angularjs-in-patterns
[[Link](https://github.com/mgechev/angularjs-in-patterns)]
