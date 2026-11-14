# Agent mail

A local mailbox for Claude Code agents and you. Each message is one email file in a Maildir. Replies form threads, and a thread can be a task.

There is no mail server. Messages go straight into the Maildir with `notmuch insert`, and `notmuch` gives the threads, the search, and the tags. Each person has a separate mailbox on their own machine.

## Files

```
| Path                     | What it is                                                     |
|--------------------------|----------------------------------------------------------------|
| `SKILL.md`               | The Claude Code skill that tells agents how to use the mailbox |
| `notmuch-config.example` | The template for your local `notmuch-config`                   |
| `notmuch-config`         | Your notmuch configuration for this mailbox (not in git)       |
| `messages/`              | The Maildir and the `.notmuch` index (not in git)              |
```

In the commands below, `<repo>` is the directory of your clone, for example `~/web/agent-mail`.

## Install

1. Install the tools:

       brew install notmuch fswatch

   On Linux, use `inotifywait -e create` from `inotify-tools` instead of `fswatch`.

2. Make your configuration from the template:

       cp <repo>/notmuch-config.example <repo>/notmuch-config

   In `notmuch-config`, set `path` to `<repo>/messages`. A relative path is relative to your home directory. You can also set `name` to your name.

3. Create the Maildir and the index:

       mkdir -p <repo>/messages/{cur,new,tmp}
       NOTMUCH_CONFIG=<repo>/notmuch-config notmuch new

4. Make the skill available to Claude Code in all projects:

       ln -s <repo> ~/.claude/skills/mailbox

## Conventions

- Teams: a team is a group of agents that work on one task, for example `task-1`. Each team has its own domain, `<team>.agents.local`. A new team needs no setup: the first message to the domain starts it.
- Addresses: `<role>@<team>.agents.local`, for example `engineer@task-1.agents.local` and `tester@task-1.agents.local`. Agents send mail only inside their own team and to you.
- You have one address for all teams: `human@agents.local`. It is the same for everyone, because each mailbox has only one human.
- Tags:
  - `task`: the first message of a task thread.
  - `done`: the task is complete.
  - `seen-<address>`: this reader has read the message, for example `seen-engineer@task-1.agents.local`. Each reader has a separate tag, because all readers share one database. Do not use `unread` for this.

Two agents with the same role in different teams (`engineer@task-1...` and `engineer@task-2...`) do not see the mail of each other. The mailbox keeps only the conversations apart. If two teams work in the same repository, give each team its own git worktree.

## Start an agent

Give the full address in the start prompt, for example:

    You are tester@task-1.agents.local. Use the mailbox skill. Read your new mail, do the work, and reply.

The skill tells the agent how to wait for new mail with `fswatch`. The Claude Code `Monitor` tool can run this command.

## Use the command line

Set the configuration in your shell first:

    export NOTMUCH_CONFIG=<repo>/notmuch-config

CAUTION: Do not set `NOTMUCH_CONFIG` in your shell profile. Then every notmuch command in every shell uses the agent mailbox instead of your personal mail.

Common commands:

    notmuch search '*'                                    # all threads
    notmuch search tag:task and not tag:done              # open tasks
    notmuch search 'to:human@agents.local and not tag:seen-human@agents.local'
    notmuch search 'to:task-1.agents.local or from:task-1.agents.local'  # one team
    notmuch show --entire-thread=true thread:<id>         # read a thread
    notmuch tag +seen-human@agents.local -- thread:<id>   # mark as read
    notmuch tag +done -- thread:<id>                      # close a task

To send a message from the command line, write the full message text (headers, an empty line, the body) to the input of `notmuch insert`. `SKILL.md` shows an example.

## Use Emacs

Homebrew installs the notmuch Emacs package in `/opt/homebrew/share/emacs/site-lisp/notmuch`.

notmuch can also hold your personal mail, so open the mailbox in a separate Emacs instance:

    NOTMUCH_CONFIG=<repo>/notmuch-config emacs --no-desktop -f notmuch &

The `--no-desktop` flag stops this instance from loading or saving your desktop (the buffer list of your main session).

Add this to your Emacs init file. It changes the settings only in the agent mail instance. If the name of your clone does not contain `agent-mail`, change the string in the first line.

```elisp
(when (string-match-p "agent-mail" (or (getenv "NOTMUCH_CONFIG") ""))
  (defun agent-mail-send ()
    "Store the current message in the agent mailbox."
    (goto-char (point-min))
    (when (re-search-forward
           (concat "^" (regexp-quote mail-header-separator) "\n") nil t)
      (replace-match "\n"))
    (unless (zerop (call-process-region (point-min) (point-max) "notmuch"
                                        nil nil nil "insert" "+seen-human@agents.local"))
      (error "notmuch insert failed")))
  (setq message-send-mail-function #'agent-mail-send
        notmuch-fcc-dirs nil
        ;; Opening a message marks it as read with your tag.
        notmuch-show-mark-read-tags '("+seen-human@agents.local")
        ;; One saved search per team on the notmuch hello screen.
        notmuch-saved-searches
        '((:name "new for me" :key "n"
           :query "to:human@agents.local and not tag:seen-human@agents.local")
          (:name "open tasks" :key "o" :query "tag:task and not tag:done")
          (:name "task-1" :key "1"
           :query "to:task-1.agents.local or from:task-1.agents.local"))))
```

The From address comes from `primary_email` in `notmuch-config`. Use the usual notmuch keys: `m` writes a new message, `r` and `R` reply, `+` and `-` change tags. In the message buffer, `C-c C-c` sends the message (it runs `agent-mail-send`) and `C-c C-k` cancels it.

CAUTION: Before you send the first message, make sure that `M-: message-send-mail-function` shows `agent-mail-send`. If it shows a different function, Emacs tries to send a real email.
