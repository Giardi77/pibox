# PiBox

The idea is simple: isolate the agent in a container, limiting the danger of a prompt injection or an hallucination.

## Dependencies

```bash
brew install lima
```

## What it does

PiBox gives you throwaway Linux VMs for running coding agents on untrusted work. Each VM is a one-shot: you copy a workspace in, you work, you pull the keepers out, you destroy the VM.

To keep spin-up fast, pibox builds a **base image** from a template once (apt packages, bun, pi, whatever else) and then clones it for every engagement. First engagement pays the provisioning cost; every subsequent one is ~15 seconds.

Commands:

- `pibox rebuild [-t <template>]` — build or rebuild a base image from a template (default: `default`)
- `pibox new <name> [src-dir] [-t <template>]` — clone a base into a fresh VM and copy `src-dir` (default: current dir) into `~/workspace`. Auto-builds the base on first use.
- `pibox start <name>` — start or resume an existing engagement VM after it was stopped
- `pibox stop <name>` — stop an existing engagement VM without deleting it
- `pibox attach <name>` or `pibox a <name>` — start the VM if needed, then SSH in and land in the workspace
- `pibox commit <name> [dest]` — start the VM if needed, then copy `~/workspace/persistent/` out to your Mac; VM keeps running
- `pibox delete <name> [dest]` or `pibox d <name> [dest]` — start the VM if needed, do a final copy of `persistent/`, then destroy the VM
- `pibox status` — show all templates, which have built bases, disk sizes, tool versions
- `pibox sync-pi <name>` — start the VM if needed, then copy `~/.pi/` from Mac into it
- `pibox list` or `pibox l` — show all VMs

The persistence rule is the whole point: **only files under `~/workspace/persistent/` survive the VM**. Everything else dies with it. This is the boundary that makes prompt injection cheap to recover from — whatever the agent did, you throw the VM away and you're clean.

SSH hosts are registered automatically. After `pibox new test`, `ssh test` just works.

## Why Lima, not OrbStack or Docker Desktop

OrbStack runs every container and Linux machine in **one shared VM with a shared kernel**. Its own docs say the VM isn't used as a strong security boundary. A container escape lands you next to every other container you're running, plus whatever host paths are shared in. Great for dev work, wrong shape for this.

Docker Desktop is similar — one VM, many containers, shared state.

Lima gives you **one real VM per instance**, isolated from each other and from your Mac. `pibox delete` destroys that VM entirely. Next engagement starts from zero.

## The flow

```bash
cd ~/projects/engagement-42
pibox new engagement-42         # auto-builds 'default' base first time (~5 min), clones it (~15s)
ssh engagement-42               # or: pibox attach engagement-42 / pibox a engagement-42
# ... run the agent, save keepers under ~/workspace/persistent/ ...
pibox commit engagement-42      # optional: checkpoint to Mac
pibox stop engagement-42        # pause without destroying the workspace VM
# later, after a Mac shutdown or any stop:
pibox start engagement-42       # resume the same workspace VM
pibox delete engagement-42      # final pull + destroy
```

Anything the agent produced that isn't under `persistent/` is gone after `delete`. By design.

## Templates and bases

Templates live in `~/.pibox/templates/` as YAML files. Each template produces its own base image named `pibox-base-<template>`. The default template is `default.yaml`; the default base is `pibox-base-default`.

```
~/.pibox/
├── templates/
│   ├── default.yaml       # general-purpose
│   ├── web.yaml           # web pentest tools
│   └── net.yaml           # network pentest tools
└── container/
    ├── default/           # metadata for pibox-base-default
    │   ├── built-at
    │   └── manifest.txt
    └── web/
        └── ...
```

```bash
# rebuild the default base after changing default.yaml
pibox rebuild

# build a specific template's base
pibox rebuild -t web

# use a specific template for an engagement
pibox new engagement-43 -t web

# see what's built and what's in each base
pibox status
```

The base is stopped automatically after build — it only runs long enough to provision itself, then sits on disk as a clone source. Clones are cheap (copy-on-write on APFS).

## Customization

**Installed tooling.** Edit `~/.pibox/templates/<name>.yaml`. Two provisioning modes:

