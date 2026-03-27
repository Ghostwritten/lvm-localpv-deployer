# 使用指南

本指南覆盖各脚本的参数、变量及在线/离线部署路径，便于在生产或离线环境中快速复用。

## 前置条件与依赖
- Kubernetes v1.23+ 集群，`kubectl` 能正常访问。
- `helm` 3.x、`kubectl`、`curl`、`envsubst`。离线镜像下载/加载需要 Docker 或 Podman。
- 可选工具：`jq` 用于 JSON 处理，`yq` 用于解析本地 Chart 版本。

## 脚本概览
- `deploy/scripts/download.sh`
  - 作用：下载 Helm chart 与相关镜像，生成 `offline-media/` 目录，可选 `--pack` 打成 tar.gz。
  - 主要参数：`--chart-version`、`--output-dir`、`--pack`、`--log-level`。
- `deploy/scripts/load-images.sh`
  - 作用：从 tar 加载镜像到 Docker/Podman，可选重标记并推送到私有仓库。
  - 主要参数：`--images-dir`、`--registry`、`--push`、`--log-level`。
- `deploy/scripts/install.sh`
  - 作用：在线或离线安装 LVM LocalPV，可创建 StorageClass。
  - 主要参数：`--offline`、`--chart-version`、`--namespace`、`--create-storageclass`、`--dry-run`、`--log-level`。
- `deploy/scripts/upgrade.sh`
  - 作用：在线或离线升级现有安装。
  - 主要参数：`--offline`、`--chart-version`、`--chart-dir`、`--dry-run`、`--force`、`--log-level`。
- `deploy/scripts/uninstall.sh`
  - 作用：卸载 LVM LocalPV，可保留 CRD/StorageClass 或强制删除 PVC。
  - 主要参数：`--namespace`、`--release`、`--keep-crds`、`--keep-storageclass`、`--delete-pvcs`、`--force`、`--log-level`。

## 在线安装步骤
1. 准备变量（示例）：
   ```bash
   export OPENEBS_CONTROLLER_NODE_NAMES="master01,master02"
   export OPENEBS_DATA_NODE_NAMES="node01,node02"
   export OPENEBS_STORAGECLASS_NAME="openebs-lvmpv"
   export OPENEBS_VG_NAME="lvmvg"
   export OPENEBS_KUBE_NAMESPACE="openebs"
   ```
2. 安装：
   ```bash
   ./deploy/scripts/install.sh --namespace ${OPENEBS_KUBE_NAMESPACE}
   ```
3. 验证：
   ```bash
   kubectl get pods -n ${OPENEBS_KUBE_NAMESPACE}
   kubectl get storageclass
   ```

## 离线安装全流程
1. **联网环境打包离线介质**
   ```bash
   ./deploy/scripts/download.sh --chart-version 1.8.0 --pack --output-dir ./offline-media
   ```
   生成的目录结构：
   ```
   offline-media/
   ├── charts/
   ├── images/
   ├── manifests/
   └── README.md
   ```
2. **传输并解压**
   将 `offline-media-1.8.0.tar.gz` 带到离线环境：
   ```bash
   tar -xzf offline-media-1.8.0.tar.gz
   ```
3. **加载镜像（可选推送私有仓库）**
   ```bash
   ./deploy/scripts/load-images.sh --images-dir ./offline-media/images --registry <your-registry> --push
   ```
4. **离线安装**
   ```bash
   export OFFLINE_INSTALL=true
   export OPENEBS_CHART_DIR="./offline-media/charts/lvm-localpv-1.8.0"
   export OPENEBS_IMAGE_REGISTRY="<your-registry>"
   export OPENEBS_LOAD_IMAGES=true
   export OPENEBS_CONTROLLER_NODE_NAMES="master01,master02"
   export OPENEBS_DATA_NODE_NAMES="node01,node02"
   export OPENEBS_STORAGECLASS_NAME="openebs-lvmpv"
   export OPENEBS_VG_NAME="lvmvg"
   ./deploy/scripts/install.sh --offline
   ```

