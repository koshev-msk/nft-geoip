# nft_geoip

GeoIP matching for nftables, built on top of the **xt_geoip database format**.

```
geoip saddr RU,BY drop
geoip daddr != CN accept
```

nftables has no native GeoIP match. The usual workaround is to generate one
huge set per country and load it with `nft -f`. On small routers that costs a
lot of RAM and a lot of time at load. `nft_geoip` instead lets the kernel read
the compact xt_geoip files (`/usr/share/xt_geoip/XX.iv4`, `XX.iv6`) directly
and do a binary search per packet, the same way `xt_geoip` does for iptables.

If you already generate the xt_geoip database for iptables, nothing about your
database workflow changes.

## Components

| Part | What it is |
|------|------------|
| `src/` | Out-of-tree kernel module `nft_geoip.ko` (OpenWrt kmod package) |
| `100-libnftnl-add-geoip-expression.patch` | Adds the `geoip` expression to libnftnl |
| `100-nftables-add-geoip-statement.patch` | Adds the `geoip` statement to the `nft` parser and printer |
| `Makefile.openwrt` | A makefile OpenWrt SDK |

All three are needed: the module evaluates packets, libnftnl talks to it over
netlink, and `nft` provides the rule syntax.

Patches are written against **libnftnl 1.3.1** and **nftables 1.1.6** (the
versions in OpenWrt 25.12.6, kernel 6.12). Other versions may need rebasing.

## Syntax

```
geoip saddr|daddr [!=] CC[,CC...]
```

- `saddr` / `daddr` selects which address is looked up.
- `!=` inverts the match.
- Country codes are ISO 3166-1 alpha-2 (plus whatever codes your database
  contains, e.g. `A1`), case-insensitive, duplicates are merged.
- At most 16 countries per statement. The limit lives in three places
  (`NFT_GEOIP_MAX` in the module, libnftnl and nft) and must be changed in all
  of them together.
- It is a *match statement*: if it does not match, evaluation of the rule stops,
  just like `limit` or `meta` matches. Combine it with a verdict or another
  statement: `geoip saddr RU counter`, `geoip saddr RU drop`.
- Only IPv4 and IPv6 packets are evaluated. In table families where the packet
  family is not IPv4/IPv6 (`bridge`, `netdev`) the statement never matches.

Example for fw4 (`/etc/nftables.d/*.nft` is included inside `table inet fw4`);
adjust interface names to your setup:

```
chain geoip_block {
	type filter hook input priority filter - 1; policy accept;
	iifname "wan" geoip saddr CN,KP drop
}
```

`geoip` becomes a reserved word in `nft`: a table or chain that is literally
named `geoip` must be quoted.

## Database format

The module reads the same flat files `xt_geoip` uses, one pair per country, from
`/usr/share/xt_geoip/` (no `LE`/`BE` subdirectories):

- `XX.iv4`: array of `{ __be32 begin, end; }`, 8 bytes per range
- `XX.iv6`: array of `{ struct in6_addr begin, end; }`, 32 bytes per range
- network byte order on disk, so the files are architecture independent
- ranges sorted ascending, non-overlapping, `end` inclusive, no header

