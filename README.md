# travelrc – take your rc-files with you when using ssh or su

## What is the problem?

Over the years, I massively configure commandline tools that I regularly use, like bash, vim, screen an so on.
I created aliases, a custom prompt, special keybindings and commands.
It fits my workflow very well.
But every time I switch to another system (ssh to a local VM, or cloud-VM, or a docker container; changing to root) none of these settings work anymore and I'm annoyed.

## How can it be solved?

I put my rc-files in a git repo and cloned it on systems that I use regularly.
It works, I just have to pull my latest changes a lot.
But it does not work on temporal systems or on systems where I do not want to clone (or can't clone) all my rc-files.
The requirements that I came up with for traveling with my rc-files are like this:

* only save travelled rc-files temporarily
* it's okay to travel with just the most important rc-files
* do not interfere with already existing rc-files (especially for shared accounts like root)
* use it with ssh, su (changing to root or another user on the same machine), docker, and probably more
* do not add too much steps in the process (no rollout via ansible, no scp then ssh), keep it ephemeral

## How did I solve it?

I searched the internet for solutions to my problem, found something (https://github.com/IngoMeyer441/sshrc) and extended and (in my opinion) improved it.

The basic idea is to minify (remove comments), compress and tar the rc-files,
convert this archive to a (long) string (base64 encode),
and add this string and commands to decompress and extract it as a command to ssh (or su/docker).

My solution requires some steps which I'll outline here.
It can be used as is, but it can also act as an inspiration for other solutions.

* create a directory (the name of the directory is used once in the `minify-trc` function below) and symlink (or copy) the rc-files, that should travel with you, into it

```bash
mkdir $HOME/.travelrc
ln -s -t $HOME/.travelrc $HOME/{.bashrc,.inputrc,.vimrc,.screenrc,.tmux.conf}
```

* add a directory with executables that should be available in the travelled session

```bash
mkdir $HOME/.travelrc/trcbin
echo '#!/bin/bash
bash --rcfile "$TBASHRC"' > $HOME/.travelrc/trcbin/trcbash
chmod +x $HOME/.travelrc/trcbin/trcbash
```

* add helper functions to your `.bashrc`

```bash
minify-trc() {
    # this function copies the trc files to a tempdir and minifies them, it returns the tempdirname on stdout
    local trcdir="${TRAVELRCDIR:-$HOME/.travelrc}" # use the travelled rc dir if already travelled
    [ -f "$trcdir/.bashrc" ] || { echo "file .bashrc inside trcdir \"$trcdir\" does not exist" >&2; return 1; }
    # remove all comments from the files to decrease size, has to be done carefully (comment characters can be inside quotes or other commands)
    echo -n "minifying trcdir... " >&2
    local trc_min_dir="$(mktemp --directory "/tmp/.trc.min.XXXXXX")"
    cp --dereference --recursive --target-directory="$trc_min_dir" "$trcdir/"*
    [ -f "$trc_min_dir/.vimrc" ] && sed --in-place -e 's/^ *".*$//' -e 's/ \+" .*$//' "$trc_min_dir/.vimrc" # special minifyer for vimrc
    find "$trc_min_dir/" -type f -exec sed --in-place -e 's/^ *#[^!].*$//' -e 's/ \+# .*$//' '{}' '+' # minify all files as if they were shell scripts
    echo "$trc_min_dir"
}

trccmd() {
    # echo all the commands to travel with rc files, arguments act as additional arguments to tar
    local trc_min_dir="$(minify-trc)"
    echo -n "packing trcdir... " >&2
    local trcvar="$(tar --create --file=- "$@" --directory="$trc_min_dir" --dereference ./ | base64 --wrap=0)" # this writes the compressed contents of $trc_min_dir base64-encoded to $trcvar
    rm -rf "$trc_min_dir" # delete the temporal directory generated by minify-trc
    echo "to ${#trcvar} bytes... " >&2
    [ ${#trcvar} -lt 65536 ] || { echo "content of trcdir \"$trcdir\" is too big, even after minifying" >&2; return 1; }
    # export $TRAVELRCDIR and create this directory, it could also be created in /tmp
    echo '
export TRAVELRCDIR=$HOME/.travelrc.travelled
readonly TRAVELRCDIR
mkdir --parents $TRAVELRCDIR
'
    # SSH_TTY should still be set to figure out whether this is a ssh session
    [ -z "$SSH_TTY" ] || echo "export SSH_TTY=$SSH_TTY"
    # decompress the files saved to $trcvar; start a bash with the travelled rc-file; only remove $TRAVELRCDIR if there is no screen or tmux session (which can still use the files); last command is true so that the returncode is always 0
    # TODO: is it save to always remove $TRAVELRCDIR because it will be recreated on the next connection?
    echo '
echo '"$trcvar"' | base64 --decode | tar --file=- --extract '"$@"' --touch --directory=$TRAVELRCDIR
chmod --quiet --recursive go= $TRAVELRCDIR
bash --rcfile $TRAVELRCDIR/.bashrc
screen -ls 1>/dev/null 2>&1 || tmux has-session 1>/dev/null 2>&1 || rm -rf $TRAVELRCDIR
true
'
}
```

* add functions to your `.bashrc` that enable travelling with the rc files via ssh, su, and docker
    * I prepended a „t“ to each of these commands, so when I type `ssh host` and realize that my rc-files are missing, I can disconnect, prepend a „t“ tho the previous command (to get `tssh host`) and everything is better :-)
    * this code has to be added after the helper functions mentioned above

```bash
tssh() {
    # use -t to force the allocation of a terminal
    ssh -t "$@" "$(trccmd --xz)"
}
_tssh_completion() {
    # when completion is requested, it will be redefined to use _ssh and then load the completion function for ssh, see https://stackoverflow.com/questions/61539494/how-does-bash-do-ssh-autocompletion
    complete -F _ssh tssh
    __load_completion "ssh" && return 124 || complete -r tssh # if loading completion is successful then return, otherwise disable completion for tssh
}
complete -F _tssh_completion tssh # this just loads the correct completion function

tdocker() {
    local dcmd="$1"
    shift
    # some docker images don't come with --xz support, so --gzip is used
    docker "$dcmd" --interactive --tty "$@" bash -c "$(trccmd --gzip)"
}
# TODO: add completion

tsu() { # travel substitude user
    local next_user="${1:-root}"
    local tsu_cmd="$(mktemp "/tmp/.tsu-cmd.XXXXXX")"
    # create a script as tsu_cmd, which should always return 0 to be able to tell whether tsu was successful
    trccmd --gzip > "$tsu_cmd"
    chmod --quiet ugo=rx "$tsu_cmd"
    # try several ways to change the user and execute the tsu_cmd
    local tsu_done="false"
    if type -P sudo &>/dev/null; then
        # the groups test might not be very accurate, but it is quiet; sudo is usually allowed for user vagrant
        # the sudo test is accurate, but creates log messages, which might not be desired, especially when sudo is not allowed
        if groups | \grep --quiet -e "sudo" -e "admin" -e "wheel" || [ "$USER" = "vagrant" ] #\
            #|| \sudo -nv &>/dev/null || \sudo -nv 2>&1 | \grep --quiet '^sudo:' # sudo is allowed with or without password
        then
            echo "trying sudo, enter your password:"
            # 'sudo -u' seems better than 'sudo su' because the latter displayed problems with IOCTL (I/O-Control)
            sudo -u "$next_user" "$tsu_cmd" && tsu_done="true"
            #sudo su "$next_user" --command "$tsu_cmd" && tsu_done="true"
        else
            read -p "sudo might not be allowed for you ($USER), try anyway? [y/N] " try_sudo_anyway
            [ "y" = "$try_sudo_anyway" ] && sudo -u "$next_user" "$tsu_cmd" && tsu_done="true"
        fi
        if ! "$tsu_done" && [ "$next_user" != "root" ]; then
            echo "trying su 'root' to call sudo '$next_user', enter root's password:"
            su "root" --command "sudo -u $next_user $tsu_cmd" && tsu_done="true"
        fi
    fi
    if ! "$tsu_done"; then
        echo "trying su '$next_user', enter $next_user's password:"
        su "$next_user" --command "$tsu_cmd" && tsu_done="true"
    fi
    if ! "$tsu_done"; then
        echo "trying su 'root' --command su '$next_user', enter root's password:"
        su "root" --command "su $next_user --command '$tsu_cmd'" && tsu_done="true"
    fi
    $tsu_done || echo "sorry, tsu did not work in this case :-("
    rm --force "$tsu_cmd"
}
complete -A user tsu # complete usernames
```

* add code to your `.bashrc` (the one that will travel) to detect whether this is a „travelled rc session“ and make some configurations, i.e. make screen and vim use the travelled rc-files, and add the „trcbin“ directory to PATH
    * I have this at the beginning of my `.bashrc`, but _after_ defaults for `EDITOR` and `INPUTRC` have been set

```bash
if [ -n "$TRAVELRCDIR" ]; then # this is a travelled rc session
    [ -f "$TRAVELRCDIR/.bashrc" ] && export TBASHRC="$TRAVELRCDIR/.bashrc"
    [ -f "$TRAVELRCDIR/.inputrc" ] && export INPUTRC="$TRAVELRCDIR/.inputrc"
    [ -f "$TRAVELRCDIR/.screenrc" ] && type -P screen &>/dev/null && alias screen="$(type -P screen) -c $TRAVELRCDIR/.screenrc -s trcbash"
    [ -f "$TRAVELRCDIR/.tmux.conf" ] && type -P tmux &>/dev/null && alias ttmux="$(type -P tmux) -f $TRAVELRCDIR/.tmux.conf new-session 'trcbash'" # this alias does not work properly when named "tmux" (it fucked up other calls of tmux)
    if [ -f "$TRAVELRCDIR/.vimrc" ] && type -P vim &>/dev/null; then
        export EDITOR="$(type -P vim) -u $TRAVELRCDIR/.vimrc"
        alias vim="$EDITOR"
    fi
    [ -d "$TRAVELRCDIR/trcbin" ] && export PATH="$TRAVELRCDIR/trcbin:$PATH" # add (prepend) trcbin to PATH
fi
```

## FAQ

* Can it fix more problems?
    * yes, you can add more executables to PATH
    * I had to execute scripts that unnecessarily used `clear`, which meant that I was not able to scroll up anymore, so I added a script to the `trcbin` directory with the name `clear`, but which did not do a real clear → problem solved (for me)
* Can it be used with zsh?
    * most likely yes, just replace bash with zsh, but zsh might not be installed on the destination-system and then you're screwed; perhaps bash works with most of the zsh configurations, I can't tell
* Is it very efficient?
    * probably not :-(
    * I've tried to make it efficient, but I think in some cases it starts much more shells then necessary (i.e. `trcbash` in the `trcbin` directory)
    * it might slow down ssh connections a few seconds, but for me having my rc-files with me outweighs this downside
* Does it work with tmux?
    * not very well (see the ttmux alias above), but when tmux is started (or a new window is created), just type `trcbash` and you'll get a proper bash ;-)
* Is this project finished?
    * I've been using it for around a year now and am still improving if necessary

## License

CC-BY-NC-SA 4.0 https://creativecommons.org/licenses/by-nc-sa/4.0/

