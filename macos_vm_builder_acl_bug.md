# `cp --no-preserve=mode` fails on Determinate Nix macOS Linux builder

## Summary

On aarch64-darwin with Determinate Nix's native Linux builder (`external-builders` →
`determinate-nixd builder`), GNU `cp --no-preserve=mode` (and `--no-preserve=all`) fails with
`setting permissions for '...': Permission denied` when copying a **read-only** file into `$out`.

The same command works for a writable source file, and for any file copied to the VM-local
`/build` (tmpfs). This breaks common nixpkgs idioms such as
`cp --no-preserve=all -r ${src}/. $out/...`, since store paths are always read-only.

## Environment

- Host: macOS (Darwin 25.6.0), aarch64-darwin
- Nix: Determinate Nix 3.23.2 (Nix 2.35.3)
- Builder: `external-builders = [{"program":"/usr/local/bin/determinate-nixd","args":["builder"],"systems":["aarch64-linux","x86_64-linux"]}]`
- Guest kernel: `Linux 6.1.143 #1-NixOS SMP aarch64`
- Guest mounts:
  ```
  tmpfs     /            tmpfs    rw,relatime
  nix-store /nix/store   virtiofs rw,relatime
  buildroot /build-info  virtiofs rw,relatime
  ```
- coreutils 9.11, build user `nixbld` (uid 30000)

## Minimal reproduction

`repro.nix`:

```nix
# Usage: nix build --impure -L -f repro.nix <attr>   (on aarch64-darwin with the Determinate Linux builder)
let
  pkgs = (builtins.getFlake "github:NixOS/nixpkgs/nixos-unstable").legacyPackages.aarch64-linux;
  mk = name: mode: target: pkgs.runCommand "vm-acl-repro-${name}" { } ''
    echo hi > /build/src
    chmod ${mode} /build/src
    mkdir -p $out
    cp --no-preserve=mode /build/src ${target}/copy
  '';
in {
  # FAILS: read-only source copied into $out (virtiofs-backed /nix/store)
  readonly-to-store = mk "ro-store" "0444" "$out";
  # OK: writable source copied into $out
  writable-to-store = mk "rw-store" "0644" "$out";
  # OK: read-only source copied into /build (VM-local tmpfs)
  readonly-to-tmpfs = mk "ro-tmpfs" "0444" "/build";
}
```

```sh
for a in readonly-to-store writable-to-store readonly-to-tmpfs; do
  nix build --no-link --impure -f repro.nix $a; echo "$a rc=$?"
done
```

Observed:

```
cp: setting permissions for '/nix/store/...-vm-acl-repro-ro-store/copy': Permission denied
readonly-to-store rc=1
writable-to-store rc=0
readonly-to-tmpfs rc=0
```

## Analysis

`strace` of `cp --no-preserve=all /build/src(0444) $out/g` inside the builder:

```
openat(AT_FDCWD, "$out/g", O_WRONLY|O_CREAT|O_EXCL, 0444) = 4
fsetxattr(4, "system.posix_acl_access", ..., 28, 0)       = -1 EACCES (Permission denied)
fchmod(4, 0644)                                           = 0
cp: setting permissions for '$out/g': Permission denied
```

The same command targeting `/build` (tmpfs):

```
fsetxattr(4, "system.posix_acl_access", ..., 28, 0)       = 0
```

1. With `--no-preserve=mode`, cp creates the file with the source mode (`0444`), then resets
   permissions through `set_acl()`, which first tries `fsetxattr(system.posix_acl_access)`.
2. On Linux, the file owner may set ACLs/xattrs regardless of the file's mode bits. Over
   virtiofs, the request is served by the macOS host. The host appears to apply macOS semantics,
   where writing an xattr needs write permission on the file, so it rejects the call with `EACCES`
   because the file is `0444`. This is inferred from the writable-vs-read-only behaviour, not
   confirmed in the host-side code.
3. cp falls back to `fchmod`, which succeeds. But gnulib only treats "ACLs unsupported" errnos
   (`ENOTSUP`, `ENOSYS`, `EINVAL`, ...) as ignorable, so `EACCES` is reported and cp exits 1.

Additional observations:

- `chmod` and `setfacl -m u::rw` (path-based; reduces to a chmod for a minimal ACL) succeed on
  the same file.
- Directories are unaffected, because cp creates them writable by the owner before fixing the mode.

## Expected behaviour

As on a native Linux filesystem: the file owner can set `system.posix_acl_access` on their own
file regardless of its mode bits. At minimum the virtiofs server should return `EOPNOTSUPP` (which
cp tolerates) rather than `EACCES`.

## Workaround

Avoid `--no-preserve=mode` when copying into the store; fix permissions afterwards instead:

```sh
cp -r "${src}"/. "$out/dir"
chmod -R u+w "$out/dir"
```

This is what jetpack-nixos now does in `pkgs/kernels/r3{6,9}/{oot-modules,devicetree}.nix` and
`mk-overlay.nix`.
