## 简化游戏部署：一个vm只需要挂载一块Ceph盘，不需要区分游戏盘和恢复盘

### 方案一：为克隆镜像创建快照并回滚

#### 1、在克隆镜像上创建一个新的快照

`rbd snap create runtime_pool_v1/sttcmnt.gs15806.v26.vm4.t0.i0@snap.gs15806.v26.vm4`

#### 2、在客户端卸载并取消映射

#### 3、执行回滚命令

`rbd snap rollback runtime_pool_v1/sttcmnt.gs15806.v26.vm4.t0.i0 --snap snap.gs15806.v26.vm4`

### 方案二：删除并重新克隆

#### 1、在客户端卸载并取消映射

```powershell
rbd-wnbd unmap runtime_pool_v1/sttcmnt.gs15806.v26.vm4.t0.i0
```

#### 2、删除当前的克隆镜像

```powershell
rbd rm runtime_pool_v1/sttcmnt.gs15806.v26.vm4.t0.i0
```

#### 3、重新从父快照克隆一个新的镜像

```powershell
rbd clone data_pool/gs15806.game.image@v26 runtime_pool_v1/sttcmnt.gs15806.v26.vm4.t0.i0
```

#### 4、在客户端重新映射并挂载

```powershell

# Windows
rbd-wnbd map runtime_pool_v1/sttcmnt.gs15806.v26.vm4.t0.i0
# 然后使用 Set-Disk, Initialize-Disk, New-Partition, Format-Volume 等命令初始化（如果需要）

```