Title: Idempotent installer script
Date:Fri 27 Jan 2023 13:07:33 UTC

Tags: programming, bash
Category: Programming


# Introduction

I tend to expiremnt with different ways to bootstrap in agnostic manner

The following code  is another attempt in managing dependency across platform

In  terms of usage  it can be  plugged  in  the beginning

# Variables

| Variable | Description|
-----------|------------
|SYS_PACKAGER_CMD|executable of the packager name|
|SYS_PACKAGER_CMD_UPDATE_KEYWORD|update command of the packager (can accept none)|
|SYS_PACKAGER_CMD_INSTALL_KEYWORD|install command of the packager (must have a value)|
|SYS_DEPS|Total dependencies (can accept none)|
|SYS_DEPS_SHARED|Dependecies with universal names across distro|
|SYS_DEPS_DEBIAN|Dependecies with names unique to Debain|
|SYS_DEPS_UBUNTU|Dependecies with names unique to Ubuntu|
|SYS_DEPS_ALPINE|Dependecies with names unique to Alpine|
|SYS_DEPS_SUSE_BASE|Dependecies with names unique to SuSe (shared between Leap and Tumbleweed)|
|SYS_DEPS_SUSE_LEAP|Dependecies with names unique to SuSe Leap (added on top of SYS_DEPS_SUSE_BASE)|
|SYS_DEPS_SUSE_TUMBLEWEED|Dependecies with names unique to SuSe Tumbleweed (added on top of SYS_DEPS_SUSE_BASE)|
|SYS_DEPS_RHEL_BASE|Dependecies with names unique to RHEL based distro(shared between fedora, RHEL, rocky, alma)|
|SYS_DEPS_RHEL_FEDORA|Dependecies with names unique to fedora (added on top of SYS_DEPS_RHEL_BASE)|
|SYS_DEPS_RHEL_RHEL|Dependecies with names unique to RHEL (added on top of SYS_DEPS_RHEL_BASE)|
|SYS_DEPS_RHEL_ROCKY|Dependecies with names unique to Rocky (added on top of SYS_DEPS_RHEL_BASE)|



# Code
```
#!/bin/sh
set -eu

SYS_TYPE="$(grep -E '^ID=' /etc/os-release|awk -F= '{print $2}'|tr -d '"')"
SYS_PACKAGER_CMD="none"
SYS_PACKAGER_CMD_UPDATE_KEYWORD="none"
SYS_PACKAGER_CMD_INSTALL_KEYWORD="none"
SYS_DEPS="none"
SYS_DEPS_SHARED=""

SYS_DEPS_DEBIAN=""
SYS_DEPS_UBUNTU=""

SYS_DEPS_ALPINE=""

SYS_DEPS_SUSE_BASE=""
SYS_DEPS_SUSE_LEAP=""
SYS_DEPS_SUSE_TUMBLEWEED=""


SYS_DEPS_RHEL_BASE=""
SYS_DEPS_RHEL_FEDORA=""
SYS_DEPS_RHEL_RHEL=""
SYS_DEPS_RHEL_ROCKY=""

if [ "${SYS_TYPE}" = "none" ]
then
  echo "System wasn't detected"
  exit 1
fi

SYS_DEPS="${SYS_DEPS_SHARED}"
case "${SYS_TYPE}" in
  "alpine")
    SYS_PACKAGER_CMD="apk"
    SYS_DEPS="${SYS_DEPS} ${SYS_DEPS_ALPINE}"
    ;;
  "debian")
    SYS_PACKAGER_CMD="apt"
    SYS_DEPS="${SYS_DEPS} ${SYS_DEPS_DEBIAN}"
    ;;
  
  "ubuntu")
    SYS_PACKAGER_CMD="apt"
    SYS_DEPS="${SYS_DEPS} ${SYS_DEPS_UBUNTU}"
    ;;
  "opensuse-tumbleweed")
    SYS_PACKAGER_CMD="zypper"
    SYS_DEPS="${SYS_DEPS} ${SYS_DEPS_SUSE_BASE} ${SYS_DEPS_SUSE_TUMBLEWEED}"
    ;;

  "opensuse-leap")
    SYS_PACKAGER_CMD="zypper"
    SYS_DEPS="${SYS_DEPS} ${SYS_DEPS_SUSE_BASE} ${SYS_DEPS_SUSE_LEAP}"
    ;;

  "fedora")
    SYS_PACKAGER_CMD="dnf"
    SYS_DEPS="${SYS_DEPS} ${SYS_DEPS_RHEL_BASE} ${SYS_DEPS_RHEL_FEDORA}"
    ;;

  "rhel")
    SYS_PACKAGER_CMD="dnf"
    SYS_DEPS="${SYS_DEPS} ${SYS_DEPS_RHEL_BASE} ${SYS_DEPS_RHEL_RHEL}"
    ;;

  "rocky")
    SYS_PACKAGER_CMD="dnf"
    SYS_DEPS="${SYS_DEPS} ${SYS_DEPS_RHEL_BASE} ${SYS_DEPS_RHEL_ROCKY}"
    ;;
esac

case "${SYS_PACKAGER_CMD}" in
  "apk")
    SYS_PACKAGER_CMD_INSTALL_KEYWORD="add"
    SYS_PACKAGER_CMD_UPDATE_KEYWORD="none"
    ;;

  "apt")
    SYS_PACKAGER_CMD_INSTALL_KEYWORD="install -y"
    SYS_PACKAGER_CMD_UPDATE_KEYWORD="update"
    ;;

  "zypper")
    SYS_PACKAGER_CMD_INSTALL_KEYWORD="install -y"
    SYS_PACKAGER_CMD_UPDATE_KEYWORD="update -y"
    ;;

  "dnf")
    SYS_PACKAGER_CMD_INSTALL_KEYWORD="install -y"
    SYS_PACKAGER_CMD_UPDATE_KEYWORD="update -y"
    ;;
esac


if [ "${SYS_DEPS}"  != "none" ]
then
  # shellcheck disable=2086
  [ "$SYS_PACKAGER_CMD_UPDATE_KEYWORD" != "none" ] && ${SYS_PACKAGER_CMD} ${SYS_PACKAGER_CMD_UPDATE_KEYWORD}

  # shellcheck disable=2086
  ${SYS_PACKAGER_CMD} ${SYS_PACKAGER_CMD_INSTALL_KEYWORD} ${SYS_DEPS}
fi
```
