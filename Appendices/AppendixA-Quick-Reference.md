# Appendix A: Bash Quick Reference

## Variables

```bash
var="value"                # Assignment (no spaces around =)
echo "$var"                # Use variable
echo "${var}"              # Explicit braces
readonly CONST="fixed"     # Constant (can't change)
unset var                  # Delete variable
export var                 # Make available to child processes
```

## Quoting

```bash
"double quotes"            # Allows $var and $(cmd) expansion
'single quotes'            # Everything literal
$'ansi\n'                  # ANSI-C (escape sequences work)
```

## Parameter Expansion

```bash
${var:-default}            # Default if unset/empty
${var:=default}            # Assign default if unset/empty
${var:?error}              # Error if unset/empty
${var:+alternate}          # Alternate if set and non-empty
${#var}                    # String length
${var:offset:length}       # Substring
${var#pattern}             # Remove shortest prefix match
${var##pattern}            # Remove longest prefix match
${var%pattern}             # Remove shortest suffix match
${var%%pattern}            # Remove longest suffix match
${var/old/new}             # Replace first match
${var//old/new}            # Replace all matches
${var/#old/new}            # Replace prefix match
${var/%old/new}            # Replace suffix match
${var^^}                   # Uppercase all
${var,,}                   # Lowercase all
${!prefix*}                # Variable names matching prefix
```

## Arrays

```bash
arr=(a b c)                # Indexed array
declare -A map             # Associative array (Bash 4+)
${arr[0]}                  # Access element
${arr[@]}                  # All elements
${#arr[@]}                 # Count elements
${!arr[@]}                 # All indices/keys
arr+=(d)                   # Append
unset 'arr[1]'             # Delete element
```

## Conditionals

```bash
if [[ condition ]]; then ...; elif [[ ... ]]; then ...; else ...; fi

# String tests
[[ -z "$str" ]]            # Empty
[[ -n "$str" ]]            # Non-empty
[[ "$a" == "$b" ]]         # Equal
[[ "$a" != "$b" ]]         # Not equal
[[ "$a" < "$b" ]]          # Lexicographic less-than
[[ "$a" == pattern* ]]     # Glob match
[[ "$a" =~ regex ]]        # Regex match

# Numeric tests
(( a == b ))    (( a != b ))    (( a < b ))
(( a > b ))     (( a <= b ))    (( a >= b ))

# File tests
[[ -f file ]]              # Regular file exists
[[ -d dir ]]               # Directory exists
[[ -e path ]]              # Exists (any type)
[[ -r file ]]              # Readable
[[ -w file ]]              # Writable
[[ -x file ]]              # Executable
[[ -s file ]]              # Non-empty file
[[ -L file ]]              # Symlink
[[ file1 -nt file2 ]]     # Newer than
```

## Loops

```bash
for item in list; do ...; done
for ((i=0; i<10; i++)); do ...; done
while [[ condition ]]; do ...; done
until [[ condition ]]; do ...; done
select opt in list; do ...; done
```

## Functions

```bash
func_name() {
    local var="$1"         # Local variable
    echo "result"          # Return data via stdout
    return 0               # Return exit code (0-255)
}
result=$(func_name arg)    # Capture output
```

## Redirection

```bash
cmd > file                 # Stdout to file (overwrite)
cmd >> file                # Stdout to file (append)
cmd 2> file                # Stderr to file
cmd &> file                # Both stdout+stderr to file
cmd > file 2>&1            # Both (POSIX compatible)
cmd < file                 # Stdin from file
cmd1 | cmd2                # Pipe stdout to next command
cmd1 |& cmd2               # Pipe stdout+stderr
cmd <<EOF ... EOF           # Here document
cmd <<< "string"           # Here string
```

## Special Variables

```bash
$0                         # Script name
$1 $2 ... ${10}            # Positional arguments
$#                         # Number of arguments
$@                         # All arguments (separate words)
$*                         # All arguments (single word)
$?                         # Last exit code
$$                         # Current PID
$!                         # Last background PID
$BASHPID                   # Current process PID (differs in subshells)
$LINENO                    # Current line number
$FUNCNAME                  # Current function name
$RANDOM                    # Random number 0-32767
$SECONDS                   # Seconds since shell started
```

## Process Control

```bash
cmd &                      # Run in background
wait                       # Wait for all background jobs
wait $pid                  # Wait for specific PID
jobs                       # List background jobs
fg %1                      # Bring job 1 to foreground
bg %1                      # Resume job 1 in background
kill $pid                  # Send SIGTERM
kill -9 $pid               # Send SIGKILL
trap 'cmd' SIGNAL          # Set signal handler
```

## Arithmetic

```bash
$(( expression ))          # Arithmetic expansion
(( x++ ))                  # Arithmetic command
(( x += 5 ))               # Compound assignment
(( x = a > b ? a : b ))   # Ternary
```

## Set Options

```bash
set -e                     # Exit on error
set -u                     # Error on unset variables
set -o pipefail            # Pipeline fails on first error
set -x                     # Debug trace
set -f                     # Disable globbing
set -C                     # No clobber
```

## Common Patterns

```bash
# Die function
die() { echo "ERROR: $*" >&2; exit 1; }

# Strict mode
set -euo pipefail

# Script directory
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"

# Temp file with cleanup
tmpfile=$(mktemp); trap 'rm -f "$tmpfile"' EXIT

# Safe cd
cd "$dir" || exit 1

# Check command exists
command -v cmd >/dev/null 2>&1 || die "cmd not found"

# Default value
: "${VAR:=default}"

# Read file line by line
while IFS= read -r line; do echo "$line"; done < file.txt
```
