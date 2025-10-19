```
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'

log() { printf '[%s] %s\n' "$(date +'%F %T')" "$*"; }
die() { echo "ERROR: $*" >&2; exit 1; }
trap 'rc=$?; echo "Script failed (rc=$rc) at line $LINENO: $BASH_COMMAND" >&2' ERR

# 0) Root check
[ "$(id -u)" -eq 0 ] || die "Hãy chạy script với quyền root."

# 1) Xác định thiết bị & FS của /
ROOT_DEV="$(findmnt -no SOURCE /)"      || die "Không xác định được thiết bị mount /"
FSTYPE="$(findmnt -no FSTYPE /)"        || die "Không xác định được FSTYPE của /"
RD_REAL="$(readlink -f "$ROOT_DEV")"
log "ROOT_DEV: $ROOT_DEV (real: $RD_REAL), FS: $FSTYPE"

# 2) Lấy VG/LV từ ROOT_DEV (thử nhiều cách)
VG=""; LV=""; LV_PATH=""
if lvs --help 2>/dev/null | grep -q lv_path; then
  OUT="$(lvs --noheadings -o vg_name,lv_name,lv_path 2>/dev/null | awk -v tgt="$RD_REAL" '
    {
      vg=$1; lv=$2; lp=$3;
      cmd="readlink -f " lp; cmd | getline rp; close(cmd);
      if (rp==tgt) { print vg, lv, lp; exit }
    }')"
  if [ -n "${OUT:-}" ]; then
    VG="$(awk '{print $1}' <<<"$OUT")"
    LV="$(awk '{print $2}' <<<"$OUT")"
    LV_PATH="$(awk '{print $3}' <<<"$OUT")"
  fi
fi
if [ -z "${VG:-}" ] || [ -z "${LV:-}" ]; then
  if [[ "$ROOT_DEV" == /dev/mapper/* ]]; then
    DMNAME="$(dmsetup info -C --noheadings -o name "$ROOT_DEV")"
    VG_ESC="${DMNAME%-*}"; LV_ESC="${DMNAME##*-}"
    VG="${VG_ESC//--/-}";  LV="${LV_ESC//--/-}"
    LV_PATH="/dev/$VG/$LV"
  else
    VG="$(basename "$(dirname "$ROOT_DEV")")"
    LV="$(basename "$ROOT_DEV")"
    LV_PATH="/dev/$VG/$LV"
  fi
fi
[ -n "${VG:-}" ] && [ -n "${LV:-}" ] || die "Không lấy được VG/LV từ $ROOT_DEV"
[ -e "$LV_PATH" ] || LV_PATH="$ROOT_DEV"
log "VG=$VG  LV=$LV  LV_PATH=$LV_PATH"

# 3) Chọn PV chính trong VG (PV có nhiều extents nhất)
PV="$(pvs --noheadings -o pv_name,pe_count --select "vg_name=$VG" \
      | awk '{print $1, $2}' | sort -k2,2nr | awk 'NR==1{print $1}')"
[ -n "${PV:-}" ] || die "Không tìm thấy PV thuộc VG $VG"
log "PV: $PV"

# 4) Xác định DISK và PARTNO một cách chắc chắn (ưu tiên regex)
DISK=""; PARTNO=""
if [[ "$PV" =~ ^(/dev/.+)p([0-9]+)$ ]]; then
  # Kiểu nvme/mmc: /dev/nvme0n1p2 -> /dev/nvme0n1 + 2
  DISK="${BASH_REMATCH[1]}"; PARTNO="${BASH_REMATCH[2]}"
elif [[ "$PV" =~ ^(/dev/[a-zA-Z]+)([0-9]+)$ ]]; then
  # Kiểu sdX: /dev/sda2 -> /dev/sda + 2
  DISK="${BASH_REMATCH[1]}"; PARTNO="${BASH_REMATCH[2]}"
else
  # Fallback thử lsblk (không bắt buộc)
  PKNAME="$(lsblk -no PKNAME "$PV" 2>/dev/null || true)"
  PARTNUM="$(lsblk -no PARTNUM "$PV" 2>/dev/null || true)"
  if [ -n "${PKNAME:-}" ] && [ -n "${PARTNUM:-}" ]; then
    DISK="/dev/${PKNAME}"; PARTNO="$PARTNUM"
  else
    DISK="$PV"; PARTNO=""
  fi
fi

if [ -n "${PARTNO:-}" ]; then
  log "PV là phân vùng: DISK=$DISK  PARTNO=$PARTNO"
else
  log "PV KHÔNG là phân vùng (có thể là nguyên đĩa): DISK=$DISK"
fi
[ -b "$DISK" ] || die "Không xác định được đĩa gốc từ PV ($PV)"

# 5) Resize partition (nếu PV là phân vùng) tới 100% đĩa
if [ -n "${PARTNO:-}" ]; then
  # Thử cài growpart (nếu chưa có) — bỏ qua lỗi cài đặt
  command -v growpart >/dev/null 2>&1 || dnf -y install cloud-utils-growpart >/dev/null 2>&1 || yum -y install cloud-utils-growpart >/dev/null 2>&1 || true

  if command -v growpart >/dev/null 2>&1; then
    log "growpart $DISK $PARTNO ..."
    growpart "$DISK" "$PARTNO" || log "growpart: không thay đổi (có thể đã tối đa)."
  else
    log "parted resizepart ${PARTNO} 100% ..."
    parted -s "$DISK" resizepart "$PARTNO" 100% || log "parted: không thay đổi (có thể đã tối đa)."
  fi
  log "Cập nhật kernel partition table..."
  partprobe "$DISK" || true
  partx -u "$DISK" || true
  udevadm settle || true
else
  log "Bỏ qua bước resize partition (PV không phải phân vùng)."
fi

# 6) pvresize để VG nhận phần trống
log "pvresize $PV ..."
pvresize "$PV" || die "pvresize thất bại"

# Kiểm tra free trước khi extend LV
VG_FREE_B="$(vgs --noheadings -o vg_free --units b --nosuffix "$VG" | awk '{$1=$1;print}')"
VG_FREE_B="${VG_FREE_B:-0}"
log "VG free sau pvresize: ${VG_FREE_B} bytes"
if [ "${VG_FREE_B:-0}" -le 0 ]; then
  log "VG chưa có free (có thể phân vùng chưa mở rộng được). Kết thúc không thay đổi LV."
  lsblk -o NAME,SIZE,FSTYPE,TYPE,MOUNTPOINT
  vgs; lvs -o+devices; df -hT /
  exit 0
fi

# 7) lvextend + filesystem (online)
log "lvextend -l +100%FREE -r $LV_PATH ..."
lvextend -l +100%FREE -r "$LV_PATH" || die "lvextend -r thất bại"

# 8) Đảm bảo grow FS (idempotent)
if [ "$FSTYPE" = "xfs" ]; then
  log "xfs_growfs / (đảm bảo)..."
  xfs_growfs / || true
elif [ "$FSTYPE" = "ext4" ] || [ "$FSTYPE" = "ext3" ]; then
  log "resize2fs $LV_PATH (đảm bảo)..."
  resize2fs "$LV_PATH" || true
fi

# 9) Báo cáo kết quả
log "Hoàn tất."
lsblk -o NAME,SIZE,FSTYPE,TYPE,MOUNTPOINT
df -hT /
vgs
lvs -o+devices

```  
