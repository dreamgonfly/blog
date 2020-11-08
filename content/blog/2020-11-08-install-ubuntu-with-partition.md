---
title: "Ubuntu 설치 시 디스크 파티션 나누기"
date: 2020-10-08
slug: "install-ubuntu-with-partition"
description: "How to install Ubuntu 18.04 with disk partitioning"
keywords: ["ubuntu"]
draft: false
tags: ["ubuntu"]
math: false
toc: true
---

Ubuntu 18.04를 재설치하며 루트 디렉터리 (`/`) 와 홈 디렉터리 (`/home`)의 파티션을 나누면서 겪었던 과정과 트러블 슈팅을 정리하여 공유합니다.

### 파티션을 나누면 좋은 이유

홈 디렉터리가 별개의 스토리지나 파티션에 있다면 데이터를 잃지 않으면서 운영체제를 재설치하기가 간편해집니다. 재설치를 할 때 홈 디렉터리는 포맷하지 않고 운영체제가 담긴 파티션만 포맷 후 새 운영체제를 설치하는 것으로 끝나기 때문입니다.

> #### 참고: 디스크 파티션이란
> 디스크의 스토리지의 영역을 나누는 것을 "디스크 파티셔닝"이라고 부릅니다. 각 파티션의 위치와 크기는 디스크의 "파티션 테이블"이라는 곳에 저장됩니다. 운영체제는 디스크를 읽을 때 이 테이블을 가장 먼저 읽으며 각 파티션은 운영체제에게 논리적으로 독립된 디스크로 인식됩니다.

## 프로세스

### 0. 백업하기

중요한 데이터를 백업하는 것을 잊지 마세요.