They can be produced from a MaxMind database with
[maxminddb-dump-country](https://github.com/vel21ripn/maxminddb-dump-country)
or with `xt_geoip_build`. The database is **not** part of this package.

Files are read when a rule is created, not when the module loads. After the
database is updated, re-create the rules (reload the firewall) to pick up the
new data. A country used by several rules is loaded once.

## Building with the OpenWrt SDK

Tested with SDK 25.12.6 (more arch, kernel 6.12.112).

1. Add the package to a feed (the example below uses a custom feed) so that it
   ends up as `.../nft-geoip/Makefile` plus `.../nft-geoip/src/`.

2. Install the userspace patches:

   ```sh
   cp 100-libnftnl-add-geoip-expression.patch \
      feeds/base/libs/libnftnl/patches/
   cp 100-nftables-add-geoip-statement.patch \
      feeds/base/network/utils/nftables/patches/
   ```

   (In a full OpenWrt tree the directories are `package/libs/libnftnl/patches/`
   and `package/network/utils/nftables/patches/`.)

3. Check the package Makefiles:

   - `libnftnl` must have `PKG_FIXUP:=autoreconf`, otherwise the change to
     `src/Makefile.am` is ignored and linking fails with
     `undefined reference to expr_ops_geoip`.
   - Bump `PKG_RELEASE` in `libnftnl` and `nftables`, otherwise `apk` considers
     the old packages up to date.
   - The nftables patch changes `parser_bison.y` and `scanner.l`, so `bison` and
     `flex` must be installed on the build host.

4. Build, libnftnl first:

   ```sh
   make package/feeds/base/libnftnl/{clean,compile} V=s
   make package/feeds/base/nftables/{clean,compile} V=s
   make package/feeds/<your-feed>/nft-geoip/{clean,compile} V=s
   ```

   `clean` matters: without it, sources already unpacked in `build_dir` stay
   unpatched. The `nftables-json` and `nftables-nojson` variants are both built.

5. Install on the device (order matters):

   ```sh
   apk add --allow-untrusted libnftnl11-*.apk
   apk add --allow-untrusted nftables-json-*.apk
   apk add --allow-untrusted kmod-nft-geoip-*.apk
   ```

   The kernel module must be built by the same SDK/kernel that runs on the
   device.

## Building without OpenWrt

Kernel module, against a configured kernel tree with `CONFIG_NF_TABLES`:

```sh
cd nft-geoip/src
make -C /path/to/kernel/build M=$PWD modules
```

Userspace, on clean release tarballs:

```sh
cd libnftnl-1.3.1 && patch -p1 < ../100-libnftnl-add-geoip-expression.patch
autoreconf -fi && ./configure && make && make install
cd ../nftables-1.1.6 && patch -p1 < ../100-nftables-add-geoip-statement.patch
./configure && make && make install
```

## Quick check

```sh
lsmod | grep nft_geoip
ls /usr/share/xt_geoip/RU.iv4
nft add table inet t
nft add chain inet t c '{ type filter hook input priority 0; }'
nft add rule inet t c geoip saddr RU counter
nft list ruleset
```

## Failure behaviour

- **Module not loaded:** the rule is rejected with
  `Could not process rule: No such file or directory` pointing at `geoip`. The
  syntax is still parsed fine because userspace knows the statement.
- **No database for a country** (missing directory or both files): same error;
  `dmesg` shows
  `nft_geoip: RU: cannot read database from /usr/share/xt_geoip (iv4: -2, iv6: -2)`.
- **Only one of `.iv4`/`.iv6` exists:** accepted. The missing family simply has
  no ranges, so `geoip saddr RU` never matches those packets and
  `geoip saddr != RU` always does.
- **File size not a multiple of the record size:** `-EINVAL` and a `dmesg`
  message.
- **Files removed after the rule was loaded:** no effect until the next reload;
  the data is already in kernel memory.

Rulesets are applied atomically. If a rule with `geoip` fails to load (module
or database missing), the whole transaction is rejected, which for `fw4` means
no firewall at all. Make sure the module is loaded before the firewall starts
and keep geo rules in a separate snippet that is easy to disable.

## Design notes

- `geoip` is a *statement* in `nft` and has no destination register in the
  kernel: it either lets evaluation continue or breaks out of the rule.
- The lookup is a plain binary search over the flat arrays, IPv4 ranges are
  converted to host order once at load.
- Files are read with `kernel_read_file_from_path()` from the sleepable rule
  creation path, so the root filesystem must be readable at that time.
- The netlink attribute ids (`NFTA_GEOIP_FLAGS = 1`, `NFTA_GEOIP_CC = 2`) are
  not part of the kernel's `nf_tables.h`; they are defined in `nft_geoip.h` and
  duplicated in the libnftnl patch. Keep them in sync.


## Memory usage comparison: `nftset` vs `nft_geoip`

A memory usage comparison was performed on an OpenWrt router with **128 MiB of RAM**, using the same set of seven countries (`CN`, `GB`, `UA`, `PL`, `LT`, `LV`, `EE`) for both implementations.

### Results

| Memory metric | `nft_geoip` | `nftset` | Difference (`nftset` − `nft_geoip`) |
|---|---:|---:|---:|
| `MemAvailable` | 60,356 KiB | 22,008 KiB | **−38,348 KiB** |
| `MemFree` | 71,080 KiB | 35,676 KiB | −35,404 KiB |
| `Slab` | 12,756 KiB | 41,824 KiB | **+29,068 KiB** |
| `SUnreclaim` | 9,756 KiB | 38,900 KiB | **+29,144 KiB** |
| `SReclaimable` | 3,000 KiB | 2,924 KiB | −76 KiB |
| `Cached` | 24,776 KiB | 19,048 KiB | −5,728 KiB |
| `VmallocUsed` | 3,012 KiB | 1,344 KiB | −1,668 KiB |
| `AnonPages` | 3,508 KiB | 3,500 KiB | −8 KiB |

All values are in KiB. The snapshots were collected after applying the respective firewall rules.

### Key observations

- **29,068 KiB (~28.4 MiB) less slab memory** was reported with `nft_geoip`.
- `nftset` showed **29,144 KiB (~28.5 MiB) more unreclaimable slab memory**.
- Available memory was **38,348 KiB (~37.5 MiB) higher** with `nft_geoip`.
- Anonymous memory usage was nearly identical in both snapshots, suggesting that the observed difference was not caused by a significant increase in userspace process memory.
- `VmallocUsed` was 1,668 KiB higher with `nft_geoip`, approximately matching the combined on-disk sizes of the selected countries' IPv4 and IPv6 GeoIP database files.

### Conclusion

In this test, `nft_geoip` showed substantially lower slab memory usage than the `nftset` implementation for the same set of countries. On a router with 128 MiB of RAM, the observed difference is significant: approximately **28.4 MiB less slab memory** and **37.5 MiB more available memory**.

These are measurements from individual snapshots, not a fully controlled benchmark. The results demonstrate a strong correlation between the implementation and memory usage, but further repeated tests under identical conditions are needed to attribute the entire difference exclusively to the GeoIP implementation.


## Status and limitations

- Kernel module builds and loads on OpenWrt 25.12.6, kernel 6.12.112
- The `nft` and libnftnl side parses and encodes the rules (checked against a
  libnftnl/nftables build outside OpenWrt); the JSON importer does not know the
  statement (JSON output works).
- No `clone` operation yet, so the expression cannot be used inside set/map
  element expressions; plain rules in chains are fine.
- Kernel API details used (`dump(..., bool reset)`, `NFT_REDUCE_READONLY`,
  `kernel_read_file_from_path`) match 6.12 and may need small changes on other
  kernel versions.

## License

Kernel module: GPL-2.0-only. The patches follow the licenses of the projects
they modify (libnftnl: GPL-2.0-or-later, nftables: GPL-2.0-only).

## Credits

- The database format and the idea come from `xt_geoip` in
  [xtables-addons](https://inai.de/projects/xtables-addons/).
- Database generation:
  [vel21ripn/maxminddb-dump-country](https://github.com/vel21ripn/maxminddb-dump-country).
