#!/bin/bash

LOG_FILE="/var/log/wp-delete-unused-themes.log"
LOCK_FILE="${LOG_FILE}.lock"
RESULT_DIR="/tmp/wp-delete-themes-$$"
RAM_PER_JOB_MB=200
WP_TIMEOUT=60

# Theme ที่ต้องการลบ
DELETE_THEMES=("twentytwentythree" "twentytwentyfour")

# Theme ที่ต้องเก็บไว้ (ห้ามลบ)
KEEP_THEMES=("twentytwentyfive" "blocksy" "blocksy-child")

log() {
    local DATE=$(date '+%Y-%m-%d %H:%M:%S')
    echo "$1"
    ( flock 200; echo "[$DATE] $1" >> "$LOG_FILE" ) 200>"$LOCK_FILE"
}

cleanup() {
    wait
    rm -rf "$RESULT_DIR"
    rm -f "$LOCK_FILE"
}
trap cleanup EXIT

mkdir -p "$RESULT_DIR"
mkdir -p "$RESULT_DIR/check"
mkdir -p "$RESULT_DIR/delete"

START_TIME=$(date +%s)

# ====================================
# เช็ค WP-CLI
# ====================================
if ! command -v wp &>/dev/null; then
    log "❌ ERROR: ไม่พบ WP-CLI"
    exit 1
fi

# ====================================
# คำนวณ MAX_JOBS อัตโนมัติ
# ====================================
CPU_CORES=$(nproc)
TOTAL_RAM_MB=$(free -m | awk '/^Mem:/{print $2}')
MAX_JOBS_BY_RAM=$(( TOTAL_RAM_MB / RAM_PER_JOB_MB ))

if [ "$CPU_CORES" -lt "$MAX_JOBS_BY_RAM" ]; then
    MAX_JOBS=$CPU_CORES
else
    MAX_JOBS=$MAX_JOBS_BY_RAM
fi

[ "$MAX_JOBS" -lt 1 ] && MAX_JOBS=1
[ "$MAX_JOBS" -gt 20 ] && MAX_JOBS=20

log "======================================"
log " DELETE UNUSED THEMES"
log " เริ่มเวลา      : $(date '+%Y-%m-%d %H:%M:%S')"
log " CPU Cores     : $CPU_CORES Core"
log " Total RAM     : $TOTAL_RAM_MB MB"
log " Auto MAX_JOBS : $MAX_JOBS"
log " ลบ Theme      : ${DELETE_THEMES[*]}"
log " เก็บ Theme     : ${KEEP_THEMES[*]}"
log "======================================"

# ====================================
# หา WordPress ทั้งหมด
# ====================================
DIRS=()
for dir in /home/*/public_html/*/; do
    if [ -f "${dir}wp-config.php" ]; then
        DIRS+=("$dir")
    fi
done

