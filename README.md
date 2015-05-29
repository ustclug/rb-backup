# rb-backup

一个简单的远程机器增量备份脚本。大概思路是这样的（运行这个脚本并存储备份文件的机器叫 Server，待备份的机器叫 Client）：
* Server 通过 rsync over SSH 从 Client 抓要备份的文件，存储到一个 btrfs subvolume 中。
* 备份后，Server 为存放备份文件的 subvolume 创建一个 snapshot，这样就可以记录历史备份了。

## Client 配置

创建一个备份用的用户，该用户至少要对待备份文件有读权限。配置 SSH，使得我们可以从 Server 上用密钥登录该用户并执行 rsync。

## Server 配置

`common.conf` 是应用到所有 profile 的全局配置文件。

`profiles/<profiles name>.conf` 是每个 profile 单独的配置文件。

配置文件格式可以看看例子：一行设置一个变量，`变量名=值` 这样。不是用 Bash 解析的，所以不要在值周围添加多余的引号。路径有空格也不需要转义。

配置文件中有用的变量有：
* `SERVER`: 必需，Client 的地址或域名
* `USER`: 必需，SSH 登录 Client 的用户名
* `SRC`: 必需，Client 上要备份文件的路径，可以重复设置多次，以设置同步多个路径
* `STORAGE`: 必需，建议放在全局配置中，Server 上存放备份文件的本地文件夹，必须在 btrfs 分区上，且执行该脚本的用户可以读写
* `LOG_DIR`: 存放日志的目录（默认是 `log/<profile name>` ）
* `SSH_KEY`: SSH 登录用的私钥（默认是 `keys/${USER}@${SERVER}`），该文件必须存在
* `KEEP_DAYS`: 短期备份的保留天数（默认为 30）
* `KEEP_LONG_COUNT`: 最多保留多少长期备份（默认为 4）
* `RSYNC_OPTS`: RSYNC 同步参数（默认值为 `-aHAX --timeout 3600 --delete`）

备份时，执行：
```
./backup <profile name>
```

要定期执行的话，放进 crontab 即可。

### Server 对备份文件的存放

该脚本会在 `${STORAGE}/<profile name>/` 文件夹中存放每个 profile 的备份文件。

第一次使用时，脚本会在上述文件夹下创建名为 `current` 的 btrfs subvolume。以后每次执行同步，脚本都会用 rsync 把最新的备份文件同步进这个 subvolume。

每次成功执行完 rsync 后，脚本就会为 `current` 目录创建一个只读的 snapshot，名为 `L_<epoch>` 或 `S_<epoch>`，其中 `<epoch>` 代表备份文件时的 UNIX 时间。
* `S_<epoch>` 是一个短期备份，过了 `${KEEP_DAYS}` 天就会被删除。
* `L_<epoch>` 是一个长期备份，每隔 `${KEEP_DAYS}` 会创建一个。同时会保留 `${KEEP_LONG_COUNT}` 个长期备份，旧的会被删除。

注意，btrfs 默认挂载参数不允许普通用户删除 subvolume。必须加上 `user_subvol_rm_allowed` 参数才行。

