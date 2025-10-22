+++
title = "나의 UPS 구매기"
date = 2025-10-23
description = "UPS 구매 및  NUT 설치 과정"
[taxonomies]
categories = ["infra"]
tags = ["UPS", "NUT", "On-Prem"]
[extra]
lang = "ko"
toc = true
+++

# UPS 구매 동기
홈 서버를 꾸린지 6개월이 넘어가며 크고 작은 홈서버의 문제를 겪게 되었다. 집은 IDC만큼 안락한 곳이 아니었다. 일상생활에서 내가 깨닫지 못했을 뿐 정말 작은 정전들은 잦게 일어나고 있었고 6개월 동안 주변 전기공사나 날씨 문제로 인해 두 번이나 서버가 꺼졌다. 최소한 Graceful Shutdown으로 서버를 보호해야겠다고 생각했다!

# UPS, Uninterruptible Power Supply
서버는 일반적인 PC와 달리 연속성과 안정성이 기대되는 장비이다. 연속성과 안정성이 확보되지 않으면 작게는 서비스가 중단되는 상황이 벌어질 수 있으며, 심각한 경우에는 파일 시스템 결함과 데이터의 손상이 일어날 수 있고, 서버 하드웨어의 손상까지 이어질 수 있다.

무정전전원장치(UPS)는 안정적으로 전력을 공급하기 위한 장치로, 전력 공급이 중단되거나 불안정할 시 UPS 내의 배터리에 저장된 전력을 이용해 안전된 전력을 공급할 수 있게 돕는다. 정전 시 UPS를 통한 Graceful shutdown으로 서버를 보호할 수 있고, 정전 상황 해제 후 관리자의 개입 없이 시스템을 재시작할 수 있어 전원 사고에 대한 걱정 없이 더 편리하게 서버를 관리할 수 있다!

# UPS 성능
요구되는 UPS의 성능을 파악하기 위해서는 다음 네 가지를 고려해야 한다.

1. 용량 : UPS가 감당할 수 있는 전력의 총량. UPS가 연결된 모든 기기의 소비전력보다 1.5배 이상의 전력량을 소비할 수 있어야 한다.
2. 출력 파형 : 정전 시, UPS가 배터리로 동작할 때 공급되는 전기의 파형. 장비의 안정성과 수명을 유지하기 위해 고려해야 하는 성능이다. 일반적으로 서버를 안정적으로 운영하기 위해서는 순수사인파가 적합하다.
3. 배터리의 용량: 정전 시, UPS가 공급하는 전원의 시간. UPS는 장비가 안정적으로 시스템을 종료할 수 있도록 종료까지의 시간을 벌어줘야 한다. 배터리의 용량과 소비전력을 기반으로 안전한 시스템 종료에 필요한 백업 시간을 보장할 수 있는 용량이어야 한다.
4. 전환 시간: 정전 감지 후, 상용전력에서 배터리 전력으로 전환하는 데 소요되는 시간. 일반적으로 4~8ms 이내가 요구되나 더 짧을수록 좋다.

이에 더하여, 서버 OS와의 호환성, UPS 장비 자체의 수명과 효율을 결정할 배터리 종류도 고려하는 것이 좋다.

# 용량 산정
UPS가 감당할 수 있는 전력의 용량을 계산해야 한다.

내 홈서버는 SBC 3개와 Mac Mini로 이루어진 단촐한 구성이다.

## 홈서버 구성 환경
- Orange Pi 5 2대, Raspberry Pi 4B 1대
- Mac Mini M4

## 예상 전력 소모량
각 기기의 최대 전력 소모량(Maximum Power Consumption)은 다음과 같다.
- Mac Mini: 최대 전력 65W
- Orange Pi 5 8GB: (권장 전원 어뎁터 5V/4A) 20W
- Raspberry Pi 4B: (권장 전원 어뎁터 5V/3A) 15W
- ASUS RT-AX3000 Router: (input power 19V/1.75A) 33.25W

보수적으로 계산하여 최대 사용량은 153.25W이며, *UPS 안정성을 위해 최대 사용량의 1.5배에서 2.0배를 선택하는 것이 권장*되므로, 300W급 이상의 제품을 선택해야 했다.

## UPS 타입
- 스탠바이 (Standby) UPS : 정전이나 전압 이상이 감지되면, 아주 짧은 시간 안에 배터리 전원으로 전환하여 전력을 공급하는 방식
- 라인 인터랙티브 (Line-Interactive) UPS: 내부에 AVR(Automatic Voltage Regulator, 자동 전압 조정 장치)가 평상시에도 전압을 안정적으로 관리해주고, 정전 시 배터리로 전력을 공급하는 방식
- 온라인 (Online) UPS : 컨버터를 통해 배터리를 충전하고, 동시에 인버터를 거친 전력을 장비에 공급하는 방식

나의 경우, 서버를 구성하는 모든 제품이 *SMPS(Switched-mode Power Supply)* 전원 어댑터를 사용하므로, *스탠바이 UPS*를 사용해도 괜찮을 것으로 예상했다.


# 배터리 종류
- 납 축전지 : 납산전지(VRLA), 밀폐납산전지(AGM)등은 저렴한 가격 덕에 가정용 UPS에서 가장 많이 사용된다. 수명이 짧고 충전속도가 느리다는 단점이 있다.
- 리튬 이온 전지 : 높은 에너지 밀도와 긴 수명으로 전자기기와 자동차에도 사용되는 배터리이다.
  - LFP(LiFePO4), 리튬인산철 : 에너지 밀도가 낮아 동일 부피 대비 용량이 작으나, 안전성이 높고 수명이 길다는 장점이 있다.
  - NMC(니켈망간코발트) : 높은 에너지 밀도로 고효율을 기대할 수 있지만, 안정성이 낮고 수명이 짧으며, 코발트의 가격 때문에 배터리가 비싸진다는 단점이 있다.
