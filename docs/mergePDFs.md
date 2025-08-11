# mergePDFs 使用与开发文档

本工具用于将被拆分为多个片段的 PDF 文件（例如 `xxx.pdf.1`, `xxx.pdf.2`, ...）自动合并回一个完整的 `xxx.pdf` 文件，并在成功合并后自动删除原有的分片文件。

- **适用场景**: 当单个 PDF 文件过大（如 GitHub 上传限制）而被拆分为多个小文件时的一键合并。
- **支持平台**: Windows（提供可执行文件）、macOS/Linux（需从源码构建）。
- **工作目录**: 程序运行时会在“当前目录”扫描并处理所有符合命名规则的分片文件。

---

## 一、快速开始（Windows）

1. 将 `mergePDFs.exe` 放到包含分片文件的同一文件夹中。
2. 双击运行 `mergePDFs.exe`，等待合并完成。
3. 合并成功后，会生成 `xxx.pdf`，且原始分片 `xxx.pdf.1`, `xxx.pdf.2`, ... 会被删除。

示例目录：

```
- mergePDFs.exe
- 义务教育教科书 · 数学一年级上册.pdf.1
- 义务教育教科书 · 数学一年级上册.pdf.2
```

运行后将生成：

```
- 义务教育教科书 · 数学一年级上册.pdf
```

---

## 二、从源码构建与运行（macOS/Linux/Windows）

本工具由 Go 语言编写，源码位于 `.cache/mergePDFs.go`。

- 安装 Go（建议 1.18+）
- 在源码目录执行：

```bash
# 进入仓库根目录
cd /path/to/repo

# 构建（当前平台）
go build -o mergePDFs ./.cache/mergePDFs.go

# 运行（在包含分片文件的目录中执行）
./mergePDFs
```

跨平台构建示例：

```bash
# 构建 Windows 版可执行文件（在 macOS/Linux 上）
GOOS=windows GOARCH=amd64 go build -o mergePDFs.exe ./.cache/mergePDFs.go
```

---

## 三、命名规则与处理逻辑

- 工具会在当前目录扫描所有文件名中包含 `.pdf.` 片段的文件，例如：
  - `xxx.pdf.1`, `xxx.pdf.2`, ...
- 对每一组同名基底的分片（基底即 `xxx.pdf`）进行排序后顺序写入到目标文件 `xxx.pdf`。
- 合并完成后，会删除对应分片文件。

注意：
- 分片必须严格按照 `原文件名.pdf.N` 的格式命名，其中 `N` 为顺序号，建议从 1 开始连续编号。
- 分片的顺序由文件名的字典序排序决定，如 `1, 2, 10` 在字符串排序中会是 `1, 10, 2`。建议使用固定宽度的编号（如 `001, 002, ...`）或确保编号不跨位数；若当前仓库提供的分片均按 `1, 2, 3...` 命名，工具已通过 `sort.Strings` 进行排序，通常不会有问题。

---

## 四、使用示例

- 仅合并当前目录下的所有分片：

```bash
# Windows
mergePDFs.exe

# macOS/Linux（需先构建）
./mergePDFs
```

- 在某个教材目录中合并该目录内所有分片：

```bash
cd "/path/to/小学/数学/人教版"
/path/to/mergePDFs.exe   # Windows
# 或
/path/to/mergePDFs       # macOS/Linux
```

---

## 五、常见问题（FAQ）

- 问：运行后没有生成合并文件？
  - 请确认当前目录下确实存在形如 `xxx.pdf.N` 的分片文件。
  - 请检查是否有写权限创建目标文件 `xxx.pdf`。

- 问：顺序不正确导致合并后的 PDF 打不开或内容错乱？
  - 检查分片命名是否连续、是否存在缺失片段。
  - 建议编号统一位数：`001, 002, ...`。

- 问：不想删除分片文件？
  - 目前工具在合并完成后会调用 `os.Remove(part)` 删除分片；如不希望删除，请参考开发者文档自行修改源码（移除该行），重新构建。

---

## 六、开发者文档（Go API）

源码位置：`.cache/mergePDFs.go`

### 1. 程序入口

```go
func main()
```
- 设置工作目录为当前目录 `"."`，并调用 `mergeSplitPDFsInDirectory`。

### 2. 目录扫描与分组

```go
func mergeSplitPDFsInDirectory(dirPath string)
```
- 读取目录 `dirPath` 下的所有文件。
- 过滤出文件名包含 `.pdf.` 的分片文件。
- 根据 `baseName := strings.Split(fileName, ".pdf.")[0] + ".pdf"` 将分片映射到同一基底文件名下。
- 对每个基底对应的分片进行字典序排序：`sort.Strings(parts)`。
- 依次调用 `mergeFiles(baseName, parts)` 合并。

参数：
- `dirPath`: 需要扫描的目录路径。

### 3. 合并写入与清理

```go
func mergeFiles(baseName string, parts []string)
```
- 创建目标文件 `baseName`（存在则覆盖）。
- 按顺序读取每个分片文件的二进制数据并写入目标文件。
- 写入成功后，删除该分片文件。

参数：
- `baseName`: 目标 PDF 文件名（含 `.pdf`）。
- `parts`: 分片文件名切片，需为正确顺序。

错误处理：
- 代码中使用 `panic(err)`；如需友好错误提示，建议替换为显式错误返回并在 `main` 中处理。

### 4. 本地构建与发布建议

- 本地构建：
  - `go build -o mergePDFs ./.cache/mergePDFs.go`
- Windows 发行版：
  - `GOOS=windows GOARCH=amd64 go build -o mergePDFs.exe ./.cache/mergePDFs.go`
- 代码改进建议：
  - 支持自定义目录参数，例如 `mergePDFs -dir <path>`。
  - 增加日志与错误信息输出，避免使用 `panic`。
  - 增加跳过删除分片的可选参数 `--keep-parts`。

---

## 七、当前仓库的公开接口说明

- 本仓库为教材资源聚合仓库，当前唯一的“公开接口”面向终端用户的是命令行工具（或 Windows 可执行文件）`mergePDFs`/`mergePDFs.exe`。
- 源码中的 Go 函数均为包内私有（非导出）函数，不直接对外提供库级别的调用接口；如需以库形式复用，可将相关函数提升为导出函数并抽取为独立包。

---

## 八、许可证与致谢

- 如果后续补充许可证文件（LICENSE），请以其为准。
- 感谢所有为公开教育资源做出贡献的参与者。