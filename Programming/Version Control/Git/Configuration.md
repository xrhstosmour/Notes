#version-control #git #configuration

- User configuration:

``` bash
git config --global user.name "<your_full_name>"
git config --global user.email <your_email>
```

- Pull to rebase by default:

``` bash
git config --global pull.rebase true
```

- Set the conflict style to `diff3`:

``` bash
git config --global merge.conflictstyle diff3
```

- Set the default push branch to current:

``` bash
git config --global push.default current
```

- Set aliases by appending below the `[alias]` section, at the `~/.gitconfig` file the below:

``` bash
autofixup = "!git log -n 50 --pretty=format:'%h | %s' --no-merges | while read -r line; do echo \"$line | $(git diff-tree --no-commit-id --name-only -r $(echo $line | awk '{print $1}') | tr '\n' ' ')\"; done | grep -v '^.* | fixup!' | fzf | cut -c -7 | xargs -o git commit --fixup"
```

**In order for the above to function correctly, you must install the [fzf](https://github.com/junegunn/fzf") package.**

- Set constants to your shell startup configuration file:

``` bash
# Constants for Git functions.

# Colors for the script's messages.
NO_COLOR='\e[0m'
BOLD_CYAN='\e[1;36m'
BOLD_GREEN='\e[1;32m'
BOLD_YELLOW='\e[1;33m'
BOLD_RED='\e[1;31m'
```

- Set functions to your shell startup configuration file:

``` bash
# Git functions.

# Function to log an info message.
# Usage:
#   log_info "Info message to log"
#   log_info -n "Info message to log without newline"
log_info() {
    local prefix="\n"
    if [[ "$1" == "-n" ]]; then
        prefix=""
        shift
    fi
    info="$1"
    echo -e "${prefix}${BOLD_CYAN}$info${NO_COLOR}" >&2
}

# Function to log a success message.
# Usage:
#   log_success "Success message to log"
#   log_success -n "Success message to log without newline"
log_success() {
    local prefix="\n"
    if [[ "$1" == "-n" ]]; then
        prefix=""
        shift
    fi
    success="$1"
    echo -e "${prefix}${BOLD_GREEN}$success${NO_COLOR}" >&2
}

# Function to log a warning message.
# Usage:
#   log_warning "Warning message to log"
#   log_warning -n "Warning message to log without newline"
log_warning() {
    local prefix="\n"
    if [[ "$1" == "-n" ]]; then
        prefix=""
        shift
    fi
    warning="$1"
    echo -e "${prefix}${BOLD_YELLOW}$warning${NO_COLOR}" >&2
}

# Function to log an error message.
# Usage:
#   log_error "Error message to log"
#   log_error -n "Error message to log without newline"
log_error() {
    local prefix="\n"
    if [[ "$1" == "-n" ]]; then
        prefix=""
        shift
    fi
    error="$1"
    echo -e "${prefix}${BOLD_RED}$error${NO_COLOR}" >&2
}

# Function to stash changes with a default name.
# Usage:
#   stash_with_default_name "Stash message" or stash_with_default_name
stash_with_default_name() {
    local name="${1:-temp_$(date +'%d_%m_%YT%H_%M_%S')}"
    git stash push -u -m "$name"
}

# Function to get the base branch, either master or main.
# Usage:
#   get_base_branch
get_base_branch() {
    local base_branch

    if git show-ref --verify --quiet refs/remotes/origin/master; then
        base_branch="origin/master"
    elif git show-ref --verify --quiet refs/remotes/origin/main; then
        base_branch="origin/main"
    else
        log_error "Neither origin/master nor origin/main exists!"
        return 1
    fi

    echo "$base_branch"
}

# Function to fetch and rebase the current branch onto master/main with autostash enabled by default.
# Usage:
#   fetch_and_rebase true/false
fetch_and_rebase() {
    local autostash_enabled=${1:-true}
    local base_branch

    # Get the base branch using the `get_base_branch` function.
    base_branch=$(get_base_branch)
    if [ $? -ne 0 ]; then
        log_error "Failed to determine the base branch!"
        return 1
    fi

    # Check for uncommitted changes if autostash is disabled.
    if [ "$autostash_enabled" = "false" ] && ! git diff-index --quiet HEAD --; then
        log_error "Commit or stash your uncommitted changes before rebasing!"
        return 1
    fi

    log_info "Fetching and rebasing onto '$base_branch'..."
    if [ "$autostash_enabled" = "true" ]; then
        git fetch && git rebase -i "$base_branch" --autosquash --autostash
    else
        git fetch && git rebase -i "$base_branch" --autosquash
    fi
}

# Function to merge the current branch to master/main.
# Usage:
#   merge
merge() {
    # Terminate script on error.
    set -e

    local remote="origin"
    local current_branch
    local upstream_branch
    local base_branch
    local branch_to_be_merged
    local new_commits_count
    local no_ff_option=""
    local merge_commit_msg=""

    # Check if a rebase is ongoing.
    if [ -d "$(git rev-parse --git-dir)/rebase-merge" ] || [ -d "$(git rev-parse --git-dir)/rebase-apply" ]; then
        log_error "Rebase in progress, operation stopped!"
        return 1
    fi

    # Checkout the specified branch if provided.
    if [ -n "$1" ]; then
        log_info "Checking out '$1' branch..."
        git checkout "$1"
    fi

    current_branch=$(git rev-parse --abbrev-ref HEAD)
    log_info "Currently on '$current_branch' branch."

    # Resolve upstream branch.
    upstream_branch=$(git rev-parse --abbrev-ref "@{upstream}" 2>/dev/null || echo "$current_branch")
    if ! git ls-remote --heads --exit-code "$remote" "$upstream_branch" &>/dev/null; then
        read -rp "Please enter the branch name for fetch/push operations: " upstream_branch
    fi
    log_info "Branch tracking the remote branch '$upstream_branch'."

    # Fetch and rebase current branch onto master/main.
    fetch_and_rebase false
    if [ $? -ne 0 ]; then
        log_error "Rebase failed, resolve conflicts/errors before running the script again!"
        return 1
    fi

    # Get the base branch using the `get_base_branch` function.
    base_branch=$(get_base_branch)

    # Force push current branch.
    log_info "Force pushing '$current_branch' to '$upstream_branch'..."
    git push "$remote" "HEAD:$upstream_branch" --force-with-lease

    # Merge to master/main.
    branch_to_be_merged="$remote/$upstream_branch"
    log_info "Merging '$current_branch' to '$branch_to_be_merged'..."
    git checkout "${base_branch#origin/}"
    git reset --hard "$base_branch"

    # Check if the branch to be merged has more than one commit.
    new_commits_count=$(git rev-list --count "$base_branch..$branch_to_be_merged")
    if [ "$new_commits_count" -gt 1 ]; then
        no_ff_option="--no-ff"
    fi

    # Check if a PR number is provided.
    if [ -n "$no_ff_option" ]; then
        merge_commit_msg="-m 'Merge branch $branch_to_be_merged'"
        if [ -n "$2" ]; then
            merge_commit_msg="$merge_commit_msg -m 'Closes #$2'"
        fi
    fi

    git merge "$branch_to_be_merged" $no_ff_option $merge_commit_msg

    # Extract the branch name without the 'origin/' prefix.
    local_branch="${base_branch#origin/}"

    log_info "The commits listed below will be pushed to '$base_branch':"
    git --no-pager log --decorate --graph --oneline "$local_branch...$base_branch"

    # Prompt the user for confirmation.
    log_warning "Would you like to push your local commits to '$base_branch'? (y/n) "
    read push_to_master
    if [[ "$push_to_master" =~ ^[Yy]$ ]]; then
        git push "$remote" "$local_branch"
        if [ $? -ne 0 ]; then
            log_error "Push to '$base_branch' failed!"
            git reset --hard HEAD^
            git checkout "$current_branch"
            return 1
        fi
        log_success "Push to '$base_branch' was successful!"
    else
        log_info "Push to '$base_branch' was aborted!"
        git reset --hard HEAD^
        git checkout "$current_branch"
    fi
}
```

- Set aliases to your shell startup configuration file:

``` bash
# Aliases for Git operations.
alias g="git"
alias gl="git log"
alias gpl="git pull"
alias gps="git push"
alias gpsf="git push -f"
alias gft="git fetch"
alias ga="git add"
alias gap="git add -p"
alias gc="git commit"
alias gcp="git cherry-pick"
alias gst="git status"
alias gd="git diff"
alias gds="git diff --staged"
alias gdsn="git diff --staged --name-only"
alias gco="git checkout"
alias gcob="git checkout -b"
alias grstr="git restore"
alias grstrs="git restore --staged"
alias grstrss="git restore --staged -p"
alias grst='git reset HEAD~'
alias grb="git rebase -i --autosquash HEAD~"
alias grba="git rebase --abort"
alias grbc="git rebase --continue"
alias grbs="git rebase --skip"
alias gbr="git branch"
alias gbrd="git branch -d"
alias gbrdf="git branch -D"
alias gbrr="git branch -m"
alias gsts="stash_with_default_name"
alias gstsl="git stash list"
alias gstsa="git stash apply"
alias gstsp="git stash pop"
alias gstsd="git stash drop"
alias gstss="git stash show"
alias gfx="git commit --fixup"
alias gafx="git autofixup"
alias gfrb="fetch_and_rebase"
alias grcl="git prune && rm -rf ./.git/gc.log && git gc --prune=now && git remote prune origin"
alias gmtm="merge"
```
