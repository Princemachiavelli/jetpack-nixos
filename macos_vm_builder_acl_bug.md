# Determinate Nix macOS Linux builder: virtiofs /nix/store issues

# Issue 1: `cp --no-preserve=mode` fails on Determinate Nix macOS Linux builder

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

---

# Issue 2: `$out` is case-insensitive on the Determinate Nix macOS Linux builder

## Summary

Inside the same Linux builder VM, `/nix/store` is case-insensitive, because it is the host's
`Nix Store` APFS volume shared over virtiofs:

```
$ diskutil info /nix | grep -E "Volume Name|Personality"
   Volume Name:               Nix Store
   File System Personality:   APFS          # not "Case-sensitive APFS"
```

A Linux build that writes two paths differing only in case into `$out` either fails or silently
loses one file. The host's `use-case-hack = true` (Nix's renaming of colliding names on macOS)
doesn't apply, because the guest writes directly to the shared volume.

Real-world trigger: NVIDIA's `nvidia-l4t-jetson-multimedia-api_39.2.1` .deb ships both

```
usr/src/jetson_multimedia_api/include/NvBufSurface.h
usr/src/jetson_multimedia_api/include/nvbufsurface.h
```

and copying the unpacked tree to `$out` (`cp -pr . $out`) fails with:

```
cp: cannot create regular file '/nix/store/...-source/src/jetson_multimedia_api/include/nvbufsurface.h': File exists
```

## Minimal reproduction

`case.nix`:

```nix
# Usage: nix build --impure -L -f case.nix <attr>   (on aarch64-darwin with the Determinate Linux builder)
let
  pkgs = (builtins.getFlake "github:NixOS/nixpkgs/nixos-unstable").legacyPackages.aarch64-linux;
  mk = name: target: pkgs.runCommand "vm-case-repro-${name}" { } ''
    mkdir -p $out ${target}
    echo upper > ${target}/Foo.h
    echo lower > ${target}/foo.h
    ls ${target}; [ "$(cat ${target}/Foo.h)" = upper ] || { echo "Foo.h was clobbered"; exit 1; }
  '';
in {
  # FAILS: two names differing only in case under $out (virtiofs-backed /nix/store)
  case-collision-store = mk "store" "$out";
  # OK: same thing on the VM-local tmpfs
  case-collision-tmpfs = mk "tmpfs" "/build/t";
}
```

Observed:

```
case-collision-store:  ls shows only "Foo.h"; "Foo.h was clobbered"; rc=1
case-collision-tmpfs:  rc=0
```

A plain shell redirect (`open(O_CREAT|O_TRUNC)`) silently overwrites the existing file, which
loses data. `cp` uses `O_EXCL`, so it fails with `EEXIST` instead.

## Expected behaviour

Linux builds should see a case-sensitive `/nix/store`, as on a native Linux builder. Options for
Determinate: back the builder's store with a case-sensitive volume, or warn when the host store
is case-insensitive.

## Workaround

- Recreate the host store on a case-sensitive APFS volume. An existing APFS volume's case
  sensitivity can't be changed in place, so this means uninstalling and reinstalling with
  `/nix/nix-installer install macos --case-sensitive`.
- Or build affected derivations on a native Linux remote builder, or substitute them from a cache.
