---
title: "Extreme Engineering: 134MB RAM으로 구동하는 FreeBSD 데스크탑 아키텍처 분석"
description: "Extreme Engineering: 134MB RAM으로 구동하는 FreeBSD 데스크탑 아키텍처 분석"
pubDate: "2026-01-21T07:55:10Z"
---

현대의 소프트웨어 생태계는 '자원 효율성'보다는 '개발 편의성'에 치우쳐 있습니다. Electron 기반 앱 하나가 수백 MB의 메모리를 점유하는 시대에, 운영체제와 GUI 환경 전체를 200MB 미만의 RAM으로 구동한다는 것은 단순한 호기심을 넘어선 **극한의 최적화 엔지니어링(Extreme Optimization Engineering)** 과제입니다.

이 글에서는 FreeBSD 15.0-RELEASE를 기반으로, 커널 튜닝과 유저스페이스 구성을 통해 데스크탑 환경을 **134MB RAM** 수준까지 경량화하는 기술적 방법론을 심층 분석합니다.

## 1. 아키텍처 설계: "무엇을 버릴 것인가?"

메모리 점유율을 최소화하기 위해서는 기본적으로 무거운 서브시스템을 배제해야 합니다. 저자는 두 가지 핵심적인 아키텍처 결정을 내렸습니다.

### 1.1 Filesystem: ZFS 대신 UFS 채택
FreeBSD의 강력한 기능인 ZFS는 ARC(Adaptive Replacement Cache)를 통해 성능을 극대화하지만, 메모리 요구량이 큽니다. 저자는 **UFS(Unix File System)와 Soft Updates Journaling**을 선택했습니다. 이는 메모리 풋프린트를 획기적으로 줄이면서도 파일 시스템의 안정성을 보장합니다. 흥미로운 점은 Netflix의 CDN 서버들도 FreeBSD와 UFS 조합을 사용한다는 사실입니다.

### 1.2 Display Server: Xorg 대신 XLibre
저자는 현재의 Xorg 구현체가 Red Hat과 FreeDesktop.org의 영향으로 불필요하게 비대해졌다고 판단했습니다. 대신 **XLibre** X11 서버를 사용하여 레거시 의존성을 제거하고 순수한 X11 프로토콜 처리에 집중했습니다.

## 2. 커널 및 부트로더 레벨 튜닝

최적화의 핵심은 부팅 단계에서 로드되는 모듈과 커널 파라미터를 제어하는 것입니다.

### 2.1 `/boot/loader.conf` 최적화
이 설정은 커널이 로드되기 전, 부트로더 단계에서 적용됩니다. 불필요한 장치 식별자 생성과 USB 관련 대기 시간을 제거하여 초기 메모리 할당을 줄입니다.

```conf
# CONSOLE COMMON
loader_logo=none
loader_menu_frame=none
screen.font="6x12"

# CONSOLE RESOLUTION (해상도 고정)
kern.vt.fb.default.mode="1920x1080"
efi_max_resolution="1920x1080"
vbe_max_resolution="1920x1080"

# ENABLE SYNAPTICS
hw.psm.synaptics_support=1

# DISABLE DISK LABELS (불필요한 /dev/diskid/* 엔트리 생성 방지)
kern.geom.label.disk_ident.enable=0
kern.geom.label.gptid.enable=0

# POWER MANAGEMENT (드라이버 없는 장치 전원 차단)
hw.pci.do_power_nodriver=3

# AHCI POWER MANAGEMENT
hint.ahcich.0.pm_level=5
# ... (각 채널별 설정)

# NETWORK QUEUE & USB TWEAKS
net.link.ifqmaxlen=2048
hw.usb.no_pf=1           # USB 패킷 필터링 비활성화
hw.usb.no_boot_wait=1    # 부팅 시 USB 열거 대기 제거
hw.usb.no_shutdown_wait=1

# DISABLE hwpstate_intel DRIVER
hint.hwpstate_intel.0.disabled=1
```

### 2.2 `/etc/sysctl.conf` 런타임 튜닝
운영체제 실행 중 커널 동작을 제어합니다. 특히 공유 메모리(Shared Memory) 처리와 스케줄러 설정이 데스크탑 반응성에 중요합니다.

```conf
# SECURITY & PID
security.bsd.see_jail_proc=0
security.bsd.unprivileged_proc_debug=0
kern.randompid=1

# DISABLE ANNOYING THINGS
vfs.usermount=1
kern.coredump=0          # 코어 덤프 비활성화 (디스크/메모리 절약)
hw.syscons.bell=0
kern.vt.enable_bell=0

# USB SUSPEND TWEAKS
hw.usb.no_suspend_wait=1
hw.usb.no_shutdown_wait=1

# DESKTOP INTERACTIVITY (스케줄러 튜닝)
kern.sched.preempt_thresh=224
kern.sched.slice=3       # Timeshare 스레드 퀀텀 조정

# IPC & MEMORY
kern.ipc.shm_allow_removed=1
security.bsd.unprivileged_idprio=1
kern.ipc.shm_use_phys=1  # 공유 메모리가 스왑으로 페이지아웃 되는 것을 방지 (성능 유지)
kern.vt.suspendswitch=0
vfs.fusefs.data_cache_mode=0 # FUSE가 Wired Memory를 사용하지 않도록 설정
```

## 3. 유저스페이스 구성: 데몬 최소화 및 세션 관리

### 3.1 `/etc/rc.conf`: 서비스 다이어트
불필요한 백그라운드 서비스를 모두 끕니다. `sendmail`을 비활성화하고, 로그 데몬 플래그를 최소화합니다.

