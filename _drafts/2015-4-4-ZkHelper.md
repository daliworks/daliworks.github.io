---
layout: post
title: "zkHelper - Zookeeper를 이용한 분산처리 도우미"
tags: zookeeper zkHelper
published: true
---

Thing+는 Apache Zookeeper[^zk] 를 이용합니다. Thing+ 서비스는 최소 수십 개 서버(이글에서는 물리적 서버가 아니라, 단위 일을 처리하는 서버 측 애플리케이션)들이 서로 협력하여 복잡한 일을 나누어서 해냅니다. 다른 대안들도 있지만, 여전히 Zookeeper는 이러한 분산 환경의 조정자 역할을 하는 가장 인기가 좋은 도구입니다.

일반적인 분산환경에서 Zookeeper를 보다 쉽게 사용할 수 있도록 Thing+ 개발팀이 공개한 [zkHelper](https://github.com/daliworks/zkHelper) 소개합니다.  

보통 여러서버는 다음의 구성으로 서로 협력하여 일을 해냅니다.

  - 주인은 한 명이고, 여러 명의 일꾼들 - (Master/Slaves)

    같은 자원에 여러 명이 읽기(read)작업을 하지만, 쓰기(write)작업은 한 명만 해야 되는 경우가 대표적인 경우입니다. 

  - 한 명은 일하고 있고, 나머지는 대기 - (Active/Standby)

    많은 양의 일을 나누어서 처리 해야되는 것은 아니지만, 장애가 생겼을 때 바로 절체(fail over)되어 바로 정상처리하는 경우입니다.

  - 모두 같은 나누어서 동시에 - (Active/Active)

    모두 협력해서 일을 잘 나누어 처리해내는 경우입니다. 



<br>
실제 Master 선정에 대한 예를 보겠습니다.  Node#1의 zkHelper를 다음과 옵션으로 초기화 하면,


```javascript
{
  basePath: 'myapps'
  node: node#1:1234 // {hostname}:{port}
}
```

이렇게 Zookeeper에 노드들이 생깁니다. 이때는 하나의 노드밖에서 없기 때문에, 다음의 /myapps 의 데이터를 보면 node#1이 Master 임을 알 수 있습니다.

```
  /myapps  { "master": "node#1:1234"}
        /nodes/node#1:1234
        /votes/n_00001 <--- node#1
```  

이때 myapps node#2가 더 수행되면, 2번 티켓을 받게 되고 이미 node#1이 1번 티켓을 가지고 있으므로 여전히 Master 지위를 유지합니다.

```
  /myapps { "master": "node#1:1234"}
        /nodes
               /node#1:1234
               /node#2:1234
        /votes
               /n_00001 <--- node#1
               /n_00002 <--- node#2
```  

node#3, node#4 가 추가되었고, 장애가 일어나서 Master인 node#1이 zookeeper 연결이 끊어진 경우에는 해당 노드(Ephemeral node)는 사라지고, 다시 master 선출하게 됩니다. 이때 Master 선출에 참여하는 모든 서버는 재시작하면서 공정하게 경쟁해서 티켓을 차례로 가져가고 티켓번호가 낮은 서버가 Master가 됩니다. 다음은 node#3이 Master가 된 경우입니다.

```
  /myapps { "master": "node#3:1234"}
        /nodes
               /node#2:1234
               /node#3:1234
               /node#4:1234
        /votes
               /n_00005 <--- node #3
               /n_00006 <--- node #2
               /n_00007 <--- node #4
```  

다음의 가정으로 설명을 이어나가겠습니다.

 - Zookeeper의 시간 단위(tick)는 2초이면, 서버가 끊어졌다고 판단하는 기준시간(sessionTimeout)은 여기서는 10초로 가정합니다.
 - Master를 선출할 상황이 되면 모든 서버는 경쟁하기 위해서 재시작합니다. 
 - Zookeeper 10초(sessionTimeout) 내에 Master가 살아나게 되면 계속 Master를 유지합니다. 단순 유지보수 목적 등으로 재시작한 경우는 기득권을 주는 것입니다. 
 - 간혹 서버 간 네트워크 사정이 좋지 않아서 끊어지는 경우에는 Master 선출과정이 빈번하게 발생합니다. 이를 최소화하기 위해 일시적으로 네트워크가 끊어지더라도 5초 후에 다시 시도하도록 합니다. 5초로 정한 이유는 zookeeper의 시간 단위(tick)는 두 배에 50%  여유를 두었습니다. 

Zookeeper 3개가 운용하고 있는 환경에서, 서버 Master 선출에 참여하여 Master 혹은 Slave가 되고 공통 설정을 가져오는 예제입니다.

```javascript
var zk = require('zkHelper'),
options = {
  basePath: '/myapps';
  configPath: '/myapps/config',
  node: require('os').hostname(),
  servers: ['zk0:2181', 'zk1:2181', 'zk2:2181'], // zk servers
  clientOptons: {
    sessionTimeout: 10000,
    retries: 3
  }
 };
 zk.init(options, function (err, zkClient) {
   var appConfiguration;
   if (zk.isMaster()) {
     console.info('i am master')
   } else {
     console.info('master', zk.getMaster() && zk.getMaster().master)
   }
   appConfiguration = zk.getConfig();
   // do something
 });
```

<br>
직접 master 선출에 참여 하지 않고, 관찰자(Observer mode)가 되어 

 - 해당 서버들이 추가되거나, 
 - 감소되거나, 
 - Master가 바뀌는 경우를 모니터링하는 예제입니다.

```javascript
var zk = require('zkHelper'),
_ = require('lodash'),
options = {
  servers: ['zk0:2181', 'zk1:2181', 'zk2:2181'], // zk servers
  clientOptons: {
    sessionTimeout: 10000,
    retries: 3
  },
  observerOnly: true
 };
zk.init(options, function (err, zkClient) {
  var observer = new Observer('/otherApp');

  observer.on('children', _.debounce(function (path, newVal, diff) {
    consol.info('Master=%j', observer.getMaster());
    logger.info('[%s] children:[%s] added=[%s] deleted=[%s]', path, newVal, diff.added, diff.deleted);
  }, 500));
  observer.on('data', function (path, newVal, oldVal) {
    if (oldVal) {
      logger.info('master change:' + path);
    }
    logger.info('Master=%j', observer.getMaster());
    logger.info('[%s] data: %j <- %j', path, newVal, oldVal);
  });
});
```

#### 각주

[^zk]: [Apache Zookeeper](https://zookeeper.apache.org/) 는 분산환경에서 여러 서버가 서로 설정을 공유하고, 식별하고, 동기화하는 등의 조정자 기능을 제공하는 오픈소스소프트웨어입니다. 그 시작은 하둡(Hadoop)의 하위 프로젝트였지만 이제는 아파치재단의 중요한 프로젝트로 진행되고 있습니다.