- 나트륨 전지: 리튬을 대체할 유력한 차세대 배터리로, 원료가 저렴하고 매우 안정적이며 화재 위험이 적다. 다만, 아직 상용화되지 않은 소재이다.

# 고려한 제품들 및 선정 이유/ 앞으로 나오길 희망하는 스펙

## APC Smart-UPS SCL500RM1UC
완벽한 APC UPS 관리 소프트웨어 지원과 순수 사인파 출력이 가능하기에 이상적인 모델이다. 리튬 이온 배터리를 이용하여 기대 수명이 긴 것도 장점이다.
그러나 SBC들로 이뤄진 내 홈서버에 사용할 목적으로는 과한 스펙과 가격인 반면, 대용량 리튬 이온 배터리는 가정에서 사용하기에 다소 위험하다고 판단했다.

## GOLDENMATE LiFePO4 Battery UPS / ECOFLOW River3
리튬 배터리들 중 열 안정성이 뛰어난 LiFePO4를 이용한 제품들이다. LiFePO4의 긴 수명과 안정성이 기대되는 제품들이다.
리튬 이온 배터리의 단점이 완화되었지만, 데이터 케이블을 통한 통신이 불가능해 서버를 위한 UPS의 기능을 하지 못하는 EPS(Emergency Power Supply,비상전원공급장치)에 가까운 제품들이다. 따라서 서버를 보호하기 위한 목적에는 적합하지 않은 제품들로 판단했다.

## APC Back-UPS ES 550
가정용 UPS 제품으로 납산전지(VRLA)를 사용한다.
순수 사인파 출력을 지원하지 않고, 배터리 핫스왑을 지원하지 않는다는 단점이 있지만,
저렴한 가격으로 사용자가 많아 문제 발생 시 커뮤니티의 도움을 받기 쉽고, 데이터 케이블 통신을 지원하여 NUT 등을 통해 가정용 데스크톱 PC와 NAS를 보호하기에 적합하다.

최근 몇 년간 자동차를 포함한 여러 분야에서 LiFePO4 배터리의 가능성이 주목받고 있다. 하지만 시중에서 쉽게 구할 수 있는 제품 중 서버 보호용으로 적합한 LiFePO4 배터리의 UPS 제품은 찾기 쉽지 않다. LiFePO4 배터리가 사용된 제품들은 APC 같은 기존의 IT 인프라 하드웨어 제조사들의 제품이 아닌 스타트업의 실험적인 제품 뿐인 것 같아 보인다. 몇 년 전에 UPS 유행 시에도 가정용 제품으로 APC Back-UPS ES 550이 거론되었던 것 같은데 아직도 이 제품 외에는 딱히 개선된 선택지가 없는 것이 아쉽다. LiFePO4 배터리나 나트륨 전지를 이용한 합리적인 가격의 안전한 가정용 UPS가 나오길 기대해 본다.

# [NUT, Network UPS Tools]("https://networkupstools.org/ddl/APC/Back-UPS_ES_550.html")
NUT은 UPS를 포함한 다양한 전원 장치를 위한 오픈소스 프로젝트이다. NUT로 UPS와 통신하여 전력 상태를 감시하고 제어할 수 있게 하여, 비상 시 설정값에 따라 서버를 종료하는 등의 대처를 할 수 있게 한다.

## [Homebrew를 활용한 NUT 설치]("https://formulae.brew.sh/formula/nut")

1. Homebrew를 통한 NUT 설치
```zsh
brew install nut
```
2. UPS와 연결
```zsh
ls /dev/cu.usbmodem*
```
해당 명령어를 통해 연결된 UPS USB 모뎀을 찾을 수 있다.

3. NUT config 수정
[`ups.conf`]("https://networkupstools.org/docs/man/ups.conf.html") : NUT와 통신할 UPS의 정보를 설정한다.
ES550 모델의 경우 `usbhid-ups` 드라이버를 사용한다. UPS가 사용하는 드라이버는 [Hardware compatibility list]("https://networkupstools.org/stable-hcl.html")에서 확인할 수 있다.
```conf
# /opt/homebrew/etc/nut/ups.conf
[ups] # UPS 이름
    driver = usbhid-ups # UPS 드라이버
    port = /dev/cu.usbhid1111 # 2에서 얻은 장치 이름
    desc = "APC ES550 for Mac" # UPS 설명
```

[`upsd.conf`]("https://networkupstools.org/docs/man/upsd.conf.html") : upsd 데이터 서버 설정. UPS 상태를 확인하려는 클라이언트의 네트워크 규칙을 위한 설정이다.
```conf
# /opt/homebrew/etc/nut/upsd.conf
LISTEN 127.0.0.1 4000
```

[`upsmon.conf`]("https://networkupstools.org/docs/man/upsmon.conf.html"): upsmon 클라이언트 설정. 지속적으로 UPS 상태를 감시하는 클라이언트 데몬이다.
```conf
MONITOR myups@localhost 1 local_admin mysecret master
SHUTDOWNCMD "/sbin/shutdown -h +0" # UPS 정전 시 Mac을 종료시키기 위한 명령어
```

4. NUT 서비스 실행
```zsh
sudo brew services start nut
```

정상적으로 연결되었다면, `upsc myups@localhost` 명령어로 UPS 정보를 확인할 수 있다.

# 향후 계획
SPOF를 피하기 위한 UPS HA 구성을 계획중이다.
