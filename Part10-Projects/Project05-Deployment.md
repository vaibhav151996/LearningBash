# Project 5: Deployment Script

## Project Overview

Build a zero-downtime deployment script for web applications with rollback capability, health checks, and notifications.

---

## Requirements

- Deploy application from Git repository or archive
- Zero-downtime deployment using symlink switching
- Automatic rollback on health check failure
- Keep N previous releases for quick rollback
- Pre/post deploy hooks
- Multi-server support

---

## Complete Implementation

```bash
#!/usr/bin/env bash
#
# deploy.sh — Zero-Downtime Deployment Tool
#
# Usage: deploy.sh [-c config] [-e environment] [-r] [-R release_id]

set -euo pipefail

readonly SCRIPT_NAME="$(basename "$0")"
readonly TIMESTAMP="$(date +%Y%m%d_%H%M%S)"

# --- Defaults ---
APP_NAME="myapp"
DEPLOY_BASE="/opt/apps"
REPO_URL=""
BRANCH="main"
KEEP_RELEASES=5
HEALTH_CHECK_URL=""
HEALTH_CHECK_RETRIES=10
HEALTH_CHECK_DELAY=3
RESTART_COMMAND=""
PRE_DEPLOY_HOOK=""
POST_DEPLOY_HOOK=""
SERVERS=("localhost")

# --- Derived Paths ---
DEPLOY_DIR=""
RELEASES_DIR=""
SHARED_DIR=""
CURRENT_LINK=""

setup_paths() {
    DEPLOY_DIR="${DEPLOY_BASE}/${APP_NAME}"
    RELEASES_DIR="${DEPLOY_DIR}/releases"
    SHARED_DIR="${DEPLOY_DIR}/shared"
    CURRENT_LINK="${DEPLOY_DIR}/current"
}

# --- Logging ---
log()  { printf '[%s] \033[0;34m%s\033[0m\n' "$(date '+%H:%M:%S')" "$*"; }
ok()   { printf '[%s] \033[0;32m✓ %s\033[0m\n' "$(date '+%H:%M:%S')" "$*"; }
warn() { printf '[%s] \033[0;33m⚠ %s\033[0m\n' "$(date '+%H:%M:%S')" "$*" >&2; }
fail() { printf '[%s] \033[0;31m✗ %s\033[0m\n' "$(date '+%H:%M:%S')" "$*" >&2; }
die()  { fail "$@"; exit 1; }

# --- Step Display ---
step() {
    local num="$1" total="$2" msg="$3"
    printf '\n[%s] \033[1m[%d/%d] %s\033[0m\n' "$(date '+%H:%M:%S')" "$num" "$total" "$msg"
}

# --- Config ---
load_config() {
    local config="$1"
    [[ -f "$config" ]] || die "Config not found: $config"
    # shellcheck source=/dev/null
    source "$config"
    setup_paths
}

# --- Deployment Steps ---

create_structure() {
    mkdir -p "$RELEASES_DIR" "$SHARED_DIR/log" "$SHARED_DIR/tmp" "$SHARED_DIR/config"
    ok "Directory structure ready"
}

fetch_code() {
    local release_dir="$1"
    
    if [[ -n "$REPO_URL" ]]; then
        log "Cloning $BRANCH from $REPO_URL"
        git clone --depth 1 --branch "$BRANCH" "$REPO_URL" "$release_dir"
        rm -rf "$release_dir/.git"
    else
        die "No REPO_URL configured"
    fi
    
    ok "Code fetched"
}

link_shared() {
    local release_dir="$1"
    
    # Link shared directories
    for dir in log tmp; do
        rm -rf "${release_dir}/${dir}"
        ln -sfn "${SHARED_DIR}/${dir}" "${release_dir}/${dir}"
    done
    
    # Link shared config files
    if [[ -d "${SHARED_DIR}/config" ]]; then
        for config_file in "${SHARED_DIR}/config"/*; do
            [[ -f "$config_file" ]] || continue
            local basename
            basename=$(basename "$config_file")
            ln -sfn "$config_file" "${release_dir}/${basename}"
        done
    fi
    
    ok "Shared resources linked"
}

run_hook() {
    local hook_name="$1" hook_cmd="$2" release_dir="$3"
    
    if [[ -n "$hook_cmd" ]]; then
        log "Running $hook_name hook..."
        (cd "$release_dir" && eval "$hook_cmd") || die "$hook_name hook failed"
        ok "$hook_name hook complete"
    fi
}

switch_release() {
    local release_dir="$1"
    local previous=""
    
    # Save current release for rollback
    if [[ -L "$CURRENT_LINK" ]]; then
        previous=$(readlink "$CURRENT_LINK")
    fi
    
    # Atomic symlink switch
    local tmp_link="${CURRENT_LINK}.tmp.$$"
    ln -sfn "$release_dir" "$tmp_link"
    mv -Tf "$tmp_link" "$CURRENT_LINK"    # Atomic rename
    
    ok "Switched to: $(basename "$release_dir")"
    
    # Save previous release path for rollback
    if [[ -n "$previous" ]]; then
        echo "$previous" > "${DEPLOY_DIR}/.previous_release"
    fi
}

restart_service() {
    if [[ -n "$RESTART_COMMAND" ]]; then
        log "Restarting service..."
        eval "$RESTART_COMMAND" || die "Service restart failed"
        ok "Service restarted"
    fi
}

health_check() {
    if [[ -z "$HEALTH_CHECK_URL" ]]; then
        warn "No health check URL configured — skipping"
        return 0
    fi
    
    log "Running health check..."
    local i
    for ((i=1; i<=HEALTH_CHECK_RETRIES; i++)); do
        local status
        status=$(curl -so /dev/null -w "%{http_code}" --max-time 5 "$HEALTH_CHECK_URL" 2>/dev/null || echo "000")
        
        if [[ "$status" == "200" ]]; then
            ok "Health check passed (HTTP $status)"
            return 0
        fi
        
        warn "Attempt $i/$HEALTH_CHECK_RETRIES: HTTP $status"
        sleep "$HEALTH_CHECK_DELAY"
    done
    
    fail "Health check failed after $HEALTH_CHECK_RETRIES attempts"
    return 1
}

cleanup_old_releases() {
    local keep="$1"
    local releases
    
    mapfile -t releases < <(ls -1dt "$RELEASES_DIR"/*/ 2>/dev/null)
    
    if (( ${#releases[@]} > keep )); then
        local to_remove=("${releases[@]:$keep}")
        for old in "${to_remove[@]}"; do
            log "Removing old release: $(basename "$old")"
            rm -rf "$old"
        done
        ok "Cleaned up $((${#to_remove[@]})) old releases"
    fi
}

# --- Rollback ---
do_rollback() {
    local target="$1"
    
    if [[ -z "$target" ]]; then
        # Rollback to previous
        [[ -f "${DEPLOY_DIR}/.previous_release" ]] || die "No previous release to rollback to"
        target=$(cat "${DEPLOY_DIR}/.previous_release")
    else
        # Rollback to specific release
        target="${RELEASES_DIR}/${target}"
    fi
    
    [[ -d "$target" ]] || die "Release not found: $target"
    
    log "Rolling back to: $(basename "$target")"
    switch_release "$target"
    restart_service
    
    if health_check; then
        ok "Rollback successful"
    else
        die "Rollback failed health check!"
    fi
}

# --- Deploy ---
do_deploy() {
    local total_steps=8
    local release_name="release_${TIMESTAMP}"
    local release_dir="${RELEASES_DIR}/${release_name}"
    
    echo ""
    echo "╔══════════════════════════════════════════╗"
    echo "║  Deploying: $APP_NAME"
    echo "║  Release:   $release_name"
    echo "║  Branch:    $BRANCH"
    echo "╚══════════════════════════════════════════╝"
    echo ""
    
    step 1 $total_steps "Creating directory structure"
    create_structure
    
    step 2 $total_steps "Fetching code"
    fetch_code "$release_dir"
    
    step 3 $total_steps "Linking shared resources"
    link_shared "$release_dir"
    
    step 4 $total_steps "Running pre-deploy hook"
    run_hook "pre-deploy" "$PRE_DEPLOY_HOOK" "$release_dir"
    
    step 5 $total_steps "Switching to new release"
    switch_release "$release_dir"
    
    step 6 $total_steps "Restarting service"
    restart_service
    
    step 7 $total_steps "Health check"
    if ! health_check; then
        fail "Health check failed — rolling back!"
        do_rollback ""
        die "Deployment failed, rolled back to previous release"
    fi
    
    step 8 $total_steps "Running post-deploy hook"
    run_hook "post-deploy" "$POST_DEPLOY_HOOK" "$release_dir"
    
    cleanup_old_releases "$KEEP_RELEASES"
    
    echo ""
    ok "Deployment complete! 🚀"
    echo ""
}

# --- List Releases ---
list_releases() {
    echo "=== Available Releases ==="
    local current=""
    [[ -L "$CURRENT_LINK" ]] && current=$(readlink "$CURRENT_LINK")
    
    ls -1dt "$RELEASES_DIR"/*/ 2>/dev/null | while read -r dir; do
        local name
        name=$(basename "$dir")
        if [[ "$dir" == "${current}/" || "$dir" == "$current" ]]; then
            printf "  * %-30s  (current)\n" "$name"
        else
            printf "    %-30s\n" "$name"
        fi
    done
}

# --- Main ---
usage() {
    cat <<EOF
Usage: $SCRIPT_NAME [OPTIONS]
  -c FILE         Load configuration
  -b BRANCH       Git branch (default: main)
  -r              Rollback to previous release
  -R RELEASE      Rollback to specific release
  -l              List releases
  -h              Help
EOF
}

ACTION="deploy"
ROLLBACK_TARGET=""

while getopts ":c:b:rR:lh" opt; do
    case $opt in
        c) load_config "$OPTARG" ;;
        b) BRANCH="$OPTARG" ;;
        r) ACTION="rollback" ;;
        R) ACTION="rollback"; ROLLBACK_TARGET="$OPTARG" ;;
        l) ACTION="list" ;;
        h) usage; exit 0 ;;
        *) usage; exit 1 ;;
    esac
done

setup_paths

case "$ACTION" in
    deploy)   do_deploy ;;
    rollback) do_rollback "$ROLLBACK_TARGET" ;;
    list)     list_releases ;;
esac
```

---

## Sample Configuration

```bash
# /etc/deploy/myapp.conf
APP_NAME="myapp"
DEPLOY_BASE="/opt/apps"
REPO_URL="git@github.com:company/myapp.git"
BRANCH="main"
KEEP_RELEASES=5
HEALTH_CHECK_URL="http://localhost:8080/health"
HEALTH_CHECK_RETRIES=10
HEALTH_CHECK_DELAY=3
RESTART_COMMAND="sudo systemctl restart myapp"
PRE_DEPLOY_HOOK="npm install --production && npm run build"
POST_DEPLOY_HOOK="npm run migrate"
```

---

**Next Project:** [Project 6: CLI Task Manager →](Project06-Task-Manager.md)
