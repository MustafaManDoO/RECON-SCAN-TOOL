# RECON-SCAN-TOOL
# THIS TOOL RECON SCAN > IPS AND SUBDOMINS <<
#!/bin/bash

# ════════════════════════════════════════════════════════════════
#  recon_scan.sh
#  RUN: ./recon_scan.sh [FILE] [DATE]
#
#  RUN ./recon_scan.sh subdomins.txt
#         ./recon_scan.sh targets.txt -t 3 -o results
# ════════════════════════════════════════════════════════════════

# ────────────────────────────────────────────────────────
RED='\033[0;31m';  GREEN='\033[0;32m'; YELLOW='\033[1;33m'
CYAN='\033[0;36m'; BLUE='\033[1;34m';  BOLD='\033[1m'; NC='\033[0m'

# ───────────────────────────────────────────
FILE="${1:-subdomins.txt}"
THREADS=10           #----------
TIMEOUT=2            #----------
OUTPUT_DIR="results_$(date +%Y%m%d_%H%M%S)"
PING_COUNT=2

# ────────────────────────────────────────────────
shift
while [[ $# -gt 0 ]]; do
  case "$1" in
    -t|--threads) THREADS="$2"; shift 2 ;;
    -T|--timeout) TIMEOUT="$2"; shift 2 ;;
    -o|--output)  OUTPUT_DIR="$2"; shift 2 ;;
    -p|--ping)    PING_COUNT="$2"; shift 2 ;;
    -h|--help)
      echo -e "${BOLD}USING:${NC} $0 [FILE] [-t threads] [-T timeout] [-o output_dir] [-p ping_count]"
      exit 0 ;;
    *) shift ;;
  esac
done

# ────────────────────────────────────────────────
if [[ ! -f "$FILE" ]]; then
  echo -e "${RED}[✘] NO IS'N FILE HERE: $FILE${NC}"
  exit 1
fi

# ──────────────────────────────────────────────
mkdir -p "$OUTPUT_DIR"
ALIVE_FILE="$OUTPUT_DIR/alive.txt"
DEAD_FILE="$OUTPUT_DIR/dead.txt"
FULL_LOG="$OUTPUT_DIR/full_log.txt"
HTML_REPORT="$OUTPUT_DIR/report.html"

# ─────────────────────────────────────────────────
clear
echo -e "${BLUE}${BOLD}"
echo "  ██████╗ ███████╗ ██████╗ ██████╗ ███╗   ██╗"
echo "  ██╔══██╗██╔════╝██╔════╝██╔═══██╗████╗  ██║"
echo "  ██████╔╝█████╗  ██║     ██║   ██║██╔██╗ ██║"
echo "  ██╔══██╗██╔══╝  ██║     ██║   ██║██║╚██╗██║"
echo "  ██║  ██║███████╗╚██████╗╚██████╔╝██║ ╚████║"
echo "  ╚═╝  ╚═╝╚══════╝ ╚═════╝ ╚═════╝ ╚═╝  ╚═══╝"
echo -e "${NC}"
echo -e "${CYAN}  ╔══════════════════════════════════════════╗"
echo -e "  ║  CEARTE:MANO <<>> CHEKE FILE IPS AND SUBDOMINS  ║"
echo -e "  ╚══════════════════════════════════════════╝${NC}"
echo ""

