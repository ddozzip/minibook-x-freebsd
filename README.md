# Chuwi MiniBook X (N150) — FreeBSD 15.1 S3 Suspend/Resume Fix

Chuwi MiniBook X (Intel N150 Twin Lake, BIOS DNN20 V2.50)에서 FreeBSD 15.1-RELEASE의
S3 suspend/resume이 완전히 동작하게 만드는 설정 모음.

## 증상

1. `acpiconf -s 3`으로 재운 뒤 깨우면 **커널이 행** — 강제 전원 리셋만 가능, 로그 없음
2. 커널 행을 해결한 뒤에도 **화면이 영영 안 들어옴** (ssh는 됨)

## 원인 분석

### 1. 커널 행: iwlwifi + sdhci resume 실패

`debug.acpi.suspend_bounce=1` (실제 S3 진입 없이 장치 suspend/resume 경로만 시험)로 확인:

- **iwlwifi (Intel AX101)**: resume 후 레지스터가 전부 `0xff` — PCIe 버스에서 장치가
  응답하지 않음 (`iwlwifi0: Queue 2 is stuck`)
- **sdhci_pci0 (SD 컨트롤러, pci0:0:26:0)**: `Controller timeout`

이 두 장치가 복구에 실패하면서 resume 경로에서 커널이 멈춘다.
BIOS의 S3 자체는 정상 (같은 기기에서 Linux deep sleep 동작 보고 있음).

→ **해결**: [`rc.d/suspend_fixup`](rc.d/suspend_fixup) — suspend 직전 두 장치를 분리,
resume 후 재연결. FreeBSD 표준 훅(`rc.suspend`/`rc.resume`의 `rcorder -k suspend/resume`)에
자동으로 걸린다.

### 2. 검은 화면: drm-kmod의 VBT MIPI GPIO op 미구현

drm 디버그(`dev.drm.__drm_debug=0x1df`)로 resume 로그를 떠보면:

```
intel_dsi_vbt_exec: Starting MIPI sequence 6 - MIPI_SEQ_BACKLIGHT_ON
intel_dsi_vbt_exec: Unsupported MIPI operation byte 3
```

operation byte 3 = **GPIO 조작**. FreeBSD drm-kmod(drm-612-kmod 6.12.85 기준)는
`mipi_exec_gpio`를 구현하지 않아 그냥 건너뛴다. VBT(블록 53)를 파싱해보면 이 패널(DSI-1,
EDID 없음, 1200x1920)의 전원 시퀀스는 GPIO에 전적으로 의존한다:

| 시퀀스 | GPIO | Linux i915 기준 실제 레지스터 |
|---|---|---|
| POWER_ON | `MIPI_AVDD_EN_1` = 1 | `PP_CONTROL(0)` 0xC7204 bit0 |
| INIT_OTP 서두 | `MIPI_RESET_1` 1→0→1 토글 | `SHOTPLUG_CTL_DDI` 0xC4030 HPD_A를 GPIO 출력으로 |
| BACKLIGHT_ON | `MIPI_BKLT_EN_2` = 1 | `PP_CONTROL(1)` 0xC7304 bit2 |

부팅 때는 BIOS(GOP)가 패널을 켜줘서 문제가 없지만, S3에서 PCH가 리셋되면
i915가 직접 이 GPIO들을 복구해야 하는데 못 한다. 결과:

- 패널 AVDD 꺼짐 + 리셋 안 됨 → 280개의 DSI 초기화 패킷이 죽은 패널로 전송됨
- 백라이트 enable 안 됨

(참고: 디스플레이 파이프 자체는 살아있다 — vblank 카운터 증가 확인됨.
 DPMS 토글, VT 전환, X 재시작으로는 복구 불가능.)

→ **해결**: [`sbin/dsi-panel-resume`](sbin/dsi-panel-resume) — Linux
`icl_native_gpio_set_value()`와 동일한 레지스터 조작을 `/dev/mem`으로 재현:
AVDD on → 리셋 토글 → 백라이트 on → `xrandr`로 패널 off/on (드라이버가 INIT_OTP
재전송). `suspend_fixup`의 resume 훅에서 자동 실행된다.

## 설치

```sh
install -o root -g wheel -m 555 rc.d/suspend_fixup /usr/local/etc/rc.d/
install -o root -g wheel -m 555 sbin/dsi-panel-resume /usr/local/sbin/
```

`boot/loader.conf`의 i915 튜너블(DC states/PSR off)은 참고용 — 이것만으로는 해결되지
않았지만 해로울 것도 없어 유지 중.

필요 패키지: `python3.12` (dsi-panel-resume의 MMIO 조작용), X11 + sddm 환경 가정.

## 진단에 쓴 도구 메모

- `debug.acpi.suspend_bounce=1` — S3 진입 없이 suspend/resume 경로만 시험
- `dev.drm.__drm_debug=0x1df` — drm 상세 로그
- `mount -t lindebugfs none /compat/linux/sys/kernel/debug` 후
  `dri/0/i915_vbt` — VBT 덤프 (블록 53 size 필드가 0인 것은 알려진 VBT 버그,
  실제 크기는 내부 u32)
- 화면 없이 커널 생존 판정: `acpiconf -s 3 && touch /root/resume_ok && reboot`

## 남은 일

- drm-kmod에 `mipi_exec_gpio` native(ICL+) 경로를 구현하는 패치가 정석.
  Linux의 `icl_native_gpio_set_value`는 레지스터 조작만 하므로 이식 가능.
  → freebsd/drm-kmod 업스트림 이슈/패치 후보
