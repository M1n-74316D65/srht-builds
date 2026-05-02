# builds.sr.ht Image Compatibility Matrix

## Quick reference

| Distro | Recommended alias | Package install command | Notes |
|---|---|---|---|
| Alpine | `alpine/latest` | `apk add PKG` | Smallest images, musl libc |
| Arch | `archlinux` | `yay -Syu PKG` | Rolling release |
| Debian stable | `debian/stable` | `sudo apt-get install PKG` | glibc, wide package selection |
| Debian testing | `debian/testing` | `sudo apt-get install PKG` | Newer packages than stable |
| Debian unstable | `debian/unstable` | `sudo apt-get install PKG` | Bleeding edge |
| Fedora | `fedora/latest` | `sudo dnf install PKG` | RPM-based |
| Ubuntu LTS | `ubuntu/lts` | `sudo apt-get install PKG` | Good GitHub Actions parity |
| NixOS | `nixos/latest` | `nix-env -iA nixos.PKG` | Reproducible; flakes supported |
| FreeBSD | `freebsd/latest` | `sudo pkg install PKG` | BSD userland |
| OpenBSD | `openbsd/latest` | `sudo pkg_add PKG` | Security-focused BSD |
| NetBSD | `netbsd/latest` | `sudo pkgin install PKG` | Broad arch support |
| Rocky Linux | `rockylinux/latest` | `sudo dnf install PKG` | RHEL-compatible |
| Guix | `guix` | `guix install PKG` | Functional package manager |

---

## Alpine Linux

**Package manager:** `apk add`  
**Default arch:** x86_64  
**Aliases:** `alpine/edge`, `alpine/latest` (3.23), `alpine/old` (3.22), `alpine/older` (3.21), `alpine/oldest` (3.20)  
**Archs:** aarch64, armhf, ppc64el, s390x, x86_64, x86  
**Update:** Daily (edge), Weekly (stable releases)  
**Note:** musl libc — some software needs porting; lightweight and fast

**Custom repos:**
```yaml
repositories:
  myrepo: "repo-url key-url key-name"
```

---

## Arch Linux

**Package manager:** `yay -Syu`  
**Alias:** `archlinux`  
**Archs:** x86_64  
**Update:** Daily (rolling release)

**Custom repos:**
```yaml
repositories:
  myrepo: "url#key-id"
```

---

## Debian

**Package manager:** `sudo apt-get install`  
**Default arch:** amd64  
**Aliases:**
- `debian/stable` / `debian/trixie` — stable
- `debian/testing` / `debian/forky` — testing (daily updates)
- `debian/unstable` / `debian/sid` — unstable (daily updates)
- `debian/oldstable` / `debian/bookworm` — oldstable (monthly updates)

**Archs:** arm64, amd64, armel, armhf, i386, ppc64el, riscv64, s390x (bookworm also: mips64el, mipsel)  
**Note:** glibc, largest package selection; good default for complex builds

**Custom repos:**
```yaml
repositories:
  myrepo: "url release component key-id"
```

---

## Fedora Linux

**Package manager:** `sudo dnf install`  
**Default arch:** x86_64  
**Aliases:** `fedora/rawhide` / `fedora/44`, `fedora/latest` / `fedora/43`, `fedora/42`  
**Archs:** aarch64, x86_64  
**Update:** Daily (rawhide/latest), Weekly (42)

**Custom repos (Fedora 41+):**
```yaml
repositories:
  myrepo: "dnf config-manager addrepo --from-repofile=URL"
```

---

## FreeBSD

**Package manager:** `sudo pkg install`  
**Default arch:** amd64  
**Aliases:** `freebsd/current` (16.0), `freebsd/latest` / `freebsd/15.x`, `freebsd/14.x`, `freebsd/13.x`  
**Archs:** aarch64, amd64, powerpc64 (13.x/14.x also: i386, powerpc)  
**Note:** Custom repositories not supported

---

## Guix System

**Package manager:** `guix install`  
**Alias:** `guix`  
**Archs:** aarch64, armhf, i686, powerpc64le, x86_64  
**Update:** Daily  
**Note:** Channels via repositories not supported in manifest

---

## NetBSD

**Package manager:** `sudo pkgin install`  
**Default arch:** amd64  
**Aliases:** `netbsd/latest` / `netbsd/10.x`, `netbsd/9.x`  
**Archs:** aarch64, amd64, armv6hf, armv7hf, armv7hfeb, i386, mipseb, mipsel, mips64eb, mips64el, ppc, sparc64  
**Note:** Custom repositories not supported

---

## NixOS

**Package manager:** `nix-env -iA nixos.PKGNAME`  
**Default arch:** x86_64  
**Aliases:** `nixos/unstable`, `nixos/latest` / `nixos/25.05`, `nixos/24.11`  
**Archs:** aarch64, armv6, armv7, x86_64  
**Update:** Daily (unstable), Weekly (stable)

**Flakes support:**
```yaml
environment:
  NIX_CONFIG: "experimental-features = nix-command flakes"
```

**Custom channels:**
```yaml
repositories:
  channel-name: "channel-url"
```

---

## OpenBSD

**Package manager:** `sudo pkg_add`  
**Default arch:** amd64  
**Aliases:** `openbsd/latest` / `openbsd/7.8`, `openbsd/old` / `openbsd/7.7`  
**Archs:** alpha, amd64, arm64, armv7, hppa, i386, landisk, loongson, luna88k, macppc, octeon, power64, sparc64  
**Note:** Binary patches applied via `syspatch`; custom repos not supported

---

## Rocky Linux

**Package manager:** `sudo dnf install`  
**Default arch:** x86_64  
**Alias:** `rockylinux/latest` / `rockylinux/8`  
**Archs:** aarch64, x86_64  
**Note:** RHEL-compatible; useful for enterprise Linux targets

---

## Ubuntu

**Package manager:** `sudo apt-get install`  
**Default arch:** amd64  
**Aliases:**
- `ubuntu/lts` / `ubuntu/noble` / `ubuntu/24.04` — current LTS
- `ubuntu/oldlts` / `ubuntu/jammy` / `ubuntu/22.04` — previous LTS
- `ubuntu/plucky` / `ubuntu/25.04`, `ubuntu/oracular` / `ubuntu/24.10` — non-LTS
- `ubuntu/focal` / `ubuntu/20.04` — older LTS

**Archs:** arm64, amd64, i386, ppc64el, s390x  
**Note:** Best choice when migrating from GitHub Actions (`ubuntu-latest` → `ubuntu/lts`)

---

## General notes

- `*` (native) arch runs at full speed; all others are emulated and may be significantly slower
- Use stable aliases (`alpine/latest`, `debian/stable`) rather than version numbers — they
  track upstream and manifests stay valid longer
- Deprecated images get a 2-week removal notice; users who accessed the image in the prior
  30 days are notified
- Multi-arch support is underway but not yet generally available