# ─────────────────────────────────────────
mapfile -t TARGETS < <(grep -vE '^\s*#|^\s*$' "$FILE" | sed 's/[[:space:]]//g')
TOTAL=${#TARGETS[@]}

if [[ $TOTAL -eq 0 ]]; then
  echo -e "${RED}[✘] ADD IN FILE SUBDOMAINS!${NC}"
  exit 1
fi

echo -e "  ${BOLD}FILE:${NC}       $FILE"
echo -e "  ${BOLD}GLOALS:${NC}     $TOTAL"
echo -e "  ${BOLD}Threads:${NC}     $THREADS"
echo -e "  ${BOLD}Timeout:${NC}     ${TIMEOUT}s"
echo -e "  ${BOLD}RESULT:${NC}  $OUTPUT_DIR/"
echo ""
echo -e "  ${YELLOW}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
echo ""

# ── Counters──────────────────────────────────────────────
ALIVE=0; DEAD=0; DONE=0
LOCK_FILE="/tmp/recon_lock_$$"

# ──────────────────────────────────────────────
scan_target() {
  local HOST="$1"
  local STATUS="DOWN"
  local LATENCY="—"
  local IP="—"
  local EXTRA=""

  # --- Ping ---
  PING_RESULT=$(ping -c "$PING_COUNT" -W "$TIMEOUT" "$HOST" 2>&1)
  if echo "$PING_RESULT" | grep -q "bytes from"; then
    STATUS="UP"
    LATENCY=$(echo "$PING_RESULT" | grep -oP 'avg.*?(\d+\.\d+)' | grep -oP '\d+\.\d+' | head -1)
    [[ -z "$LATENCY" ]] && LATENCY=$(echo "$PING_RESULT" | grep -oP 'time=\K[\d.]+' | head -1)
    LATENCY="${LATENCY:-<1}ms"
  fi

  # --- DNS Resolve ---
  if command -v dig &>/dev/null; then
    IP=$(dig +short "$HOST" 2>/dev/null | grep -oP '\d+\.\d+\.\d+\.\d+' | head -1)
  elif command -v nslookup &>/dev/null; then
    IP=$(nslookup "$HOST" 2>/dev/null | grep -oP 'Address: \K\S+' | tail -1)
  fi
  [[ -z "$IP" ]] && IP="$HOST"

  # --- HTTP Check (UP) ---
  if [[ "$STATUS" == "UP" ]]; then
    if command -v curl &>/dev/null; then
      HTTP_CODE=$(curl -sk --max-time "$TIMEOUT" -o /dev/null -w "%{http_code}" "http://$HOST" 2>/dev/null)
      HTTPS_CODE=$(curl -sk --max-time "$TIMEOUT" -o /dev/null -w "%{http_code}" "https://$HOST" 2>/dev/null)
      [[ "$HTTP_CODE" =~ ^[2345] ]]  && EXTRA+=" HTTP:${HTTP_CODE}"
      [[ "$HTTPS_CODE" =~ ^[2345] ]] && EXTRA+=" HTTPS:${HTTPS_CODE}"
    fi
  fi

  # ------
  (
    flock 200
    DONE=$((DONE + 1))
    PCT=$(( DONE * 100 / TOTAL ))
    BAR=$(printf '█%.0s' $(seq 1 $((PCT / 5))))
    SPACE=$(printf '░%.0s' $(seq 1 $((20 - PCT / 5))))

    if [[ "$STATUS" == "UP" ]]; then
      ALIVE=$((ALIVE + 1))
      echo -e "  ${GREEN}[✔ UP]${NC}   ${BOLD}$HOST${NC}  ${CYAN}($IP)${NC}  ${YELLOW}$LATENCY${NC}${EXTRA}"
      echo "$HOST $IP $LATENCY $EXTRA" >> "$ALIVE_FILE"
    else
      DEAD=$((DEAD + 1))
      echo -e "  ${RED}[✘ DOWN]${NC} $HOST"
      echo "$HOST" >> "$DEAD_FILE"
    fi
    echo "[$STATUS] $HOST | IP:$IP | Latency:$LATENCY |$EXTRA" >> "$FULL_LOG"
    echo -ne "  ${CYAN}Progress: [${BAR}${SPACE}] ${PCT}% (${DONE}/${TOTAL})${NC}\r"
  ) 200>"$LOCK_FILE"
}

export -f scan_target
export FILE PING_COUNT TIMEOUT TOTAL DONE ALIVE DEAD
export ALIVE_FILE DEAD_FILE FULL_LOG LOCK_FILE
export RED GREEN YELLOW CYAN BLUE BOLD NC

# ────────────────────────────────────────
echo "" > "$ALIVE_FILE"; echo "" > "$DEAD_FILE"; echo "" > "$FULL_LOG"

printf '%s\n' "${TARGETS[@]}" | xargs -P "$THREADS" -I{} bash -c 'scan_target "$@"' _ {}

rm -f "$LOCK_FILE"

# ───────────────────────────────────────────
ALIVE_COUNT=$(grep -cvE '^\s*$' "$ALIVE_FILE" 2>/dev/null || echo 0)
DEAD_COUNT=$(grep -cvE '^\s*$' "$DEAD_FILE" 2>/dev/null || echo 0)

echo -e "\n"
echo -e "  ${YELLOW}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
echo -e "  ${BOLD}📊 $Summary of results{NC}"
echo -e "  ${YELLOW}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
echo -e "  ${BOLD}إجمالي الأهداف:${NC}  $TOTAL"
echo -e "  ${GREEN}${BOLD}✔ شغالين (UP):${NC}   $ALIVE_COUNT"
echo -e "  ${RED}${BOLD}✘ ميتين (DOWN):${NC}  $DEAD_COUNT"
echo -e "  ${BOLD}نسبة الحياة:${NC}     $(( ALIVE_COUNT * 100 / TOTAL ))%"
echo ""
echo -e "  ${CYAN}📁 FILES:${NC}"
echo -e "     alive.txt   → RESULT ON"
echo -e "     dead.txt    → RESULT DEAD"
echo -e "     full_log.txt → RESULTS"
echo -e "  ${YELLOW}━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━${NC}"
echo ""

# ── HTML Report ─────────────────────────────────────────────
cat > "$HTML_REPORT" << HTMLEOF
<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
<meta charset="UTF-8">
<title>Recon Report</title>
<style>
  body{font-family:monospace;background:#0d1117;color:#c9d1d9;margin:0;padding:2rem}
  h1{color:#58a6ff;border-bottom:1px solid #21262d;padding-bottom:.5rem}
  .stats{display:flex;gap:1rem;margin:1rem 0}
  .card{background:#161b22;border:1px solid #21262d;border-radius:8px;padding:1rem;flex:1;text-align:center}
  .card span{display:block;font-size:2rem;font-weight:bold}
  .up{color:#3fb950}.down{color:#f85149}.total{color:#58a6ff}
  table{width:100%;border-collapse:collapse;margin-top:1rem}
  th{background:#161b22;padding:.5rem;text-align:right;border-bottom:1px solid #21262d;color:#8b949e}
  td{padding:.5rem;border-bottom:1px solid #21262d}
  .badge-up{background:#0d4a23;color:#3fb950;padding:2px 8px;border-radius:4px}
  .badge-down{background:#4a0d0d;color:#f85149;padding:2px 8px;border-radius:4px}
  .ts{color:#8b949e;font-size:.85rem}
</style>
</head>
<body>
<h1>🔍 Recon Scan Report</h1>
<p class="ts">📅 $(date) | 📂 $FILE</p>
<div class="stats">
  <div class="card"><span class="total">$TOTAL</span>إجمالي الأهداف</div>
  <div class="card"><span class="up">$ALIVE_COUNT</span>WORKS ✔</div>
  <div class="card"><span class="down">$DEAD_COUNT</span>DEAD ✘</div>
  <div class="card"><span class="total">$(( ALIVE_COUNT * 100 / TOTAL ))%</span>نسبة الحياة</div>
</div>
<table>
<tr><th>الهدف</th><th>IP</th><th>Latency</th><th>HTTP</th><th>الحالة</th></tr>
HTMLEOF

grep -vE '^\s*$' "$FULL_LOG" | while IFS='|' read -r STATUS_LINE IP_LINE LAT_LINE EXTRA_LINE; do
  HOST=$(echo "$STATUS_LINE" | grep -oP '\] \K\S+')
  STATUS=$(echo "$STATUS_LINE" | grep -oP '^\[\K[^]]+')
  IP=$(echo "$IP_LINE" | grep -oP 'IP:\K\S+')
  LAT=$(echo "$LAT_LINE" | grep -oP 'Latency:\K\S+')
  HTTP=$(echo "$EXTRA_LINE" | grep -oP 'HTTP[S]?:\d+' | tr '\n' ' ')
  [[ "$STATUS" == "UP" ]] && BADGE='<span class="badge-up">UP ✔</span>' || BADGE='<span class="badge-down">DOWN ✘</span>'
  echo "<tr><td>$HOST</td><td>$IP</td><td>$LAT</td><td>$HTTP</td><td>$BADGE</td></tr>" >> "$HTML_REPORT"
done

cat >> "$HTML_REPORT" << HTMLEOF
</table>
</body>
</html>
HTMLEOF

echo -e "  ${GREEN}✔ HTML Report:${NC} $HTML_REPORT"
echo ""