```conf
# SILENCE
rc_startmsgs=NO
rc_info=NO

# NETWORK (Static IP 설정 예시)
ifconfig_vtnet0="inet 10.1.1.71/24 up"
defaultrouter="10.1.1.1"

# DAEMONS
update_motd=NO
local_unbound_enable="NO"
dbus_enable=YES          # GUI 앱을 위한 필수 IPC
syslogd_flags='-s -s'    # 원격 로깅 비활성화 (보안 및 리소스)
sendmail_enable=NO
sendmail_submit_enable=YES
sendmail_outbound_enable=NO
sendmail_msp_queue_enable=NO

# FS & OTHERS
fsck_y_enable=YES
clear_tmp_enable=NO
clear_tmp_X=YES
savecore_enable=NO       # 커널 크래시 덤프 저장 안 함
```

### 3.2 Display Manager 제거 및 `~/.xinitrc`
메모리를 12MB 가량 점유하는 `xdm`, `gdm` 같은 로그인 매니저를 사용하지 않습니다. 대신 `xinit`을 통해 직접 X 세션을 시작합니다.

**핵심 트릭:** X 세션이 종료되면 사용자도 자동으로 로그아웃되도록 스크립트를 구성하여 보안을 유지합니다.

```bash
# ... (환경 변수 설정 생략)

# WM 실행
exec dbus-launch --exit-with-session openbox-session 1> /dev/null 2> /dev/null & WM=${!}

# UI 요소 (Tint2, Dzen2) 시작
( sleep 1 && ~/scripts/__openbox_restart_dzen2.sh 1> /dev/null 2> /dev/null ) &
( sleep 1 && ~/scripts/__openbox_restart_tint2.sh 1> /dev/null 2> /dev/null ) &

# WM 종료 대기
wait ${WM}

# LOGOUT LOGIC (X11 종료 시 쉘 및 로그인 프로세스 강제 종료)
XINIT=$( ps -o pid,comm | awk '/xinit/ {print $1}' )
SHELL=$( ps -o ppid -p ${XINIT} | grep -v PPID )
LOGIN=$( ps -o ppid -p ${SHELL} | grep -v PPID )
kill -9 ${XINIT} ${SHELL} ${LOGIN} 2> /dev/null

exit 0
```

## 4. 결과 분석: 206MB vs 134MB의 비밀

### 4.1 초기 측정 (6GB RAM 환경)
6GB RAM이 할당된 VM에서 테스트했을 때, 시스템은 **206MB** (xterm, htop 제외 보정치)를 사용했습니다. 이는 여유 메모리가 많을 때 FreeBSD 커널이 버퍼와 캐시를 넉넉하게 잡기 때문입니다.

![Minimal RAM Desktop](https://vermaden.wordpress.com/wp-content/uploads/2026/01/minimal-ram-desktop.png)

### 4.2 제약 환경 측정 (220MB RAM 환경) - **핵심 발견**
저자는 VM의 물리 메모리를 220MB로 제한하고 다시 테스트했습니다. 이 경우 놀라운 결과가 나타났습니다.

*   **Plain FreeBSD:** 82 MB RAM 사용 (기존 115 MB 대비 감소)
*   **Full Desktop (Openbox):** **134 MB RAM 사용**

![134MB Result](https://vermaden.wordpress.com/wp-content/uploads/2026/01/minimal-ram-220-openbox.png)

이는 **"Free memory is wasted memory"**라는 철학을 가진 운영체제들이 가용 메모리에 따라 동적으로 커널 구조체와 캐시 크기를 조절한다는 것을 증명합니다. 하드웨어 리소스가 제한될 때 FreeBSD는 더욱 공격적으로 메모리를 관리하여 134MB라는 극소량의 메모리로도 GUI 환경을 구동해냈습니다.

## 5. Community Consensus & Alternatives

기술 커뮤니티에서는 이러한 최적화에 대해 다음과 같은 통찰을 공유했습니다:

1.  **Tiny Core Linux의 존재:** 더 극한의 다이어트를 원한다면 Tiny Core Linux가 25MB 미만의 RAM으로 GUI를 구동할 수 있습니다. 하지만 이는 편의성(Copy & Paste 불가 등)을 심각하게 희생한 결과입니다.
2.  **Window Maker:** Openbox 외에도 Window Maker가 매우 가벼운 대안으로 거론되었습니다. 테스트 결과 Window Maker는 약 48MB, Openbox는 24MB 정도를 점유하여 Openbox가 더 가벼운 것으로 확인되었습니다.
3.  **suckless tools:** `dwm`과 `st` 조합도 강력한 경쟁자이지만, 설정의 난이도와 기능성 측면에서 Openbox/Tint2 조합이 훌륭한 균형점을 보여줍니다.

## Verdict

이 실험은 단순한 '기록 세우기'가 아닙니다. 레거시 하드웨어의 수명 연장, 임베디드 시스템 설계, 그리고 클라우드 인스턴스의 비용 최적화(Micro Instance) 관점에서 시사하는 바가 큽니다. FreeBSD의 **UFS + XLibre + Openbox** 조합은 현대적인 컴퓨팅 환경에서도 불필요한 레이어를 제거하면 얼마나 효율적인 시스템을 구축할 수 있는지 보여주는 모범 사례입니다.

## References

*   **Original Article:** [200 MB RAM FreeBSD Desktop](https://vermaden.wordpress.com/2026/01/18/200-mb-ram-freebsd-desktop/)
*   **Hacker News Thread:** [Discussion on Y Combinator](https://news.ycombinator.com/item?id=46665987)
*   **Vendefoul Wolf Linux:** [Inspiration Source](https://vendefoul-wolf-linux.sourceforge.io/index_en.html)
