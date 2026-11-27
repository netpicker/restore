# Netpicker Restore YAML – File Transfer Definitions

Restore YAML files tell Kibbitzer how to copy a file **to** a network device and check that it arrived intact. They are used when an automation job calls `device.write_file()`, for example to stage a firmware image. Copying an image does not activate it or reload the device.

> **Out of scope:** Transferium installation and networking. Netpicker's *backup* YAML (root key `protocols`) is a different format and is not covered here.

## Contents

1. [Platforms](#1-platforms)
2. [File structure](#2-file-structure)
3. [Command entries](#3-command-entries)
4. [Prompts](#4-prompts)
5. [Methods and timeouts](#5-methods-and-timeouts)
6. [Placeholders](#6-placeholders)
7. [Verification](#7-verification)
8. [Errors](#8-errors)
9. [Creating or changing a definition](#9-creating-or-changing-a-definition)
10. [Troubleshooting](#10-troubleshooting)
11. [Open questions](#11-open-questions)

**Required / Optional** in the tables below reflects how the shipped files are built. Parser-enforced rules have not been confirmed yet (see [§11](#11-open-questions)).

## 1. Platforms

| Platform | File | Protocols | Verify command |
| --- | --- | --- | --- |
| Cisco IOS | [cisco_ios.yml](cisco_ios.yml) | SCP, TFTP | `verify /md5 {dst_file}` |
| Cisco IOS XE | [cisco_xe.yml](cisco_xe.yml) | SCP, TFTP | `verify /md5 {dst_file}` |
| Cisco NX-OS | [cisco_nxos.yml](cisco_nxos.yml) | SCP | `show file {dst_file} md5sum` |
| Cisco IOS XR | [cisco_xr.yml](cisco_xr.yml) | SCP, TFTP | `show md5 file {dst_file}` |
| Arista EOS | [arista_eos.yml](arista_eos.yml) | SCP | `verify /md5 {dst_file}` |
| Juniper Junos | [juniper_junos.yml](juniper_junos.yml) | SCP, TFTP | `file checksum md5 {dst_file}` |
| Westermo OS | [westermo_os.yml](westermo_os.yml) | SCP | `file cfg`, then `dir` (MD5 matched in output) |

> `cisco_ios.yml` and `cisco_xe.yml` are identical. Changing one does not change the other.

## 2. File structure

```yaml
.transfer-prompts: &prompts   # shared prompt answers
verify:                        # integrity check
transfers:                     # one command list per protocol
  scp:
  tftp:
errors:                        # output pattern → exception
```

| Section | Required? | Purpose |
| --- | --- | --- |
| `transfers` | **Required** | Command list per protocol (`scp`, `tftp`) |
| `verify` | **Required** for an integrity check (present in every shipped file) | Checks the copied file against `{md5}` |
| `errors` | Recommended | Maps device output to exceptions |
| `.transfer-prompts` | Optional | Prompt answers reused with a YAML merge |

## 3. Command entries

Each item under `transfers.<protocol>` or `verify` is a command entry. Entries in a list run in order.

| Parameter | Required? | Purpose |
| --- | --- | --- |
| `command` | **Required** | CLI command to send (`""` sends Enter) |
| `method` | Optional | How the command runs (see [§5](#5-methods-and-timeouts)) |
| `kwargs` | Optional | Settings for the method, such as timeouts and patterns |
| `prompts` | Conditional: needed when the device asks a question | Output pattern → response |

**Example** (`cisco_ios.yml`, TFTP) uses all four parameters:

```yaml
.transfer-prompts: &prompts
  "Destination file": ""
  "Destination filename": ""
  "file already existing": ""
  "Do you want to overwrite": ""
  "Do you want to over\\s?write": ""

transfers:
  tftp:
    - command: "copy tftp://{src_ip}/{src_file} {dst_file}"
      method: send_command_timing
      prompts:
        <<: *prompts
      kwargs:
        read_timeout: 1800
        last_read: 60
```

> **Multiple commands:** when a platform needs several steps, list them one after another. `westermo_os.yml` is the repository example: it runs `file cfg`, then the `import proto scp …` command, then `end`.

## 4. Prompts

### Response forms

| Form | Repository example | Effect |
| --- | --- | --- |
| String | `"Password": "{password}"` (XR, Junos) | Sends the string; `""` sends Enter |
| Nested command entry | `assword:` with `command`, `method`, `kwargs` (IOS/XE, NX-OS, EOS) | Sends `command` with its own method and settings |
| `exc` | `'= {md5}': {exc: TransferOK}` | Raises a named exception or signal. `TransferOK` = verified |
| `msg` | `.: {msg: 'digest does not match'}` | Diagnostic message for the match |
| `true` | `"{md5}": true` (XR, Junos verify) | Marks the match as successful |

### Matching rules

| Rule | Detail |
| --- | --- |
| Substring or regex | Keys match part of the output; regex is allowed (`"assword\|over wr"`) |
| Case matters | XR and Junos list `"Password"` and `"password"` separately. Truncated keys (`assword`, `verwrite`, `stination file`) avoid this. Don't expand them |
| Escaping | In double quotes, write `\s` as `\\s` (`"Do you want to over\\s?write"`) |
| Order | Put specific patterns before general ones. IOS/XE verify relies on `= {md5}` being checked before the catch-all `.` |
| Overwrite answer | IOS/XE: `""` · EOS, NX-OS, XR: `y` · Junos: `yes` |

### Shared prompts

Defined once and merged with `<<: *prompts`, a standard YAML merge. Keys written next to the merge are added or override merged keys. Example from `cisco_nxos.yml`:

```yaml
.transfer-prompts: &prompts
  "Enter vrf":
    command: "{device.properties.vrf_mgmt}"
    kwargs:
      last_read: 5
  "Destination file":
    command: ""
    kwargs:
      last_read: 5
  "verwrite":
    command: "y"
    kwargs:
      last_read: 5

transfers:
  scp:
    - command: "copy scp://{username}@{src_ip}/{src_file} {dst_file}"
      prompts:
        <<: *prompts
        "re you sure you want to continue": "yes"
        "nter username": "{username}"
        assword:
          command: "{password}"
          method: send_command_progressing
          kwargs:
             expect_string: 'opy complete'
             error_string: 'opy fail'
```

> **Host keys:** EOS, NX-OS, XR, Junos and Westermo answer the SSH host-key prompt with `yes`, so they trust any file server on first use. Confirm the server's host key on the device before relying on this.

## 5. Methods and timeouts

### Methods

| Method | Use for | Used in |
| --- | --- | --- |
| `send_command` | Waiting for a specific pattern or the CLI prompt | IOS/XE prompt steps, verify |
| `send_command_timing` | Commands that finish when output goes quiet | IOS/XE TFTP |
| `send_command_progressing` | Long copies with explicit success/error patterns | SCP password step (IOS/XE, NX-OS, EOS) |

Only the step that starts the copy normally needs `send_command_progressing`. Short setup steps use `send_command`.

### `kwargs`

All optional. Times are in seconds.

| Parameter | Applies to | Purpose | Repository example |
| --- | --- | --- | --- |
| `expect_string` | `send_command`, `send_command_progressing` | Output that means "done" | `bytes copied` (IOS/XE), `opy complete` (NX-OS) |
| `error_string` | `send_command_progressing` | Output that means "failed" | `opy fail` (NX-OS) |
| `read_timeout` | All | Longest read time | `60` (IOS/XE verify), `1800` (IOS/XE TFTP) |
| `last_read` | `send_command_timing` | Quiet time before reading stops | `60` (IOS/XE TFTP) |
| `heartbeat_timeout` | `send_command_progressing` | Longest gap with no progress | `300` (IOS/XE), `360` (EOS) |

**IOS SCP flow** (`cisco_ios.yml`, identical in `cisco_xe.yml`): waits for the password or overwrite prompt, then runs the copy as a progressing command.

```yaml
transfers:
  scp:
    - command: "copy scp://{username}@{src_ip}/{src_file} {dst_file}"
      prompts:
        "stination file":
          command: ""
          method: send_command
          kwargs:
            expect_string: "assword|over wr"
        "over wr":
          command: ""
          method: send_command
          kwargs:
            expect_string: "assword"
        assword:
          command: "{password}"
          method: send_command_progressing
          kwargs:
            heartbeat_timeout: 300
            expect_string: 'bytes copied'
            error_string: 'fail'
```

**Timeout lessons:**

| Lesson | Detail |
| --- | --- |
| Use real success output | Take `expect_string` from a successful manual copy. Don't start verification while the copy is still running |
| Two timeout levels | The job's overall `timeout` is separate from each command's `read_timeout`/`heartbeat_timeout`. Raising only one may not fix an "absolute timer expired" error |
| Size for the largest image | XR and Junos SCP rely on `read_timeout: 180` with no completion pattern, and their verify steps set no timeout. Likely too short for large firmware |
| Keep error patterns specific | IOS/XE `error_string: 'fail'` and EOS `Error` are broad and may match harmless output |

## 6. Placeholders

| Placeholder | Value |
| --- | --- |
| `{src_ip}` | File server address |
| `{src_file}` | Source file path/name on the server |
| `{dst_file}` | Destination path/name on the device |
| `{username}` | Transfer username |
| `{password}` | Transfer password |
| `{md5}` | Expected MD5 checksum |
| `{device.properties.<name>}` | Device property, e.g. `vrf_mgmt` (NX-OS). Must be set on every device using the file |

- Placeholders inside prompt keys are read as regex, so a `.` in a value matches any character.
- **VRF:** NX-OS uses `{device.properties.vrf_mgmt}`, but EOS hardcodes `vrf management` in its copy command.

## 7. Verification

| Platform | Success match | Mismatch match |
| --- | --- | --- |
| IOS, IOS XE | `'= {md5}'` → `exc: TransferOK` | `.` → `msg: digest does not match` |
| EOS | `'= {md5}'` → `exc: TransferOK` | — |
| NX-OS | `'{md5}'` → `exc: TransferOK` | — |
| IOS XR, Junos | `"{md5}": true` | Listed under `errors` (`digest does not match`) |
| Westermo | `'{md5}'` in `dir` output → `exc: TransferOK` | `.` → `msg: digest does not match` |

Example (`cisco_ios.yml`):

```yaml
verify:
  - command: "verify /md5 {dst_file}"
    kwargs:
      read_timeout: 60
    prompts:
      '= {md5}':
        exc: TransferOK
      .:
        msg: 'digest does not match'
```

Keep the success form each platform already uses (`exc: TransferOK` or `true`). A returned CLI prompt is not proof of integrity.

## 8. Errors

| Exception | Meaning |
| --- | --- |
| `kibbitzer.exceptions.FileTransferError` | Transfer failed (file missing, permission, space, connectivity) |
| `kibbitzer.exceptions.InvalidCommand` | Device rejected the command |

Excerpt (`cisco_nxos.yml`):

```yaml
errors:
  kibbitzer.exceptions.FileTransferError:
    - Permission denied
    - No space left on
  kibbitzer.exceptions.InvalidCommand:
    - "^% (.*)"
```

Keep existing patterns and add new ones only from output you have actually seen.

## 9. Creating or changing a definition

1. Start from the closest platform file.
2. Run a manual copy on a test device. Capture the prompts, success output, errors and checksum output.
3. Write commands, prompt answers and verify steps using the patterns above.
4. Check YAML syntax, indentation, anchors and duplicate keys. Valid YAML does not mean a working transfer.
5. Commit and push to the restore repository managed by Netpicker's **Gitd** service. Don't edit runtime copies inside the Agent or Kibbitzer containers.
6. Test a small file, an existing destination file (overwrite path) and a full-size firmware image. Check both success and failure.

## 10. Troubleshooting

| Symptom | Check |
| --- | --- |
| `No schemas found` | A definition for the device's platform exists and is visible to the running instance |
| `File server is not configured` | File server settings. Fails before any transfer command runs |
| Stalls at password/overwrite prompt | Real CLI output vs. `prompts` and `expect_string`: case, whitespace, overwrite answer |
| Absolute timer expired / read timeout | Which command failed; both command-level and overall job timeouts |
| Copy completes but job fails | Completion pattern, then verify output |
| MD5 mismatch | Copy finished, destination path, expected checksum, verify timeout |
| Unexpected TFTP attempt | Configured protocols; remove unwanted `tftp` blocks if appropriate |
| Arista fails on VRF | EOS hardcodes `vrf management` |
| YAML changes have no effect | Gitd repository revision vs. the definition actually in use; duplicate config sources |
| Transfer seems slow | Copy time vs. total job time, including backup and compliance steps. Skip those for jobs that don't change configuration |

**Logs:** set `JOB_EXEC_LOG_LEVEL: AUDIT` in the Netpicker Compose configuration and read the job output with the Agent/Kibbitzer logs. Redact credentials before sharing, especially for Westermo, which puts `{password}` in the command line.

## 11. Open questions

These can't be confirmed from the YAML files and need confirmation from the Kibbitzer maintainers:

| Question | Why it matters |
| --- | --- |
| How is a protocol chosen? Is there a fallback order? | Explains unexpected TFTP attempts |
| Which pattern wins when several prompt keys match? | IOS/XE verify and XR prompts depend on it |
| What is the default `method`? | XR and Junos set none but use `last_read` |
| Which fields does the parser require? Are `.`-prefixed keys ignored? | Firms up the Required/Optional labels |
| How does a pushed change reach the running instance? | Avoids "changes have no effect" |
