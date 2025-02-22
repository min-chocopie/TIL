<h1>Container Networking</h1>
컨테이너 네트워킹은 컨테이너가 서로 통신하거나, 외부 네트워크와 통신할 수 있는 기능을 의미한다. <br>

<h2>Network Driver</h2>
도커에서는 네트워킹을 위해 가상 네트워크 인터페이스를 생성하고 설정하는 여러 가지 네트워크 드라이버를 제공한다. <br>
어느 드라이버를 사용하느냐에 따라 컨테이너가 어떤 방식으로 네트워크에 연결될지 결정된다. <br><br>

| 네트워크 모드 | 설명 | IP주소 할당 | 컨테이너간 통신 | 외부 통신 |
|------------|----|-----------|-------------|---------|
| bridge | 가상 네트워크 사용 | ✅ 있음 | ✅ 가능 (같은 네트워크 내) | ✅ 가능 (NAT 사용) |
| host | 호스트 네트워크 공유 | ❌ 없음 |  | ✅ 가능 |
| none | 네트워크 없음 | ❌ 없음 | ❌ 불가능 | ❌ 불가능 |

<h2>bridge</h2>
bridge는 도커의 기본 네트워크 드라이버로 <b>같은 브리지 네트워크에 연결된 컨테이너들 간의 통신</b>을 가능하게 한다.<br>
도커를 시작하면 기본적으로 기본 브리지 네트워크가 자동으로 생성되고, 새로 실행되는 컨테이너는 별도로 지정하지 않는 한 이 기본 브리지 네트워크에 연결된다. <br>

<br>
아래의 bridge라는 이름을 가진 네트워크가 도커에서 자동으로 생성한 기본 브리지 네트워크이다. 
<br>

```bash
$ docker network ls
NETWORK ID     NAME       DRIVER    SCOPE
6082a93edf28   bridge     bridge    local # 기본 브리지 네트워크
da4b73de334b   host       host      local
b290cbb72d71   none       null      local
```

<h3>같은 브리지 네트워크 내 컨테이너간 통신</h3>

bridge를 살펴보기 위해 두 개의 컨테이너를 배포한다. 
```bash
$ docker run -dit --name container1 alpine sh
afee68c5c901ca1b1a6fa705f562d4da3fcc8422e76b0e2a531297be9ddb08b6

$ docker run -dit --name container2 alpine sh
0175352e128ea3eefc4ef098b1ba990cc5e57762cccc67898d13e260a820bc35

$ docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED          STATUS          PORTS     NAMES
0175352e128e   alpine    "sh"      33 seconds ago   Up 32 seconds             container2
afee68c5c901   alpine    "sh"      37 seconds ago   Up 36 seconds             container1
```

브리지 네트워크 ID를 가지고 `docker network inspect`를 통해 브리지 네트워크의 자세한 내용을 확인하면 네트워크는 `172.17.0.0/16` 범위의 서브넷을 가지고 있으며, 앞서 실행한 두 개의 컨테이너가 속해있는 것을 볼 수 있다. <br>

```bash
$ docker network inspect 6082a93edf28
[
  {
    "Name": "bridge",
    "Id": "6082a93edf28e25a41a5eb33fda0c39181926ef3d3d0af3790052e42fae6f4bd",
    "Scope": "local",
    "Driver": "bridge",
    "IPAM": {
      "Driver": "default",
      "Config": [
        {
          "Subnet": "172.17.0.0/16",
          "Gateway": "172.17.0.1"
        }
      ]
    },
    "Containers": {
      "0175352e128ea3eefc4ef098b1ba990cc5e57762cccc67898d13e260a820bc35": {
        "Name": "admiring_curie",
        "IPv4Address": "172.17.0.3/16",
      },
      "afee68c5c901ca1b1a6fa705f562d4da3fcc8422e76b0e2a531297be9ddb08b6": {
        "Name": "trusting_cohen",
        "IPv4Address": "172.17.0.3/16",
      }
    },
  }
]
```

두 컨테이너를 `docker inspect` 명령어를 통해 네트워크 부분을 확인해보면 브리지 네트워크를 사용중이고 브리지 네트워크의 서브넷 대역에 속한 IP가 자동으로 할당되었다.

```bash
$ docker inspect -f '{{range $k, $v := .NetworkSettings.Networks}}{{$k}} {{$v.IPAddress}}{{end}}' container1 
bridge 172.17.0.2

$ docker inspect -f '{{range $k, $v := .NetworkSettings.Networks}}{{$k}} {{$v.IPAddress}}{{end}}' container2
bridge 172.17.0.3
```

