# Platform File Transfer Definitions

This repository contains platform-specific file transfer definitions used by Kibbitzer / Netpicker for copying files to network devices and verifying the transferred file integrity.

Each YAML file defines how a specific network platform performs file transfers using protocols such as TFTP or SCP, how interactive device prompts are answered, how file checksums are verified, and which device output should be treated as an error.

## Purpose

Network operating systems handle file transfers differently. Even for similar commands, the prompts, verification commands, overwrite behaviour, VRF handling, and error messages can vary per vendor and platform.

These files provide a structured way to define those differences per platform.

Typical use cases include:

- Uploading files to a network device
- Downloading or staging files through TFTP or SCP
- Verifying file integrity after transfer
- Handling interactive CLI prompts
- Detecting transfer failures from device output

## File Structure

A platform file usually contains the following sections:

```yaml
.transfer-prompts: &prompts
  ...

verify:
  ...

transfers:
  ...

errors:
  ...
