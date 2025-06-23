---
author: Sat Naing
pubDatetime: 2025-06-23T11:17:00Z
title: 基于XtraBackup的Docker化MySQL备份与恢复权威指南
slug: dockerized-mysql-backup-and-recovery-guide-with-xtrabackup
featured: false
draft: false
tags:
  - docs
description: 基于XtraBackup的Docker化MySQL备份与恢复权威指南
---

## 方案概述

本指南提供了一个基于 Percona XtraBackup 的、企业级的 Docker 化 MySQL 备份与恢复方案。该方案采用 **Sidecar（边车）模式**，将备份逻辑与主数据库服务分离，实现了高内聚、低耦合的系统架构，确保了备份过程的安全、高效与可靠。

**核心特性:**

- **物理备份**: 利用 XtraBackup 进行在线热备，对生产环境性能影响极小。
- **全量 + 增量**: 支持全量备份与增量备份，在保证恢复速度的同时，有效节省存储空间。
- **跨平台兼容**: 方案经过验证，可同时在 Linux, macOS 及 Windows (通过WSL 2) 环境下稳定运行。
- **自动化**: 提供清晰的自动化路径，可与 `cron` (Linux/macOS) 或任务计划程序 (Windows) 无缝集成。


## 步骤 1: 部署环境 (`docker-compose.yml`)
首先，在您的项目根目录下创建 `docker-compose.yml` 文件。这个文件定义了主数据库服务 (`rcbp-mysql`) 和专门用于备份的 Sidecar 服务 (`rcbp-mysql-backup`)。

```yaml
version: '3.7'

services:
  rcbp-mysql:
    # 使用一个具体的、兼容性更好的版本号来避免硬件不兼容问题
    image: mysql:5.7.44
    container_name: rcbp-mysql
    restart: always
    # 将服务加入网络，以便sidecar容器可以通过服务名访问它
    networks:
      - rcbp-network
    environment:
      - MYSQL_ROOT_PASSWORD=my_password
      - TZ=Asia/Shanghai
    volumes:
      # 注意：使用相对路径 `./` 确保了项目的可移植性
      - ./data/mysql/log:/var/log/mysql
      - ./data/mysql/data:/var/lib/mysql
      - ./data/mysql/conf:/etc/mysql/conf.d
    ports:
      - 3306:3306
    command: 
      --default-authentication-plugin=mysql_native_password
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_general_ci
      --explicit_defaults_for_timestamp=true
      --lower_case_table_names=1
      --max_allowed_packet=128M
      --sql-mode="STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION"

  # --- 新增的备份Sidecar服务 ---
  rcbp-mysql-backup:
    # 使用一个具体的、兼容性更好的版本号来解决CPU不支持x86-64-v2的问题
    image: percona/percona-xtrabackup:2.4.28
    container_name: rcbp-mysql-backup
    networks:
      - rcbp-network
    volumes:
      # 以只读方式挂载 MySQL 数据目录，这是物理备份所必需的
      - ./data/mysql/data:/var/lib/mysql:ro
      # 挂载一个用于存放所有备份文件的目录
      - ./data/mysql_backups:/backups
      # 将我们的备份脚本挂载到容器的可执行路径中
      - ./backup.sh:/usr/bin/backup.sh
    environment:
      # 将密码和时区作为环境变量传入
      - MYSQL_ROOT_PASSWORD=my_password
      - TZ=Asia/Shanghai
    # 使用此命令让容器保持运行，以便我们随时可以通过 exec 执行备份命令
    command: ["sleep", "infinity"]

# 定义共享网络
networks:
  rcbp-network:
    driver: bridge
```

> ***重要提示：关于 `CPU does not support x86-64-v2` 错误***
>
> - **错误原因**: 这个错误表示您宿主机的CPU硬件比较旧，不支持 `x86-64-v2` 指令集。而您尝试拉取的Docker镜像（例如 `mysql:5.7` 或 `percona/percona-xtrabackup:2.4` 的最新构建版本）在编译时要求CPU必须支持此新特性，导致无法运行。
> - **解决方案**: 我们已将 `docker-compose.yml` 中的镜像标签修改为具体的、已知的、在旧版编译环境中构建的版本（`mysql:5.7.44` 和 `percona/percona-xtrabackup:2.4.28`），这可以解决在旧硬件上的兼容性问题。