## 升级
- 在线：
  ```bash
  ./deploy/scripts/upgrade.sh --chart-version 1.8.0
  ```
- 离线：
  ```bash
  export OFFLINE_INSTALL=true
  export OPENEBS_CHART_DIR="./offline-media/charts/lvm-localpv-1.8.0"
  export OPENEBS_IMAGE_REGISTRY="<your-registry>"
  export OPENEBS_LOAD_IMAGES=true
  ./deploy/scripts/upgrade.sh --offline
  ```

## 卸载
- 保留 CRD/StorageClass（默认保留）：
  ```bash
  ./deploy/scripts/uninstall.sh --namespace openebs
  ```
- 强制删除且清理 PVC（会导致数据丢失）：
  ```bash
  ./deploy/scripts/uninstall.sh --namespace openebs --delete-pvcs --force
  ```

## 环境变量参考
### 必填
| 变量 | 说明 |
| --- | --- |
| `OPENEBS_CONTROLLER_NODE_NAMES` | 控制面节点列表，逗号分隔。 |
| `OPENEBS_DATA_NODE_NAMES` | 数据面节点列表，逗号分隔。 |
| `OPENEBS_STORAGECLASS_NAME` | 创建的 StorageClass 名称。 |
| `OPENEBS_VG_NAME` | 节点上存在的 LVM VG 名称。 |

### 常用可选
| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `OPENEBS_KUBE_NAMESPACE` | `openebs` | 安装命名空间。 |
| `OPENEBS_CHART_VERSION` | 空 | 指定安装/升级的 chart 版本。 |
| `OPENEBS_CHART_DIR` | 空 | 离线使用的 chart 目录或 tgz 路径。 |
| `OPENEBS_IMAGE_REGISTRY` | `registry.k8s.io/` | 离线/私有镜像仓库前缀，末尾不带 `/`。 |
| `OPENEBS_LOAD_IMAGES` | `false` | 离线模式自动加载镜像。 |
| `OPENEBS_IMAGES_DIR` | 空 | 离线镜像目录，默认自动探测 `offline-media/images`。 |
| `OPENEBS_CREATE_STORAGECLASS` | `false` | 安装后自动创建 StorageClass。 |
| `OPENEBS_STORAGECLASS_YAML` | 空 | 自定义 StorageClass 模板路径。 |

### 资源配置（可选，均有默认值）
| 变量 | 默认 |
| --- | --- |
| `OPENEBS_CONTROLLER_RESOURCE_LIMITS_CPU` | `500m` |
| `OPENEBS_CONTROLLER_RESOURCE_LIMITS_MEMORY` | `512Mi` |
| `OPENEBS_CONTROLLER_RESOURCE_REQUESTS_CPU` | `500m` |
| `OPENEBS_CONTROLLER_RESOURCE_REQUESTS_MEMORY` | `512Mi` |
| `OPENEBS_NODE_RESOURCE_LIMITS_CPU` | `500m` |
| `OPENEBS_NODE_RESOURCE_LIMITS_MEMORY` | `512Mi` |
| `OPENEBS_NODE_RESOURCE_REQUESTS_CPU` | `500m` |
| `OPENEBS_NODE_RESOURCE_REQUESTS_MEMORY` | `512Mi` |

## 日志位置
每个脚本都会在 `/tmp/openebs-lvmlocalpv_<action>-<timestamp>.log` 写入详细日志；若 `/tmp` 不可写则落在当前目录。

## 建议与提示
- 若只想预览操作，使用 `--dry-run` 查看将要执行的 Helm/Kubernetes 命令。
- 离线安装时务必确认 `OPENEBS_IMAGE_REGISTRY` 与镜像加载到的仓库一致。
- 升级前可使用 `upgrade.sh` 自动备份当前 release 的 Helm values（输出到 `/tmp/openebs-lvmlocalpv-backup-*`）。
