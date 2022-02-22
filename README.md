# sysupgrade-extras
sysupgrade-extras.sh
Originaly proposed by @vgaetera

REFS:
- https://openwrt.org/license
- https://openwrt.org/docs/guide-user/advanced/sysupgrade_extras

====== Sysupgrade extras ======
{{page>meta:infobox:wip&noheader&nofooter&noeditbtn}}
{{section>meta:infobox:howto_links#cli_skills&noheader&nofooter&noeditbutton}}

===== Introduction =====
  * This instruction extends the functionality of [[docs:techref:sysupgrade|Sysupgrade]] on [[docs:guide-user:installation:openwrt_x86|x86]] target.
  * Follow the [[docs:guide-user:advanced:sysupgrade_extras#automated|automated]] section for quick setup.

===== Features =====
  * Automate upgrading to the latest firmware release.
  * Automate expanding the root partition and filesystem.

===== Options =====
| Option | Description |
| --- | --- |
| ''**-I**'' | Initialize Opkg profiles. |
| ''**-A**'' | Initialize automated upgrade. |

===== Instructions =====
```
# Configure profile
mkdir -p /etc/profile.d
cat << "EOF" > /etc/profile.d/sysupgrade.sh
sysupgrade() {
local SYSUP_CMD="${1}"
case "${SYSUP_CMD}" in
(-I) opkg save
sysupgrade_init ;;
(-A) . /etc/openwrt_release
sysupgrade_auto ;;
(*) command sysupgrade "${@}" ;;
esac
}

sysupgrade_init() {
uci -q batch << EOI
set opkg.init='opkg'
add_list opkg.init.ipkg='fdisk'
add_list opkg.init.ipkg='losetup'
add_list opkg.init.ipkg='resize2fs'
add_list opkg.init.ipkg='f2fs-tools'
commit opkg
EOI
}

sysupgrade_auto() {
local SYSUP_VER="$(sysupgrade_ver)"
local SYSUP_URL="$(sysupgrade_url)"
local SYSUP_DEV="$(sysupgrade_dev)"
local SYSUP_PROF="$(sysupgrade_prof)"
local SYSUP_FS="$(sysupgrade_fs)"
local SYSUP_TYPE="$(sysupgrade_type)"
local SYSUP_IMG="$(sysupgrade_img)"
uclient-fetch -P /tmp "${SYSUP_URL}/${SYSUP_IMG}"
case "${SYSUP_IMG}" in
(*.gz) gunzip -k /tmp/"${SYSUP_IMG}" ;;
esac
sysupgrade -i /tmp/"${SYSUP_IMG%.gz}"
}

sysupgrade_ver() {
case "${DISTRIB_RELEASE}" in
(SNAPSHOT) echo "../snapshots" ;;
(*) uclient-fetch -O - \
"https://api.github.com/repos/openwrt/openwrt/tags" \
| jsonfilter -e "$[*]['name']" \
| sed -e "/-rc/d;s/^v//;q" ;;
esac
}

sysupgrade_url() {
echo "https://downloads.openwrt.org/\
releases/${SYSUP_VER}/targets/${DISTRIB_TARGET}"
}

sysupgrade_dev() {
ubus call system board \
| jsonfilter -e "$['board_name']"
}

sysupgrade_prof() {
case "${DISTRIB_TARGET}" in
(x86*) echo "generic" ;;
(*) echo "${SYSUP_DEV//,/_}" ;;
esac
}

sysupgrade_fs() {
case "${DISTRIB_RELEASE}" in
(SNAPSHOT) ubus call system board \
| jsonfilter -e "$['rootfs_type']" ;;
(*) if grep -q -e "\s/\soverlay\s" /etc/mtab
then echo "squashfs"
else echo "ext4"
fi ;;
esac
}

sysupgrade_type() {
case "${DISTRIB_TARGET}" in
(x86*) echo "combined" ;;
(*) echo "sysupgrade" ;;
esac
}

sysupgrade_img() {
uclient-fetch -O - "${SYSUP_URL}/profiles.json" \
| jsonfilter -e "$['profiles']['${SYSUP_PROF}']['images']\
[@['type']='${SYSUP_TYPE}'&&@['filesystem']='${SYSUP_FS}']\
['name']"
}
EOF

# Configure uci-defaults
cat << "EOF" > /etc/uci-defaults/70-root-resize
if [ ! -e /etc/root-resize ] \
&& lock -n /var/lock/root-resize \
&& [ -e /etc/opkg-restore-init ]
then
BOOT="$(sed -n -e "/\s\/boot\s.*$/{s///p;q}" /etc/mtab)"
DISK="${BOOT%%[0-9]*}"
PART="$((${BOOT##*[^0-9]}+1))"
ROOT="${DISK}${PART}"
OFFS="$(fdisk "${DISK}" -l -o "device,start" \
| sed -n -e "\|^${ROOT}\s*|s///p")"
echo -e "p\nd\n${PART}\nn\np\n${PART}\n${OFFS}\n\nn\np\nw" \
| fdisk "${DISK}"
touch /etc/root-resize
lock -u /var/lock/root-resize
fi
exit 1
EOF
cat << "EOF" > /etc/uci-defaults/80-ext4-resize
if grep -q -e "\s/\sext4\s" /etc/mtab \
&& [ ! -e /etc/ext4-resize ] \
&& lock -n /var/lock/ext4-resize \
&& [ -e /etc/opkg-restore-init ]
then
BOOT="$(sed -n -e "/\s\/boot\s.*$/{s///p;q}" /etc/mtab)"
DISK="${BOOT%%[0-9]*}"
PART="$((${BOOT##*[^0-9]}+1))"
ROOT="${DISK}${PART}"
LOOP="$(losetup -f)"
losetup "${LOOP}" "${ROOT}"
fsck.ext4 -y "${LOOP}"
resize2fs "${LOOP}"
touch /etc/ext4-resize
lock -u /var/lock/ext4-resize
reboot
fi
exit 1
EOF
cat << "EOF" > /etc/uci-defaults/80-f2fs-resize
if grep -q -e "\s/\soverlay\s" /etc/mtab \
&& [ ! -e /etc/f2fs-resize ] \
&& lock -n /var/lock/f2fs-resize \
&& [ -e /etc/opkg-restore-init ]
then
LOOP="$(losetup -n -O "NAME" \
| sed -n -e "1p")"
ROOT="$(losetup -n -O "BACK-FILE" "${LOOP}" \
| sed -e "s|^|/dev|")"
OFFS="$(losetup -n -O "OFFSET" "${LOOP}")"
LOOP="$(losetup -f)"
losetup -o "${OFFS}" "${LOOP}" "${ROOT}"
fsck.f2fs -f "${LOOP}"
mount "${LOOP}" /mnt
umount "${LOOP}"
resize.f2fs "${LOOP}"
touch /etc/f2fs-resize
lock -u /var/lock/f2fs-resize
reboot
fi
exit 1
EOF
cat << "EOF" >> /etc/sysupgrade.conf
/etc/uci-defaults
EOF
```

===== Examples =====
```
# Automated Sysupgrade
sysupgrade -I
sysupgrade -A
```

===== Automated =====
```
uclient-fetch -O sysupgrade-extras.sh "https://openwrt.org/_export/code/docs/guide-user/advanced/sysupgrade_extras?codeblock=0"
. ./sysupgrade-extras.sh
```