*在启动前，请确保在项目根目录下已手动创建 `./data/mysql/` 和 `./data/mysql_backups/` 文件夹。*

## 步骤 2: 创建核心备份脚本 (`backup.sh`)

在与 `docker-compose.yml` 文件相同的目录下，创建 `backup.sh` 文件。这是整个方案的核心，其中包含了最终修正的、最可靠的逻辑，包括加固后的自动清理功能。

```sh
#!/bin/bash
# 任何命令执行失败则立即退出脚本
set -e

# --- 配置区域 ---
MYSQL_PASSWORD="${MYSQL_ROOT_PASSWORD:-my_password}" 
MYSQL_USER="root"
MYSQL_HOST="rcbp-mysql" 
BACKUP_BASE_DIR="/backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# --- 备份保留策略配置 ---
# 定义您希望保留的完整备份周期的数量。例如，设置为2，将始终保留最新的2个全量备份及其相关的增量备份。
FULL_BACKUPS_TO_KEEP=2

# --- 内部函数定义 ---

# 显示脚本用法
function show_usage() {
    echo "A robust backup script for XtraBackup (Final Version with Cleanup)."
    echo ""
    echo "Usage: $0 [full|incremental|prepare|cleanup]"
    echo "  full         - Perform a full backup and automatically clean up old backups."
    echo "  incremental  - Perform an incremental backup based on the latest backup."
    echo "  prepare      - Prepares the latest full backup and all subsequent incrementals for restoration."
    echo "  cleanup      - Manually trigger the cleanup of old backups."
}

# --- 新增：清理旧备份的函数 (最终加固版) ---
function perform_cleanup() {
    echo ">>> [$(date)] Starting cleanup of old backups..."
    
    # 获取所有全量备份目录，按名称（时间）正序排序
    ALL_FULL_BACKUPS=$(find "${BACKUP_BASE_DIR}" -mindepth 1 -maxdepth 1 -type d -name "*_full" | sort)
    
    # 计算有多少个全量备份
    BACKUP_COUNT=$(echo "${ALL_FULL_BACKUPS}" | wc -l)
    
    if [ ${BACKUP_COUNT} -le ${FULL_BACKUPS_TO_KEEP} ]; then
        echo ">>> Number of full backups (${BACKUP_COUNT}) is less than or equal to retention number (${FULL_BACKUPS_TO_KEEP}). No cleanup needed."
        return
    fi
    
    # 获取需要删除的全量备份列表 (除了最新的N个之外的所有备份)
    FULL_BACKUPS_TO_DELETE=$(echo "${ALL_FULL_BACKUPS}" | head -n -$((FULL_BACKUPS_TO_KEEP)))

    for full_backup_to_delete in ${FULL_BACKUPS_TO_DELETE}; do
        # 获取这个待删除备份的下一个全量备份，以确定清理范围的上限
        next_full_backup=$(echo "${ALL_FULL_BACKUPS}" | grep -A 1 -F "${full_backup_to_delete}" | tail -n 1)
        
        # 获取所有备份目录，并按名称（时间）排序
        all_dirs_sorted=$(find "${BACKUP_BASE_DIR}" -mindepth 1 -maxdepth 1 -type d | sort)
        
        echo "--- Cleaning up backup cycle starting with ${full_backup_to_delete}"

        # 遍历所有已排序的目录，找到位于两个全量备份之间的增量备份
        for dir in ${all_dirs_sorted}; do
            # 检查当前目录是否在待删除的全量备份和下一个全量备份之间
            if [[ "${dir}" > "${full_backup_to_delete}" ]] && \
               ([[ "${dir}" < "${next_full_backup}" ]] || [[ "${full_backup_to_delete}" == "${next_full_backup}" ]]); then
                # `[[ "${full_backup_to_delete}" == "${next_full_backup}" ]]` 处理了正在删除最后一个备份周期的情况
                
                # 确认这是一个增量备份目录后才删除
                if [[ "${dir}" == *"_inc" ]]; then
                    echo "--- Deleting associated incremental backup: ${dir}"
                    rm -rf "${dir}"
                fi
            fi
        done
        
        # 最后，删除旧的全量备份目录本身
        echo "--- Deleting old full backup: ${full_backup_to_delete}"
        rm -rf "${full_backup_to_delete}"
        echo "---"
    done

    echo ">>> Cleanup completed."
}

# --- 执行全量备份函数 ---
function perform_full_backup() {
    echo ">>> [$(date)] Starting full backup..."
    local FULL_BACKUP_DIR="${BACKUP_BASE_DIR}/${TIMESTAMP}_full"
    mkdir -p "${FULL_BACKUP_DIR}"
    
    xtrabackup --backup \
               --user="${MYSQL_USER}" \
               --password="${MYSQL_PASSWORD}" \
               --host="${MYSQL_HOST}" \
               --target-dir="${FULL_BACKUP_DIR}"
               
    echo ">>> Full backup completed successfully in: ${FULL_BACKUP_DIR}"
    
    # 每次全备后，自动调用清理函数
    perform_cleanup
}

# --- 执行增量备份函数 ---
function perform_incremental_backup() {
    echo ">>> [$(date)] Starting incremental backup..."
    # 找到最新的一个备份目录（无论是全量还是增量）作为基础
    LATEST_BACKUP=$(find "${BACKUP_BASE_DIR}" -mindepth 1 -maxdepth 1 -type d -name "*_*" | sort -r | head -n 1)
    
    if [ -z "${LATEST_BACKUP}" ]; then
        echo "Error: No previous backup found. Please perform a full backup first."
        exit 1
    fi
    
    echo ">>> Base for this incremental backup is: ${LATEST_BACKUP}"
    local INCREMENTAL_BACKUP_DIR="${BACKUP_BASE_DIR}/${TIMESTAMP}_inc"
    mkdir -p "${INCREMENTAL_BACKUP_DIR}"

    xtrabackup --backup \
               --user="${MYSQL_USER}" \
               --password="${MYSQL_PASSWORD}" \
               --host="${MYSQL_HOST}" \
               --target-dir="${INCREMENTAL_BACKUP_DIR}" \
               --incremental-basedir="${LATEST_BACKUP}"

    echo ">>> Incremental backup completed successfully in: ${INCREMENTAL_BACKUP_DIR}"
}

# --- 准备备份以供恢复 (最终修正版) ---
function prepare_backups() {
    echo ">>> [$(date)] Preparing backups for restore..."
    
    # 找到最新的全量备份作为恢复的基础
    LATEST_FULL=$(find "${BACKUP_BASE_DIR}" -mindepth 1 -maxdepth 1 -type d -name "*_full" | sort -r | head -n 1)

    if [ -z "${LATEST_FULL}" ]; then
        echo "Error: No full backup found to prepare. Exiting."
        exit 1
    fi
    echo ">>> Found latest full backup to use as base: ${LATEST_FULL}"
    
    echo "---"
    echo "--- Step 1: Applying logs to base full backup (${LATEST_FULL}) with --apply-log-only ---"
    xtrabackup --prepare --apply-log-only --target-dir="${LATEST_FULL}"
    
    # 查找所有目录，然后筛选出真正的增量备份并排序
    ALL_DIRS=$(find "${BACKUP_BASE_DIR}" -mindepth 1 -maxdepth 1 -type d | sort)
    INCREMENTALS=""
    
    # 标志位，表示是否在全量备份之后
    found_full=false
    for dir in ${ALL_DIRS}; do
        if [[ "${dir}" == "${LATEST_FULL}" ]]; then
            found_full=true
            continue
        fi

        if [[ "${found_full}" == "true" ]]; then
            # 通过检查文件内容来确认这确实是一个增量备份
            if [ -f "${dir}/xtrabackup_checkpoints" ] && grep -q 'backup_type.*incremental' "${dir}/xtrabackup_checkpoints"; then
                INCREMENTALS="${INCREMENTALS}${dir} "
            fi
        fi
    done
    
    if [ -z "${INCREMENTALS}" ]; then
        echo ">>> No incremental backups found for this cycle. Proceeding to final prepare."
    else
        echo ">>> Found the following incremental backups to apply:"
        echo "${INCREMENTALS}"
        
        for inc in ${INCREMENTALS}; do
            echo "---"
            echo "--- Step 2: Applying incremental backup: ${inc} ---"
            xtrabackup --prepare --apply-log-only \
                       --target-dir="${LATEST_FULL}" \
                       --incremental-dir="${inc}"
        done
    fi
    
    echo "---"
    echo "--- Step 3: Finalizing preparation on ${LATEST_FULL} (final log application) ---"
    xtrabackup --prepare --target-dir="${LATEST_FULL}"

    echo ">>> Preparation complete. Data in ${LATEST_FULL} is now ready to be restored."
}


# --- 主逻辑入口 ---
case "$1" in
    full)
        perform_full_backup
        ;;
    incremental)
        perform_incremental_backup
        ;;
    prepare)
        prepare_backups
        ;;
    cleanup) # 新增手动清理入口
        perform_cleanup
        ;;
    *)
        show_usage
        exit 1
        ;;
esac
```

