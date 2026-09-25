# mergerfs-tools

Optional tools to help manage data in a mergerfs pool.

## INSTALL

All of these suplimental tools are self contained Python3 apps. Make sure you have Python 3 installed and either run `make install` or copy the file to `/usr/local/bin` or wherever you keep your binarys and make it executable (chmod +x).

## TOOLS
### mergerfs-tools

A single unified front-end which wraps the tools below. It exposes one new
option vocabulary and translates internally to the original scripts; you never
type their original flags. Data-changing commands execute by default, use
`--dry-run` to preview. It is only a wrapper, the individual scripts remain
usable directly.

`balance` has no preview mode and always executes. `ctl` always writes on
`add`/`remove`/`set`. `fsck` only fixes with `--fix`. `vacate` only copies and
never deletes source data.

[Download latest](https://raw.githubusercontent.com/trapexit/mergerfs-tools/master/src/mergerfs-tools)

```
用法: mergerfs-tools <命令> [选项] <目录>

mergerfs 辅助工具的统一入口。所有选项都是本工具自定义的，底层各脚本的
原始参数由本工具内部翻译，你无需了解。

命令:
  ctl add|remove|list|get|set|info   操作运行中的 mergerfs 挂载
  fsck                               审计/修复文件的权限与属主
  dup                                在分支间复制文件，使每个文件有 N 份
  dedup                              删除分支间的重复文件
  balance                            按剩余空间在分支间搬移文件
  consolidate                        把单个目录的文件归拢到一个分支
  trash                              在每个分支创建 .Trash 目录
  vacate <pool> <branch>             安全下架分支：移出池并把数据复制进池

通用选项:
  -n, --dry-run        只预览，不执行
  -v, --verbose        提高输出详细度（可重复）
  -h, --help           显示本帮助
  --include GLOB       包含匹配的文件/目录/路径（可重复）
  --exclude GLOB       排除匹配的文件/目录/路径（可重复）

各命令专属选项:
  ctl            --mount PATH
  fsck           --fix manual|newest|nonroot    --check-size
  dup            --copies N    --keep MODE    --prune
  dedup          --keep MODE   --ignore COND  --strict
  balance        --free-gap PCT   --min-size SZ   --max-size SZ
  consolidate    --dir-max-files N   --dir-max-size SZ

取值:
  --keep    manual | oldest | newest | largest | smallest | most-space | mergerfs
            （dup 不支持 manual、most-space）
  --ignore  none | same-size | different-size | same-time | different-time |
            same-hash | different-hash | same-short-hash | different-short-hash
  SIZE      整数 + K/M/G/T，例如 16G

过滤参数的处理（执行前校验）:
  目标目录必须存在，否则报错。
  不含通配符的值会在目标目录下探测：不存在则报错；存在则按文件/目录分派，
  底层无法处理的组合直接报错（如 dup 给目录、consolidate 给文件、
  balance 按目录名排除、dup/dedup 的值里带 '/'）。
  含通配符（* ? [）的值无法预先探测，交由底层匹配器处理。

说明:
  会改动数据的命令默认直接执行，加 --dry-run 只预览。
  fsck 只有给出 --fix 才修复，否则只审计。
  vacate 只复制、从不删除源数据。
  ctl 的写操作立即生效，不支持 --dry-run；balance 没有预览模式。

示例:
  # 预览去重：保留最新文件，并排除 .git 目录
  mergerfs-tools dedup --dry-run --keep newest --exclude .git /mnt/pool

  # 实际去重：只处理 .mkv，且要求内容 md5 相同
  mergerfs-tools dedup --include '*.mkv' --ignore same-hash /mnt/pool

  # 复制文件，使每个文件至少有两份
  mergerfs-tools dup --copies 2 /mnt/pool

  # 审计某目录的权限/属主不一致
  mergerfs-tools fsck --verbose /mnt/pool/media

  # 把各分支的使用率差距压到 5% 以内
  mergerfs-tools balance --free-gap 5 /mnt/pool

  # 把某目录的文件归拢到单个分支（先预览）
  mergerfs-tools consolidate --dry-run /mnt/pool/music

  # 操作运行中的挂载：添加 / 移除分支
  mergerfs-tools ctl --mount /mnt/pool add path /mnt/disk3
  mergerfs-tools ctl --mount /mnt/pool remove path /mnt/disk3

  # 查看挂载信息
  mergerfs-tools ctl --mount /mnt/pool info
  mergerfs-tools ctl --mount /mnt/pool list values

  # 在各分支创建 .Trash 目录（需要 root）
  sudo mergerfs-tools trash /mnt/pool

  # 下架 /mnt/disk3：先预览，确认后去掉 --dry-run 执行
  mergerfs-tools vacate --dry-run /mnt/pool /mnt/disk3
  mergerfs-tools vacate /mnt/pool /mnt/disk3
```

### mergerfs.ctl

A wrapper around the mergerfs xattr interface.

[Download latest](https://raw.githubusercontent.com/trapexit/mergerfs-tools/master/src/mergerfs.ctl)

```
$ mergerfs.ctl -h
usage: mergerfs.ctl [-h] [-m MOUNT] {add,remove,list,get,set,info} ...

positional arguments:
  {add,remove,list,get,set,info}

optional arguments:
  -h, --help            show this help message and exit
    -m MOUNT, --mount MOUNT
                            mergerfs mount to act on
$ mergerfs.ctl info
- mount: /storage
  version: 2.14.0
  pid: 1234
  srcmounts:
    - /mnt/drive0
    - /mnt/drive1
$ mergerfs.ctl -m /storage add path /mnt/drive2
$ mergerfs.ctl info
- mount: /storage
  version: 2.14.0
  pid: 1234
  srcmounts:
    - /mnt/drive0
    - /mnt/drive1
    - /mnt/drive2
```

### mergerfs.fsck

Audits permissions and ownership of files and directories in a mergerfs mount and allows for manual and automatic fixing of them.

It's possible that files or directories can be duplicated across multiple drives and that their metadata become out of sync. Permissions, ownership, etc. This can cause some strange behavior depending on the mergerfs policies used. This tool helps find and fix those inconsistancies.

[Download latest](https://raw.githubusercontent.com/trapexit/mergerfs-tools/master/src/mergerfs.fsck)

```
$ mergerfs.fsck -h
usage: mergerfs.fsck [-h] [-v] [-s] [-f {manual,newest,nonroot}] dir

audit a mergerfs mount for inconsistencies

positional arguments:
  dir                   starting directory

  optional arguments:
    -h, --help            show this help message and exit
    -v, --verbose         print details of audit item
    -s, --size            only consider if the size is the same
    -f {manual,newest,nonroot}, --fix {manual,newest,nonroot}
                          fix policy
$ mergerfs.fsck -v -f manual /path/to/dir
```

### mergerfs.dup

Duplicates files & directories across branches in a pool. The file selected for duplication is picked by the `dup` option. Files will be copied to drives with the most free space. Deleted from others if `prune` is enabled.

See usage for more. Run as `root`. Requires `rsync` to be installed.

[Download latest](https://raw.githubusercontent.com/trapexit/mergerfs-tools/master/src/mergerfs.dup)

```
usage: mergerfs.dup [<options>] <dir>

Duplicate files & directories across multiple drives in a pool.
Will print out commands for inspection and out of band use.

positional arguments:
  dir                    starting directory

optional arguments:
  -c, --count=           Number of copies to create. (default: 2)
  -d, --dup=             Which file (if more than one exists) to choose to
                         duplicate. Each one falls back to `mergerfs` if
                         all files have the same value. (default: newest)
                         * newest   : file with largest mtime
                         * oldest   : file with smallest mtime
                         * smallest : file with smallest size
                         * largest  : file with largest size
                         * mergerfs : file chosen by mergerfs' getattr
  -p, --prune            Remove files above `count`. Without this enabled
                         it will update all existing files.
  -e, --execute          Execute `rsync` and `rm` commands. Not just
                         print them.
  -I, --include=         fnmatch compatible filter to include files.
                         Can be used multiple times.
  -E, --exclude=         fnmatch compatible filter to exclude files.
                         Can be used multiple times.
```


### mergerfs.dedup

Finds and removes duplicate files across mergerfs pool's branches. Use the
`ignore`, `dedup`, and `strict` options to target specific use cases.

[Download latest](https://raw.githubusercontent.com/trapexit/mergerfs-tools/master/src/mergerfs.dedup)

```
usage: mergerfs.dedup [<options>] <dir>

Remove duplicate files across branches of a mergerfs pool. Provides
multiple algos for determining which file to keep and what to skip.

positional arguments:
  dir                    Starting directory

optional arguments:
  -v, --verbose          Once to print `rm` commands
                         Twice for status info
                         Three for file info
  -i, --ignore=          Ignore files if... (default: none)
                         * same-size      : have the same size
                         * different-size : have different sizes
                         * same-time      : have the same mtime
                         * different-time : have different mtimes
                         * same-hash      : have the same md5sum
                         * different-hash : have different md5sums
  -d, --dedup=           What file to *keep* (default: newest)
                         * manual        : ask user
                         * oldest        : file with smallest mtime
                         * newest        : file with largest mtime
                         * largest       : file with largest size
                         * smallest      : file with smallest size
                         * mostfreespace : file on drive with most free space
  -s, --strict           Skip dedup if all files have same value.
                         Only applies to oldest, newest, largest, smallest.
  -e, --execute          Will not perform file removal without this.
  -I, --include=         fnmatch compatible filter to include files.
                         Can be used multiple times.
  -E, --exclude=         fnmatch compatible filter to exclude files.
                         Can be used multiple times.

# mergerfs.dedup /path/to/dir
# Total savings: 10.0GB

# mergerfs.dedup -e -d newest /path/to/dir
mergerfs.dedup -v -d newest /media/tmp/test
rm -vf /mnt/drive0/test/foo
rm -vf /mnt/drive1/test/foo
rm -vf /mnt/drive2/test/foo
rm -vf /mnt/drive3/test/foo
# Total savings: 10.0B
```


### mergerfs.balance

Will move files from the most filled drive (percentage wise) to the least filled drive. Will do so till the most and least filled drives come within a user defined percentage range (defaults to 2%).

Run as `root`. Requires `rsync` to be installed.

[Download latest](https://raw.githubusercontent.com/trapexit/mergerfs-tools/master/src/mergerfs.balance)

```
usage: mergerfs.balance [-h] [-p PERCENTAGE] [-i INCLUDE] [-e EXCLUDE]
                        [-I INCLUDEPATH] [-E EXCLUDEPATH] [-s EXCLUDELT]
                        [-S EXCLUDEGT]
                        dir

balance files on a mergerfs mount based on percentage drive filled

positional arguments:
  dir                   starting directory

optional arguments:
  -h, --help            show this help message and exit
  -p PERCENTAGE         percentage range of freespace (default 2.0)
  -i INCLUDE, --include INCLUDE
                        fnmatch compatible file filter (can use multiple
                        times)
  -e EXCLUDE, --exclude EXCLUDE
                        fnmatch compatible file filter (can use multiple
                        times)
  -I INCLUDEPATH, --include-path INCLUDEPATH
                        fnmatch compatible path filter (can use multiple
                        times)
  -E EXCLUDEPATH, --exclude-path EXCLUDEPATH
                        fnmatch compatible path filter (can use multiple
                        times)
  -s EXCLUDELT          exclude files smaller than <int>[KMGT] bytes
  -S EXCLUDEGT          exclude files larger than <int>[KMGT] bytes

# mergerfs.balance /media
from: /mnt/drive1/foo/bar
to:   /mnt/drive2/foo/bar
rsync ...
```


### mergerfs.consolidate

Consolidate **files** in a **single** mergerfs directory onto a **single** drive, recursively. This does **NOT** move all files at and below that directory to 1 drive. If you want to move data between drives simply use normal rsync or similar. This tool is only useful in niche usecases where the person wants to colocate files of their TV, music, etc. files onto a single drive *after the fact.* If you really wanted that you should probably use path preservation. For most people there is only downsides to using path preservation or colocating files.

Run as `root`. Requires `rsync` to be installed.

[Download latest](https://raw.githubusercontent.com/trapexit/mergerfs-tools/master/src/mergerfs.consolidate)

```
usage: mergerfs.consolidate [<options>] <dir>

positional arguments:
  dir                    starting directory

optional arguments:
  -m, --max-files=       Skip directories with more than N files.
                         (default: 256)
  -M, --max-size=        Skip directories with files adding up to more
                         than N. (default: 16G)
  -I, --include-path=    fnmatch compatible path include filter.
                         Can be used multiple times.
  -E, --exclude-path=    fnmatch compatible path exclude filter.
                         Can be used multiple times.
  -e, --execute          Execute `rsync` commands as well as print them.
  -h, --help             Print this help.
```

## SUPPORT

#### Contact / Issue submission
* github.com: https://github.com/trapexit/mergerfs-tools/issues
* email: trapexit@spawn.link
* twitter: https://twitter.com/_trapexit

#### Support development

This software is free to use and released under a very liberal license. That said if you like this software and would like to support its development donations are welcome.

* PayPal: https://paypal.me/trapexit
* GitHub Sponsors: https://github.com/sponsors/trapexit
* Patreon: https://www.patreon.com/trapexit
* SubscribeStar: https://www.subscribestar.com/trapexit
* Ko-Fi: https://ko-fi.com/trapexit
* Open Collective: https://opencollective.com/trapexit
* Bitcoin (BTC): bc1qjwlywkqxgrxql3m7a7fvcsf3z3t98jvtekqp2j
* Bitcoin Cash (BCH): qrvymmkvuk7703m7cx0pqxc3mz4mmsn6ngn9xw52kc
* Bitcoin SV (BSV): 1FkFuxRtt3f8LbkpeUKRZq7gKJFzGSGgZV
* Bitcoin Gold (BTG): Gfk8QbMJFgpMTcY7uB63axy6HU7uTPPWNj
* Basic Attention Token (BAT): 0x6241857fa5fb7667FB7a792b13E83fDEabe96f7F
* Chainlink (LINK): 0x6241857fa5fb7667FB7a792b13E83fDEabe96f7F
* Dash (DASH): Xu2U3Nd3G4hM5TRQUBcP4DHJFzXH93xB84
* Dogecoin (DOGE): DGFBPsRBYL8wHbgnvKbYkVn5FvAe854p1c
* Ethereum (ETH): 0x6241857fa5fb7667FB7a792b13E83fDEabe96f7F
* Filecoin (FIL): f1wpypkjcluufzo74yha7p67nbxepzizlroockgcy
* LBRY Credits (LBC): bFusyoZPkSuzM2Pr8mcthgvkymaosJZt5r
* Litecoin (LTC): LfL7jLNYuVpy7v5TyRyc3yRZ2uhqc4UoR3
* Monero (XMR): 45BBZMrJwPSaFwSoqLVNEggWR2BJJsXxz7bNz8FXnnFo3GyhVJFSCrCFSS7zYwDa9r1TmFmGMxQ2HTntuc11yZ9q1LeCE8f
* Tezos (XTZ): tz1ZxerkbbALsuU9XGV9K9fFpuLWnKAGfc1C
* Zcash (ZEC): t1bjbVBK7tx9EGBrnD2wDfjGV9yZrcyfMmr
* Other crypto currencies: contact me for address

## LINKS

* https://spawn.link
* https://github.com/trapexit/mergerfs
* https://github.com/trapexit/mergerfs/wiki
* https://github.com/trapexit/mergerfs-tools
* https://github.com/trapexit/scorch
* https://github.com/trapexit/bbf
* https://github.com/trapexit/backup-and-recovery-howtos
