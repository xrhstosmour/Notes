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

- To ensure the following functions work correctly, install the [fzf](https://github.com/junegunn/fzf") package and add them to your shell startup configuration file. Also both [color](obsidian://open?vault=notes&file=Programming%2FShell%2FConfiguration%2FConstants%2FColors) constants and [log](obsidian://open?vault=notes&file=Programming%2FShell%2FConfiguration%2FFunctions%2FLogs) functions should be defined.

``` bash
# Function to stash changes with a default name.
# Usage:
#   git_stash "Stash message" or git_stash
git_stash() {
    local name="${1:-temp_$(date +'%d_%m_%YT%H_%M_%S')}"
    git stash push -u -m "$name"
}

# Function to get the default branch.
# Usage:
#   git_get_default_branch
git_get_default_branch() {
    local default_branch=""

    # Attempt to get the default branch from the remote repository
    default_branch=$(git remote show origin | grep 'HEAD branch' | awk '{print $NF}')
    if [ -n "$default_branch" ]; then
        default_branch="origin/$default_branch"
    else
        echo "Could not determine the default branch!" >&2
        return 1
    fi

    echo "$default_branch"
}

# Function to fetch and rebase the current branch onto master/main with autostash enabled by default.
# Usage:
#   git_fetch_and_rebase true/false
git_fetch_and_rebase() {
    local autostash_enabled=${1:-true}
    local default_branch

    # Get the base branch using the `git_get_default_branch` function.
    default_branch=$(git_get_default_branch)
    if [ $? -ne 0 ]; then
        log_error "Failed to determine the base branch!"
        return 1
    fi

    # Check for uncommitted changes if autostash is disabled.
    if [ "$autostash_enabled" = "false" ] && ! git diff-index --quiet HEAD --; then
        log_error "Commit or stash your uncommitted changes before rebasing!"
        return 1
    fi

    log_info "Fetching and rebasing onto \`$default_branch\`..."
    if [ "$autostash_enabled" = "true" ]; then
        git fetch && git rebase -i "$default_branch" --autosquash --autostash
    else
        git fetch && git rebase -i "$default_branch" --autosquash
    fi
}


# Function to fixup a commit using fzf.
# Pressing:
#   - ENTER will fixup the commit
#   - TAB will show the commit changes
#   - ? will toggle the preview and show more details
# Usage:
#   git_auto_fix_up
git_auto_fix_up() {
    # Get the name of the current branch.
    current_branch=$(git rev-parse --abbrev-ref HEAD)

    # Get the base branch.
    default_branch=$(git_get_default_branch)
    if [ $? -ne 0 ]; then
        return 1
    fi

    # Get the log of the current branch excluding commits from the upstream branch.
    commits_list=$(git log --oneline --pretty=format:'%h | %s' --no-merges $default_branch..$current_branch)

    # Check if the commit list is empty.
    if [ -z "$commits_list" ]; then
        echo "No commits found!"
        return 1
    fi

    # Loop through the array.
    echo "$commits_list" | while IFS= read -r line; do

        # Get commit hash and message.
        commit_hash=$(echo "$line" | awk '{print $1}')
        commit_message=$(echo "$line" | cut -d' ' -f3-)

        # Exclude lines where commit message starts with fixup!
        if [[ "$commit_message" != fixup!* ]]; then

            # Print the commit hash and message.
            echo -e "${BOLD_YELLOW}$commit_hash${NO_COLOR} ${BOLD_GREEN}|${NO_COLOR} $commit_message"
        fi

    done | fzf --ansi --bind 'enter:execute(git commit --fixup {1})+abort,tab:execute(git diff {1}^!),?:toggle-preview' --preview '

        # Keep the commit hash and message.
        commit_hash={1}
        commit_message=$(echo {2..} | cut -d"|" -f2- | sed "s/^ //")

        # Get the author, date, and files paths for the commit.
        author=$(git show -s --format="%an" $commit_hash)
        date=$(git show -s --format="%ad" --date=format-local:"%d/%m/%Y at %H:%M:%S" $commit_hash)
        files=$(git diff-tree --no-commit-id --name-only -r $commit_hash)

        # Color constants are not working in the preview window so we use the hardcoded ANSI escape codes.
        # Add "- " in front of each file line with green color.
        formatted_files=""
        while IFS= read -r file; do
            formatted_files+="\e[1;32m-\e[0m ${file}\n"
        done <<< "$files"

        echo -e "\e[1;33mHash:\e[0m $commit_hash"
        echo -e "\e[1;33mMessage:\e[0m $commit_message\n"
        echo -e "\e[1;36mAuthor:\e[0m $author"
        echo -e "\e[1;36mDate:\e[0m $date\n"

        # Show "Files:" and the list of files only if there are any files.
        if [ -n "$files" ]; then
            echo -e "\e[1;32mFiles:\e[0m"
            echo -e "$formatted_files"
        fi
    ' --preview-window=right:50%:hidden:wrap
}

# Function to show/list git stashes and interact with them using fzf.
# Pressing:
#   - ENTER will apply the stash
#   - DELETE will drop the stash
#   - TAB will show the file changes
#   - ? will toggle the preview and show more details
# Usage:
#   git_stash_list
git_stash_list() {

    # Get the stash list as an array.
    stash_list=$(git stash list -n 50 --pretty=format:'%h|%s')

    # Check if the stash list is empty.
    if [ -z "$stash_list" ]; then
        log_error "No stashes found!"
        return 1
    fi

    # Loop through the array.
    echo "$stash_list" | while IFS= read -r line; do

        # Extract the stash hash by splitting based on "|".
        stash_hash=$(echo "$line" | cut -d'|' -f1 | xargs)

        # Extract the stash message and keep it clean, without the branch.
        stash_message=$(echo "$line" | cut -d'|' -f2- | xargs)
        stash_message=$(echo "$stash_message" | sed 's/^On [^:]*: //')

        # Print the branch related to the stash and its message.
        echo -e "${BOLD_YELLOW}$stash_hash${NO_COLOR} ${BOLD_GREEN}|${NO_COLOR} $stash_message"
    done | fzf --ansi --bind 'enter:execute(git stash apply $(git log -g stash --format="%h %gd" | grep -m 1 {1} | awk "{print \$2}"))+abort,delete:execute(git stash drop $(git log -g stash --format="%h %gd" | grep -m 1 {1} | awk "{print \$2}"))+abort,tab:execute(git stash show -p {1}),?:toggle-preview' --preview '

        # Extract stash hash and message from the selection.
        stash_hash={1}
        stash_message=$(echo {2..} | cut -d"|" -f2- | sed "s/^ //")

        # Get the stash index in the stash@{index} format.
        stash_index=$(git log -g stash --format="%h %gd" | grep -m 1 "$stash_hash" | awk "{print \$2}")

        # Get the branch from git stash list excluding the stash message.
        branch=$(git stash list --pretty=format:"%s" | grep -m 1 "$stash_message" | sed "s/.*On \(.*\): $stash_message/\1/" | xargs)

        # Get the date of the stash.
        date=$(git show -s --format="%ad" --date=format-local:"%d/%m/%Y at %H:%M:%S" $stash_hash)

        # Get the list of files affected by the stash.
        files=$(git stash show -p $stash_hash --name-only)

        # Color constants are not working in the preview window so we use the hardcoded ANSI escape codes.
        # Add "- " in front of each file line with green color.
        formatted_files=""
        while IFS= read -r file; do
            formatted_files+="\e[1;32m-\e[0m ${file}\n"
        done <<< "$files"

        echo -e "\e[1;33mIndex:\e[0m $stash_index"
        echo -e "\e[1;33mHash:\e[0m $stash_hash"
        echo -e "\e[1;33mBranch:\e[0m $branch\n"
        echo -e "\e[1;36mMessage:\e[0m $stash_message"
        echo -e "\e[1;36mDate:\e[0m $date\n"

        # Show "Files:" and the list of files only if there are any files.
        if [ -n "$files" ]; then
            echo -e "\e[1;32mFiles:\e[0m"
            echo -e "$formatted_files"
        fi
    ' --preview-window=right:50%:hidden:wrap
}

# Function to show git log for the current branch and interact with it using fzf.
# Pressing:
#   - ENTER will reset the current branch to the selected commit
#   - TAB will show the commit changes
#   - ? will toggle the preview and show more details
# Usage:
#   git_log_current_branch
git_log_current_branch() {

    # Get the name of the current branch.
    current_branch=$(git rev-parse --abbrev-ref HEAD)

    # Get the base branch.
    default_branch=$(git_get_default_branch)
    if [ $? -ne 0 ]; then
        return 1
    fi

    # Get the log of the current branch excluding commits from the upstream branch.
    log_list=$(git log --oneline --pretty=format:'%h | %s' $default_branch..$current_branch)

    # Check if the log list is empty.
    if [ -z "$log_list" ]; then
        echo "No commits found!"
        return 1
    fi

    # Loop through the array.
    echo "$log_list" | while IFS= read -r line; do

        # Extract the commit hash and message.
        commit_hash=$(echo "$line" | awk '{print $1}')
        commit_message=$(echo "$line" | cut -d' ' -f3-)

        # Print the commit hash and message.
        echo -e "${BOLD_YELLOW}$commit_hash${NO_COLOR} ${BOLD_GREEN}|${NO_COLOR} $commit_message"
    done | fzf --ansi --bind 'enter:execute(git reset --hard {1})+abort,tab:execute(git diff {1}^!),?:toggle-preview' --preview '
        # Extract commit hash and message from the selection.
        commit_hash={1}
        commit_message=$(echo {2..} | cut -d"|" -f2- | sed "s/^ //")

        # Get the author, date, and files paths for the commit.
        author=$(git show -s --format="%an" $commit_hash)
        date=$(git show -s --format="%ad" --date=format-local:"%d/%m/%Y at %H:%M:%S" $commit_hash)
        files=$(git diff-tree --no-commit-id --name-only -r $commit_hash)

        # Color constants are not working in the preview window so we use the hardcoded ANSI escape codes.
        # Add "- " in front of each file line with green color.
        formatted_files=""
        while IFS= read -r file; do
            formatted_files+="\e[1;32m-\e[0m ${file}\n"
        done <<< "$files"

        echo -e "\e[1;33mHash:\e[0m $commit_hash"
        echo -e "\e[1;33mMessage:\e[0m $commit_message\n"
        echo -e "\e[1;36mAuthor:\e[0m $author"
        echo -e "\e[1;36mDate:\e[0m $date\n"

        # Show "Files:" and the list of files only if there are any files.
        if [ -n "$files" ]; then
            echo -e "\e[1;32mFiles:\e[0m"
            echo -e "$formatted_files"
        fi
    ' --preview-window=right:50%:hidden:wrap
}

# Function to show all branches not merged or deleted and interact with them using fzf.
# Pressing:
#   - ENTER will checkout to the selected branch
#   - DELETE will delete the selected branch
#   - TAB will show the diff of the selected branch
#   - ? will toggle the preview and show more details
# Usage:
#   git_list_branches
git_list_branches() {

    # Get the base branch.
    default_branch=$(git_get_default_branch)
    if [ $? -ne 0 ]; then
        return 1
    fi

    # Get the current branch.
    current_branch=$(git branch --show-current)

    # Get the list of all branches.
    all_branches=$(git branch -av --format='%(refname:short)')

    # Get the list of all local not pushed branches.
    not_pushed_local_branches=$(git for-each-ref --format="%(refname:short) %(push:track)" refs/heads | grep '\[gone\]' | awk '{print $1}')

    # Get the list of all merged branches.
    merged_branches=$(git branch -a --merged $default_branch | sed 's/^[* ]*//')

    # Exclude merged branches from all branches.
    branch_list=$(echo "$all_branches" | grep -v -F -f <(echo "$merged_branches"))

    # Exclude remote branches that have a corresponding local branch.
    local_branches=$(git branch --format='%(refname:short)')
    branch_list=$(echo "$branch_list" | grep -v -F -f <(echo "$local_branches" | sed 's/^/origin\//'))

    # Exclude `origin/HEAD` and add the default branch to the list.
    branch_list=$(echo "$branch_list" | grep -v '^origin/HEAD$')
    branch_list=$(echo -e "$branch_list\n$default_branch")

    # Add the not pushed local branches to the list.
    branch_list=$(echo -e "$branch_list\n$not_pushed_local_branches")

    # Check if the branch list is empty.
    if [ -z "$branch_list" ]; then
        echo "No branches found!"
        return 1
    fi

    # Loop through the array.
    echo "$branch_list" | while IFS= read -r line; do

        # Check if the branch is the current one.
        if [[ "$line" == "$current_branch" ]]; then

            # Print the current branch name in green.
            echo -e "${BOLD_GREEN}$line${NO_COLOR}"
        else
            # Print the branch name in yellow.
            echo -e "${BOLD_YELLOW}$line${NO_COLOR}"
        fi
    done | fzf --ansi --bind 'enter:execute(git checkout {1})+abort,delete:execute(git branch -d {1})+abort,tab:execute(git diff {1})+abort,?:toggle-preview' --preview '
        # Extract branch name from the selection.
        branch_name={1}

        # Get the author, date, and files for the branch.
        author=$(git log -1 --pretty=format:"%an" $branch_name)
        date=$(git log -1 --pretty=format:"%ad" --date=format-local:"%d/%m/%Y at %H:%M:%S" $branch_name)
        files=$(git ls-tree -r $branch_name --name-only)

        # Color constants are not working in the preview window so we use the hardcoded ANSI escape codes.
        # Add "- " in front of each file line with green color.
        formatted_files=""
        while IFS= read -r file; do
            formatted_files+="\e[1;32m-\e[0m ${file}\n"
        done <<< "$files"

        echo -e "\e[1;33mBranch:\e[0m $branch_name\n"
        echo -e "\e[1;36mAuthor:\e[0m $author"
        echo -e "\e[1;36mDate:\e[0m $date\n"

        # Show "Files:" and the list of files only if there are any files.
        if [ -n "$files" ]; then
            echo -e "\e[1;32mFiles:\e[0m"
            echo -e "$formatted_files"
        fi
    ' --preview-window=right:50%:hidden:wrap
}

# Function to cherry-pick specific commits from different branches.
# Show a list of the branch | commit hash | commit message.
# Pressing:
#   - ENTER will cherry-pick the commit
#   - ? will toggle the preview and show more details
# Usage:
#   git_cherry_pick
git_cherry_pick() {
    # Get the current branch.
    current_branch=$(git branch --show-current)

    # Get the list of commits from all remote branches except the current one.
    commits_list=$(git log --oneline --pretty=format:'%h | %s' --all --remotes | grep -v "origin/$current_branch")

    # Check if the commit list is empty.
    if [ -z "$commits_list" ]; then
        echo "No commits found!"
        return 1
    fi

    # Loop through the array.
    echo "$commits_list" | while IFS= read -r line; do

        # Get commit hash and message.
        commit_hash=$(echo "$line" | awk '{print $1}')
        commit_message=$(echo "$line" | cut -d' ' -f3-)

        # Print the commit hash and message.
        echo -e "${BOLD_YELLOW}$commit_hash${NO_COLOR} ${BOLD_GREEN}|${NO_COLOR} $commit_message"
    done | fzf --ansi --bind 'enter:execute(git cherry-pick {1})+abort,?:toggle-preview' --preview '
        # Keep the commit hash and message.
        commit_hash={1}
        commit_message=$(echo {2..} | cut -d"|" -f2- | sed "s/^ //")

        # Get the branch, author, date, and files paths for the commit.
        branch=$(git branch -a --contains $commit_hash | grep 'remotes/origin/' | sed 's#remotes/##')

        author=$(git show -s --format="%an" $commit_hash)
        date=$(git show -s --format="%ad" --date=format-local:"%d/%m/%Y at %H:%M:%S" $commit_hash)
        files=$(git diff-tree --no-commit-id --name-only -r $commit_hash)

        # Color constants are not working in the preview window so we use the hardcoded ANSI escape codes.
        # Add "- " in front of each file line with green color.
        formatted_files=""
        while IFS= read -r file; do
            formatted_files+="\e[1;32m-\e[0m ${file}\n"
        done <<< "$files"

        echo -e "\e[1;33mHash:\e[0m $commit_hash"
        echo -e "\e[1;33mBranch:\e[0m $branch"
        echo -e "\e[1;33mMessage:\e[0m $commit_message\n"
        echo -e "\e[1;36mAuthor:\e[0m $author"
        echo -e "\e[1;36mDate:\e[0m $date\n"

        # Show "Files:" and the list of files only if there are any files.
        if [ -n "$files" ]; then
            echo -e "\e[1;32mFiles:\e[0m"
            echo -e "$formatted_files"
        fi
    ' --preview-window=right:50%:hidden:wrap
}

# Function to merge the current branch to master/main.
# Usage:
#   git_merge_to_default_branch
git_merge_to_default_branch() {
    local remote="origin"
    local current_branch
    local upstream_branch
    local default_branch
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
        log_info "Checking out \`$1\` branch..."
        git checkout "$1"
    fi

    current_branch=$(git rev-parse --abbrev-ref HEAD)
    log_info "Currently on \`$current_branch\` branch."

    # Resolve upstream branch.
    upstream_branch=$(git rev-parse --abbrev-ref "@{upstream}" 2>/dev/null || echo "$current_branch")
    if ! git ls-remote --heads --exit-code "$remote" "$upstream_branch" &>/dev/null; then
        read -rp "Please enter the branch name for fetch/push operations: " upstream_branch
    fi
    log_info "Branch tracking the remote branch \`$upstream_branch\`."

    # Fetch and rebase current branch onto master/main.
    git_fetch_and_rebase false
    if [ $? -ne 0 ]; then
        log_error "Rebase failed, resolve conflicts/errors before running the script again!"
        return 1
    fi

    # Get the base branch using the `git_get_default_branch` function.
    default_branch=$(git_get_default_branch)

    # Force push current branch.
    log_info "Force pushing \`$current_branch\` to \`$upstream_branch\`..."
    git push "$remote" "HEAD:$upstream_branch" --force-with-lease

    # Merge to master/main.
    branch_to_be_merged="$remote/$upstream_branch"
    log_info "Merging \`$current_branch\` to \`$branch_to_be_merged\`..."
    git checkout "${default_branch#origin/}"
    git reset --hard "$default_branch"

    # Check if the branch to be merged has more than one commit.
    new_commits_count=$(git rev-list --count "$default_branch..$branch_to_be_merged")
    if [ "$new_commits_count" -gt 1 ]; then
        no_ff_option="--no-ff"
    fi

    # Check if a PR number is provided.
    if [ -n "$no_ff_option" ]; then
        merge_commit_msg="-m Merge branch \`$branch_to_be_merged\`"
        if [ -n "$2" ]; then
            merge_commit_msg="$merge_commit_msg -m Closes #$2"
        fi
    fi

    git merge "$branch_to_be_merged" $no_ff_option $merge_commit_msg

    # Extract the branch name without the 'origin/' prefix.
    local_branch="${default_branch#origin/}"

    log_info "The commits listed below will be pushed to \`$default_branch\`:"
    git --no-pager log --decorate --graph --oneline "$local_branch...$default_branch"

    # Prompt the user for confirmation.
    log_warning "Would you like to push your local commits to \`$default_branch\`? (y/n) "
    read push_to_master
    if [[ "$push_to_master" =~ ^[Yy]$ ]]; then
        git push "$remote" "$local_branch"
        if [ $? -ne 0 ]; then
            log_error "Push to \`$default_branch\` failed!"
            git reset --hard HEAD^
            git checkout "$current_branch"
            return 1
        fi
        log_success "Push to \`$default_branch\` was successful!"
    else
        log_info "Push to \`$default_branch\` was aborted!"
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
alias gls="git_log_current_branch"
alias gpl="git pull"
alias gft="git fetch"
alias gftpl="git fetch && git pull"
alias gps="git push"
alias gpsf="git push -f"
alias ga="git add"
alias gap="git add -p"
alias gc="git commit"
alias gca="git commit --amend"
alias gcp="git cherry-pick"
alias gcpc="git_cherry_pick_commit"
alias gcpa="git cherry-pick --abort"
alias gst="git status"
alias gd="git diff"
alias gds="git diff --staged"
alias gdsn="git diff --staged --name-only"
alias gco="git checkout"
alias gcob="git checkout -b"
alias gsob="git switch --orphan"
alias grstr="git restore"
alias grstrs="git restore --staged"
alias grstrss="git restore --staged -p"
alias grst='git reset HEAD~'
alias grsta='git reset --hard'
alias gfrb="git_fetch_and_rebase"
alias grb="git rebase -i --autosquash HEAD~"
alias grba="git rebase --abort"
alias grbc="git rebase --continue"
alias grbs="git rebase --skip"
alias gbr="git branch"
alias gbrls="git_list_branches"
alias gbrd="git branch -d"
alias gbrdf="git branch -D"
alias gbrr="git branch -m"
alias gsts="git_stash"
alias gstsl="git_stash_list"
alias gstsa="git stash apply"
alias gstsp="git stash pop"
alias gstsd="git stash drop"
alias gfx="git commit --fixup"
alias gafx="git_auto_fix_up"
alias grp="git prune && rm -rf ./.git/gc.log && git gc --prune=now && git remote prune origin"
alias gmr="git revert -m 1"
alias grc="git revert"
alias gm="git_merge_to_default_branch"
```
