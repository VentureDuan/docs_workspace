根据您笔记中的内容以及常见的 RBD 操作，我为您整理了一份更全面的 RBD 客户端命令合集，并附上了详细注释。

### RBD 客户端命令合集

#### 1. **镜像管理**

- **列出存储池中的镜像**

  ```bash
  rbd ls -p <pool-name>
  ```

  **注释**：列出指定存储池中的所有 RBD 镜像。

- **查看镜像详细信息**

  ```bash
  rbd info <pool-name>/<image-name>
  ```

  **注释**：显示镜像的大小、格式、特性、父镜像（如果是克隆的）等详细信息。

- **创建镜像**

  ```bash
  rbd create <pool-name>/<image-name> --size <size-in-MB/G>
  ```

  **注释**：在指定存储池中创建一个指定大小的新镜像。例如：`rbd create data_pool/new.image --size 10240`（创建 10GB 镜像）。

- **删除镜像**

  ```bash
  rbd rm <pool-name>/<image-name>
  ```

  **注释**：删除指定的镜像。如果镜像有快照，需要先删除所有快照。

- **调整镜像大小**

  ```bash
  rbd resize <pool-name>/<image-name> --size <new-size>
  ```

  **注释**：增大或缩小镜像的容量。缩小容量时需谨慎，可能导致数据丢失。

#### 2. **快照管理**

- **创建快照**

  ```bash
  rbd snap create <pool-name>/<image-name>@<snap-name>
  ```

  **注释**：为指定镜像创建一个快照。快照名称通常包含版本号，如 `@v26`。

- **列出镜像的快照**

  ```bash
  rbd snap ls <pool-name>/<image-name>
  ```

  **注释**：显示指定镜像的所有快照列表。

- **保护快照**

  ```bash
  rbd snap protect <pool-name>/<image-name>@<snap-name>
  ```

  **注释**：将快照设置为保护状态，这是克隆镜像的前提条件。保护后的快照不能被删除。

- **取消保护快照**

  ```bash
  rbd snap unprotect <pool-name>/<image-name>@<snap-name>
  ```

  **注释**：解除快照的保护状态，之后才能删除该快照。

- **回滚到快照**

  ```bash
  rbd snap rollback <pool-name>/<image-name>@<snap-name>
  ```

  **注释**：将镜像内容恢复到指定快照的状态。此操作会覆盖镜像的当前数据。

- **删除快照**

  ```bash
  rbd snap rm <pool-name>/<image-name>@<snap-name>
  ```

  **注释**：删除指定的快照。如果快照处于保护状态，需要先执行 `unprotect`。

#### 3. **克隆操作**

- **克隆镜像**

  ```bash
  rbd clone <parent-pool>/<parent-image>@<snap-name> <child-pool>/<child-image>
  ```

  **注释**：基于一个受保护的快照创建一个新的克隆镜像。克隆镜像与父快照是写时复制（COW）关系，节省存储空间。这是您笔记中游戏部署的关键步骤。

- **展平克隆镜像**

  ```bash
  rbd flatten <pool-name>/<image-name>
  ```

  **注释**：如果镜像是克隆而来的，此命令会断开它与父快照的依赖关系，使其成为一个独立的镜像。之后可以删除父快照。

#### 4. **映射与挂载（客户端操作）**

- **映射 RBD 镜像到本地块设备**

  ```bash
  rbd map <pool-name>/<image-name> --id <client-id>
  ```

  **注释**：在 Linux 客户端上将 RBD 镜像映射为一个本地块设备（如 `/dev/rbd0`）。`--id` 指定用于认证的 Ceph 客户端 ID（如 `admin`）。

- **查看已映射的设备**

  ```bash
  rbd showmapped
  ```

  **注释**：列出当前客户端上所有已映射的 RBD 镜像及其对应的本地设备文件。

- **取消映射设备**

  ```bash
  rbd unmap /dev/rbdX
  ```

  **注释**：或使用 `rbd unmap <pool-name>/<image-name>`。断开 RBD 镜像与本地块设备的关联。

- **检查镜像状态**

  ```bash
  rbd status <pool-name>/<image-name>
  ```

  **注释**：检查镜像的当前状态，例如是否被锁定、由哪个客户端使用等。这在排查映射或挂载问题时很有用。

#### 5. **高级与维护命令**

- **启用/禁用镜像特性**

  ```bash
  rbd feature enable/disable <pool-name>/<image-name> <feature-name>
  ```

  **注释**：管理镜像特性，如 `exclusive-lock`（排他锁）、`object-map`（对象映射，用于快速差异操作）等。

