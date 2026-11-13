# ogg_utils — Oracle GoldenGate 26ai Shell Utilities

A lightweight Bash helper library for DBAs who manage **Oracle GoldenGate 26ai (Microservices Architecture)** on Linux.

When you source `.ogg_utils` from your shell profile, you get:

- **`oggenv`**: one command that reads the GoldenGate **deployment registry** (`deploymentRegistry.dat`), lists your deployments, shows their configuration, and loads the correct environment variables for the deployment you choose.
- **Tab completion** for `oggenv`. It completes flags, deployment names, parameter names and registry paths.
- **Helper functions** (`ggac_` and `cdgg`) that open the GoldenGate Admin Client or change to the GoldenGate home for whichever deployment is currently loaded.

---

## Table of Contents

1. [Why this exists](#why-this-exists)
2. [Requirements](#requirements)
3. [Installation](#installation)
4. [Quick start](#quick-start)
5. [Command reference](#command-reference)
6. [How it works](#how-it-works)
   - [Registry discovery](#1-registry-discovery-_ogg_find_registry)
   - [Argument parsing](#2-argument-parsing)
   - [The embedded Python handler](#3-the-embedded-python-handler)
   - [Environment loading](#4-environment-loading-env-mode)
   - [Tab completion](#5-tab-completion-_oggenv_complete)
   - [Helper functions](#6-helper-functions-ggac_-and-cdgg)
   - [Sourcing guard](#7-sourcing-guard)
7. [Examples](#examples)
8. [Environment variables exported](#environment-variables-exported)
9. [Exit codes and error handling](#exit-codes-and-error-handling)
10. [Troubleshooting](#troubleshooting)
11. [Change history](#change-history)
12. [Author](#author)

---

## Why this exists

In GoldenGate Microservices, one **Service Manager** can host several **deployments**. Each deployment has its own home, configuration, variable data, SSL wallet and environment settings (for example, `TNS_ADMIN`, `LD_LIBRARY_PATH` and `JAVA_HOME`). The Service Manager keeps all of this in a JSON file called `deploymentRegistry.dat`.

Without a helper, you have to open that file by hand, find the correct paths and export each variable yourself before you run `adminclient` or other GoldenGate binaries. This is slow and easy to get wrong, especially on servers with several deployments.

`oggenv` makes the registry the **single source of truth**. It reads the registry and configures your shell to match it.

---

## Requirements

| Requirement | Notes |
|---|---|
| Linux | Tested on enterprise Linux distributions (RHEL/OL family) |
| Bash 4.2+ | Needed for `[[ -v VAR ]]` and arrays |
| Python 3 | Must be on the `PATH` as `python3`. Uses only the standard library (`json`, `shlex`, `sys`). |
| Oracle GoldenGate 26ai Microservices | Requires a Service Manager with a valid `deploymentRegistry.dat` |
| Read access to the registry | The OS user (usually `oracle`) needs permission to read `deploymentRegistry.dat` and `/etc/oggInst.loc` |

---

## Installation

1. Copy `ogg_utils.sh` from this repo to the GoldenGate OS user's home directory as a hidden file:

   ```bash
   cp ogg_utils.sh /home/oracle/.ogg_utils
   chmod 644 /home/oracle/.ogg_utils
   ```

2. Source the file from `~/.bashrc`:

   ```bash
   if [[ -f /home/oracle/.ogg_utils ]]; then
       . /home/oracle/.ogg_utils
   fi
   ```

3. Reload your shell:

   ```bash
   source ~/.bashrc
   ```

4. Verify the installation:

   ```bash
   type oggenv          # should report "oggenv is a function"
   complete -p oggenv   # should report "complete -F _oggenv_complete oggenv"
   type ggac_ cdgg      # both should report "is a function"
   ```

> **Note:** You must **source** this file, not run it. `oggenv` exports variables into your **current** shell. If you run the file as a script by mistake (for example, `bash ogg_utils.sh`), it stops with an error that tells you to source it instead. See [Sourcing guard](#7-sourcing-guard).

> **Upgrading from an earlier version:** older versions defined `ggac_` and `cdgg` as aliases. The new file removes those aliases automatically when it is sourced. If your own `~/.bashrc` also defines them, delete those lines.

---

## Quick start

```bash
oggenv                      # Which deployments exist on this host?
oggenv <TAB><TAB>           # ...or let tab completion show you
oggenv ogg_dep01            # Load the environment for ogg_dep01
adminclient                 # Now runs from the right OGG_HOME
ggac_                       # ...or use the helper
cdgg                        # cd to the loaded OGG_HOME
```

---

## Command reference

```text
Usage:
  oggenv                                  List available deployment names
  oggenv <deployment>                     Export environment variables for <deployment>
  oggenv -l, --list                       Show full configuration for all deployments
  oggenv -g, --get <deployment> [param]   Show configuration for <deployment> or raw value of [param]
  oggenv -r, --registry <path>            Specify custom SM home or deploymentRegistry.dat path
  oggenv -h, --help                       Show usage
```

| Option | Mode | Description |
|---|---|---|
| *(none)* + `<deployment>` | `env` | Exports the deployment's environment into the current shell. This is the default mode. |
| *(none)*, no deployment | `env` | Prints the available deployments to stderr and returns `1`, with a hint to run the command again with a name |
| `-l`, `--list` | `list` | Shows a formatted block for **every** deployment. Service Manager is listed first. |
| `-g`, `--get <dep>` | `get` | Shows a formatted block for one deployment |
| `-g`, `--get <dep> <param>` | `get` | Prints **only the raw value** of one parameter, which is useful in scripts |
| `-r`, `--registry <path>` | modifier | Uses a specific registry instead of auto-discovery. Combine it with any mode. |
| `-h`, `--help` | — | Prints usage |

Flags and positional arguments can appear **in any order**. For example, `oggenv ogg_dep01 -r /u01/app/ogg/sm` is the same as `oggenv -r /u01/app/ogg/sm ogg_dep01`.

---

## How it works

### 1. Registry discovery (`_ogg_find_registry`)

This internal helper resolves the path to `deploymentRegistry.dat`. Both `oggenv` and the tab-completion function use it, so they always agree on which registry to read.

It checks these sources in order:

| # | Condition | Result |
|---|---|---|
| 1 | No `-r` given **and** `/etc/oggInst.loc` exists | Reads `inventory_loc=` from `/etc/oggInst.loc` and uses `<inventory_loc>/deploymentRegistry.dat` |
| 2 | `-r` points to a **file** | Uses that file as is |
| 3 | `-r` points to a **Service Manager home** | Uses `<path>/etc/conf/deploymentRegistry.dat` |
| 4 | None of the above | Returns `1`. `oggenv` then prints `Could not locate deploymentRegistry.dat.` |

`/etc/oggInst.loc` is the GoldenGate central inventory pointer, and it typically looks like this:

```text
inventory_loc=/u01/app/oraInventory/ogg
inst_group=oinstall
```

`awk` takes the value, and `xargs` trims any whitespace around it.

### 2. Argument parsing

`oggenv` uses a `while`/`case` loop over `$@`:

- Mode flags (`-l`, `-g`) set `mode`.
- `-r` takes the next argument as the registry path. If that argument is missing, the function fails.
- Unknown flags such as `-x` are rejected.
- Everything else goes into a `positional` array. `positional[0]` is the deployment name, and `positional[1]` is the parameter name (used by `--get`).

All variables are declared `local`, so the function never leaves temporary variables in your shell.

### 3. The embedded Python handler

Bash is poor at parsing JSON, so the function passes the work to an **inline Python 3 script** through a quoted heredoc (`<<'EOF'`). Because the heredoc is quoted, Bash does no expansion inside it. Arguments go to Python as `sys.argv`:

```text
python3 - <mode> <registry_file> <deployment> <param>
```

The Python code:

- Loads the registry JSON.
- Treats every **top-level key whose value is a JSON object** as a deployment. Scalar metadata keys are ignored.
- Sorts deployments with **`ServiceManager` first**, then the others alphabetically, ignoring case.
- Displays a fixed set of keys: `id`, `status`, `enabled`, `oggHome`, `oggEtcHome`, `oggConfHome`, `oggVarHome`, `oggDataHome` and `oggSslHome`. It also displays the deployment's `environment` array, which holds name/value pairs.

How each mode behaves:

- **`list`**: prints a block for each deployment.
- **`get`**: prints one deployment block. If you also give a parameter, it prints only that value. It first looks for the parameter among the top-level keys, then among the `environment` entries. Dictionary and list values are printed as indented JSON.
- **`env`**: prints a series of `export` statements, which Bash then evaluates (see below).

Python writes errors to **stderr** and exits with a non-zero code. Bash captures the exit code (`rc=$?`) and returns it to the caller.

### 4. Environment loading (env mode)

In `env` mode, the Python script prints shell code instead of report text. For example:

```bash
export OGG_DEPLOYMENT=ogg_dep01
export OGG_HOME=/u01/app/ogg/26ai
export OGG_ETC_HOME=/u01/app/ogg/deployments/ogg_dep01/etc
export OGG_CONF_HOME=/u01/app/ogg/deployments/ogg_dep01/etc/conf
export OGG_VAR_HOME=/u01/app/ogg/deployments/ogg_dep01/var
export OGG_DATA_HOME=/u01/app/ogg/deployments/ogg_dep01/var/lib/data
export OGG_SSL_HOME=/u01/app/ogg/deployments/ogg_dep01/etc/ssl
export TNS_ADMIN=/u01/app/oracle/network/admin
export LD_LIBRARY_PATH=/u01/app/ogg/26ai/lib/instantclient:/u01/app/ogg/26ai/lib
export JAVA_HOME=/u01/app/ogg/26ai/jdk
case ":$PATH:" in *":/u01/app/ogg/26ai/bin:"*) ;; *) export PATH="/u01/app/ogg/26ai/bin:$PATH" ;; esac
```

Bash then runs `eval "$output"` in the **current shell**, and the function confirms with:

```text
Environment loaded for deployment: ogg_dep01
```

Safety and design details:

- **`shlex.quote()`** quotes every value. Paths that contain spaces or shell metacharacters are therefore exported literally and never interpreted by the shell during `eval`.
- **Idempotent `PATH` update:** the `case ":$PATH:"` guard prepends `$OGG_HOME/bin` only if it is not already in `PATH`. You can run `oggenv` many times without `PATH` growing.
- Empty registry values are skipped, so existing variables are never overwritten with blanks.

### 5. Tab completion (`_oggenv_complete`)

This function is registered with `complete -F _oggenv_complete oggenv`. It completes based on context:

| Cursor position | Completes |
|---|---|
| After `-r` / `--registry` | File and directory paths |
| Word starts with `-` | `-l --list -g --get -r --registry -h --help` |
| First positional argument | Deployment names read live from the registry |
| Second positional argument, with `-g`/`--get` | Common parameter names: `id status enabled oggHome oggEtcHome oggConfHome oggVarHome oggDataHome oggSslHome TNS_ADMIN LD_LIBRARY_PATH JAVA_HOME` |

The completion function scans the words typed so far. If you already gave a `-r <path>`, it reads deployment names from **that** registry, so completion stays correct on hosts with more than one Service Manager. If something fails, such as a missing registry or bad JSON, it fails quietly and no error text interrupts your prompt.

### 6. Helper functions (`ggac_` and `cdgg`)

| Function | What it does |
|---|---|
| `ggac_ [args]` | Runs `$OGG_HOME/bin/adminclient` and passes any arguments to it |
| `cdgg` | Changes directory to `$OGG_HOME` |

Both are **functions**, not aliases, so they read `$OGG_HOME` **each time they run**. After `oggenv` switches to another deployment, they immediately use the new home, and you don't need to source the file again.

They are always defined, even when `OGG_HOME` is not set at login. If `OGG_HOME` is empty, they print `Error: OGG_HOME is not set. Run 'oggenv <deployment>' first.` and return `1`. `ggac_` also checks that `adminclient` exists and is executable before it runs it.

Before defining them, the file runs `unalias ggac_ cdgg`. This removes aliases from earlier versions, which would otherwise conflict with the function definitions when you re-source the file in an existing shell.

### 7. Sourcing guard

At the top of the file, a check compares `${BASH_SOURCE[0]}` with `$0`. The two values are the same only when the file is **executed**. In that case the script prints the error below and exits with code `1`. It does not quietly set variables in a subshell that are then lost.

```text
$ bash ogg_utils.sh
Error: ogg_utils.sh must be sourced, not executed.
Usage: . ogg_utils.sh
```

The shebang is `#!/usr/bin/env bash`, without `-e`. Bash ignores the shebang when a file is sourced, and `set -e` would make an interactive shell close on any failing command, so the file does not use it.

---

## Examples

The examples below use a sample host with one Service Manager and two deployments. Your hostnames and paths will be different.

### Show the deployments on this host

```text
$ oggenv
Available deployments:
  - ServiceManager
  - ogg_dep01
  - ogg_dep02

Please run the command again and provide a deployment name:
  oggenv <deployment_name>
```

### Load a deployment's environment

```text
$ oggenv ogg_dep01
Environment loaded for deployment: ogg_dep01

$ echo $OGG_DEPLOYMENT $OGG_HOME
ogg_dep01 /u01/app/ogg/26ai

$ which adminclient
/u01/app/ogg/26ai/bin/adminclient
```

### Use tab completion

```text
$ oggenv <TAB><TAB>
ServiceManager  ogg_dep01  ogg_dep02

$ oggenv ogg_d<TAB>
$ oggenv ogg_dep0

$ oggenv --get ogg_dep01 ogg<TAB><TAB>
oggConfHome  oggDataHome  oggEtcHome  oggHome  oggSslHome  oggVarHome
```

### Show the full configuration of every deployment

```text
$ oggenv --list
Reading registry: /u01/app/oraInventory/ogg/deploymentRegistry.dat
================================================================================
Deployment   : ServiceManager
  ID         : 5b0f3c2e-xxxx-xxxx-xxxx-xxxxxxxxxxxx
  Status     : running
  Enabled    : True
  oggHome    : /u01/app/ogg/26ai
  oggEtcHome : /u01/app/ogg/sm/etc
  oggConfHome: /u01/app/ogg/sm/etc/conf
  oggVarHome : /u01/app/ogg/sm/var
  oggDataHome: /u01/app/ogg/sm/var/lib/data
  oggSslHome : /u01/app/ogg/sm/etc/ssl
--------------------------------------------------------------------------------
Deployment   : ogg_dep01
  ID         : 9a7d1e44-xxxx-xxxx-xxxx-xxxxxxxxxxxx
  Status     : running
  Enabled    : True
  oggHome    : /u01/app/ogg/26ai
  ...
  Environment:
    - TNS_ADMIN=/u01/app/oracle/network/admin
    - LD_LIBRARY_PATH=/u01/app/ogg/26ai/lib/instantclient:/u01/app/ogg/26ai/lib
    - JAVA_HOME=/u01/app/ogg/26ai/jdk
--------------------------------------------------------------------------------
...
```

### Show a single deployment

```bash
oggenv -g ogg_dep02
```

### Get one raw value (for scripts)

`--get` with a parameter prints **only the value**, with no headers. You can use it directly in command substitution:

```bash
$ oggenv -g ogg_dep01 status
running

$ oggenv -g ogg_dep01 TNS_ADMIN        # environment[] entries work too
/u01/app/oracle/network/admin

# Use it in a script
VAR_HOME=$(oggenv -g ogg_dep01 oggVarHome) || exit 1
tail -f "$VAR_HOME/log/ER-events.log"

# Check every deployment's status
for d in ogg_dep01 ogg_dep02; do
    printf '%-12s %s\n' "$d" "$(oggenv -g "$d" status)"
done
```

### Use a specific registry (multiple Service Managers, or no `/etc/oggInst.loc`)

```bash
# Point to a Service Manager home
oggenv -r /u01/app/ogg/sm ogg_dep01

# ...or point directly to the registry file
oggenv --registry /u01/app/ogg/sm/etc/conf/deploymentRegistry.dat --list

# Flag order does not matter
oggenv ogg_dep01 -r /u01/app/ogg/sm
```

### Switch between deployments in one session

```bash
oggenv ogg_dep01
adminclient          # work on dep01
oggenv ogg_dep02     # variables are overwritten; PATH is not duplicated
adminclient          # work on dep02
```

### Use the helper functions

```text
$ ggac_
Error: OGG_HOME is not set. Run 'oggenv <deployment>' first.

$ oggenv ogg_dep01
Environment loaded for deployment: ogg_dep01
$ cdgg && pwd
/u01/app/ogg/26ai
$ ggac_              # starts adminclient from ogg_dep01's OGG_HOME

$ oggenv ogg_dep02   # switch deployments; the helpers follow automatically
$ ggac_              # now starts adminclient from ogg_dep02's OGG_HOME
```

### Error handling in automation

```bash
if ! oggenv ogg_dep01 >/dev/null; then
    echo "Failed to load GoldenGate environment" >&2
    exit 1
fi
```

---

## Environment variables exported

| Variable | Source in registry |
|---|---|
| `OGG_DEPLOYMENT` | The deployment name you passed |
| `OGG_HOME` | `oggHome` |
| `OGG_ETC_HOME` | `oggEtcHome` |
| `OGG_CONF_HOME` | `oggConfHome` |
| `OGG_VAR_HOME` | `oggVarHome` |
| `OGG_DATA_HOME` | `oggDataHome` |
| `OGG_SSL_HOME` | `oggSslHome` |
| *(any name)* | Every `name`/`value` pair in the deployment's `environment` array, for example `TNS_ADMIN`, `LD_LIBRARY_PATH` and `JAVA_HOME` |
| `PATH` | `$OGG_HOME/bin` is prepended if it is not already present |

---

## Exit codes and error handling

| Situation | Message (stderr) | Return code |
|---|---|---|
| Success | — | `0` |
| Registry not found | `Error: Could not locate deploymentRegistry.dat.` | `1` |
| `-r` given without a path | `Error: --registry requires a path argument.` | `1` |
| Unknown flag | `Error: Unknown option '-x'. Use --help for usage.` | `1` |
| No deployment given (`env` / `--get`) | List of available deployments, plus a hint | `1` |
| Deployment does not exist | `Error: Deployment '<name>' not found in <file>.` | `1` |
| Parameter does not exist | `Error: Parameter '<param>' not found for '<name>'.` | `1` |

On any error in `env` mode, **nothing is exported**. The `eval` step runs only when the Python handler succeeds.

---

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `Could not locate deploymentRegistry.dat` | `/etc/oggInst.loc` is missing or points somewhere else. Use `-r <SM home>`, and check `inventory_loc` in `/etc/oggInst.loc`. |
| `python3: command not found` | Install Python 3, or make sure it is on the `PATH` for the GoldenGate user |
| Python `PermissionError` traceback | The OS user cannot read the registry. Check file ownership and permissions. |
| Python `JSONDecodeError` traceback | The registry is corrupt or partly written. Check it with `python3 -m json.tool <file>`. |
| Tab completion does nothing | Run `complete -p oggenv` to confirm it is registered. Make sure the file was **sourced** and the shell is interactive Bash. |
| `ggac_` / `cdgg` print `OGG_HOME is not set` | Load a deployment first: `oggenv <dep>` |
| `ggac_` prints `adminclient not found or not executable` | The deployment's `oggHome` in the registry is wrong, or the binary is missing. Check it with `oggenv -g <dep> oggHome`. |
| `must be sourced, not executed` | You ran the file. Use `. ~/.ogg_utils` instead. |
| `ggac_`/`cdgg` still behave the old way | An old alias is defined *after* this file in your profile and overrides the function. Check with `type ggac_`, then remove those alias lines from `~/.bashrc`. |

---

## Change history

| Date | Initials | Note |
|---|---|---|
| 2026-10-08 | pb | Initial release; includes the basic `ogg_list_deployments` |
| 2026-11-12 | pb | Replaced `ogg_list_deployment` with `oggenv` |
| 2026-11-13 | pb | Added Tab-key auto-completion to `oggenv` |
| 2026-11-13 | pb | Fixed the header `.bashrc` example path; removed the ineffective `-e` from the shebang; added a guard against running the file directly; corrected the `oggenv` description; converted the `ggac_`/`cdgg` aliases to functions that resolve `$OGG_HOME` at run time |

---

## Author

**Parsa Bahrami** (pb)
