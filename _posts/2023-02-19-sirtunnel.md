---
layout: single
title: 로컬 서버를 인터넷에 공유하고 싶을 때 - SirTunnel로 터널링 서버 구축하기
date: 2023-02-19 21:00:00 +0900
categories: software
---

웹 서비스를 개발하는 과정에서, 로컬에서 개발 중인 서버를 배포를 하지 않고 인터넷에 공유하고 싶은 경우가 있습니다. 이럴 때에는 보통 ngrok과 같은 터널링 서비스를 통해 로컬 서버를 인터넷에서 접근할 수 있도록 하는데요, 오늘은 ngrok의 대안으로 SirTunnel이라는 오픈 소스 프로젝트를 이용해 터널링 서버를 직접 구축하는 방법에 대해 적어 보았습니다.

## 들어가기 전에

### 터널링

로컬 서버를 터널링을 이용해 인터넷에 공유하는 기본 원리는 단순합니다. 인터넷에 연결된 터널링 서버가 받는 요청을 그대로 로컬 서버에 전달해, 로컬 서버가 요청을 대신 처리하게 해서, 로컬 서버가 마치 인터넷에 직접 연결된 것 처럼 보이게 하는 방식입니다. 그림으로 그려보자면 아래와 같습니다.

![인터넷 밖의 외부 사용자가 요청을 보내면, 그 요청이 안전한 터널 연결로 로컬로 전달된다](/assets/images/2023-02-19/intro_diagram.png)

로컬 서버를 정식으로 배포하기 전에 인터넷에 공유하고 싶은 경우는 다양합니다. 예를 들어보자면:

- 서버가 외부 서비스로부터 웹훅 요청을 받는 것을 테스트하려는 경우
- 프론트엔드가 모바일/앱 내 웹뷰 환경에서도 정상 작동하는지 확인하려는 경우
- 배포하기 전에 다른 개발자나 내부 사용자가 결과를 리뷰할 수 있도록 하는 경우

### ngrok

가장 쉽고 빠르게 터널링 서버를 사용하는 방법은 상용 서비스를 사용하는 것입니다. 대표적인 서비스 중 하나는 ngrok이 있습니다.

예시를 위해, 쿼리 파라미터로 들어온 `message` 인자를 응답으로 돌려주는 간단한 서버를 로컬의 3000번 포트에서 실행해 보았습니다.

![로컬 서버에 요청을 보내고 응답을 받음](/assets/images/2023-02-19/intro_tunnel_local.png)

