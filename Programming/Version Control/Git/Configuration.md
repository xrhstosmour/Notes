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

- Set functions to your shell startup configuration file:

``` bash
# Git functions.
stash_with_default_name() {
	local name="${1:-temp_$(date +'%d_%m_%YT%H_%M_%S')}"
	git stash push -u -m "$name"
}

git_fetch_and_rebase() {
	if git show-ref --verify --quiet refs/remotes/origin/master; then
		git fetch && git rebase -i origin/master --autosquash --autostash
	elif git show-ref --verify --quiet refs/remotes/origin/main; then
		git fetch && git rebase -i origin/main --autosquash --autostash
	else
		echo "Neither origin/master nor origin/main exists!"
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
alias gfrb="git_fetch_and_rebase"
alias grcl="git prune && rm -rf ./.git/gc.log && git gc --prune=now && git remote prune origin"
```