두 컨테이너는 같은 브리지 네트워크를 사용하고 있기 때문에 위에서 확인한 IP를 가지고 `ping`을 보내면 정상적으로 응답이 오는 것을 확인할 수 있다.
```bash
$ docker exec -it container1 ping -c 3 172.17.0.3

PING 172.17.0.3 (172.17.0.3): 56 data bytes
64 bytes from 172.17.0.3: seq=0 ttl=64 time=0.463 ms
64 bytes from 172.17.0.3: seq=1 ttl=64 time=0.433 ms
64 bytes from 172.17.0.3: seq=2 ttl=64 time=0.232 ms

--- 172.17.0.3 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.232/0.376/0.463 ms

$ docker exec -it container2 ping -c 3 172.17.0.2

PING 172.17.0.2 (172.17.0.2): 56 data bytes
64 bytes from 172.17.0.2: seq=0 ttl=64 time=0.377 ms
64 bytes from 172.17.0.2: seq=1 ttl=64 time=0.117 ms
64 bytes from 172.17.0.2: seq=2 ttl=64 time=0.719 ms

--- 172.17.0.2 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.117/0.404/0.719 ms
```

<h3>다른 브리지 네트워크 내 컨테이너간 통신</h3>
브리지는 같은 브리지 네트워크에 연결된 컨테이너들 간의 통신을 가능하게 한다. <br>
해당 브리지 네트워크에 연결되지 않은 컨테이너들과는 격리(Isolation)을 제공하며 Docker의 브리지 드라이버는 호스트 머신에 자동으로 네트워크 규칙을 설치하여 서로 다른 브리지 네트워크에 있는 컨테이너들이 직접 통신하지 못하도록 한다.
<br><br>

테스트를 위해 사용자 정의 브리지 네트워크인 `my_bridge`를 생성 후 정보를 확인하면 기본 브리지 네트워크와 다른 서브넷 대역을 할당받은 것을 알 수 있다.

```bash
# 사용자 정의 브리지 네트워크 생성
$ docker network create my_bridge
5fb1286b07bae3ecb8aad9f9fb98f6cfc1ad3b6fefd915159d859efa862bae58

# 생성된 브리지 네트워크 확인
$ docker network ls
NETWORK ID     NAME        DRIVER    SCOPE
6082a93edf28   bridge      bridge    local
da4b73de334b   host        host      local
5fb1286b07ba   my_bridge   bridge    local # 새로 생성된 브리지 네트워크
b290cbb72d71   none        null      local

$ docker network inspect 5fb1286b07ba
[
  {
    "Name": "my_bridge",
    "Scope": "local",
    "Driver": "bridge",
    "IPAM": {
      "Config": [
        {
          "Subnet": "172.18.0.0/16", # 기본 브리지 네트워크(172.17.0.0/16)와는 다른 서브넷 대역 할당
          "Gateway": "172.18.0.1"
        }
      ]
    },
  }
]
```

이제 이 위에 컨테이너를 실행해본 후 기존의 브리지 네트워크와 통신이 되는지 `ping`을 날려본다. <br>
서로 다른 네트워크에서 실행중이므로 통신이 실패하는 것을 볼 수 있다. 
```bash
# 새로 생성한 브리지 네트워크에 컨테이너 실행
$ docker run -dit --name container3 --network my_bridge alpine sh
7c69f93e8ff712e895b9a41ca9c94635d4b494f44bd2f4f95418d199a2d20661

# ping을 보낼 컨테이너 IP 주소 확인
$ docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container1
$ docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container3
172.17.0.2 # container1 (기본 bridge 네트워크)
172.18.0.2 # container3 (사용자 정의 my_bridge 네트워크)

# container3에서 container1로 ping 실패
$ docker exec container3 ping -c 3 172.17.0.2

PING 172.17.0.2 (172.17.0.2): 56 data bytes

--- 172.17.0.2 ping statistics ---
3 packets transmitted, 0 packets received, 100% packet loss
```

<h2>host</h2>
host는 컨테이너가 호스트의 네트워크 스택을 그대로 사용하는 방식이다. <br>
bridge 처럼 별도의 가상 네트워크를 만들지 않고 호스트 네트워크의 인터페이스를 직접 사용하기 때문에 컨테이너에 별도의 IP가 할당되지 않고 포트로 구분하게 된다. <br><br>

`docker network inspect` 명령어를 통해 호스트 네트워크를 확인해보면 bridge와 달리 별도의 서브넷이나 게이트웨이가 설정되지 않는 것을 볼 수 있다. <br>
또한 호스트 네트워크를 사용하는 컨테이너에도 IP 주소가 할당되지 않는다. <br>

```bash
$ docker run -d --network host --name nginx nginx              
b684d7302d7eeff3ee15ad2146f68716046f9e982108e6697a488ec754813318

$ docker network inspect da4b73de334b              
[
  {
    "Name": "host",
    "Scope": "local",
    "Driver": "host",
    "IPAM": {
      "Config": null # 브리지 네트워크처럼 Subnet이나 Gateway가 별도로 설정되지 않음
    },
    "Containers": {
      "0e4a98bfa22fda1bcbac57bad63109fedc7bd3eba95248a4b4cce5f47f9fda07": {
        "Name": "nginx",
        "EndpointID": "5e12e78d59f7923c814f51e8fd88b22815620cb385d85d051f7e86438dd7f295",
        "MacAddress": "",
        "IPv4Address": "",
        "IPv6Address": ""
      }
    },
  }
]

$ docker inspect nginx --format '{{ .NetworkSettings.IPAddress }}'
# 빈칸 출력 -> 컨테이너에 IP가 할당되지 않음을 뜻함
```

