# runnn

Automatically re-run a command after SIGINT (ctrl-c).

Send SIGKILL and exit by doing ctrl-c twice quickly.

Usage:

 * `runnn sh -c 'while :; do echo "foo" && sleep 3; done'`
 * `runnn mydevserver`

```bash
runnn() {
  # Usage: runnn <cmd> [args...]
  #
  # Config:
  #   RUNNN_WINDOW=2    # seconds for "double Ctrl+C"
  #   RUNNN_GRACE=1     # seconds to wait after TERM before KILL on double Ctrl+C

  local window="${RUNNN_WINDOW:-2}"
  local grace="${RUNNN_GRACE:-1}"

  (( $# )) || { echo "usage: runnn <cmd> [args...]" >&2; return 2; }

  local child_pid="" pgid=""
  local last_int=0
  local want_restart=0
  local want_quit=0
  local in_handler=0

  _runnn_kill() {
    # Send to process group if we have it, else to the PID.
    local sig="$1"
    if [[ -n "$pgid" ]]; then
      kill -s "$sig" -- "-$pgid" 2>/dev/null || true
    elif [[ -n "$child_pid" ]]; then
      kill -s "$sig" -- "$child_pid" 2>/dev/null || true
    fi
  }

  _runnn_on_int() {
    # Re-entrancy guard (can happen with repeated Ctrl+C)
    (( in_handler )) && return
    in_handler=1

    local now
    now=$(date +%s)

    if (( now - last_int < window )); then
      want_quit=1
      want_restart=0
      # Forward SIGINT like normal, then escalate so wrapper can actually finish.
      _runnn_kill INT
      _runnn_kill TERM
    else
      want_restart=1
      want_quit=0
      # Forward SIGINT to child/group.
      _runnn_kill INT
    fi

    last_int=$now
    in_handler=0
  }

  trap _runnn_on_int INT

  while :; do
    want_restart=0
    want_quit=0

    "$@" &
    child_pid=$!

    # Determine the child's process group id (pgid) so signals hit the whole tree.
    pgid="$(ps -o pgid= "$child_pid" 2>/dev/null | tr -d '[:space:]')"
    [[ "$pgid" =~ ^[0-9]+$ ]] || pgid=""

    # Wait until the child is actually gone.
    # 'wait' can return early if interrupted; we loop until kill -0 says it's dead.
    local rc=0
    while :; do
      wait "$child_pid"
      rc=$?
      kill -0 "$child_pid" 2>/dev/null || break
    done

    if (( want_quit )); then
      # Ensure the child is really terminated; if not, KILL after grace.
      local deadline=$(( $(date +%s) + grace ))
      while kill -0 "$child_pid" 2>/dev/null && (( $(date +%s) < deadline )); do
        sleep 0.05
      done
      if kill -0 "$child_pid" 2>/dev/null; then
        _runnn_kill KILL
      fi

      trap - INT
      return 130
    fi

    if (( want_restart )); then
      continue
    fi

    trap - INT
    return "$rc"
  done
}
```
