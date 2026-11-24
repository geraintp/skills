---
name: plannotator-docker-sandbox
description: Launching Plannotator inside a Docker Sandbox (`IS_SANDBOX=1`) so the host browser can open it on a stable published port. Use before starting any Plannotator session here (annotate, review, last, or a plan review on exiting plan mode), alongside the other plannotator skills, and when the host cannot open a Plannotator URL.
---

# Plannotator in a Docker Sandbox

Plannotator runs inside this sandbox (`$SANDBOX_NAME`), and the human opens it on the host at `http://localhost:<port>`. For that to work, the session must listen on `0.0.0.0` on a **published port**, meaning one in the range 19432–19435 (see [Host side](#host-side)). A session bound to `127.0.0.1` is unreachable from the host. Plannotator binds there by default, so every launch needs this skill.

The `plannotator` skill decides which command to run and how to wait on it. This skill decides how the session listens.

**Local links only.** The only link you give the user is the session's `http://localhost:<port>` URL. Plannotator sessions stay on this machine, so never create `share.plannotator.ai` links. That covers the share link remote mode prints, `plannotator guide share`, and the UI's share action. If the user wants a Guided Review as a file, use `plannotator guide export`.

## Steps

1. **Configure the sandbox once.** Sandbox shells source `/etc/sandbox-persistent.sh`, so variables set there also cover plan reviews that hooks launch:
   - `PLANNOTATOR_REMOTE=1` makes Plannotator bind `0.0.0.0` on 19432, or on `PLANNOTATOR_PORT` if that is set.
   - `PLANNOTATOR_SHARE=disabled` turns off share links, including the one remote mode prints.

   ```bash
   f=/etc/sandbox-persistent.sh
   grep -q PLANNOTATOR_REMOTE "$f" || echo 'export PLANNOTATOR_REMOTE=1' >> "$f"
   grep -q PLANNOTATOR_SHARE "$f" || echo 'export PLANNOTATOR_SHARE=disabled' >> "$f"
   ```

   This step is done when `bash -c 'echo $PLANNOTATOR_REMOTE $PLANNOTATOR_SHARE'` prints `1 disabled`.

   A hook-launched plan review has no `PLANNOTATOR_PORT`, so it always takes 19432. Close any manual session on 19432 before you exit plan mode.

2. **Pick the port**: the first published port with nothing listening on it.

   ```bash
   for p in 19432 19433 19434 19435; do
     (exec 3<>/dev/tcp/127.0.0.1/$p) 2>/dev/null || { export PLANNOTATOR_PORT=$p; break; }
   done
   ```

   This gives 19432 unless another session already holds it, so the URL stays the same between runs. If `PLANNOTATOR_PORT` is still unset, every published port is busy. Show the user `plannotator sessions` and ask them to close a session, or to run `sbx port publish <sandbox> 19436:19436` on the host. Write the actual value of `$SANDBOX_NAME` in place of `<sandbox>`. Once they confirm, launch on 19436.

3. **Launch.** Put the remote-mode, share and port variables in front of the command the `plannotator` skill chose:

   ```bash
   PLANNOTATOR_REMOTE=1 PLANNOTATOR_SHARE=disabled PLANNOTATOR_PORT=$PLANNOTATOR_PORT plannotator <command> ...
   ```

   This step is done when stderr prints `Plannotator session ready: http://localhost:<port>`. If stderr says `port <n> is already in use`, another session took the port after step 2 checked it, so go back to step 2. Plannotator exits on a port collision. It does not move to another port by itself.

   A `plannotator` tool does not accept a port. When you launch through it, skip straight to step 4 and check port 19432.

4. **Verify the bind**: run this while the session is still running.

   ```bash
   grep -E "^ *[0-9]+: [0-9A-F]{8}:$(printf '%04X' $PLANNOTATOR_PORT) .* 0A " /proc/net/tcp | awk '{print $2}'
   ```

   - `00000000:…` means `0.0.0.0`. Give the user the printed `http://localhost:<port>` URL.
   - `0100007F:…` means `127.0.0.1`: remote mode did not apply. Close the session and relaunch it through the [proxy fallback](#proxy-fallback).

## Host side

`sbx` runs only on the host. From inside the sandbox you cannot publish or list ports, so treat 19432–19435 as the only ports the host can reach. The user publishes them once, each on the same port number on the host, and the mapping survives sandbox restarts:

```bash
sbx port publish <sandbox> 19432:19432 19433:19433 19434:19434 19435:19435
```

## Proxy fallback

Use this when Plannotator will only bind `127.0.0.1`. Run Plannotator on an internal port, 1000 above the picked port, and forward the picked port to it:

```bash
PLANNOTATOR_SHARE=disabled PLANNOTATOR_PORT=$((p + 1000)) plannotator <command> ... &
socat TCP-LISTEN:$p,bind=0.0.0.0,fork,reuseaddr TCP:127.0.0.1:$((p + 1000)) &
```

Give the user `http://localhost:$p`. Kill the `socat` process when the session exits.