> ***重要提示:***
>
> - **对于Linux/macOS用户**: 创建此文件后，请务必在宿主机上执行 `chmod +x backup.sh` 命令，为脚本赋予执行权限。
> - **对于Windows用户**: 请确保此脚本以 `LF` 换行符格式保存，而非 `CRLF`。

## 步骤 3: 日常备份与灾难恢复流程

#### 启动服务

在项目根目录运行 `docker-compose up -d`。

#### 日常备份操作

- **执行全量备份**: `docker-compose exec rcbp-mysql-backup backup.sh full`
- **执行增量备份**: `docker-compose exec rcbp-mysql-backup backup.sh incremental`

#### 灾难恢复流程

当需要恢复数据时，请严格按照以下顺序操作：

**1，准备数据 (`Prepare`)**: **这是恢复前最关键的一步，且只能对一个备份周期执行一次。** 它会将所有相关的增量备份合并到最新的全量备份中。

```sh
docker-compose exec rcbp-mysql-backup backup.sh prepare
```

**2，停止数据库并清空数据目录**:

```sh
docker-compose stop rcbp-mysql
# 在项目根目录下执行
rm -rf ./data/mysql/data/*
```

**3，恢复文件 (`Copy-back`)**: 找到 `prepare` 步骤中准备好的全量备份目录名，并执行以下命令。