먼저 [ngrok](https://ngrok.com/) 웹페이지에 접속해 로그인하고 설정 과정을 거친 후, 로컬에서 `ngrok http 3000` 명령어를 입력하면, `https://random-id-asdf.ngrok.io`와 같은 형식의 URL이 생성됩니다. 이제 이 URL로 요청하면, 로컬의 3000번 포트에 있는 서버가 요청을 받아 처리하게 됩니다.

![ngrok에서 제공한 url을 통해 로컬 서버와 같은 응답을 받음](/assets/images/2023-02-19/intro_tunnel_ngrok.png)

ngrok은 상용 서비스이기 때문에, 사용하기는 편리하지만 커스텀한 설정이 필요할 때 아쉬운 경우가 있습니다. 예를 들어서, 매번 연결할 때 난수 URL이 아니라 고정된 URL값을 사용하고자 한다면 ngrok의 유료 플랜을 사용해야 합니다.

## Sirtunnel 사용해보기

ngrok과 같은 상용 서비스에 대한 대안으로, 오늘 소개할 Sirtunnel을 사용해서 터널링 서버를 직접 운영해볼 수 있습니다. Sirtunnel은 Caddy라는 웹 서버를 기반으로 하고, 최소한의 설정으로 사용할 수 있도록 하는 데에 초점을 둔 터널링 솔루션입니다.

이번 챕터에서는 Sirtunnel과 보유 중인 도메인을 사용해 터널링 서버를 구축하는 방법을 설명해 보겠습니다. [Sirtunnel 레포지토리의 README](https://github.com/anderspitman/SirTunnel/blob/master/README.md)에 설명된 내용을 조금 더 단계별로 풀어서 작성한 것이기에, 해당 문서도 같이 참고하면 좋습니다.

### EC2

- 먼저, AWS EC2(또는 다른 인터넷에 연결된 가상 머신) 한 대를 준비합니다. 저는 t4g.nano 인스턴스에 Ubuntu 22.04 LTS 버전으로 테스트했습니다.

![EC2](/assets/images/2023-02-19/demo_ec2.png)
{: style="color:gray; font-size: 80%; text-align: center;"}

- EC2 내에서 Sirtunnel 레포지토리를 클론 받습니다.

```bash
git clone https://github.com/anderspitman/SirTunnel.git
```

- 클론받은 폴더에서 `./install.sh` 를 실행합니다. 그러면 해당 폴더에 Caddy 바이너리가 다운로드됩니다.
  - ARM 인스턴스(ex. EC2의 t4g.\*)에서는 해당 스크립트의 `amd64`를 `arm64`로 수정한 후 실행해야 합니다.

![install.sh 실행](/assets/images/2023-02-19/demo_install_caddy.png)
{: style="width: 80%; margin: auto;"}

- SirTunnel 폴더를 PATH에 추가합니다. 예를 들어서 Ubuntu에서는 아래와 같이 추가할 수 있습니다.
  - `new_path`의 `/home/ubuntu/SirTunnel` 부분을 SirTunnel 레포지토리가 클론된 곳으로 설정하면 됩니다.

```bash
new_path='PATH="/home/ubuntu/SirTunnel:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin"'
echo "$new_path" | sudo tee /etc/environment
```

### DNS

EC2 설정을 한 후에는, 원하는 도메인으로 해당 EC2 인스턴스에 접근할 수 있도록 DNS를 설정합니다. 예를 들어서, 제가 도메인을 구매한 Namecheap에서는 아래와 같이 `dev.dlwocks31.me` 도메인이 제 EC2 인스턴스로 향햐도록 설정할 수 있습니다.

![dns 설정 화면](/assets/images/2023-02-19/demo_dns.png)

### Sirtunnel 실행

그 후에는 다시 EC2로 돌아와, Sirtunnel이 설치된 폴더에서 `./run_server.sh`를 실행해 주세요. 스크립트가 정상적으로 실행되었다면, 아래와 같이 Caddy 서버가 실행되면서 출력하는 로그를 확인할 수 있습니다. _([screen](https://linuxize.com/post/how-to-use-linux-screen/)등을 이용해 백그라운드에서 스크립트를 실행해도 됩니다.)_

![run_server.sh 실행하기](/assets/images/2023-02-19/demo_run_server.png)

마지막으로, 로컬에 돌아와 ssh를 통해 터널링 서버와 로컬 사이의 터널을 열어줍니다. 도메인이 `dev.dlwocks31.me` 라면, 아래와 같은 ssh 명령어를 실행하면 됩니다. 명령어가 정상적으로 실행됬다면, `Tunnel created successfully` 라는 문구를 확인할 수 있습니다.

```bash
ssh -tR 9001:localhost:3000 ubuntu@dev.dlwocks31.me sirtunnel.py dev.dlwocks31.me 9001
# 해석: ssh 연결을 통해 서버의 9001번 포트를 로컬의 3000번 포트로 포워딩하고,
# 그 후에는 서버에서 `sirtunnel.py dev.dlwocks31.me 9001`를 실행.
```

그 후에 `dev.dlwocks31.me`로 접속하면, 로컬에서 실행되고 있는 서버로 정상 접근되는 것을 확인할 수 있습니다! HTTPS 연결은 덤입니다.
![dev.dlwocks31.me로 접속에 성공한 모습](/assets/images/2023-02-19/demo_success.png)

## Sirtunnel의 동작 방식

앞서 Sirtunnel을 이용해 로컬 서버를 인터넷을 통해 공유할 수 있는 것을 확인했습니다. 그런데, Sirtunnel은 정확히 어떻게 동작하는 걸까요? Caddy는 왜 사용하는 걸까요? 이 챕터에서는 Sirtunnel과 Caddy의 동작 방식을 조금 더 자세히 알아보도록 하겠습니다.

### Caddy

Sirtunnel의 동작 원리에 대해 이야기하기 위해, 먼저 Sirtunnel이 기반을 두고 있는 웹 서버인 [Caddy](https://caddyserver.com/)에 대해 먼저 살펴봅시다.

![Caddy 홈페이지](/assets/images/2023-02-19/caddy.png)

Caddy는 Go 언어로 작성된 오픈소스 웹 서버입니다. Caddy에는 여러 특별한 기능들이 있지만, 그 중에는 Sirtunnel이 사용하고 있는 두 가지 중요한 기능이 있습니다.

첫 번째는 Config API입니다. Caddy의 Config API를 사용하면, Caddy의 설정을 실행 중에도 동적으로 수정할 수 있습니다.

![Caddy Config API 설명](/assets/images/2023-02-19/caddy_config_api.png)
_Caddy 홈페이지 - Config API에 대한 소개_
{: style="color:gray; font-size: 80%; text-align: center;"}

두 번쨰는 자동 HTTPS 설정입니다. Caddy는 다른 웹 서버와는 달리 기본적으로 HTTPS를 사용하며, SSL 인증서를 자동으로 발급하고 갱신할 수 있는 기능이 내장되어 있습니다.

![Caddy https](/assets/images/2023-02-19/caddy_https.png)
_Caddy 홈페이지 - 자동 HTTPS에 대한 소개_
{: style="color:gray; font-size: 80%; text-align: center;"}

### Sirtunnel이 Caddy를 이용하는 법

SirTunnel은 이런 Caddy의 특성을 이용해 최소한의 코드만으로 HTTPS로 연결되는 터널링 서버를 구축합니다. [Sirtunnel 레포지토리](https://github.com/anderspitman/SirTunnel)를 살펴보았을 때, 주요 로직은 [`sirtunnel.py`](https://github.com/anderspitman/SirTunnel/blob/master/sirtunnel.py) 에 있는 단 50줄 가량의 파이썬 코드인 것을 확인할 수 있습니다.

위에서 보았던, 로컬에서 실행한 ssh 명령어를 다시 살펴봅시다.

```bash
ssh -tR 9001:localhost:3000 ubuntu@dev.dlwocks31.me sirtunnel.py dev.dlwocks31.me 9001
# 해석: ssh 연결을 통해 서버의 9001번 포트를 로컬의 3000번 포트로 포워딩하고,
# 그 후에는 서버에서 `sirtunnel.py dev.dlwocks31.me 9001`를 실행.
```

이 명령어를 실행하면 Caddy와 ssh가 같이 작동하면서 터널링이 시작됩니다. 구체적으로는, 로컬에서 터널링 서버로 ssh 연결이 맺어지면서 `sirtunnel.py`가 실행되고, 이 코드는 Caddy의 Config API를 이용해 Caddy의 설정을 동적으로 변경합니다. 그러면 Caddy는 서버로 들어온 트래픽을 9001번 포트로 전달하고, 9001번 포트로 전달된 트래픽은 다시 ssh 연결을 통해 로컬 서버로 전달되는 것입니다.

![Caddy api config](/assets/images/2023-02-19/caddy_api_config.png)
_`sirtunnel.py` 실행 이후에, Caddy의 설정이 변경된 것을 이렇게 Config API를 통해 확인할 수 있음_
{: style="color:gray; font-size: 80%; text-align: center;"}

또한, Caddy의 설정이 변경된 후에는, Caddy의 자동 HTTPS 설정 덕분에 `dev.dlwocks31.me`로 들어온 트래픽이 자동으로 HTTPS로 들어오게 됩니다. Caddy 설정이 변경된 후, Caddy 서버의 로그를 살펴보면 아래와 같이 자동으로 HTTPS 인증서를 가져온 흔적을 찾아볼 수 있습니다.

![Caddy Auto Https](/assets/images/2023-02-19/caddy_auto_https.png)

## 응용하기 - 원하는 서브도메인으로 연결

위에서 확인했다시피, Sirtunnel은 기본적으로 Caddy를 최대한으로 활용하는 짧은 파이썬 스크립트입니다. 그래서 Sirtunnel은 기능이 단순하지만, 반면에 내부 동작을 잘 이해하고 있다면 기능을 확장하는 것 또한 간단하다는 장점도 있습니다.

예를 들어서, 한 도메인 내에서 원하는 서브도메인으로 터널 연결을 하고 싶다면, 이를 약간의 명령어 변경으로 쉽게 달성할 수 있습니다.

먼저, 아래와 같이 DNS를 설정해 `*.dev.dlwocks31.me` 도메인을 EC2로 연결합니다.

![*.dev DNS 설정](/assets/images/2023-02-19/apply_dns.png)

그 후에는 ssh 명령어를 아래와 같이 조금 변경한 후 실행합니다.

```bash
ssh -tR 9002:localhost:3000 ubuntu@dev.dlwocks31.me sirtunnel.py abc.dev.dlwocks31.me 9002
# 기존 ssh 명령어와 달라진 점은: 서버의 9001번 포트를 포워딩하던 것을 9002번으로 수정했고,
# 도메인 또한 abc.dev.dlwocks31.me 로 수정. ("abc"는 원하는 도메인으로 교체 가능.)
```

그러면 아래와 같이 바로 `abc.dev.dlwocks31.me` 로 접근이 가능한 것을 확인할 수 있습니다.

![abc.dev.dlwocks31.me 도메인으로도 접근이 가능한 모습](/assets/images/2023-02-19/apply_working.png)

이 때 Caddy의 설정을 보면, `dev.dlwocks31.me`로 들어온 요청과 `abc.dev.dlwocks31.me`로 들어온 요청을 처리하는 리버스 프록시가 별개로 설정된 것을 확인할 수 있습니다.
![*.dev DNS 설정](/assets/images/2023-02-19/apply_caddy_config.png)

이렇게 여러 서브도메인으로 터널링 연결이 열려 있으면, Caddy가 각각의 도메인으로 오는 트래픽을 별개로 처리하기 때문에, 서버 포트만 서로 다르다는 전제 하에 (위에서 9001, 9002번으로 설정한 부분) 동시에 여러 명이 터널 연결을 열어도 모두 문제없이 작동합니다. (실제로 실무 환경에서 사용할 때는, 서버 포트를 충분히 큰 범위 내에서 랜덤으로 지정하는 방식 등으로 충돌을 피할 수 있습니다.)

## 마치며

이렇게 Caddy와 Sirtunnel을 이용해 간편하게 직접 터널링 서버를 구축해볼 수 있었습니다. 저는 전 회사에서 터널링 기능이 필요해 Sirtunnel 서버를 직접 구축하고 운영했었는데요, Sirtunnel을 직접 설정해 보면서 Caddy나 EC2 설정에 대해서 많이 배울 수 있었던 기억이 있습니다. 또한, 상용 서비스들과 비교했을 때에도 딱 필요한 기능만 쓸 수 있어 편리했고, 동시에 상용 서비스를 구독하는 것 대비 비용 절감이 되는 점도 좋았습니다.

이렇게 로컬 서버를 인터넷에 공유하기 위해 터널링이 필요할 때, Sirtunnel을 한번 사용해보는 것을 추천드립니다. 또, Github의 [anderspitman/awesome-tunneling](https://github.com/anderspitman/awesome-tunneling) 레포지토리에는 Sirtunnel이나 Ngrok과 유사한 다른 터널링 솔루션들도 많이 나열되어 있는데요, 여기에 있는 다른 대안들에 대해서도 더 알아보는 것도 추천드립니다.
