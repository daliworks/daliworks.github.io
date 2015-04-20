---
layout: post
title: "Nightwatch.js - Part1 : 테스트 자동화"
tags: nightwatch test
published: true
---

**[나이트와치](http://nightwatchjs.org)**는 웹 개발 코드에 대한 단위 테스트가 아니라 브라우저 기반에 실제 사용자의 환경과 동일하게 테스트할 수 있는 자동화 툴입니다.

웹 개발에서 서버와 클라이언트의 단위 테스트에 대한 방법에 대해서는 좋은 툴들과 다양한 방법들이 있습니다. [Thing+](https://www.thingplus.net) 서비스에서도 서버와 클라이언트에서 모두 사용하고 있지만 단위 테스트만으로는 실제 사용자의 웹 환경에서 발생할 수 있는 오류를 놓치는 경우가 발생합니다.

이를 방지하기 위해 사용자의 환경과 동일하게 하는 End-to-End 테스트가 반드시 필요하며, 나이트와치는 쉬운 문법과 설정으로 다양한 사용자 환경을 만들어 테스트를 할 수 있는 유용한 툴입니다.

## 나이트와치의 구조

나이트와치는 브라우저 기반의 테스트 프레임웍을 제공하는 오픈소스 프로젝트인 [Selenium](http://www.seleniumhq.org)을 이용한 [node.js](https://nodejs.org) 기반의 자동화 툴입니다.

Selenuim은 웹 테스트를 위한 여러 프로젝트를 오픈 소스로 진행하고 있으며, 나이트와치는 그중에 
[Selenium Webdriver](http://docs.seleniumhq.org/projects/webdriver/)를 사용해서 Selenium 서버와 통신으로 브라우저를 제어합니다.

![나이트와치 구조](http://nightwatchjs.org/img/operation.png)


## 나이트와치 환경 설정


### 1) Node.js와 나이트와치 설치

나이트와치는 node.js 기반이기 때문에 먼저 node.js설치가 필요합니다.
설치 방법은 [node.js](https://nodejs.org) 공식 사이트의 설치 가이드를 참고하면 됩니다. 

node.js 설치 후에 나이와치를 npm install을 이용해서 설치합니다. 전체 시스템에서 나이트와치를 사용하기 위해서는 `-g` 옵션을 이용합니다.

```sh
$ npm install nightwatch
```


### 2) Selenium 서버 설치

Selenium 서버는 자바 기반으로 되어 있어서 아래 자바를 우선 설치합니다.

- [Java JDK >= 6](http://www.oracle.com/technetwork/java/javase/downloads/index.html)
- `java` 와 `jar` 경로 설정

자바 설치가 되면 [Selenium 다운로드 페이지](http://selenium-release.storage.googleapis.com/index.html)에서 최신 버전을 다운 받습니다. 현재는 selenium-server-standalone-2.45.0.jar이 최신 버전입니다.

다운로드 받은 jar파일은 다음 명령으로 실행을 할 수 있습니다.

```sh
$ java -jar ./selenium-server-standalone-2.45.0.jar
```

Selenium이 정상적으로 실행되었다는 메시지가 나오면서 대기 상태가 되면 정상 동작입니다.


### 3) 브라우저 드라이버 설치

브라우저 드라이버는 Selenium 서버와 브라우저의 통신을 위한 [WebDriver's wire protocol](https://code.google.com/p/selenium/wiki/JsonWireProtocol)을 지원하는 플러그인 입니다. 테스트를 진행할 브라우저별로 해당하는 드라이버를 설치해 주어야 합니다.

- Safari, Firefox : selenium 서버에 내장
- Chrome : [다운로드](https://sites.google.com/a/chromium.org/chromedriver/downloads) 페이지에서 최선버전(현재 2.15) 중에 운영체제에 맞는 버전을 다운로드 후에 실행 가능한 PATH 에 추가
- Internet Explorer : [드라이버 정보](http://selenium-release.storage.googleapis.com/index.html?path=2.45/) 페이지에서 다운로드 후에 실행 가능한 PATH 에 추가

*전체 지원하는 브라우저 드라이버는 [Selenium 다운로드 페이지](http://www.seleniumhq.org/download/)의 Third Party Drivers 항목에서 찾을 수 있습니다.*


## 나이트와치 실행

드라이버까지 설치를 완료했으면 이제 나이트와치 실행에 필요한 환경을 설정해야 하는데 [나이트와치 github](https://github.com/beatfactor/nightwatch)에서 제공하는 `nightwatch.json` 예제 파일을 기준으로 몇 가지 중요한 옵션들을 살펴 보겠습니다.

```json
{
  "src_folders" : ["tests"],
  "output_folder" : "reports",
  "custom_commands_path" : "",
  "custom_assertions_path" : "",
  "page_objects_path" : "",
  "globals_path" : "",

  "selenium" : {
    "start_process" : false,
    "server_path" : "",
    "log_path" : "",
    "host" : "127.0.0.1",
    "port" : 4444,
    "cli_args" : {
      "webdriver.chrome.driver" : "",
      "webdriver.ie.driver" : ""
    }
  },

  "test_settings" : {
    "default" : {
      "launch_url" : "http://localhost",
      "selenium_port"  : 4444,
      "selenium_host"  : "localhost",
      "silent": true,
      "screenshots" : {
        "enabled" : false,
        "path" : ""
      },
      "desiredCapabilities": {
        "browserName": "firefox",
        "javascriptEnabled": true,
        "acceptSslCerts": true
      }
    },

    "chrome" : {
      "desiredCapabilities": {
        "browserName": "chrome",
        "javascriptEnabled": true,
        "acceptSslCerts": true
      }
    }
  }
}
```


#### nightwatch.json

- `src_folders`:  테스트 파일들의 경로 { string | string array }
- `output_folder`: 실행 후 생성되는 XML형태의 리포트가 저장되는 경로 { file path }
- `custom_commands_path`: 사용자가 정의 Command가 있는 경로 { file path }
- `selenium`: *Selenium 서버 실행 설정*
    + `start_process`: Selenium 서버 자동 실행 여부 { true | false }
    + `server_path`: Selenium Jar파일이 있는 경로 { file path }
    
- `test_settings`: *테스트 환경 설정*
    + `default`: 테스트 환경 이름(nightwatch 실행시에 옵션으로 사용)
        * `launch_url`: 테스트할 페이지의 URL(테스트 파일에서도 설정 가능)
        * `desiredCapabilities`: Selenium Webdriver에 **실행할 브라우저**와 옵션을 지정 ([전체 옵션 리스트](https://code.google.com/p/selenium/wiki/DesiredCapabilities))

         ```json
         "desiredCapabilities" : {
           "browserName" : "firefox", // {chrome|firefox|internet explorer|opera|safari}
           "acceptSslCerts" : true
         }
         ```

nightwatch.json 파일의 작성이 완료되면 아래와 같이 나이트와치를 실행합니다.

```sh
$ nightwatch --config ./nightwatch.json --env integration
```

`--config` 옵션은 `nightwatch.json`의 경로를 지정하고 `--env` 옵션은 위에서 설정한 `test_settings`의 리스트 중에서 하나를 입력합니다. `--env` 옵션이 지정되지 않으면 default가 실행됩니다. `--help` 로 옵션에 대한 상세 정보를 확인 할 수 있습니다.

## 테스트 케이스 작성

`nightwatch.json`의 `src_folders` 설정 경로에 아래의 샘플을 파일로 저장하고 나이트와치를 실행하면 Selenium 서버가 브라우저를 실행하여 테스트 케이스를 자동으로 진행하는 것을 볼 수 있습니다.

```javascript
module.exports = {
  'step one' : function (browser) {
    browser
      .url('http://www.google.com')
      .waitForElementVisible('body', 1000)
      .setValue('input[type=text]', 'nightwatch')
      .waitForElementVisible('button[name=btnG]', 1000)
  },
  'step two' : function (browser) {
    browser
      .click('button[name=btnG]')
      .pause(1000)
      .assert.containsText('#main', 'Night Watch')
      .end();
  }
};
```


샘플 소스의 `"step one"`, `"step two"` 각 함수의 `browser` 인수는 나이트와치의 Object입니다. 이 Object를 통해서 나이트와치에서 제공하는 API를 호출하고 필요한 테스트를 진행합니다. 그리고 테스트 마지막에는 반드시 end()를 호출하여서 Selenium 서버와의 세션을 종료해야 다음 테스트를 진행 할 수 있습니다.

테스트 케이스는 `Page Object pattern` 형태로 인지하기 편하고 API 자체는 직관적이라서 나이트와치의 [API Reference](http://nightwatchjs.org/api)를 참고 하면 어렵지 않게 작성 가능합니다.

그리고 앨리먼트 지정 방식은 CSS selector 와 XPath 두 가지 방법을 모두 제공합니다. 기본 설정은 CSS selector로 되어 있기 때문에 XPath를 사용하기 위해서는 nightwatch.json의 `test_settings`에  `"use_xpath": true` 를 추가하거나 테스트 케이스 내부에서도 다음과 같이 변경이 가능합니다.

```javascript
this.demoTestGoogle = function (browser) {
  browser
    .useXpath() // every selector now must be xpath
    .click("//tr[@data-recordid]/span[text()='Search Text']")
    .useCss() // we're back to CSS now
    .setValue('input[type=text]', 'nightwatch')
};
```


## 글을 마치며

나이트와치 환경 설정과 간단한 테스트 파일 작성까지 해 보았습니다. 나이트와치에는 여기에 소개하지 못한 웹 환경에 따른 사용자 정의 명령어, 동시에 여러개의 브라우저를 병렬적으로 실행하는 옵션 등 다양한 기능을 제공합니다. 
최근에도 나이트와치는 지속적인 개발이 진행되면서 안정성과 다양한 기능이 추가되고 있습니다. 좀 더 상세한 내용은 나이트와치 [홈페이지](http://nightwatchjs.org)에 문서화가 잘 되어 있습니다.

Part2에서는 테스트 케이스 작성에 필요한 주요 API들과 실제 나이트와치 적용을 위한 테스트 환경 구축에 대해서 좀 더 상세히 얘기해 보겠습니다.


#### 참고
- [nightwatch homepage](http://nightwatchjs.org)
- [nightwatch github](https://github.com/beatfactor/nightwatch)
- [selenium homepage](http://www.seleniumhq.org)
- [CSS Selector Reference](http://www.w3schools.com/cssref/css_selectors.asp)
- [XPath Tutorial](http://www.w3schools.com/xpath/)