TOTAL=${#DIRS[@]}
log "พบ WordPress ทั้งหมด: $TOTAL เว็บ"
log "======================================"

# ====================================
# PHASE 1: เช็คก่อนว่าเว็บไหนมี Theme ที่ต้องลบ
# ====================================
log " PHASE 1: กำลังตรวจสอบ..."
log "======================================"

check_site() {
    local dir="$1"
    local LOG_FILE="$2"
    local LOCK_FILE="$3"
    local RESULT_DIR="$4"
    local SITE=$(echo "$dir" | awk -F'/' '{print $5"/"$7}')
    local UNIQUE="${BASHPID}_$(date +%s%N)"

    _log() {
        local DATE=$(date '+%Y-%m-%d %H:%M:%S')
        echo "$1"
        ( flock 200; echo "[$DATE] $1" >> "$LOG_FILE" ) 200>"$LOCK_FILE"
    }

    if [ ! -d "$RESULT_DIR" ]; then return; fi

    local THEMES_DIR="${dir}wp-content/themes"
    local FOUND=0

    for theme in "twentytwentythree" "twentytwentyfour"; do
        if [ -d "${THEMES_DIR}/${theme}" ]; then
            FOUND=1
            break
        fi
    done

    if [ "$FOUND" -eq 1 ]; then
        _log "⚠️  NEEDS DELETE: $SITE"
        ( flock 200; echo "$dir" >> "$RESULT_DIR/needs_delete.txt" ) 200>"$LOCK_FILE"
        touch "${RESULT_DIR}/check/needsdelete_${UNIQUE}" 2>/dev/null
    else
        _log "✅ OK (ไม่มี Theme ที่ต้องลบ): $SITE"
        touch "${RESULT_DIR}/check/ok_${UNIQUE}" 2>/dev/null
    fi
}

export -f check_site

declare -a PIDS=()
for dir in "${DIRS[@]}"; do
    check_site "$dir" "$LOG_FILE" "$LOCK_FILE" "$RESULT_DIR" &
    PIDS+=($!)
    if [ "${#PIDS[@]}" -ge "$MAX_JOBS" ]; then
        wait "${PIDS[0]}"
        PIDS=("${PIDS[@]:1}")
    fi
done
for pid in "${PIDS[@]}"; do wait "$pid"; done

OK=$(find "$RESULT_DIR/check" -name "ok_*" 2>/dev/null | wc -l)
NEEDSDELETE=$(find "$RESULT_DIR/check" -name "needsdelete_*" 2>/dev/null | wc -l)

log "======================================"
log " สรุปผล PHASE 1"
log " รวมทั้งหมด            : $TOTAL เว็บ"
log " ✅ ไม่มี Theme ที่ต้องลบ : $OK เว็บ"
log " ⚠️  ต้องลบ Theme        : $NEEDSDELETE เว็บ"
log "======================================"

if [ "$NEEDSDELETE" -eq 0 ]; then
    log "✅ ทุกเว็บไม่มี Theme ที่ต้องลบแล้ว"
    END_TIME=$(date +%s)
    ELAPSED=$(( END_TIME - START_TIME ))
    log " เวลาที่ใช้ : $(( ELAPSED / 60 )) นาที $(( ELAPSED % 60 )) วินาที"
    log "======================================"
    exit 0
fi

# ====================================
# PHASE 2: ลบ Theme เฉพาะเว็บที่มี
# ====================================
log " PHASE 2: กำลังลบ Theme จาก $NEEDSDELETE เว็บ..."
log "======================================"

delete_themes_site() {
    local dir="$1"
    local LOG_FILE="$2"
    local LOCK_FILE="$3"
    local RESULT_DIR="$4"
    local WP_TIMEOUT="$5"
    local SITE=$(echo "$dir" | awk -F'/' '{print $5"/"$7}')
    local UNIQUE="${BASHPID}_$(date +%s%N)"

    _log() {
        local DATE=$(date '+%Y-%m-%d %H:%M:%S')
        echo "$1"
        ( flock 200; echo "[$DATE] $1" >> "$LOG_FILE" ) 200>"$LOCK_FILE"
    }

    _wp() {
        timeout "$WP_TIMEOUT" wp --path="$dir" "$@" --allow-root 2>/dev/null
    }

    if [ ! -d "$RESULT_DIR" ]; then return; fi

    local THEMES_DIR="${dir}wp-content/themes"
    local FAILED=0

    # ตรวจสอบ Active Theme ก่อนลบ
    local ACTIVE_THEME
    ACTIVE_THEME=$(_wp theme list --status=active --field=name 2>/dev/null)

    for theme in "twentytwentythree" "twentytwentyfour"; do
        if [ -d "${THEMES_DIR}/${theme}" ]; then

            # ⚠️ ถ้า Theme นั้นกำลัง Active อยู่ → ข้ามไม่ลบ
            if [ "$ACTIVE_THEME" = "$theme" ]; then
                _log "⚠️  SKIPPED (Active Theme): $SITE → $theme"
                ( flock 200; echo "$dir ($theme)" >> "$RESULT_DIR/active_theme_sites.txt" ) 200>"$LOCK_FILE"
                touch "${RESULT_DIR}/delete/skipped_${UNIQUE}_${theme}" 2>/dev/null
                continue
            fi

            # ลบ Theme
            if _wp theme delete "$theme"; then
                _log "✅ DELETED: $SITE → $theme"
            else
                _log "❌ FAILED DELETE: $SITE → $theme"
                FAILED=1
            fi
        fi
    done

    if [ "$FAILED" -eq 1 ]; then
        touch "${RESULT_DIR}/delete/failed_${UNIQUE}" 2>/dev/null
    else
        touch "${RESULT_DIR}/delete/success_${UNIQUE}" 2>/dev/null
    fi
}

export -f delete_themes_site

declare -a PIDS=()
while IFS= read -r dir; do
    delete_themes_site "$dir" "$LOG_FILE" "$LOCK_FILE" "$RESULT_DIR" "$WP_TIMEOUT" &
    PIDS+=($!)
    if [ "${#PIDS[@]}" -ge "$MAX_JOBS" ]; then
        wait "${PIDS[0]}"
        PIDS=("${PIDS[@]:1}")
    fi
done < "$RESULT_DIR/needs_delete.txt"
for pid in "${PIDS[@]}"; do wait "$pid"; done

# ====================================
# คำนวณเวลารวม
# ====================================
END_TIME=$(date +%s)
ELAPSED=$(( END_TIME - START_TIME ))

SUCCESS=$(find "$RESULT_DIR/delete" -name "success_*" 2>/dev/null | wc -l)
FAILED=$(find "$RESULT_DIR/delete" -name "failed_*" 2>/dev/null | wc -l)
SKIPPED=$(find "$RESULT_DIR/delete" -name "skipped_*" 2>/dev/null | wc -l)

log "======================================"
log " สรุปผลรวม"
log " รวมทั้งหมด               : $TOTAL เว็บ"
log " ✅ ไม่มี Theme ที่ต้องลบ   : $OK เว็บ"
log " ✅ ลบสำเร็จ               : $SUCCESS เว็บ"
log " ❌ ลบไม่สำเร็จ            : $FAILED เว็บ"
log " ⚠️  ข้าม (Active Theme)   : $SKIPPED เว็บ"
log " เวลาที่ใช้                : $(( ELAPSED / 60 )) นาที $(( ELAPSED % 60 )) วินาที"
log " Log อยู่ที่               : $LOG_FILE"
log "======================================"

# แจ้งเว็บที่ข้ามเพราะ Theme กำลัง Active
if [ -f "$RESULT_DIR/active_theme_sites.txt" ]; then
    log "======================================"
    log " ⚠️  เว็บที่ข้าม (Theme กำลังใช้งานอยู่):"
    while IFS= read -r line; do
        log " ⚠️  $line"
    done < "$RESULT_DIR/active_theme_sites.txt"
    log " ⚠️  ต้องเปลี่ยน Active Theme ก่อนแล้วค่อยลบ"
    log "======================================"
fi
```

---

## Step by Step

---

## STEP 1: สร้างไฟล์บน GitHub
```
https://github.com/ufavision/server-scripts