```sh
# 使用绝对路径来避免 Docker 解析错误，并增加了 --datadir 参数。
# 请将 target-dir 后的目录名替换为您实际准备好的备份目录名。
docker run --rm --user root \
  -v "$(pwd)/data/mysql_backups:/backups" \
  -v "$(pwd)/data/mysql/data:/var/lib/mysql" \
  percona/percona-xtrabackup:2.4.28 \
  xtrabackup --copy-back --target-dir=/backups/YOUR_PREPARED_FULL_BACKUP_DIR --datadir=/var/lib/mysql
```

**4，修复文件权限**: **这是保证MySQL能正常启动的强制步骤。**

```sh
# 在Linux/macOS上执行:
sudo chown -R 999:999 ./data/mysql/data
```

**5，重启数据库服务**:

```sh
docker-compose start rcbp-mysql
```

## 步骤 4: 自动化与备份策略

#### 自动化任务设置

在您的**宿主机**上使用计划任务工具来定时执行备份命令。

**Linux / macOS 用户**: 使用 `cron`。运行 `crontab -e` 并添加类似以下内容：

```sh
# CRON 任务示例 (最终修正版)
# 增加了 -T 参数以解决 "not a TTY" 错误，并使用 docker-compose 的绝对路径。

# 定义项目和docker-compose的路径，方便管理
PROJECT_PATH=/path/to/your/compose/project
# !! 重要: 替换为您的 docker-compose 的真实路径
DOCKER_COMPOSE_PATH=/usr/local/bin/docker-compose 

# 每周日凌晨 2:00 执行一次全量备份 (并自动清理旧备份)
0 2 * * 0 cd $PROJECT_PATH && $DOCKER_COMPOSE_PATH exec -T rcbp-mysql-backup backup.sh full >> /var/log/mysql_backup.log 2>&1

# 每天凌晨 3:00 执行一次增量备份 (周一至周六)
0 3 * * 1-6 cd $PROJECT_PATH && $DOCKER_COMPOSE_PATH exec -T rcbp-mysql-backup backup.sh incremental >> /var/log/mysql_backup.log 2>&1
```

