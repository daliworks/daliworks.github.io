---
layout: post
title: "Nightwatch.js - Part2 : 테스트 자동화"
tags: nightwatch test
published: true
---

지난 [Nightwatch.js - Part1](http://techblog.daliworks.net/Nightwatchjs/)에 이어서 좀더 상세한 나이트와치의 사용법에 대해서 알아 보겠습니다.


## 나이트와치 API
나이트와치의 실행을 위한 API는 Assert, Commands, Selenium Portocol, Expect(version 0.7 에서 추가됨)로 분류 되어 있습니다.


#### 1) Assert
Assert는 일반적으로 프로그램에서 사용하는 assertion 기능입니다. `.assert`와 `.verify` 두 가지 함수로 분리되는데 결과에 따라서 프로그램을 중지하거나 로그만 남기고 계속 진행 하는 두 가지를 필요에 따라서 구분해서 사용하면 됩니다.

주로 attribute, text, css class, element에 화면 표시 검증 등에 대해서 사용합니다.


#### 2) Commands
테스트를 위한 다양한 동작들을 제공하는 Selenium protocol의 API들을 하나 또는 그 이상을 묶어서 사용하기 편하게 제공하는 API들입니다. 웹 페이지 내의 element에 대해서 입력, 값 읽기, 클릭 등의 명령을 사용하고 callback 함수를 제공하여서 동작후에 Assert API로 결과에 대해서 검증할 수 있습니다.

```javascript
this.demoTest = function (browser) {
  
  browser.click("#main ul li a.first", function(response) {
    this.assert.ok(browser === this, "Check if the context is right.");
    this.assert.ok(typeof response == "object", "We got a response object.");   
  });
    
};
```


#### 3) Selenium Protocol
Selenium Protocol은 [Selenium JsonWireProtocol](http://code.google.com/p/selenium/wiki/JsonWireProtocol)에 대해 맵핑된 API들을 제공합니다.

실 사용에서는 대부분 Commands API를 사용하면 문제가 없지만, Frame이나 기타 몇 가지에 대해서는 제공하지 않기 때문에 테스트를 위해서는 필요할 수 있습니다.


아래에 샘플은 실제 테스트에서 Commands API로 구현이 불가능하여 작성한 Selenium Protocol 예제 입니다.

```javascript
client.elements('css selector', '.widget-container', function (result) {
  _.forEach(result.value, function (elemt, index) {
    client.elementIdAttribute(elemt.ELEMENT, 'id', function (attr) {
      // test each widget
    });
  });
});
```


#### 4) Expect
Expect API는 `0.7`버전에 추가된 부분으로 Assert API를 대체하여 BDD-style로 assertion 기능을 제공합니다.

```javascript
this.demoTest = function (browser) {
  // start with identifying the element
  // and then assert the element is present
  browser.expect.element('#main').to.be.present;

  // or assert the element is visible
  browser.expect.element('#main').to.be.visible;
};
```


 간단히 나이트와치의 API 구성에 대해서 설명을 했는데 전체 API는 Nightwatchjs의 [API Reference](http://nightwatchjs.org/api)에서 볼 수 있습니다.



## Custom Commands
위에 API 기반으로 테스트 케이스를 작성하다 보면 반복적으로 같은 동작의 API들을 호출해야 할 때가 많은데 이럴때 유용하게 이용할 수 있는것이 Custom Commands 입니다.

아래는 nightwatchjs 페이지에서 가져온 Custom Commands 샘플코드로 입력 받은 이미지 파일을 resize 합니다.

```javascript
exports.command = function(file, callback) {
  var self = this, imageData, fs = require('fs');

  try {
    var originalData = fs.readFileSync(file);
    var base64Image = new Buffer(originalData, 'binary')
      .toString('base64');
    imageData = 'data:image/jpeg;base64,' + base64Image;
  } catch (err) {
    console.log(err);
    throw "Unable to open file: " + file;
  }

  this.execute(
    function(data) { // execute application specific code
      App.resizePicture(data);
      return true;
    },

    [imageData], // arguments array to be passed

    function(result) {
      if (typeof callback === "function") {
        callback.call(self, result);
      }
    }
  );

  return this; // allows the command to be chained.
};
```


작성된 파일 이름을 resizePicture.js로 만듭니다. 위에 예제에서 보면 실제 호출되는 함수의 이름을 따로 지정하지 않았는데 테스트 케이스에서는 해당 파일의 이름이 호출할 함수 이름이 됩니다. 

그리고 실행 설정 파일인  `nightwatch.json`의   `custom_commands_path` property의 값을 생성한 파일이 있는 경로로 지정합니다. 그리고 테스트 케이스에서 아래와 같이 사용하면 됩니다.

```javascript
module.exports = {
  "testing resize picture" : function (browser) {
    browser
      .url("http://app.host")
      .waitForElementVisible("body")
      .resizePicture("/var/www/pics/moon.jpg")
      .assert.element(".container .picture-large")
      .end();
  }
};
```



## 테스트 실행 옵션
전체 테스트 케이스의 구조와 실행 시에 지난 번에 다루지 못했던 옵션에 대해서 몇 가지에 대해서 알아 보겠습니다.

아래는 Nightwatchjs 기반에서 Thing+ 서비스에서 실제 사용하고 있는 테스트 구조입니다.

```
config/
  ├── default.js
lib/
  ├── selenium-server-standalone.jar
custom-commands/
  ├── $ready.js
  ├── $click.js
tests/
  ├── dashboard
  |   ├── dashboard_test.js
  |   └── widget_test.js
  ├── chart
  |   └── chart_test.js
  
```

`config` 폴더는 테스트 환경에 따라서 변경되는 테스트 항목에 대해서 [node-config](https://github.com/lorenwest/node-config)를 이용하여 다르게 동작할 수 있도록 추가했습니다.

`lib` 폴더에는 실행되는 selenium server의 실행 jar파일을 두고 `custom-commands` 폴더에는 테스트 케이스에서 사용되는 Custom Commands를 `$`를 붙여서 구분해서 사용합니다.

그리고 tests에 실제 테스트 케이스를 폴더로 구분해서 분리해 두었습니다.

이렇게 테스트 케이스 작성과 구조를 만들고 전체 테스트를 하거나 일부 테스트를 진행할 경우 Command-line, 또는 Grunt를 이용하여 웹 애플리케이션과 빌드와 연동하여 실행할 수 있습니다.

### 1) Command-line
`--group`, `--skipgroup` 두 가지 옵션을 이용하면 tests 폴더의 특정 하위폴더만 지정해서 실행하거나 지정된 하위폴더만 제외하고 테스트를 진행할 수 있습니다.

```sh
# to run only dashboard directory
$ nightwatch --group dashboard

# to skip runing dashboard directory
$ nightwatch --skipgroup dashboard
```


그리고 하나의 테스트 케이스 파일만 실행할 때는 테스트 케이스 파일에 정의해 둔 `tags`를 이용해서 실행할 수도 있습니다.

```javascript
module.exports = {
  tags: ['login', 'sanity'],
  'demo login test': function (client) {
     // test code
  }
};
```

`tags` 지정 실행방법

```sh
$ nightwatch --tag login

# specify multipl tags
$ nightwatch --tag login --tag something_else
```



### 2) Grunt

나이트와치에서는 웹 애플리케이션의 가장 일반적인 빌드 툴인 [Grunt](http://gruntjs.com/)를 이용한 실행도 기본으로 제공하고 있습니다.

Gruntfile.js에서 다음과 같이 나이트와치를 로딩합니다.

```javascript
module.exports = function(grunt) {
  var nightwatch = require('nightwatch');
  nightwatch.initGrunt(grunt);
  // ...
};
```


그리고 실해을 위한 grunt task를 작성하는데 세가지 옵션을 제공합니다.

- `options` : 현재 실행 폴더에 대한 cwd 옵션만 제공
- `argv` : command-line의 옵션을 지정
- `settings` : 나이트와치 실행 환경을 지정 (`nightwatch.json` 설정 파일에서 `test_setting`에 설정한 환경 이름)


```javascript
grunt.initConfig({
  nightwatch: {
    options: {
      cwd: './'
    },
    browserstack: {
      argv: {
        env: 'browserstack'
      },
      settings: {
        silent: true
      }
    }
  }
});
```

위와 같이 Grunt의 task를 정의한 다음 Grunt를 실행하면 됩니다.

```sh
$ grunt nightwatch:browserstack
```



## 글을 마치며

지난 나이트와치의 간단한 소개에 이이서 실제적인 테스트 케이스 작성에 필요한 API와 실행 방법등을 살펴 보았습니다.