- **清理孤儿状态的锁**

  ```bash
  rbd lock list <pool-name>/<image-name>
  rbd lock remove <pool-name>/<image-name> <lock-id> <locker>
  ```

  **注释**：如果客户端异常崩溃导致镜像被锁定，可以列出锁并手动移除。

- **导出/导入镜像**

  ```bash
  rbd export <pool-name>/<image-name> <local-file-path>
  rbd import <local-file-path> <pool-name>/<image-name>
  ```

  **注释**：将镜像导出到本地文件进行备份，或从本地文件导入创建新镜像。

- **差异比较**

  ```bash
  rbd diff <pool-name>/<image-name>
  ```

  **注释**：显示自创建以来镜像中已更改的数据块列表。常用于与快照比较。

### 使用建议
1. **关键操作前先检查**：在执行 `rm`、`resize`、`rollback` 等可能影响数据的操作前，先用 `info`、`snap ls` 等命令确认目标。
2. **克隆工作流**：这是您笔记中的核心流程：`创建基础镜像` -> `创建快照` -> `保护快照` -> `克隆` -> `映射使用`。
3. **权限与认证**：大多数命令需要指定 `--id` 和正确的密钥环文件（通过 `ceph auth get` 获取）。确保客户端有相应存储池的读写权限。

您可以将这些命令补充到您的 [[RBD 命令合集]] 笔记中，形成一个更完整的参考手册。


- mon_host替换成真实ceph节点IP，以逗号分隔多个节点
- keyring文件的内容登录到ceph节点执行指令`ceph auth get client.admin`
- ceph集群管理界面使用https://<node_ip>:8443，用户名admin，密码viu@1234
- cep机器登录密码，用户名root， 密码viu@1234
- cgvmagnet使用部署在Zone侧的ceph_agent，ip信息通常为：Nginx的IP，可在部署文档的ecn信息栏中使用`nginx+pdns`等关键词机型搜索，port为`30194`

```powershell

rbd map data_pool/g8.game.image --id admin

rbd map data_pool/g8.game.image --id cloudgame_storage


rbd status --id admin runtime_pool_v62/sttcmnt.gs18579.v27.vm4.t0.i1


ceph auth get client.admin -o keyring


pdns
admin
admin
http://10.141.194.46:8000/domain/ops.vrviu.com



RBD操作：

查看存储池
rados lspools

查看存储池下所有镜像
rbd ls -p runtime_pool_v1
rbd ls -p data_pool

	创建快照
rbd snap create {pool-name}/{image-name}@{snap-name}

rbd snap create data_pool/gs15806.game.image@v26


将快照设置为保护状态【为后续克隆做准备】
rbd snap protect data_pool/gs15806.game.image@v26

获取镜像信息
rbd info data_pool/gs15806.game.image@v26

克隆镜像
rbd clone data_pool/gs15806.game.image@v26 runtime_pool_v1/sttcmnt.gs15806.v26.vm4.t0.i0

rbd clone data_pool/gs15806.game.image@v26 runtime_pool_v1/sttcmnt.gs15806.v26.vm4.t0.i1

rbd status --id admin runtime_pool_v1/sttcmnt.gs15806.v26.vm4.t0.i0

rbd status --id admin runtime_pool_v1/sttcmnt.gs15806.v26.vm4.t0.i1

rbd map --id admin runtime_pool_v1/sttcmnt.gs15806.v26.vm4.t0.i0

rbd map --id admin runtime_pool_v1/sttcmnt.gs15806.v26.vm4.t0.i1


ceph-dokan.exe  -l x 

c:/ProgramData/Ceph/ceph.conf

```

```powershell
robocopy $dstM $dstN /E /COPY:DAT /MIR /R:1 /W:1 /MT:16 /NFL /NDL /NP /LOG+:"$logDir"
```

```

http://dal.ops.vrviu.com:40082/v1/t_game_virtual_machine_game_pool_info?offset=0&limit=1&query=vmid:3518

10.141.194.46

http://10.141.194.46:40082/v1/t_game_virtual_machine_game_pool_info?offset=0&limit=1&query=vmid:3518

```

1、打开命令行窗口，输入DiskPart  
2、运行命令List Disk，列出服务器加载的所有硬盘  
3、运行命令Select Disk (硬盘编号)，选中需进行修改的硬盘  
4、运行命令ATTRIBUTES DISK CLEAR READONLY，清除磁盘只读属性  
5、编辑完成就可对该硬盘进行正常读写操作。



adb connect 10.0.5.103

adb -s 10.0.5.103 shell