- `mode: system` — runs as root. Use for `apt install`, system services, anything needing sudo.
- `mode: user` — runs as your VM user. Use for `pipx install`, shell config, agent installs, anything under `$HOME`.

Add your coding agent in the user block. Keep the template fat if you want — disk is thin-provisioned.

After editing a template, `pibox rebuild -t <template>` to apply the changes. Existing VMs keep running on the old base; new ones use the new base.

**Resources.** Set per invocation via env vars, or change the defaults in the template itself.

```bash
PIBOX_CPUS=12 PIBOX_MEMORY=48 PIBOX_DISK=400 pibox rebuild -t big
```

Defaults: 8 CPUs, 16 GiB RAM, 200 GiB disk (thin-provisioned).

**Paths and behavior.** Override via env vars:

```
PIBOX_HOME             pibox state dir            (default: ~/.pibox)
PIBOX_PERSIST_ROOT     where commits go           (default: ~/pibox-output)
PIBOX_SSH_CONFIG_DIR   where SSH host files live  (default: ~/.ssh/pibox.config)
PIBOX_CPUS             VM CPUs                    (default: 8)
PIBOX_MEMORY           VM memory in GiB           (default: 16)
PIBOX_DISK             VM disk in GiB             (default: 200)
PIBOX_SKIP_PI_SYNC     set to skip ~/.pi sync on 'new'
```

**Pi config sync.** `pibox new` automatically tars `~/.pi/` from your Mac into the VM (skills, extensions, themes, prompt templates — session history is excluded). Disable with `PIBOX_SKIP_PI_SYNC=1` if you want a clean pi config in the VM. Sessions and logs are never synced.

## Talking to services on the host

Sometimes the agent inside needs to reach something on your Mac — an MCP server, a local API, a proxy. Lima exposes your host at `host.lima.internal` from inside the VM.

```bash
# on your Mac
my-service --host 127.0.0.1 --port 8765

# from inside the VM
curl http://host.lima.internal:8765/
```

**Bind to `127.0.0.1`, never `0.0.0.0`.** Lima handles the VM-to-host bridge. You don't want this service exposed to your LAN.

This punches a hole in the isolation boundary on purpose, so be honest about what's on the other side. A read-only docs server is fine. An MCP that can send email as you or read `~/.ssh/` defeats the sandbox — a prompt-injected agent will happily call every tool that's exposed. When in doubt, run the service **inside** the VM instead. It's almost always the right answer for filesystem, code execution, or scratch-database MCPs.

If you do expose something on the host: put auth on it (even a static bearer token), scope macOS firewall rules to the Lima subnet, and log requests. Assume every call might be coming from an injected prompt.

## Where things live

```
~/.pibox/templates/        # YAML templates you edit
~/.pibox/container/<t>/    # metadata about each built base
~/.lima/pibox-base-<t>/    # the base VMs (stopped, used as clone sources)
~/.lima/<name>/            # engagement VMs (running)
~/pibox-output/            # pulled persistent/ contents (your deliverables)
~/.ssh/pibox.config/hosts  # auto-managed SSH entries
```

## Initial setup

```bash
brew install lima
mkdir -p ~/.pibox/templates
# drop your template as ~/.pibox/templates/default.yaml
sudo install -m 755 pibox /usr/local/bin/pibox
```

The script will add `Include pibox.config/*` to `~/.ssh/config` the first time it runs. If the file doesn't exist, it creates it.

On first `pibox new`, if no base exists for the requested template, pibox auto-runs `rebuild -t <template>` before cloning.

## What it doesn't do

- No automatic cleanup of `~/pibox-output/`. Old commits sit there until you delete them.
- No live sync between host and VM. Edits on the Mac during an engagement don't appear in the VM — that's the point. If you need to bring in new files mid-engagement, `scp` them over or start a fresh VM.
- No host mounts. The template explicitly disables them. If you find yourself wanting to mount something, reconsider — you're weakening the boundary you set up this tool to have.
- No sharing of bases across machines. The base VMs are Lima instances, not portable disk images. If you want that, a `packer`/`virt-customize` build pipeline producing a shared `.img` file is the next step up — not currently built in.