<h3>host 네트워크 모드에서의 포트</h3>

앞서 살펴본 브리지 네트워크에서는 컨테이너마다 고유한 IP 주소가 할당되고 `docker run -p` 를 통해 컨테이너 포트를 호스트 포트와 매핑해줘야 했다.
```bash
# 컨테이너 내부에서는 80포트를 사용하고 호스트에서 8080으로 접근할 수 있도록 포트 매핑
$ docker run -d -p 8080:80 --name bridge-nginx nginx
163239f66e3f425de336c3ab173934f1cfbc8a12c56d546207542edcd5fa5c50

# PORTS 부분을 보면 포트가 매핑되어 있는 것을 볼 수 있음                              
$ docker ps
CONTAINER ID   IMAGE     COMMAND                  CREATED         STATUS         PORTS                  NAMES
163239f66e3f   nginx     "/docker-entrypoint.…"   2 seconds ago   Up 2 seconds   0.0.0.0:8080->80/tcp   bridge-nginx

$ curl localhost:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

그러나 호스트 네트워크에서는 컨테이너가 호스트와 동일한 네트워크 환경을 사용해 호스트의 포트를 직접 사용하므로 `-p` 옵션으로 매핑하지 않아도 된다.
```bash
$ docker run -d --network host --name nginx nginx

# PORTS를 보면 포트가 매핑되어 있지 않음
$ docker ps
docker ps
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS          PORTS                  NAMES
160a8cf31f52   nginx     "/docker-entrypoint.…"   10 minutes ago   Up 10 minutes                          nginx
```

그러나 호스트 네트워크 모드에서는 포트를 통해서 컨테이너를 구분하기 때문에 같은 포트를 사용하는 컨테이너는 동시에 실행될 수 없다. <br>
앞에서 80 포트를 사용하는 nginx 컨테이너를 하나 더 실행하면 어떻게 될까? <br>

```bash
$ docker run -d --network host --name nginx2 nginx

# docker ps로 확인해보면 실행한 nginx 컨테이너가 바로 종료된 것을 볼 수 있다. 
$ docker ps -a
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS                     PORTS     NAMES
632e793100ed   nginx     "/docker-entrypoint.…"   12 seconds ago   Exited (1) 9 seconds ago             nginx2
160a8cf31f52   nginx     "/docker-entrypoint.…"   13 minutes ago   Up 13 minutes                        nginx

$ docker logs 632e793100ed
nginx: [emerg] bind() to [::]:80 failed (98: Address already in use)
2025/02/22 12:12:47 [notice] 1#1: try again to bind() after 500ms
2025/02/22 12:12:47 [emerg] 1#1: still could not bind()
nginx: [emerg] still could not bind()
```

컨테이너가 실행되지 않고 바로 종료되는 것을 볼 수 있다. 로그를 확인해보면 `Address already in use` 라는 에러를 뱉는 것을 볼 수 있다.

<h3>브리지 네트워크와 호스트 네트워크 컨테이너 간 통신</h3>

그럼 브리지 네트워크를 사용하는 컨테이너와 호스트 컨테이너를 사용하는 컨테이너는 서로 통신할 수 있을까? <br>
테스트를 위해 브리지 네트워크를 사용하는 컨테이너를 배포한 후 bridge -> host로 통신해보면 호스트 IP 주소를 통해 접근이 가능한 것을 볼 수 있다. 

```bash
# bridge 네트워크에서 컨테이너 실행
$ docker run -it --rm alpine sh

# 컨테이너 내에서 host의 IP 주소를 이용해 컨테이너에 접근시 nginx의 HTML이 출력됨
> wget -qO- http://$(hostname -I | awk '{print $1}'):80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

<h2>none</h2>
none의 경우 이름 그대로 컨테이너에 네트워크 인터페이스를 할당하지 않는 모드이다. <br>
컨테이너가 외부와 네트워크 통신을 하지 않도록 완전히 차단되며 당연하게 IP 주소가 할당되지 않는다. <br>
완전히 독립된 환경에서 실행해야 하는 컨테이너에 적합한 모드이다. <br><br>

테스트를 위해 none 네트워크 모드로 컨테이너를 실행 후 ping 테스트를 진행하면 `Network unreachable`이라는 메세지가 뜨며 불가능한 것을 볼 수 있다.

```bash
# none 네트워크 모드로 컨테이너를 실행 후 ping 테스트 진행시 불가능한 것 확인 가능
docker run --rm --network none alpine ping -c 3 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
ping: sendto: Network unreachable
```
