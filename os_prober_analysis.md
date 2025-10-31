# os-prober脚本功能分析报告

## 1. 脚本整体结构和目的

### 脚本基本信息
- **文件**: `/usr/bin/os-prober`
- **语言**: Bash shell脚本
- **主要功能**: 检测系统中安装的操作系统

### 脚本目的
os-prober是一个用于检测硬盘分区中已安装操作系统的工具，它是Linux系统中GRUB引导程序的重要组件。它的主要目的是：
- 扫描硬盘分区查找已安装的操作系统
- 识别操作系统的类型、版本和挂载点
- 为GRUB等引导程序提供操作系统列表信息

### 脚本入口流程
1. 导入公共函数库 `/usr/share/os-prober/common.sh`
2. 调用 `newns "$@"` - 进入新的命名空间
3. 创建临时目录
4. 设置日志输出函数
5. 初始化dmraid映射表

## 2. 主要函数功能分析

### 2.1 log_output函数
```bash
log_output () {
    if type log-output >/dev/null 2>&1; then
        log-output -t os-prober --pass-stdout $@
    else
        $@
    fi
}
```
**功能**: 提供统一的日志输出机制，支持管道重定向

### 2.2 on_sataraid函数
```bash
on_sataraid () {
    local parent="${1%/*}"
    local device="/dev/${parent##*/}"
    if grep -q "$device" "$OS_PROBER_TMP/dmraid-map"; then
        return 0
    fi
    return 1
}
```
**功能**: 检查设备是否为SATA RAID成员
**参数**: 设备路径
**返回值**: 0(是RAID)或1(非RAID)

### 2.3 partitions函数
**功能**: 获取系统中所有可用分区的列表
**处理逻辑**:
- 排除whole_disk属性设置的分区
- 排除属于SATA RAID的分区
- 添加Serial ATA RAID设备
- 检测LVM卷上的系统
- 支持GNU/Linux以外的操作系统

### 2.4 parse_proc_swaps函数
**功能**: 解析 `/proc/swaps` 文件，提取swap设备映射信息
**输出格式**: 设备路径 + " swap"标识

### 2.5 parse_proc_mdstat函数
**功能**: 解析 `/proc/mdstat` 文件，识别RAID成员设备
**输出格式**: 设备路径列表

## 3. 分区检测逻辑分析

### 3.1 分区枚举流程
脚本通过`partitions()`函数实现分区检测：

1. **Linux系统检测**
   ```bash
   if [ -d /sys/block ]; then
       # 检测常规分区
       for part in /sys/block/*/*[0-9]; do
           if [ -f "$part/start" ] && \
              [ ! -f "$part/whole_disk" ] && ! on_sataraid $part; then
               # 处理分区
           fi
       done
   ```

2. **RAID设备检测**
   ```bash
   # 检测Serial ATA RAID设备
   if type dmraid >/dev/null 2>&1 && \
      dmraid -s -c >/dev/null 2>&1; then
       for raidset in $(dmraid -sa -c); do
           for part in /dev/mapper/"$raidset"*[0-9]; do
               echo "$part"
           done
       done
   fi
   ```

3. **LVM卷检测**
   ```bash
   # 检测LVM卷上的操作系统
   if type lvs >/dev/null 2>&1; then
       echo "$(LVM_SUPPRESS_FD_WARNINGS=1 log_output lvs --noheadings --separator : -o vg_name,lv_name |
           sed "s|-|--|g;s|^[[:space:]]*\(.*\):\(.*\)$|/dev/mapper/\1-\2|")"
   fi
   ```

### 3.2 分区过滤规则
- 排除设置whole_disk属性的分区
- 排除属于SATA RAID的分区
- 排除已经在软件RAID中的设备

## 4. RAID检测和处理

### 4.1 dmraid集成
- 扫描dmraid配置的RAID设备
- 生成dmraid映射表存储在`$OS_PROBER_TMP/dmraid-map`

### 4.2 软件RAID检测
通过解析`/proc/mdstat`识别MD RAID成员：
```bash
parse_proc_mdstat () {
    while read line; do
        for word in $line; do
            dev="${word%%\[*}"
            # 检查设备路径格式
            # 输出RAID设备路径
        done
    done
}
```

## 5. 操作系统检测流程

