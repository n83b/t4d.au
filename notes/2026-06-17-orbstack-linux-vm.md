# Using OrbStack's Linux VM on macOS

[OrbStack](https://orbstack.dev) is a fast, lightweight native Mac app for running Linux machines and Docker containers. Unlike the alternatives, it is built entirely in Swift — no Electron, no Java, no cross-platform cruft — so it feels right at home on macOS. It launches in under two seconds, integrates cleanly with the macOS filesystem, and sips battery compared to Docker Desktop.

## Starting the Linux Machine

When OrbStack is running you get a default Debian-based Linux machine named `orb`. You can open a full shell into it directly from Terminal:

```bash
ssh orb
```

Or use the `orb` CLI to drop into a specific machine or distribution:

```bash
# List all machines
orb list

# Start an interactive shell in the default machine
orb shell

# Start a shell in a named machine
orb shell my-ubuntu
```

## The `orb` Command — Running Linux on Mac Files

The real magic is the `orb run` command. It lets you execute a Linux binary against files that live on your **Mac filesystem**, without copying anything across. OrbStack mounts your home directory into the VM automatically at `/Users/<you>`, so paths just work.

```bash
# Run a Linux command against a Mac file — no copying needed
orb run grep -rn "TODO" ~/Sites/t4d.au

# Use a Linux build tool on a local project directory
orb run make -C ~/Projects/my-c-project

# Run a specific distro's toolchain against a Mac path
orb run -m ubuntu:24.04 apt list --installed
```

The pattern is simply:

```
orb run [flags] <linux-command> [args…]
```

Flags worth knowing:

| Flag | Purpose |
| :--- | :--- |
| `-m <machine>` | Target a specific machine or image (e.g. `ubuntu:24.04`) |
| `-u <user>` | Run as a specific user inside the VM |
| `--` | Explicitly separate `orb` flags from the command being run |

## Practical Examples

```bash
# Check a file's line endings with Linux `file`
orb run file ~/Downloads/mystery-script.sh

# Use Linux `stat` (different output to macOS stat)
orb run stat ~/Sites/t4d.au/index.html

# Run a quick Python script using the VM's Python
orb run python3 ~/Scripts/parse-logs.py

# Pipe output back to macOS commands as normal
orb run cat /etc/os-release | grep PRETTY_NAME
```

Because the filesystem is shared over a high-performance virtio-fs layer, reads and writes go directly to your Mac disk — there is no sync step and no stale copies to worry about.

## Managing Machines

```bash
# Create a new named machine running Ubuntu 24.04
orb create ubuntu:24.04 my-ubuntu

# Stop a machine (it resumes instantly next time)
orb stop my-ubuntu

# Delete a machine entirely
orb delete my-ubuntu
```

## Why OrbStack

OrbStack is the tool I reach for when I need a genuine Linux environment without leaving the Mac workflow. It is a native Swift app, integrates with macOS networking and DNS out of the box, and the `orb run` command in particular makes it trivially easy to validate scripts and use Linux-specific tooling against real project files.
