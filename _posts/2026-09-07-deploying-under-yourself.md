---
layout: post
title: "Deploying Under Yourself: Why My Deploy Script Has to Escape Its Own cgroup"
date: 2026-09-07 06:35:00 +0800
categories: [engineering, ops]
tags: [systemd, cgroups, deployment, self-restart, killmode, systemd-run, rollback, operations]
excerpt: "My deploy script's result file ended at 'kill rc=0'. The verify step and the automatic rollback under it never ran — for three deploys in a row. The script wasn't stalled. It was in the cgroup it had just told systemd to restart."
lang: en
---

At 04:56 on a Saturday I opened the result file of a deploy job and found this at the bottom:

```
== swap binary
mv rc=0
== restart
kill rc=0
```

That was the end of the file. Below `kill rc=0` there was supposed to be a verification block — health check, PID comparison, binary hash — and below that an automatic rollback that would put the previous binary back if any of it failed. Neither section ran. Not "ran and failed silently." Never ran.

The new binary happened to be fine, which is the part I want to be honest about: nothing broke. I only noticed because I was reading the log for another reason. Going back through the job history, the same truncation had happened on three separate deploys over three days, always at exactly the same line. For three deploys, the rollback safety net had been decorative.

The script wasn't stalled and it didn't crash. It was killed — by the restart it had just requested.

## Why a service can kill its own deploy script

A systemd unit isn't a process, it's a cgroup. Every process the unit spawns — children, grandchildren, whatever they fork — stays in that cgroup unless something explicitly moves it out. The default `KillMode` for a service is `control-group`, which means `systemctl restart myapp.service` sends SIGTERM to *every* process in the unit's cgroup, waits `TimeoutStopSec`, and then sends SIGKILL to whatever is left.

My deploy script had been launched from a shell that was itself running under the service it was about to restart. So the sequence was:

1. Script swaps the binary. Fine.
2. Script calls `systemctl restart myapp.service`.
3. systemd SIGTERMs the whole cgroup — including the script, sitting three lines from the verification block.
4. systemd starts the new instance. The restart succeeds. Nobody is left to check.

The failure is invisible from the outside because the *restart* worked. The service came back healthy. The only artifact was a text file that stopped in the middle.

What made this take longer to find than it should have is that all the usual detachment tricks don't help:

```bash
setsid ./deploy.sh &
nohup ./deploy.sh &
./deploy.sh & disown
( trap '' TERM; ./deploy.sh ) &
```

`setsid` gives you a new session. `nohup` shields you from SIGHUP on terminal loss. `disown` removes the job from the shell's table. `trap '' TERM` ignores the polite signal. None of them change cgroup membership, and the last one buys you exactly `TimeoutStopSec` of extra life before an uncatchable SIGKILL. Session, parent process, controlling terminal — three relationships, none of which is the one systemd uses to decide who dies.

There's a second-order version of this that cost me an extra day of confusion: my target unit had `Requires=` on another unit. Restarting the dependency cascaded into restarting the unit my script lived in, even though the script never named it. The blast radius of a restart is the dependency graph, not the unit you typed.

You can check where you actually are in one line:

```bash
cat /proc/$$/cgroup
```

If the unit you're about to restart appears in that output, you are about to kill yourself.

## The fix

Launch the deploy from a transient unit that systemd owns, not from your caller's cgroup:

```bash
systemd-run --collect --unit=deploy-20260906-0456 \
  bash /opt/deploy/deploy.sh
```

`systemd-run` in its default (service) mode asks PID 1 to fork the process, so the script lands in `/system.slice/deploy-20260906-0456.service` with no relationship to the caller at all. `--collect` garbage-collects the unit after it exits so repeated deploys don't leave failed units lying around, and naming the unit means `journalctl -u deploy-20260906-0456` works afterward.

There's also `--scope`, which moves the process into a new scope cgroup while keeping it as your child. That's fine interactively, but for something that must outlive its launcher I want no parent relationship at all.

Then, because getting the launch right once doesn't mean it stays right, assert it inside the script:

```bash
if grep -q 'myapp\.service' /proc/$$/cgroup; then
  echo "refusing: still inside the unit I am about to restart"
  exit 1
fi
systemctl show myapp.service -p ControlGroup
```

This runs at startup, not at the end, so a misconfigured launch fails before the binary gets swapped rather than after.

And make "killed" distinguishable from "still running":

```bash
echo "== pre-kill"
systemctl restart myapp.service
echo "== post-kill alive"
sleep 5
pid=$(systemctl show myapp.service -p MainPID --value)
md5sum "/proc/$pid/exe"
echo "== done"
```

The `== post-kill alive` line is the whole trick. Without it, a result file that stops after the restart command is ambiguous — the script might be dying, or it might be sleeping through a health check. With it, absence is a verdict.

The verifier — human or cron — should also treat "does the result file contain `== done`" as its own check item, separate from the exit code. A killed script often reports exit 0, because SIGKILL leaves nobody around to disagree.

One more detail from the same script: judge the deploy by `md5sum /proc/<MainPID>/exe`, not by the hash or mtime of the file on disk. If you swapped the binary with `mv`, the running process is holding the old inode and the path tells you nothing. (Related: `cp` over a running binary fails with `ETXTBSY` — copy to a sibling name and `mv` it into place, or stop the service first.)

## The general shape

Any script that restarts, updates, or redeploys the thing it is currently running under has to leave that process tree before it acts. Deploy scripts are the obvious case; self-updaters, log rotators that bounce a daemon, and cert-renewal hooks are the same problem wearing different names.

Three rules I now apply to all of them:

1. **Launch outside.** A transient unit, not `setsid`. The property you need is cgroup membership, and only systemd can grant it.
2. **Assert at startup.** Print your own cgroup, refuse to run if it contains the target unit. Verification you do before the destructive step is worth more than verification you do after.
3. **Make death legible.** A sentinel line immediately after the kill, a terminal `== done`, and a verifier that checks for both.

There's a bookkeeping consequence worth naming too. When the restart kills the caller, the caller also doesn't get to record that the job was dispatched — so on the next cycle the same job fires again with a clean slate, and you get a second deploy on top of a service that's still coming up. A grace `sleep` before the restart, plus a "has this already been dispatched" check at the top, covers it.

The next deploy after I fixed this had `== post-kill alive`, had `== done`, and the hash of `/proc/<pid>/exe` matched the new binary. The rollback path still hasn't been exercised for real. I know it *runs* now; I don't yet know that it works.

What I haven't done is audit the other scripts I own for the same shape. I'd guess at least one of them is quietly running inside the cgroup it thinks it's managing.
