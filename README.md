# Agent mail

This is a local mailbox for you and your Claude Code agents. Each message is stored as a single email file in a Maildir. Replies are grouped into threads, and a thread can also work as a task.

There is no mail server. Messages are added directly to the Maildir with `notmuch insert`, and `notmuch` provides the threading, search, and tagging. Each person has their own separate mailbox on their own machine.

## Files

```
| Path                     | What it is                                                     |
|--------------------------|----------------------------------------------------------------|
| `SKILL.md`               | The Claude Code skill that tells agents how to use the mailbox |
| `notmuch-config.example` | The template for your local `notmuch-config`                   |
| `notmuch-config`         | Your notmuch configuration for this mailbox (not in git)       |
| `messages/`              | The Maildir and the `.notmuch` index (not in git)              |
```

In the commands below, `<repo>` is the directory of your clone of this repostiory, for example `~/code/agent-mail`.

## Install

1. Install the required tools:

       brew install notmuch fswatch

   On Linux, use `inotifywait -e create` from `inotify-tools` instead of `fswatch`.

2. Copy and update the notmuch configuration template:

       cp <repo>/notmuch-config.example <repo>/notmuch-config

   Edit the `notmuch-config` to set `path` to `<repo>/messages` and other parameters (see the file for more information).

3. Create the Maildir and the index:

       mkdir -p <repo>/messages/{cur,new,tmp}
       NOTMUCH_CONFIG=<repo>/notmuch-config notmuch new

4. Make the skill available to Claude Code in all projects:

       ln -s <repo> ~/.claude/skills/mailbox

   You can skip this, but then you need to let the agent know about the skill explicitly.

## Conventions

Conventions:

- **Teams/Tasks:** A team is a group of agents that work together on one task, for example `task-1` means a "team of agents working on the task-1".
  - Each team has its own domain, `<team>.agents.local`.
  - You do not need to manually set up a new team, it works automatically when the first message is sent to its `<team>.agents.local` domain.
- **Addresses:** Agent addresses have the format `<role>@<team>.agents.local`, for example `engineer@task-1.agents.local` and `tester@task-1.agents.local`.
- **Your address:** You have one address for all teams/tasks: `human@agents.local`.
- **Tags:**
  - `task`: marks the first message of a task thread.
  - `done`: marks a task as complete.
  - `seen-<address>`: shows that this reader has read the message, for example `seen-engineer@task-1.agents.local`. Each email address has a separate "seen" tag.

## Start an agent

Let the agent know what their email address is in the prompt, for example:

    Use the mailbox skill, your email is tester@task-1.agents.local. Read your mail, do the work, and reply with results.

The skill explains to the agent how to use `fswatch` to watch for the email. Claude Code has `Monitor` tool that can run this command.

## How to access messages. Method 1: Use the Command Line

Set the configuration in your shell first:

    export NOTMUCH_CONFIG=<repo>/notmuch-config

Common commands:

    notmuch search '*'                                    # all threads
    notmuch search tag:task and not tag:done              # open tasks
    notmuch search 'to:human@agents.local and not tag:seen-human@agents.local'
    notmuch search 'to:task-1.agents.local or from:task-1.agents.local'  # one team
    notmuch show --entire-thread=true thread:<id>         # read a thread
    notmuch tag +seen-human@agents.local -- thread:<id>   # mark as read
    notmuch tag +done -- thread:<id>                      # close a task

To send a message from the command line, send the full message text as the input to `notmuch insert`. See the example of the message in the `SKILL.md`.

## How to access messages. Method 1: Use Emacs

Install the [notmuch emacs](https://notmuchmail.org/notmuch-emacs/) package and use the extra configuration below to access agent messages.

If you also use notmuch for your personal mail, run a separate Emacs instance to access agent communcation like this:

    NOTMUCH_CONFIG=<repo>/notmuch-config emacs --no-desktop -f notmuch &

Settings below in the Emacs init file help setting up notmuch to work with agent mail:

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

The From address is the same as `primary_email` in `notmuch-config`.

To manage mail, use the usual notmuch keys:
* `m` to write a new message
* `r` and `R` to reply
* `+` and `-` to change tags.

In the message buffer, `C-c C-c` sends the message (`M-x agent-mail-send`) and `C-c C-k` cancels sending.

Before you send the first message, it is good to make sure that `M-: message-send-mail-function` returns `agent-mail-send`.
If it shows a different function, Emacs will try to send a real email.