### 1. 부팅하기
USB 등 Ubuntu 설치 장치(Ubuntu Installation media)로 컴퓨터를 부팅합니다. 저는 [UNetbootin](https://unetbootin.github.io/)로 만든 부팅용 Live USB(bootable Live USB drive)를 사용했습니다.

![Installation type](/images/install-ubuntu-with-partition/KURnS.png)

BIOS/UEFI 모드로 진입해서 부트 메뉴에서 부트 로더를 선택합니다. 이 때 USB에 부트 로더가 BIOS와 UEFI 모드 두 종류 있는 것을 볼 수 있습니다. 제 경우에는 `Samsumg Type-C 1100` (제가 가진 USB 이름)과 `UEFI: Samsumg Type-C 1100`가 있었습니다. 이 중 UEFI 모드를 선택합니다.

### 2. 설치 유형 선택하기

![Installation type](/images/install-ubuntu-with-partition/KURnS.png)

설치 과정 중 4번째까지 진행하면 설치 유형 선택 창이 나타납니다. 이 과정이 제일 중요합니다. 파티션을 나누어 설치하기 위해서는 여기서 "Something else"를 선택합니다.

### 3. 파티션 나누기

![Partitioning](/images/install-ubuntu-with-partition/3DBJC.png)

`/dev/sda`와 같은 이름의 스토리지를 볼 수 있습니다. 파티션을 처음부터 다시 나누고 싶다면 `New Parition Table...`을 선택합니다. 이렇게 하면 모든 디스크 영역이 free space로 바뀌게 됩니다. 이미 있는 파티션을 수정하여 새 파티션을 만들 수도 있습니다.

> #### 참고: `/dev/sda`의 의미
> `/dev/`는 device의 약자로 Unix에서 모든 장치 파일을 담고 있는 디렉터리입니다. Unix는 접근 가능한 모든 것을 읽고 쓸 수 있는 파일로 취급합니다. `sd`는 `SCSI device`라는 뜻입니다. SCSI는 Small Computer System Interface의 약자로 하드 디스크 등 주변기기와 컴퓨터를 연결하기 위한 인터페이스입니다. 이후 `sd`는 데이터를 담는 모든 장치를 뜻하는 용어로 쓰이게 되었습니다. `sda`, `sdb`, `sdc` 등으로 장치를 구분합니다.

### 3-1. Swap 파티션

![Swap Partition](/images/install-ubuntu-with-partition/lNmDo.png)

Swap 파티션은 메모리가 부족하거나 컴퓨터가 잠자기 모드일 때 메모리 페이지를 담는 파티션입니다. 사용성을 위해 Swap 파티션을 만드는 것을 추천합니다. Swap 파티션의 크기는 메인 메모리 크기보다 커야 잠자기 모드를 사용할 수 있습니다. 또한, Swap 파티션의 위치는 디스크의 끝에 둘 수도 있지만 이 경우 속도가 느려집니다.

### 3-2 Root 파티션

![Root partition](/images/install-ubuntu-with-partition/f9AS5.png)

루트 파일 시스템 `/`을 위한 파티션을 만듭니다. 이 곳에는 커널, 부트 파일, 시스템 파일, 커멘드 라인 유틸리티, 라이브러리, 시스템 설정과 로그 파일 등이 들어갑니다. 보통 10 ~ 20GB면 충분하지만 도커 이미지가 디폴트로 시스템 영역에 저장되기 때문에 저는 첨부된 사진보다 더 큰 영역을 할당했습니다. Root 파티션을 생성할 때 제가 쓴 파라미터들은 다음과 같습니다.

- Type for the new partition: Primary
- Location for the new partition: Beginning of this space
- Used as: Ext4 journaling file system
- Mount point: /


> #### 참고: Primary vs. Logical
> 파티션에는 Primary와 Logical 두 가지 유형이 있습니다. 가장 큰 차이는 primary 파티션만이 BIOS가 부트 로더를 찾는 위치로 지정할 수 있다는 것입니다. 즉, primary 파티션에서만 부팅할 수 있으므로 운영체제는 주로 primary 파티션에 담깁니다. 일반적으로, 디스크 드라이브는 최대 4개의 primary 파티션을 갖거나 3개의 primary, 1개의 extended 파티션을 가질 수 있습니다. Logical 파티션의 개수에는 제한이 없습니다.

> ![Logical vs. Primary](/images/install-ubuntu-with-partition/logical-vs-primary-3.png)

> 참고로, parimary 파티션과 logical 파티션의 구분은 MBR 디스크에서만 존재합니다. GPT 디스크에는 primary 파티션만 있습니다.

### 3-3 Home 파티션

Home 파티션을 만드는 방법은 Root 파티션과 동일합니다. 파일 시스템을 다른 형식으로 지정하는 것도 얼마든지 가능합니다. Home 파티션에는 모든 남은 스토리지 용량을 할당합니다.

### 3-4 EFI 파티션

UEFI 모드로 설치할 때는 반드시 독립된 EFI 파티션이 필요합니다. 이 파티션은 FAT32 포맷으로 구성하고 300MB ~ 500MB 정도의 공간만 할당하면 됩니다.

ESP라고도 불리는 EFI System Partition은 컴퓨터가 부팅될 때 UEFI 펌웨어가 운영체제와 유틸리지를 시작하기 위해 필요한 파일들을 저장하는 곳입니다. ESP에는 부트 로더나 커널 이미지, 디바이스 드라이버, 운영체제 전에 실행되는 시스템 유틸리티 프로그램 등이 들어있습니다.

![EFI partition](/images/install-ubuntu-with-partition/efi-partition.png)

참고로 BIOS 모드 설치 시에는 EFI 시스템 파티션 대신 `/boot` 디렉터리에 마운트된 ext4 형식의 파티션이 필요합니다.

### 3-5 부트 로더 설치 장치

부트 로더를 설치할 장치는 디폴트로 둡니다. 직접 설정한다면 특정 파티션이 아니라 디스크 전체를 선택해야 합니다.

![Partitioning result](/images/install-ubuntu-with-partition/psm5Z.png)

## 트러블 슈팅

모든 일이 순조롭게 흘러간다면 이 블로그 글을 쓰는 일도 없었을 것입니다. OS 재설치를 하며 겪은 이슈들과 해결한 방법을 정리해 보겠습니다.

### Separate Boot loader Code Error

파티셔닝을 하고 설치를 진행 중에 다음과 같은 에러 메시지가 떴습니다.

> “Ubuntu Error: The Partition table format in use on your disks normally requires you to create a separate partition for boot loader code. This partition should be marked for use as a Reserved BIOS boot area and should be at least 1MB in size. Fix this or else you will get errors during the Ubuntu Installation process”.

원인은 간단했습니다. UEFI가 아닌 BIOS를 통해 Ubuntu를 설치하려고 했기 때문입니다. Legacy Mode라고도 불리는 BIOS는 부팅을 하기 위해 독립된 Grub 파티션이 필요하기 때문에 위와 같은 에러가 생긴 것입니다.

해결 방법은 부트 메뉴에서 Ubuntu 설치를 시작할 때 UEFI 이름이 붙은 USB 장치를 선택하는 것입니다. USB를 꽂고 부트 메뉴를 시작하면 같은 USB가 두 가지 이름으로 있는 것을 볼 수 있습니다. 하나는 그냥 USB 이름이고 다른 하나는 UEFI라는 태그가 붙은 이름입니다. 그냥 USB 이름을 선택하면 BIOS 모드로 로딩합니다. UEFI 태그가 붙은 이름을 선택하면 UEFI 모드로 로딩합니다. 설치 과정을 취소하고 재시작한 뒤 UEFI 모드를 선택하면 됩니다.

### grub-efi-amd64-signed failed

설치가 거의 끝나갈 즈음에 다음과 같은 에러 메시지가 뜨며 설치가 중단되었습니다.

> grub-efi-amd64-signed failed installation /target/ Ubuntu 18.04

이 에러는 여러 원인에 의해 발생할 수 있습니다. 제 경우에는 설치 USB를 굽기 전에 제대로 포맷하지 않았던 문제였습니다. USB를 포맷한 후 다시 Ubuntu 설치 장치를 만드니 에러가 해결되었습니다.

### Turning off Secure Boot

운영체제 설치 후, `nvidia-smi`가 작동하지 않는 문제가 있었습니다. UEFI 설정에 Secure Boot가 Windows optimized로 되어 있어서 Other OS로 변경하니 이슈가 사라졌습니다.

## References

- https://askubuntu.com/questions/343268/how-to-use-manual-partitioning-during-installation
- https://askubuntu.com/questions/840434/how-to-reinstall-ubuntu-but-keep-personal-files
- https://superuser.com/questions/558156/what-does-dev-sda-in-linux-mean
- https://www.easeus.com/partition-master/logical-vs-primary.html
- https://www.linuxquestions.org/questions/linux-newbie-8/the-major-difference-between-ext4-and-reiserfs-933194/
- https://tecrobust.com/separate-boot-loader-code-error-and-fix/
- https://askubuntu.com/questions/789998/16-04-new-installation-gives-grub-efi-amd64-signed-failed-installation-target#comment1187206_789998
- https://askubuntu.com/questions/827491/is-separate-efi-boot-partition-required