**如何找到 `docker-compose` 的绝对路径？**

在您的服务器终端中运行 `which docker-compose` 命令。它会输出类似 `/usr/local/bin/docker-compose` 的路径，请将这个路径用于上面的 `DOCKER_COMPOSE_PATH` 变量。这样做可以确保 `cron` 在其受限的环境中也能准确找到并执行命令。



**Windows 用户**: 使用 **任务计划程序 (Task Scheduler)**。创建一个新任务，操作设置为“启动程序”，程序为 `powershell.exe`，参数为 `cd D:\path\to\project; docker-compose exec -T rcbp-mysql-backup backup.sh full`。



#### 备份策略建议

选择合适的备份频率是在**可接受的数据丢失量 (RPO)**、**恢复时间 (RTO)** 和 **存储成本** 之间的权衡。以下是一些常见的策略，您可以根据业务需求选择。

| 业务场景               | 全量备份频率                   | 增量备份频率                         | 优点                                                         | 缺点                                          |
| ---------------------- | ------------------------------ | ------------------------------------ | ------------------------------------------------------------ | --------------------------------------------- |
| **标准/通用型 (推荐)** | **每周一次**  (例如，周日凌晨) | **每天一次**  (例如，周一至周六凌晨) | 平衡了恢复时间和存储成本，恢复时最多应用6个增量包，RPO为24小时。 | 每天有最高24小时的数据丢失风险。              |
| **核心业务/高流量型**  | **每天一次**  (例如，每日凌晨) | **每小时一次**  或 **每4小时一次**   | 数据丢失风险极低(RPO为1小时)，恢复速度非常快(备份链条短)。   | 存储空间消耗大，对系统负载更频繁。            |
| **低更新/归档型**      | **每月一次**                   | **每周一次**                         | 存储成本和系统负载都非常低。                                 | 数据丢失风险大(RPO为一周)，恢复时间可能较长。 |

**我们强烈建议您从“标准/通用型”策略开始**，在运行一段时间后，根据您的数据变化量和对数据丢失的容忍度，再决定是否需要调整备份间隔。

## 附录

### 附录 A: Windows用户特别说明

本方案在Windows上通过Docker Desktop (WSL 2)可完美运行，请注意以下几点：

- **执行权限 (`chmod`)**: **此为Linux/macOS下的必需步骤**。在Windows上通常可以跳过 `chmod +x backup.sh` 这一步。如果遇到权限问题，请使用 **Git Bash** 来执行此命令。
- **路径变量**: 在PowerShell中，请使用 `${pwd}` 来代替 `$(pwd)` 获取当前绝对路径。
- **`sudo`**: Windows上没有 `sudo` 命令，执行 `docker run` 时请直接去掉。
- **文件权限修复 (`chown`)**: 这是最特殊的一步。您需要在PowerShell或CMD中输入 `wsl` 进入您的Linux子系统，然后在子系统环境中对挂载的目录执行 `chown`。例如：`sudo chown -R 999:999 /mnt/d/path/to/your/project/data/mysql/data`。

