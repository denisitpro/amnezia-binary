# amnezia-binary

Pre-built **linux/amd64** userspace binaries for [AmneziaWG](https://github.com/amnezia-vpn):

| Asset | Source |
|---|---|
| `amneziawg-go-linux-amd64` | [amnezia-vpn/amneziawg-go](https://github.com/amnezia-vpn/amneziawg-go) (static Go datapath) |
| `awg-linux-amd64` | [amnezia-vpn/amneziawg-tools](https://github.com/amnezia-vpn/amneziawg-tools) (`awg` CLI) |
| `awg-quick` | same (`awg-quick` bash helper) |
| `SHA256SUMS` | checksums for the three assets above |

## Why this repo exists

Upstream `amneziawg-go` publishes **no GitHub release assets**. An Ansible role that only wants to
`get_url` a binary + checksum cannot consume it without compiling on the target host.

This repository does **not** fork either upstream project and changes none of their code. A GitHub
Actions workflow clones each upstream repo at a pinned tag, builds the artifacts, and publishes them
as release assets with **stable, predictable names**.

## Scope

- **linux/amd64 only.** arm64 is out of scope for now.
- Userspace only (`amneziawg-go` + tools). No kernel module.
- No man pages, bash completion, or shipped systemd units — the consuming Ansible role owns the unit.

## `version.txt` format

Two lines, one pinned upstream git tag each:

```
amneziawg-go=v3.0.2
amneziawg-tools=v3.0.20260730
```

- Keys must be exactly `amneziawg-go` and `amneziawg-tools`.
- Values are used **verbatim** as `git clone --branch` refs.
- Blank lines and `#` comments are allowed.
- A malformed file **fails the workflow** — nothing is guessed or defaulted.

### Matched pair — do not bump one alone

The two versions are a **matched pair**. Bumping only one side is wrong:

- `amneziawg-go` **v3.0.2** is AmneziaWG 3.0. Its UAPI (`device/uapi.go`) accepts
  `content_padding_addition`, `header_protection_key`, `keepalive_timeout`,
  `max_handshake_attempts`, `reject_after_time`, `rekey_after_time`, `rekey_timeout`
  in addition to the 2.0 key set.
- `amneziawg-tools` only learned to **send** those keys in **v3.0.20260730**
  (commit message: `feat: AmneziaWG 3.0 (#60)`). The previous tag `v1.0.20260618-2` is 2.0-only.

Pairing a 3.0 datapath with 2.0 tools **silently loses every 3.0 feature**. Always bump both
entries together when moving between major protocol generations.

### How to bump

1. Edit `version.txt` with the new matched pair of tags.
2. Merge to `main` (a push that changes `version.txt` triggers the workflow).
3. Wait for the release `awg-<go_tag>-tools-<tools_tag>` to appear.

Manual re-run: **Actions → Build and publish… → Run workflow**. Use `force=true` only when you
intentionally want to overwrite an existing release with the same tag. Without `force`, an
already-published tag is left alone (idempotent — no silent clobber).

## Release assets

Release tag pattern:

```text
awg-<amneziawg-go-tag>-tools-<amneziawg-tools-tag>
```

Example for the pins above: `awg-v3.0.2-tools-v3.0.20260730`.

Asset URLs (replace owner/repo if forked):

```text
https://github.com/denisitpro/amnezia-binary/releases/download/awg-v3.0.2-tools-v3.0.20260730/amneziawg-go-linux-amd64
https://github.com/denisitpro/amnezia-binary/releases/download/awg-v3.0.2-tools-v3.0.20260730/awg-linux-amd64
https://github.com/denisitpro/amnezia-binary/releases/download/awg-v3.0.2-tools-v3.0.20260730/awg-quick
https://github.com/denisitpro/amnezia-binary/releases/download/awg-v3.0.2-tools-v3.0.20260730/SHA256SUMS
```

### Build notes

| Binary | How it is built |
|---|---|
| `amneziawg-go-linux-amd64` | `golang:1.25.12` container; ldflags match upstream Dockerfile: `-linkmode external -extldflags "-fno-PIC -static"`. Fully static ELF. |
| `awg-linux-amd64` | Built and staged-installed inside `ubuntu:22.04` (**glibc 2.35**). Dynamically linked to that glibc. Intended consumer host: **Ubuntu 24.04 (glibc 2.39)** — older glibc binaries run on newer glibc, not the reverse. |
| `awg-quick` | Harvested from `make -C src install … WITH_WGQUICK=yes` (`src/wg-quick/linux.bash` → `awg-quick`). Architecture-independent bash. |

`awg-quick` reads `/etc/amnezia/amneziawg/<iface>.conf` and falls back to
`${WG_QUICK_USERSPACE_IMPLEMENTATION:-amneziawg-go}` when `ip link add type amneziawg` fails
(no kernel module). `amneziawg-go` serves its UAPI socket at `/var/run/amneziawg/<iface>.sock`.

## Ansible example

```yaml
- name: Ensure AmneziaWG userspace binaries are installed
  vars:
    awg_release: "awg-v3.0.2-tools-v3.0.20260730"
    awg_base_url: "https://github.com/denisitpro/amnezia-binary/releases/download/{{ awg_release }}"
    # Fill these from SHA256SUMS of that release (do not invent checksums).
    amneziawg_go_sha256: "REPLACE_WITH_SHA256_FROM_SHA256SUMS"
    awg_sha256: "REPLACE_WITH_SHA256_FROM_SHA256SUMS"
    awg_quick_sha256: "REPLACE_WITH_SHA256_FROM_SHA256SUMS"
  block:
    - name: Install amneziawg-go
      ansible.builtin.get_url:
        url: "{{ awg_base_url }}/amneziawg-go-linux-amd64"
        dest: /usr/local/bin/amneziawg-go
        mode: "0755"
        checksum: "sha256:{{ amneziawg_go_sha256 }}"

    - name: Install awg
      ansible.builtin.get_url:
        url: "{{ awg_base_url }}/awg-linux-amd64"
        dest: /usr/local/bin/awg
        mode: "0755"
        checksum: "sha256:{{ awg_sha256 }}"

    - name: Install awg-quick
      ansible.builtin.get_url:
        url: "{{ awg_base_url }}/awg-quick"
        dest: /usr/local/bin/awg-quick
        mode: "0755"
        checksum: "sha256:{{ awg_quick_sha256 }}"
```

Pin `awg_release` and the three SHA256 digests in your inventory/group_vars when you promote a new
release. Fetch digests from the release’s `SHA256SUMS` asset (or the release body table).

## License

Build automation in this repository is provided as-is. The binaries themselves are covered by their
respective upstream licenses (see amneziawg-go and amneziawg-tools).
