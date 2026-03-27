# lvm-localpv-deployer

面向 OpenEBS LVM LocalPV 的 Helm 部署脚本集合，支持在线/离线安装、升级、卸载以及离线介质打包。脚本源自原仓库的 `deploy/scripts`，单独抽离方便在离线环境或自建镜像仓库中使用。

## 功能特性
- 在线/离线安装与升级：自动拉取或使用本地 chart、镜像完成部署。
- 离线介质打包：一键下载 Helm chart 与镜像并生成可传输的离线包。
- 镜像导入与推送：支持 Docker/Podman，必要时重标记并推送到私有仓库。
- 资源与节点选择：通过环境变量配置节点标签、资源配额、命名空间等。
- 幂等操作与日志：脚本具有幂等性，并将日志输出到 `/tmp/openebs-lvmlocalpv_*.log` 便于排查。

## 仓库结构
- [deploy/scripts/download.sh](deploy/scripts/download.sh)：下载 Helm chart 与镜像，生成离线介质并可选打包。
- [deploy/scripts/load-images.sh](deploy/scripts/load-images.sh)：从 tar 文件加载镜像，可选重标记/推送到私有仓库。
- [deploy/scripts/install.sh](deploy/scripts/install.sh)：在线或离线安装 LVM LocalPV，可创建 StorageClass。
- [deploy/scripts/upgrade.sh](deploy/scripts/upgrade.sh)：在线或离线升级已安装的 LVM LocalPV 版本。
- [deploy/scripts/uninstall.sh](deploy/scripts/uninstall.sh)：卸载 LVM LocalPV，可选保留 CRD/StorageClass。

## 前置条件
- 已连接的 Kubernetes 集群（推荐 v1.23 及以上），`kubectl` 已配置好 kubeconfig。
- `helm` 3.x、`kubectl`、`curl`、`envsubst` 可用；离线打包需 Docker 或 Podman。
- 离线场景建议预先准备一个可访问的私有镜像仓库，用于推送打包好的镜像。

## 快速开始
### 在线安装
1. 设置必需环境变量（示例）：
   ```bash
   export OPENEBS_CONTROLLER_NODE_NAMES="master01,master02"
   export OPENEBS_DATA_NODE_NAMES="node01,node02"
   export OPENEBS_STORAGECLASS_NAME="openebs-lvmpv"
   export OPENEBS_VG_NAME="lvmvg"
   ```
2. 执行安装：
   ```bash
   ./deploy/scripts/install.sh --namespace openebs
   ```
3. 验证：
   ```bash
   kubectl get pods -n openebs
   kubectl get storageclass
   ```

### 离线模式总览
1. 在联网环境打包离线介质（可指定版本）：
   ```bash
   ./deploy/scripts/download.sh --chart-version 1.8.0 --pack --output-dir ./offline-media
   ```
2. 将 `offline-media-<version>.tar.gz` 传输到离线环境并解压：
   ```bash
   tar -xzf offline-media-1.8.0.tar.gz
   ```
3. 加载镜像到本地或私有仓库（可选推送）：
   ```bash
   ./deploy/scripts/load-images.sh --images-dir ./offline-media/images --registry <your-registry> --push
   ```
4. 离线安装（示例）：
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

### 升级
- 在线：`./deploy/scripts/upgrade.sh --chart-version 1.8.0`
- 离线：设置 `OFFLINE_INSTALL=true` 与 `OPENEBS_CHART_DIR` 后执行 `./deploy/scripts/upgrade.sh --offline`

### 卸载
- 保留 CRD/StorageClass：`./deploy/scripts/uninstall.sh`
- 强制删除并清理 PVC（危险）：`./deploy/scripts/uninstall.sh --delete-pvcs --force`

## 关键环境变量
| 名称 | 必填 | 说明 | 示例 |
| --- | --- | --- | --- |
| `OPENEBS_CONTROLLER_NODE_NAMES` | 是 | 运行控制面组件的节点列表（逗号分隔） | `master01,master02` |
| `OPENEBS_DATA_NODE_NAMES` | 是 | 运行数据面节点 DaemonSet 的节点列表 | `node01,node02` |
| `OPENEBS_STORAGECLASS_NAME` | 是 | 创建的 StorageClass 名称 | `openebs-lvmpv` |
| `OPENEBS_VG_NAME` | 是 | 节点上的 LVM VG 名称 | `lvmvg` |
| `OPENEBS_KUBE_NAMESPACE` | 否 | 安装命名空间，默认 `openebs` | `openebs` |
| `OPENEBS_IMAGE_REGISTRY` | 否 | 离线/私有仓库前缀（结尾不要带 `/`） | `registry.example.com` |
| `OPENEBS_CHART_DIR` | 否 | 离线安装使用的 chart 路径或 `.tgz` | `./offline-media/charts/lvm-localpv-1.8.0` |
| `OPENEBS_LOAD_IMAGES` | 否 | 离线模式自动加载镜像，`true/false` | `true` |

更多可调参数（资源限制、StorageClass 模板等）详见 [docs/usage.md](docs/usage.md)。

## 日志与排错
- 每个脚本在 `/tmp/openebs-lvmlocalpv_*.log` 写入详细日志，可在失败后先行查看。
- 如使用 `--dry-run`，脚本只输出计划的命令，不会对集群做修改。

## 贡献
欢迎 issue / PR。若有其他安装场景或兼容性需求，可在 issue 中说明。
