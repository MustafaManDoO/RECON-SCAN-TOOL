# IN USE DOWNLOAD THIS TOOLS AND ---.sh AND chmod +x ---,sh AND ./RECON.sh [FILE]
# ════════════════════════════════════════════════════════════════
#!/bin/bash

# ════════════════════════════════════════════════════════════════
# recon_scan.sh
#
# Usage:
#   ./recon_scan.sh targets.txt
#   ./recon_scan.sh targets.txt -t 50 -T 5 -o results
# ════════════════════════════════════════════════════════════════

# ───────────────────────── COLORS ─────────────────────────
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
CYAN='\033[0;36m'
BLUE='\033[1;34m'
BOLD='\033[1m'
NC='\033[0m'

# ───────────────────────── CONFIG ─────────────────────────
FILE="${1:-subdomins.txt}"
THREADS=20
TIMEOUT=5
OUTPUT_DIR="results_$(date +%Y%m%d_%H%M%S)"

shift 2>/dev/null

while [[ $# -gt 0 ]]; do
    case "$1" in
        -t|--threads)
            THREADS="$2"
            shift 2
            ;;
        -T|--timeout)
            TIMEOUT="$2"
            shift 2
            ;;
        -o|--output)
            OUTPUT_DIR="$2"
            shift 2
            ;;
        -h|--help)
            echo "Usage:"
            echo "./recon_scan.sh targets.txt -t 50 -T 5 -o results"
            exit 0
            ;;
        *)
            shift
            ;;
    esac
done

# ───────────────────────── CHECK FILE ─────────────────────────
if [[ ! -f "$FILE" ]]; then
    echo -e "${RED}[✘] File not found:${NC} $FILE"
    exit 1
fi

# ───────────────────────── OUTPUT ─────────────────────────
mkdir -p "$OUTPUT_DIR"

ALIVE_FILE="$OUTPUT_DIR/alive.txt"
DEAD_FILE="$OUTPUT_DIR/dead.txt"
FULL_LOG="$OUTPUT_DIR/full_log.txt"

touch "$ALIVE_FILE" "$DEAD_FILE" "$FULL_LOG"

# ───────────────────────── BANNER ─────────────────────────
clear

echo -e "${BLUE}${BOLD}"
echo " ██████╗ ███████╗ ██████╗ ██████╗ ███╗   ██╗"
echo " ██╔══██╗██╔════╝██╔════╝██╔═══██╗████╗  ██║"
echo " ██████╔╝█████╗  ██║     ██║   ██║██╔██╗ ██║"
echo " ██╔══██╗██╔══╝  ██║     ██║   ██║██║╚██╗██║"
echo " ██║  ██║███████╗╚██████╗╚██████╔╝██║ ╚████║"
echo " ╚═╝  ╚═╝╚══════╝ ╚═════╝ ╚═════╝ ╚═╝  ╚═══╝"
echo -e "$ CEARTE BY >> MANO <<{NC}"

echo -e "${CYAN}Targets File:${NC} $FILE"
echo -e "${CYAN}Threads:${NC} $THREADS"
echo -e "${CYAN}Timeout:${NC} ${TIMEOUT}s"
echo -e "${CYAN}Output:${NC} $OUTPUT_DIR"
echo ""

# ───────────────────────── LOAD TARGETS ─────────────────────────
mapfile -t TARGETS < <(
    grep -vE '^\s*$|^\s*#' "$FILE" \
    | awk '{print $1}' \
    | sort -u
)

TOTAL=${#TARGETS[@]}

if [[ $TOTAL -eq 0 ]]; then
    echo -e "${RED}[✘] No targets found.${NC}"
    exit 1
fi

echo -e "${YELLOW}[*] Loaded $TOTAL targets${NC}"
echo ""

# ───────────────────────── SCAN FUNCTION ─────────────────────────
scan_target() {

    HOST=$(echo "$1" | awk '{print $1}')

    STATUS="DOWN"
    IP="-"
    TITLE="-"
    HTTP_CODE="-"

    # ───── DNS Resolve ─────
    if command -v dig &>/dev/null; then
        IP=$(dig +short "$HOST" A | head -n 1)
    fi

    [[ -z "$IP" ]] && IP="-"

    # ───── HTTPS CHECK ─────
    HTTP_RESULT=$(curl -skL \
        --connect-timeout "$TIMEOUT" \
        --max-time "$TIMEOUT" \
        -o /dev/null \
        -w "%{http_code}" \
        "https://$HOST" 2>/dev/null)

    if [[ "$HTTP_RESULT" =~ ^[12345][0-9][0-9]$ ]]; then
        STATUS="UP"
        HTTP_CODE="$HTTP_RESULT"
    else

        # ───── HTTP CHECK ─────
        HTTP_RESULT=$(curl -skL \
            --connect-timeout "$TIMEOUT" \
            --max-time "$TIMEOUT" \
            -o /dev/null \
            -w "%{http_code}" \
            "http://$HOST" 2>/dev/null)

        if [[ "$HTTP_RESULT" =~ ^[12345][0-9][0-9]$ ]]; then
            STATUS="UP"
            HTTP_CODE="$HTTP_RESULT"
        fi
    fi

    # ───── TITLE EXTRACT ─────
    if [[ "$STATUS" == "UP" ]]; then

        TITLE=$(curl -skL \
            --connect-timeout "$TIMEOUT" \
            --max-time "$TIMEOUT" \
            "https://$HOST" 2>/dev/null \
            | grep -iPo '(?<=<title>)(.*?)(?=</title>)' \
            | head -n 1)

        [[ -z "$TITLE" ]] && TITLE="-"

        echo "$HOST | $IP | $HTTP_CODE | $TITLE" >> "$ALIVE_FILE"

        echo -e "${GREEN}[UP]${NC} $HOST ${CYAN}[$IP]${NC} ${YELLOW}[$HTTP_CODE]${NC} ${BOLD}$TITLE${NC}"

    else

        echo "$HOST" >> "$DEAD_FILE"

        echo -e "${RED}[DOWN]${NC} $HOST"
    fi

    echo "[$STATUS] $HOST | IP:$IP | CODE:$HTTP_CODE | TITLE:$TITLE" >> "$FULL_LOG"
}

export -f scan_target
export TIMEOUT ALIVE_FILE DEAD_FILE FULL_LOG
export RED GREEN YELLOW CYAN BLUE BOLD NC

# ───────────────────────── START SCAN ─────────────────────────
printf '%s\n' "${TARGETS[@]}" \
| xargs -P "$THREADS" -I{} bash -c 'scan_target "$@"' _ {}

# ───────────────────────── SUMMARY ─────────────────────────
ALIVE_COUNT=$(wc -l < "$ALIVE_FILE")
DEAD_COUNT=$(wc -l < "$DEAD_FILE")

echo ""
echo -e "${YELLOW}════════════════════════════════════════════${NC}"
echo -e "${GREEN}${BOLD}UP:${NC} $ALIVE_COUNT"
echo -e "${RED}${BOLD}DOWN:${NC} $DEAD_COUNT"
echo -e "${CYAN}${BOLD}Results:${NC} $OUTPUT_DIR"
echo -e "${YELLOW}════════════════════════════════════════════${NC}"
echo ""
