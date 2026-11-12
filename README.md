# LINUX-ANDSHELLSCRIPTING
#!/bin/bash
PKG_LIST='packages.txt'
LOG_FILE='logs/install_update.log'
REPORT='reports/software_report.txt'
mkdir -p logs reports
installed=0; skipped=0; failed=0

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') | $1" | tee -a "$LOG_FILE"
}

if [ "$EUID" -ne 0 ]; then
    echo 'Please run as root (use sudo)'
    exit 1
fi

install_package() {
    if dpkg -s "$1" &>/dev/null; then
        log "SKIPPED   | $1 is already installed"
        skipped=$((skipped+1))
    elif apt-get install -y "$1" &>>"$LOG_FILE"; then
        log "INSTALLED | $1"
        installed=$((installed+1))
    else
        log "FAILED    | $1"
        failed=$((failed+1))
    fi
}

log 'Software management cycle started'
while read -r pkg; do
    [ -n "$pkg" ] && install_package "$pkg"
done < "$PKG_LIST"

apt-get update &>>"$LOG_FILE"
upgradable=$(apt list --upgradable 2>/dev/null | grep -c upgradable)
apt-get upgrade -y &>>"$LOG_FILE"
log "UPDATED   | upgrade completed ($upgradable packages were upgradable)"

apt-get autoremove -y &>>"$LOG_FILE"
apt-get autoclean &>>"$LOG_FILE"
log 'CLEANED   | unused packages removed'

{
    echo 'SOFTWARE MANAGEMENT REPORT'
    echo "Generated: $(date)"
    echo "Installed: $installed"
    echo "Skipped:   $skipped"
    echo "Failed:    $failed"
    echo "Upgradable packages found: $upgradable"
} > "$REPORT"
log 'Cycle completed successfully'
