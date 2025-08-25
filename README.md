# iStoreOS ARM64 RootFS 提取器

这个项目使用GitHub Actions自动从iStoreOS官方镜像中提取ARM64架构的rootfs文件系统，并将其发布到GitHub Releases中。

## 功能特性

- 🚀 **自动化提取**: 使用GitHub Actions自动下载和处理iStoreOS镜像
- 📦 **智能分区识别**: 自动识别镜像中的rootfs分区
- 🔄 **定期更新**: 支持定期检查和提取最新版本
- 📋 **详细日志**: 提供完整的提取过程日志
- ✅ **文件校验**: 自动生成SHA256校验和
- 📝 **发布说明**: 自动生成详细的发布说明

## 使用方法

### 手动触发提取

https://fw0.koolcenter.com/iStoreOS/armsr

1. 前往项目的 `Actions` 页面
2. 选择 `Extract iStoreOS ARM64 RootFS` 工作流
3. 点击 `Run workflow`
4. 可选择自定义以下参数：
   - **镜像URL**: iStoreOS镜像文件的下载链接
   - **发布标签**: 用于Release的版本标签

### 自动定期提取

工作流默认配置为每周日凌晨2点自动执行，您可以在 `.github/workflows/extract-rootfs.yml` 中修改执行频率。

### 下载提取的RootFS

1. 前往项目的 `Releases` 页面
2. 选择最新的发布版本
3. 下载 `rootfs.tar.gz` 文件

### 使用Docker镜像（推荐）

🚀 **直接使用预构建镜像**：

```bash
# 拉取最新镜像
docker pull ghcr.io/YOUR_USERNAME/istoreos:latest

# 运行容器
docker run -d --name istoreos --privileged ghcr.io/YOUR_USERNAME/istoreos:latest

# 旁路由模式（需要网络配置）
docker run -d --name istoreos-gateway --privileged --network host ghcr.io/YOUR_USERNAME/istoreos:latest
```

> ⚠️ **注意**: 请将 `YOUR_USERNAME` 替换为您的GitHub用户名（例如：`jianjunx`）

🔗 **查看可用镜像版本**：[GitHub Container Registry](https://github.com/YOUR_USERNAME/istore-arm-rootfs/pkgs/container/istoreos)

## 使用提取的RootFS

📖 **详细指南**: 查看 [Docker使用指南](DOCKER_GUIDE.md) 了解完整的Docker部署和旁路由配置步骤。

### 快速开始
```bash
# 基本解压命令
mkdir -p ./rootfs
tar -xzf rootfs.tar.gz -C ./rootfs
```

🐳 **Docker部署**: 推荐使用Docker来运行iStoreOS，支持旁路由模式。详见 [Docker使用指南](DOCKER_GUIDE.md)。

### 手动构建Docker镜像（备选方案）
```bash
# 下载rootfs文件后，创建Dockerfile
cat > Dockerfile << 'EOF'
FROM scratch
ADD rootfs.tar.gz /
CMD ["/sbin/init"]
EOF

# 构建和运行
docker build -t istoreos .
docker run -d --name istoreos --privileged istoreos
```


## 工作流程说明

1. **环境准备**: 安装必要的工具（wget, gzip, fdisk, parted, squashfs-tools）
2. **下载镜像**: 从指定URL下载iStoreOS镜像文件
3. **解压镜像**: 解压.gz压缩的镜像文件
4. **分区分析**: 分析镜像的分区结构
5. **RootFS提取**: 自动识别并挂载rootfs分区，创建tar.gz压缩包
6. **发布文件**: 生成校验和和发布说明，创建GitHub Release

## 目录结构

```
istore-arm-rootfs/
├── .github/
│   └── workflows/
│       └── extract-rootfs.yml    # GitHub Actions工作流配置
├── README.md                     # 项目说明文档
└── LICENSE                       # 开源许可证
```

## 技术细节

### 支持的镜像格式
- 压缩格式：gzip (.img.gz)
- 分区表：GPT或MBR
- 文件系统：ext4, squashfs等Linux常见文件系统

### 自动识别逻辑
工作流会自动遍历镜像中的所有分区，通过检查目录结构（如/bin, /etc, /usr等）来识别rootfs分区。

### 安全性
- 所有操作在隔离的GitHub Actions环境中执行
- 使用只读方式挂载分区，确保原始镜像不被修改
- 自动清理临时文件，不保留敏感信息

## 自定义配置

### 修改镜像源
编辑 `.github/workflows/extract-rootfs.yml` 文件中的默认URL：

```yaml
default: 'https://your-custom-mirror.com/path/to/image.img.gz'
```

### 调整执行频率
修改cron表达式来改变自动执行的频率：

```yaml
schedule:
  - cron: '0 2 * * 0'  # 每周日凌晨2点
```

## 故障排除

### 常见问题

1. **分区识别失败**
   - 检查镜像文件是否完整下载
   - 确认镜像格式是否受支持

2. **挂载失败**
   - 可能是分区表损坏或格式不支持
   - 查看Actions日志获取详细错误信息

3. **权限问题**
   - 确保GitHub仓库有创建Release的权限
   - 检查GITHUB_TOKEN是否正确配置

### 日志查看
前往GitHub Actions页面查看详细的执行日志，所有关键步骤都有相应的输出信息。

## 许可证

本项目采用MIT许可证，详见 [LICENSE](LICENSE) 文件。

## 贡献

欢迎提交Issue和Pull Request来改进这个项目！

## 相关链接

- [iStoreOS官方网站](https://www.istoreos.com/)
- [GitHub Actions文档](https://docs.github.com/en/actions)
- [Docker官方文档](https://docs.docker.com/)