### 5.1 检测阶段1：初始化
1. **执行初始化脚本**
   ```bash
   for prog in /usr/lib/os-probes/init/*; do
       if [ -x "$prog" ] && [ -f "$prog" ]; then
           "$prog" || true
       fi
   done
   ```

2. **构建系统映射表**
   - 挂载点映射：`/proc/mounts` → `$OS_PROBER_TMP/mounted-map`
   - 交换分区映射：`/proc/swaps` → `$OS_PROBER_TMP/swaps-map`
   - RAID设备映射：`/proc/mdstat` → `$OS_PROBER_TMP/raided-map`

### 5.2 检测阶段2：分区扫描
**主循环逻辑**：
```bash
for partition in $(partitions); do
    # 1. 设备映射验证
    if ! mapped="$(mapdevfs "$partition")"; then
        continue
    fi
    
    # 2. 跳过RAID成员
    if grep -q "^$mapped" "$OS_PROBER_TMP/raided-map" ; then
        continue
    fi
    
    # 3. 跳过活动交换分区
    if grep -q "^$mapped " "$OS_PROBER_TMP/swaps-map" ; then
        continue
    fi
    
    # 4. 执行操作系统检测
    if ! grep -q "^$mapped " "$OS_PROBER_TMP/mounted-map" ; then
        # 未挂载分区检测
        for test in /usr/lib/os-probes/*; do
            "$test" "$partition" && break
        done
    fi
done
```

### 5.3 检测阶段3：挂载分区检测
对于已挂载的分区：
```bash
if [ "$mpoint" != "/target/boot" ] && [ "$mpoint" != "/target" ] && [ "$mpoint" != "/" ]; then
    type=$(grep "^$mapped " "$OS_PROBER_TMP/mounted-map" | head -n1 | cut -d " " -f 3)
    for test in /usr/lib/os-probes/mounted/*; do
        if [ -f "$test" ] && [ -x "$test" ]; then
            if "$test" "$partition" "$mpoint" "$type"; then
                break
            fi
        fi
    done
fi
```

### 5.4 新命名空间检测(newns模式)
针对挂载在`/mnt/`或`/media/`的分区，采用特殊处理：
```bash
if [ -n "$OS_PROBER_NEWNS" ] && { [ "${mpoint#/mnt/}" != "$mpoint" ] || [ "${mpoint#/media/}" != "$mpoint" ]; }; then
    debug "using unmounted detection for newns partition: $partition at $mpoint"
    # 使用非挂载检测模式
    for test in /usr/lib/os-probes/*; do
        if "$test" "$partition"; then
            break
        fi
    done
fi
```

**特殊处理原因**: 解决符号链接指向本地根目录导致版本信息错误的问题

## 6. 挂载点和文件系统处理

### 6.1 挂载点映射构建
- 解析`/proc/mounts`获取当前挂载信息
- 生成映射表格式：`设备路径 挂载点 文件系统类型`

### 6.2 挂载点过滤
排除系统关键挂载点：
- `/target/boot` - 目标系统boot分区
- `/target` - 目标系统根分区
- `/` - 当前系统根分区

### 6.3 文件系统类型识别
从挂载映射表中提取文件系统类型，传递给检测脚本

## 7. 脚本核心功能总结

### 7.1 主要功能
1. **系统分区发现**: 自动发现系统中所有可用分区
2. **RAID支持**: 支持硬件RAID(dmraid)和软件RAID(mdadm)
3. **LVM支持**: 检测LVM卷上的操作系统
4. **多操作系统检测**: 支持检测Linux、Windows、macOS等多种操作系统
5. **挂载状态检测**: 区分已挂载和未挂载分区，采用不同检测策略

### 7.2 技术特点
- **模块化设计**: 使用外部检测脚本在`/usr/lib/os-probes/`目录
- **命名空间隔离**: 通过newns实现进程隔离
- **错误容错**: 完善的错误处理和异常情况处理
- **日志支持**: 完整的调试和日志输出机制

### 7.3 输出格式
脚本通过外部检测脚本生成标准格式的输出：
```
设备路径:操作系统名称:操作系统版本:挂载点:类型
```

### 7.4 应用场景
- GRUB引导程序更新时的操作系统检测
- 系统安装程序中的多系统检测
- 磁盘管理工具中的系统信息获取
- 备份工具中的分区识别

这是一个成熟、健壮的系统工具，为Linux多系统环境提供了可靠的操作系统检测能力。