### 附录 B: 备份与恢复生命周期说明

为了确保备份策略的长期稳定和可靠，理解备份的生命周期至关重要。

#### 恢复后为什么必须进行一次新的全量备份？

**这是一个强制性的最佳实践。**

- **备份链断裂**: 您的备份系统依赖一个基于日志序列号（LSN）的连续“链条” (`全量 -> 增量1 -> 增量2 ...`)。当您将数据库恢复到过去某个时间点时，这个链条就已经断裂了。恢复后的数据库，其历史与发生灾难前的数据库不再连续。
- **建立新的恢复基线**: 成功恢复并验证数据无误后，您的首要任务是**立即**对这个“新生”的、健康的数据库执行一次全新的全量备份 (`backup.sh full`)。这将为未来的所有增量备份创建一个全新的、可靠的起点，开启一个完整、有效的新备份周期。

#### 何时删除旧的备份？

**请勿在恢复成功后立即删除用于恢复的备份集。** 正确的做法是基于**“备份保留策略 (Retention Policy)”**进行自动化清理。

我们已将此策略内置于 `backup.sh` 脚本中。

- **自动清理**: 每次成功执行一次新的全量备份 (`backup.sh full`)后，脚本会自动调用清理函数。
- **可配置的保留策略**: 您可以通过修改 `backup.sh` 脚本顶部的 `FULL_BACKUPS_TO_KEEP=2` 变量来调整保留策略。例如，设置为 `2` 将始终保留最近的2个完整备份周期。
- **手动清理**: 如果需要，您也可以随时手动执行 `docker-compose exec -T rcbp-mysql-backup backup.sh cleanup` 来触发一次清理。

### 附录 C: 最终建议

**测试，测试，再测试**: 未经测试的备份等于没有备份。请务必定期在测试环境中演练您的恢复流程。

**异地容灾**: 为了抵御硬件故障等物理灾难，请务必将您的备份目录 (`./data/mysql_backups`) 定期同步到另一台物理服务器或云存储（如阿里云OSS）。

### 附录 D: 跨平台传输备份文件的最佳实践

当您需要将在一台Linux机器上创建的备份，复制到另一台Windows机器上进行恢复或验证时，**请务必遵循以下流程，以避免文件损坏。**

**核心原则：解压操作必须在一个能够正确处理Linux文件权限的环境中进行。**

#### 错误的做法 (高风险)

**绝对不要**在目标Windows机器上用 `WinRAR`, `7-Zip` 或其他图形化解压工具来解压 `.tar.gz` 包。这极有可能导致文件权限丢失或内容损坏，最终使恢复失败。

#### 正确的流程

**1，在源机器 (A机, Linux) 上压缩**: 进入您的备份目录 (`./data/mysql_backups/`)，将需要传输的整个备份周期打包。

```sh
# 将一个全量包和一个增量包打包
tar -zcvf mysql_backup_cycle.tar.gz 2025..._full/ 2025..._inc/
```

**2，传输文件**: 通过任何方式将 `mysql_backup_cycle.tar.gz` 文件复制到目标机器（B机, Windows）的某个临时目录下，例如 `D:\temp_backups`。

**3，在目标机器 (B机, Windows) 上解压**:

**使用Docker解压** 这是最可靠、最干净的方法，因为它利用了您已有的Docker环境来确保100%的兼容性，无需安装任何额外软件。

a.  打开PowerShell或CMD。

b.  使用以下 `docker run` 命令来启动一个临时的Linux容器完成解压工作。

```powershell
# 请将下面的 "D:\temp_backups" 和 "D:\path\to\project\data\mysql_backups" 替换为您真实的Windows绝对路径
# 在CMD中，请将行尾的 ^ 替换为 \
docker run --rm ^
  -v "D:\temp_backups:/source" ^
  -v "D:\path\to\project\data\mysql_backups:/destination" ^
  alpine ^
  tar -zxvf /source/mysql_backup_cycle.tar.gz -C /destination
```

