# FreeSWITCH on Fedora 42 — From Zero to Running

**Status:** Tested on Fedora 42 (x86_64), September 2025  
**Install mode:** Build from source (recommended for Fedora)

FreeSWITCH publishes Debian packages; Fedora users should build from source.  
This guide gets you a working FreeSWITCH 1.10.x with sample configs, sounds, a systemd service, firewall rules, and SELinux notes — plus fixes for the most common `configure`/`make` failures on Fedora.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [1) OS Prep](#1-os-prep)
- [2) Build Dependencies](#2-build-dependencies)
- [3) Install SpanDSP 3 (required)](#3-install-spandsp-3-required)
- [4) Get FreeSWITCH Source](#4-get-freeswitch-source)
- [5) Configure, Build, Install](#5-configure-build-install)
- [6) Create Service User & Permissions](#6-create-service-user--permissions)
- [7) Systemd Service](#7-systemd-service)
- [8) Firewall (firewalld)](#8-firewall-firewalld)
- [9) SELinux (Enforcing) Notes](#9-selinux-enforcing-notes)
- [10) Verify](#10-verify)
- [Optional Libraries / Modules](#optional-libraries--modules)
  - [SQLite (required)](#sqlite-required)
  - [FFmpeg for `mod_av` (video)](#ffmpeg-for-mod_av-video)
  - [Lua / LuaJIT for `mod_lua`](#lua--luajit-for-mod_lua)
  - [ldns for `mod_enum`](#ldns-for-mod_enum)
  - [Other optional libs (png, freetype2, gumbo, libfvad, tpl)](#other-optional-libs-png-freetype2-gumbo-libfvad-tpl)
- [Troubleshooting Cheatsheet](#troubleshooting-cheatsheet)
- [Uninstall / Revert](#uninstall--revert)
- [FAQ](#faq)

---

## Prerequisites

- Fedora 42 (fresh or maintained)
- Sudo privileges
- Internet connectivity (to fetch sources and sounds)

---

## 1) OS Prep

```bash
sudo dnf upgrade -y
sudo reboot
```

## 2) Build Dependencies
Toolchain + common libs used by FreeSWITCH:
```bash
# Dev toolchain
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y git cmake autoconf automake libtool pkgconf-pkg-config \
  gcc gcc-c++ make wget which

# Core libs
sudo dnf install -y libedit-devel libuuid-devel libcurl-devel openssl-devel \
  pcre-devel libxml2-devel ncurses-devel zlib-devel yasm gnutls-devel

# Audio/codec & media
sudo dnf install -y libsndfile-devel opus-devel speex-devel speexdsp-devel \
  libogg-devel libvorbis-devel libtheora-devel libjpeg-turbo-devel \
  libtiff-devel portaudio-devel

# Databases (recommended)
sudo dnf install -y unixODBC-devel postgresql-devel

# **Critical since FS 1.10.4+ (unbundled)**
sudo dnf install -y sofia-sip-devel
sudo dnf install -y ldns ldns-devel # install ldns runtime + headers on Fedora
sudo dnf install -y ffmpeg-free ffmpeg-free-devel #FFmpeg dev headers (libavformat + libswscale).
sudo dnf install -y sqlite sqlite-devel
sudo apt install -y libpq-dev 
sudo dnf install -y postgresql-devel
sudo dnf install -y lua-devel


```
## 3) Install SpanDSP 3 (required)
Fedora 42’s repo ships spandsp 0.0.6 — too old. Build SpanDSP 3 from source:
```bash
cd /usr/local/src
sudo git clone https://github.com/freeswitch/spandsp
cd spandsp
sudo ./bootstrap.sh
sudo ./configure --libdir=/usr/local/lib64
sudo make -j"$(nproc)"
sudo make install

# Make runtime linker see /usr/local paths
printf '/usr/local/lib64\n/usr/local/lib\n' | sudo tee /etc/ld.so.conf.d/local-spandsp.conf >/dev/null
sudo ldconfig

# pkg-config search path (add to ~/.bashrc to persist)
export PKG_CONFIG_PATH=/usr/local/lib64/pkgconfig:/usr/local/lib/pkgconfig:$PKG_CONFIG_PATH

# Sanity check
pkg-config --modversion spandsp     # expect 3.0.0 (or 3.0.0+git)
```
## 4) Get FreeSWITCH Source
```bash
cd /usr/local/src
sudo git clone https://github.com/signalwire/freeswitch.git
cd freeswitch
git config pull.rebase true
```
## 5) Configure, Build, Install
```bash
# Autotools
sudo ./bootstrap.sh -j

# Configure (enable common DB backends)
sudo ./configure 

# Build and install
sudo make -j"$(nproc)"
sudo make install

# Install English prompts & MOH (adjust quality tiers as you like)
sudo make cd-sounds-install cd-moh-install
```



