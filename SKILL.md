---
name: mailbox
description: Read and send threaded messages (tasks, questions, reviews) between agents and the user in the local notmuch mailbox. Use when you start work, when you wait for input, or when you need to tell another role something.
---

# Mailbox

- Run every `notmuch` command with `NOTMUCH_CONFIG=<dir>/notmuch-config`, where `<dir>` is the directory of this `SKILL.md` file. Without it, notmuch can open the personal mail of the user. Never read or change that mail.
- Your start prompt gives your address, `<role>@<team>.agents.local`. Below, `ME` is this full address.
- Send mail only to addresses in your own team domain and to the user, `human@agents.local`.
- Do not use the `unread` tag, because all readers share it. Use your own tag, `seen-ME`.

New mail for you:

    notmuch search 'to:ME and not tag:seen-ME'

After you read a thread:

    notmuch tag +seen-ME -- thread:<id>

Send (the `+seen-ME` tag stops you from reading your own message as new):

    notmuch insert +seen-ME <<EOF
    From: ME
    To: engineer@<team>.agents.local, human@agents.local
    Subject: ...
    Date: $(date -R)
    Message-ID: <$(uuidgen)@agents.local>
    In-Reply-To: <parent-message-id>   (replies only)

    Body.
    EOF

- To start a new task thread, add `+task`. To close a thread, run `notmuch tag +done -- thread:<id>`.
- Reply only when you are in `To:` or when you have new facts.
- If notmuch reports a locked database, wait 1 second and try again.
- To wait for new mail, run `fswatch -1 "$(notmuch config get database.path)/new"`.
