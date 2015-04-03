---
layout: post
title: SensorJS - 웹개발자가 사물과 대화하는 방식
---

여기는 사물 인터넷(Internet of Things)기술을 이야기하는 블로그입니다. 사물의 연결에 대한 이야기로 시작해 보겠습니다.
많은 개발자가 사물 인터넷에 관심을 가지고 여러 가지 쉽게 시도해 볼 수 있는 도구들이 있지만, 여전히 사물과 연결하고 대화하는 것은 어려운 일로 여겨집니다.

> 웹 개발자도 센서와 쉽게 대화할 수 없을까?

이러한 의문에 대한 Thing+ 개발팀의 시도가 바로 [Sensor.js](https://github.com/daliworks/sensorjs) 오픈소스소프트웨어입니다.
[Express.js](http://expressjs.com/)를 통해 웹 응용프로그램을 쉽게 만들 수 있습니다. 마찬가지로, Sensor.js를 통해 센서 응용프로그램을 쉽게 만들 수 있기를 기대합니다. Express.js에 친숙한 분들은 쉽게 Sensor.js 앱을 이해하실 수 있습니다. 두 경우 모두 동일한 [Connect](https://github.com/senchalabs/connect)의 패턴을 따랐으니까요. Connect는 미들웨어(middleware)라 불리는 플러그인을 이용해서 쉽게 웹서버를 만드는 방법을 제공합니다.

> 센서 드라이버를 웹 개발자도 작성할 수 없을까?

웹개발자가 가장 어려운 하는 부분이 각 센서별로 너무 다른 방식으로 초기화하고, 값을 읽어 내고 해석해내는 드라이버를 제작하는 부분입니다. 이 부분도 Sensor.js를 이용해 자바스크립트로  개발할 수 있도록 할 수있습니다. 예를 들어, [HTU21D](http://www.meas-spec.com/product/humidity/HTU21D.aspx)는 유선 센서 네트워크인 I²C[^1]에 연결하여 사용할 수 있는 습도센서입니다. 이 센서만의 특별한 방식으로 초기화하고, 값을 읽어내고, 이것을 Sensor.js가 원하는 형태로 전달하는 과정을 직접 구현한 드라이버 예를 참고하세요 - [HTU21D 드라이버](https://github.com/daliworks/sensorjs/blob/master/lib/sensor/driver/digitalHumidity/HTU21D.js)

![BBB와 HTU21D](/assets/bbb+sensors.jpg)
_BeagleBone Black에 HTU21D등의 여러 센서 연결_

 ✓ Sensorjs내에 이미 활용할 수 있는 많은 센서드라이 들이 포함되어 있습니다. - [기본 센서드라이버들](https://github.com/daliworks/sensorjs/blob/master/lib/sensor/README.md)

 ✓ 또한, 독자적으로 만들어진 드라이버들을 사용할 수 있습니다. 예를 들어, 카메라 연결하려면, [sensorjs-foscam](https://github.com/daliworks/sensorjs-foscam)를, TI SensorTag를 사용하려면 [sensorjs-ble](https://github.com/daliworks/sensorjs-ble)를 활용해 보세요. 

Sensor.js의 사용 설명 슬라이드입니다. 센서 애플리케이션을 어떻게 작성하였는지, 센서 드라이버를 직접 제작하는 방법을 구체적으로 설명하고 있습니다.
[Sensor.js App and Driver](/assets/sensorjs_slides/index.html)


---
[^1]: I2C는 두 가닥의 선(하나는 데이터, 하나는 동기화를 위한 clock)으로 데이터를 주고받는 유선 센서 네트워크입니다. 두 가닥의 선만 필요하고, 여러 개의(112개까지 가능) 센서를 연결할 수 있어서, 게이트웨이 장비에 직접 센서를 연결할 때 많이 사용됩니다.  http://en.wikipedia.org/wiki/I%C2%B2C 
