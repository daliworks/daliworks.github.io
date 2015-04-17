---
layout: post
title: "zkHelper - Zookeeper를 이용한 분산처리 도우미"
tags: zookeeper zkHelper
published: true
---

Thing+ 서비스는 최소 수십 개 서버(이글에서는 물리적 서버가 아니라, 단위 일을 처리하는 서버 측 애플리케이션)들이 서로 협력하여 복잡한 일을 나누어서 해냅니다.

보통 여러대의서버들은 다음의 구성으로 서로 협력하여 일을 해냅니다.

- 주인은 한 명이고, 여러 명의 일꾼들 - (Master/Slaves)

  같은 자원에 여러 명이 읽기(read)작업을 하지만, 쓰기(write)작업은 한 명만 해야 되는 경우가 대표적인 경우입니다.

- 한 명은 일하고 있고, 나머지는 대기 - (Active/Standby)

  많은 양의 일을 나누어서 처리 해야되는 것은 아니지만, 장애가 생겼을 때 바로 절체(fail over)되어 바로 정상처리하는 경우입니다.

- 모두 같은 나누어서 동시에 - (Active/Active)

  모두 협력해서 일을 잘 나누어 처리해내는 경우입니다.

이러한 많은 서버를 관리하고 역할을 지정하기 위해 Thing+는 Apache Zookeeper[^zk] 를 이용합니다. 다른 대안들도 있지만, 여전히 Zookeeper는 이러한 분산 환경의 조정자 역할을 하는 가장 인기 좋은 도구입니다.

Zookeeper는 다음의 특성을 가지고 있습니다.
- znode를 단위로 관리된다.
- znode는 파일 시스템과 유사한 디렉토리 구조(path)를 가진다.
- znode는 부모(parent) 노드를 가져야 하며, 루트(root) 노드의 path는 '/'이다.
- znode에 데이터를 저장할 수 있다.
- znode의 path의 변화 또는 저장된 데이터의 변화를 통보받을 수 있다.
- znode는 Zookeeper와 클라이언트의 연결이 끊어지면 자동으로 삭제되도록 설정(Ephemeral 노드) 할 수 있다.

Thing+ 개발팀은 이러한 Zookeeper의 특성을 사용하여 분산환경에서 Zookeeper를 보다 쉽게 사용할 수 있도록 [zkHelper](https://github.com/daliworks/zkHelper)를 공개하였습니다.
 - [zkHelper]는 node.js 기반으로 구현된 오픈소스 입니다.

먼저 [zkHelper]를 사용하여 실제 Master를 선정하는 경우 znode가 어떻게 변경되는지를 살펴보겠습니다.

1) Node#1 서버 애플리케이션에서 zkHelper를 다음의 옵션으로 초기화 하면,

```javascript
{
  basePath: 'myapps'
  node: node#1:1234 // {hostname}:{port}
}
```

아래와 같이 Zookeeper에 노드들이 생깁니다.
 - /myapps 노드는 zkHelper에서 입력한 basePath 정보로 만들어지며, /myapps 노드에는 master에 대한 데이터도 함께 저장됩니다.
   현재는 하나의 노드만 존재하기 때문에 /myapps 노드에 저장된 데이터를 보면 node#1이 Master로 선정되었음을 알 수 있습니다.
 - /myapps/nodes 노드에는 zkHelper에서 입력한 node 정보, 즉 node#1:1234가 자식 노드로 저장됩니다.
 - /myapps/votes 노드에는 master 선출을 위한 티켓 정보가 자식 노드로 저장되어 있습니다.
   master는 /myapps/votes 노드의 자식 노드들 중 티켓 정보가 가장 빠른 노드가 선출됩니다.

```
/myapps <-- { "master": "node#1:1234"}
       /nodes/node#1:1234
       /votes/n_00001 <--- node#1:1234
```

2) node#2 서버 애플리케이션이 추가로 수행되면, /myapps/votes/n_00002로 2번 티켓을 받게 됩니다.
그러나 이미 node#1이 1번 티켓(가장 빠른 티켓)을 가지고 있으므로 여전히 Master 지위를 유지합니다.

```
/myapps <-- { "master": "node#1:1234"}
       /nodes
             /node#1:1234
             /node#2:1234
       /votes
             /n_00001 <--- node#1:1234
             /n_00002 <--- node#2:1234
```

3) 마찬가지로 node#3, node#4 서버 애플리케이션이 추가로 더 수행되면 /myapps/nodes 노드와 /myapps/votes 노드에 자식 노드가 생성될 것이고 Master도 변경되지 않습니다.

4) 장애가 발생하여서 Master 인 node#1 서버 애플리케이션이 Zookeeper와 연결이 끊어지면, 해당 노드(Ephemeral 노드로 등록되어 있으므로)는 사라지게 됩니다. Master가 사라졌으므로 다시 master 선출하게 되는데, 이때 Master 선출에 참여하는 모든 서버는 재시작하면서 공정하게 경쟁해서 티켓을 차례로 가져가고 티켓번호가 낮은 서버가 Master가 됩니다.
   
   다음은 node#3이 Master가 된 경우의 znode 상태 입니다.

```
/myapps <-- { "master": "node#3:1234"}
       /nodes
             /node#2:1234
             /node#3:1234
             /node#4:1234
       /votes
             /n_00005 <--- node #3
             /n_00006 <--- node #2
             /n_00007 <--- node #4
```

이번에는 Master 선출과 관련하여 [zkHelper]의 규칙을 알아보겠습니다.

- Zookeeper의 시간 단위(tick)는 2초이면, 서버가 끊어졌다고 판단하는 기준시간(sessionTimeout)은 여기서는 10초로 가정합니다.
- Master를 선출할 상황이 되면 모든 서버는 경쟁하기 위해서 재시작합니다.
- Zookeeper 10초(sessionTimeout) 내에 Master가 살아나게 되면 계속 Master를 유지합니다. 단순 유지보수 목적 등으로 재시작한 경우는 기득권을 주는 것입니다.
- 간혹 서버 간 네트워크 사정이 좋지 않아서 끊어지는 경우에는 Master 선출과정이 빈번하게 발생합니다. 이를 최소화하기 위해 일시적으로 네트워크가 끊어지더라도 5초 후에 다시 시도하도록 합니다. 5초로 정한 이유는 zookeeper의 시간 단위(tick)는 두 배에 50% 여유를 두었습니다.

마지막으로 [zkHelper]의 사용방법을 예제를 통해 알아보겠습니다.

1) Zookeeper 3개가 운용하고 있는 환경에서, 서버 Master 선출에 참여하여 Master 혹은 Slave가 되고 공통 설정을 가져오는 예제입니다.

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

2) 직접 master 선출에 참여 하지 않고, 관찰자(Observer mode)가 되어 모니터링하는 예제입니다.
- 서버 애플리케이션들이 추가되거나 감소되는지 모니터링
- Master가 바뀌는 지를 모니터링

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

#### 참고
[ZooKeeper Overview](https://cwiki.apache.org/confluence/display/ZOOKEEPER/ProjectDescription)
