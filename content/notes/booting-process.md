+++
title = "리눅스 부팅 프로세스"
date = 2025-02-12
description = "전원을 켠 후 부터 로그인 프롬프트가 실행되기까지"
[taxonomies]
categories = ["linux"]
tags = ["linux"]
[extra]
lang = "ko"
toc = true
+++

Power ON -> Firmware -> MBR/GPT -> Bootloader -> Kernel -> Init -> Run levels -> User Commands


## 1. Firmware : BIOS/UEFI
OS를 부팅하는 데 사용되는 Firmware는 부팅 프로세스를 시작하기 전 커널과 RAM 디스크를 로드하는 역할을 한다. Firmware에는 다음 두 가지 시스템이 있다.
- BIOS, Basic Input/Output System(입출력 시스템)의 약자이다.
- UEFI, Unified Extensible Firmware Interface(통합 확장 펌웨어 인터페이스)의 약자이다.

Firmware는 POST(power-on self-test)를 실행한다.
POST는 하드웨어 구성 요소(메모리와 CPU 등) 및 주변 장치 확인, 펌웨어 자체 무결성 검사, 기본적인 시스템 기능 점검 등 컴퓨터가 작동하기 적절한 상태에 있는지 확인하는 테스트이다.
이후, Firmware는 부트로더 프로그램을 검색, 프로그램을 메모리에 로드 후 실행한다.

### UEFI, Unified Extensible Firmware Interface
- 최신 firmware 표준이다.
- FAT12, FAT16, FAT32를 포함한 다양한 파일 시스템을 지원한다.
- ESP,EFI System Partition에 의존한다. 이 파티션에 부트로더가 저장된다.
- EFI 응용 프로그램을 사용하는 방식이다.
- 보안 부팅, 빠른 부팅 속도등 BIOS 보다 향상된 기능을 지원한다.

## 2. MBR/GPT : Bootloader 로드 및 실행
펌웨어 인터페이스에 따라 드라이브 메타데이터를 저장하는 방식이 나뉜다.
- BIOS는 MBR(Master Booting Record) 방식을 주로 사용한다.
- UEFI는 GPT(GUID Partition Table) 방식을 주로 사용한다.

### GPT, GUID Partiton Table
- 기존의 MBR 방식은 2TB 이상의 하드 드라이브를 지원하지 않지만 GPT는 8ZB를 지원한다.
- 최대 128개의 파티션을 만들 수 있다.
- GPT는 파티션 정보 복사본을 디스크 여러 곳에 저장하여 데이터 안정성이 높고 파티션 손상시 복구가 용이하다.
- GPT 방식을 사용할 경우, Bootloader가 ESP 파티션에 설치되어 UEFI에 의해 실행된다.

## 3. Bootloader : Kernel 로드 및 실행
Bootloader는 리눅스 커널을 메모리에 로드하고 실행하는 역할을 한다. GRUB(Grand Unified Bootloader)가 주로 사용된다.
과거에는 LILO(Linux Loader)도 사용되었지만 지금은 멀티부트 환경과 UEFI를 지원하지 않아 잘 사용되지 않는다.
GRUB 외에 Syslinux, systemd-boot 등의 부트로더도 사용된다. 또한 최근에는 UKI(Unified Kernel Image) 등을 활용하여 아예 부트로더를 사용하지 않고 바로 커널을 부팅하기도 한다.
펌웨어, 아키텍처, 배포판 등에 따라 지원하는 부트로더가 다르므로, 설치 환경에 맞는 부트로더를 선택해야 한다.

### GRUB2, Grand Unified Bootloader
- GRUB을 사용하면 부팅시 커널 매개변수를 설정하는 것이나 다른 커널을 선택하는 것이 가능하다.
- GRUB의 설정파일은 `/boot/grub/grub.cfg`이다.

## 4. Kernel : 하드웨어 초기화 및 initramfs 실행
Bootloader는 Kernel이 포함된 vmlinux(리눅스 커널 이미지) 이미지를 ESP 파티션 또는 루트 파티션에서 찾아 메모리에 로드 후 부팅한다.
Kernel은 루트 파일 시스템을 마운트하기 위해 필요한 정보를 Bootloader에서 전달받는다.
Kernel은 실제 루트 파티션을 마운트하기 전, initramfs(초기 RAM 파일 시스템)을 임시 파일 시스템으로 사용한다.
initramfs에는 커널이 루트 파일 시스템에 접근하기 위해 필요한 드라이버 모듈 등이 저장되어 있다.
Kernel은 이를 활용하여 실제 루트 파일 시스템을 마운트한다.

## 5. Init : Init 프로세스 시작 및 시스템 초기화
init 프로세스는 시스템의 첫 번째 프로세스(PID 1)로 부팅 과정에서는 시스템 초기화에 사용된다.
Ubuntu나 Debian, Rocky Linux 등의 대부분의 리눅스 시스템에서는 Init 시스템으로 systemd를 채택하고 있다.
Systemd 외에도 SysVinit, Upstart 등의 Init 시스템도 사용된다.

### Runlevel: 리눅스 실행 레벨
0. Halt : 종료
1. Single user Mode : 단일 사용자 모드
2. Multiuser, without NFS : NFS가 없는 다중 사용자
3. Full multiuser mode : 전체 다중 사용자 모드
4. Unused : 미사용
5. X11 : 기본값, GUI 환경
6. Reboot : 재부팅

## 6. Runlevel : Target 실행
리눅스 시스템 부팅 시 시작하는 프로그램의 실행 순서를 Runlevel이라고 한다.
시스템은 정의된 Runlevel에 따라 서비스와 시스템 로깅, 사용자 인터페이스 등 다양한 프로세스를 시작한다.

---
참고 문서 :
1) [Arch boot process - Arch Wiki]("https://wiki.archlinux.org/title/Arch_boot_process")
2) Silberschatz, Abraham, et al. 운영체제. 10판, 퍼스트북, 2020.
3) [The Linux Booting Process - 6 Steps Described in Detail - freeCodeCamp]("https://www.freecodecamp.org/news/the-linux-booting-process-6-steps-described-in-detail/")
