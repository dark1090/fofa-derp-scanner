# FOFA DERP Scanner

一个用于自动化扫描 FOFA 上的 Tailscale DERP 节点，进行可用性测试，并生成可以在 Tailscale 后台中直接使用的自定义 `derpMap` 节点的工具流。

**当前版本：v1.0.0**

## 整体流程概述

这个项目的目标是从海量的公开 DERP 节点中，筛选出可以在国内或特定网络环境下正常连接和中继的优质节点。主要步骤如下：

1. **信息收集（FOFA）**：从 FOFA 获取符合条件的 DERP 节点 IP 及端口信息。
2. **格式转换**：将 FOFA 导出的 JSON 资产转换成 DERP Prober 需要的标准 `derp.json` 格式。
3. **探测存活**：使用 docker 容器 `tailscale-derpprober` 对节点进行大批量并发探测。
4. **结果提取**：提取探测成功的优质节点，并按 ID 范围进行格式化。
5. **合入应用**：将生成的结果复制到 Tailscale 管理后台，供设备使用。

## 第一步：信息收集 (FOFA)

1. 打开 [FOFA 官网](https://fofa.so/)
2. 使用以下查询条件进行搜索（可以根据需要调整 `country`）：
   ```fofa
   body="Tailscale" && body="DERP server" && country="CN"
   ```
3. 在页面上方点击**导出**，选择 **JSON** 格式。假设保存为 `fofa_assets.json`。

## 第二步：格式转换

将上一步下载的 FOFA 数据转换为 `derp.json` 文件：

```bash
# 进入项目目录
cd fofa-derp-scanner

# 运行转换脚本 (请确保已经安装 Python3)
python3 scripts/convert_assets.py --input /path/to/fofa_assets.json --output config/derp.json --start 900
```
*(这会将每个节点的 RegionID 从 900 开始自动递增分配)*

## 第三步：探测节点存活 (DERP Prober)

本项目包含一个 `docker-compose.yml` 文件。在这个文件中，我们将本地的 `config` 文件夹挂载到了容器内部：

### 1. 调整 Docker 路径配置
打开项目根目录下的 `docker-compose.yml`，查看 `volumes` 部分：
```yaml
    volumes:
      # ./config 代表当前目录下的 config 文件夹，你可以修改为你的绝对路径
      - ./config:/config
```
如果你在第二步中把 `derp.json` 生成到了当前项目的 `config` 目录下，那么这里**不需要**修改任何路径。

### 2. 启动探测容器
```bash
docker-compose up -d
```

### 3. 获取成功探测报告
1. 在浏览器中打开 [http://localhost:8030/](http://localhost:8030/)
2. 点击进入 `success` 页面（这里列出了所有成功连接的节点）。**注意：探测需要时间，请耐心等待几分钟直至列表稳定。**
3. 使用浏览器的插件（例如：[SingleFile](https://chrome.google.com/webstore/detail/singlefile/mpiodijhokgodhhofbcjdecpffjipkle)）将**整个 success 页面保存为一个 HTML 文件**。假设保存为 `success_report.html`。

## 第四步：提取最终成功节点

现在我们需要把那个庞大繁杂的 HTML 报告，转换回精简的 JSON：

```bash
python3 scripts/extract_success.py \
  --html /path/to/success_report.html \
  --json config/derp.json \
  --output config/derp-success-prober.json \
  --start 900 \
  --end 999
```
*(在这个例子中，我们只保留 Region ID 在 900 到 999 之间的节点，你可以通过 `--start` 和 `--end` 参数自定义)*

运行完毕后，最终筛选出的高质量节点配置就躺在 `config/derp-success-prober.json` 里了！

## 第五步：合入 Tailscale 后台

1. 打开 [Tailscale Access Controls (ACLs)](https://login.tailscale.com/admin/acls/file) 页面。
2. 打开刚才生成的 `config/derp-success-prober.json` 文件。
3. 在你的 ACL JSON 根节点中，找到（或添加）`"derpMap"` 字段：

```json
{
    // ... 其他 ACL 配置 ...

    "derpMap": {
        "OmitDefaultRegions": false, // 注意：改为true
        "Regions": {
            // ---> 将 derp-success-prober.json 中的 "Regions" 里面的内容，原样复制粘贴到这里 <---
        }
    }
}
```
4. 点击 **Save** 保存。

## 常见问题与维护维护

如果你在使用过程中发现网络卡顿，中继 (relay) 无法连通：

1. **排查节点**：在设备终端执行 `tailscale status`
2. 或者在 Windows PowerShell 中执行：
   ```powershell
   (Get-NetTCPConnection -OwningProcess (Get-Process tailscaled | Select-Object -Last 1).Id -State Established).RemoteAddress
   ```
   查看当前正在连接的是哪个 IP 的节点。
3. **剔除节点**：去 Tailscale ACLs 后台，在 `derpMap.Regions` 下找到对应 IP 的节点块，将其删除，然后 `Ctrl+S` 保存即可。Tailscale 客户端会自动刷新并连接下一个可用节点。

---

> 开源协议：MIT
> 感谢 [helloworlde/tailscale-derpprober](https://github.com/helloworlde/tailscale-derpprober) 及linuxdo的各位佬提供的优秀探测镜像支持！
