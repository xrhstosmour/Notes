#constants #shell #bash #logs

Here are some useful log functions for various shell script usages. Ensure [color](obsidian://open?vault=notes&file=Programming%2FShell%2FConfiguration%2FConstants%2FColors) constants are defined before using them:

``` bash
# Base function to log a message with a specific color.
# Usage:
#   log_base COLOR "Message to log"
#   log_base COLOR -n "Message to log without newline"
log_base() {
    local color="$1"
    local prefix="\n"
    shift

    # Check if the message should be printed without a newline.
    if [[ "$1" == "-n" ]]; then
        prefix=""
        shift
    fi

    local message="$1"
    echo -e "${prefix}${color}${message}${NO_COLOR}" >&2
}

# Function to log an info message.
# Usage:
#   log_info "Info message to log"
#   log_info -n "Info message to log without newline"
log_info() {
    log_base "$BOLD_CYAN" "$@"
}

# Function to log a success message.
# Usage:
#   log_success "Success message to log"
#   log_success -n "Success message to log without newline"
log_success() {
    log_base "$BOLD_GREEN" "$@"
}

# Function to log a warning message.
# Usage:
#   log_warning "Warning message to log"
#   log_warning -n "Warning message to log without newline"
log_warning() {
    log_base "$BOLD_YELLOW" "$@"
}

# Function to log an error message.
# Usage:
#   log_error "Error message to log"
#   log_error -n "Error message to log without newline"
log_error() {
    log_base "$BOLD_RED" "$@"
}
```
