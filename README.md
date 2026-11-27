# 地震勘探工具箱（seis_toolbox）

Win11 风格的地震勘探小工具集合。当前已装四个工具：SEG-Y 裁切、坐标转换与 KMZ、GPS 时间换算、生产进度图。
界面统一为亮蓝主题（主色 `#0067C0`，浅蓝渐变页头，全直角控件），样式集中在 `ui/app.qss`，页头由 `ui/theme.py` 的 `page_header()` 生成。
设计口径见 `00_边界与实现方案.md`；总览与启动见 `使用说明.md`，四个工具的手册分别在 `tools/segy_crop/`、`tools/coord_mini/`、`tools/gps_time/`、`tools/progress_map/` 下。

生产进度图由独立程序并入：核心算法在 `tools/progress_map/core/`（SPS 设计点、日报扫描、线对应、快照统计、图面渲染），界面在 `tools/progress_map/ui/`。
原 QMainWindow 改为 `ProgressMapPanel(QWidget)` 嵌入主窗堆栈，页头、状态行与工程页签都留在面板内部；工程默认落在 `%LOCALAPPDATA%\SeisToolbox\progress_map`。

## 运行

```powershell
cd seis_toolbox
python main.py
```

依赖 PySide6 + matplotlib + numpy + openpyxl + tzdata（见 `requirements.txt`）。`tzdata` 是 Windows 上 zoneinfo 的 IANA 时区库，不能被漏装；打包时由 `--collect-data tzdata` 带入。

## 打包

### onedir 免安装包

```powershell
python -m PyInstaller --noconfirm --clean SeisToolbox.spec
```

程序名、版本、发布者与版权署名统一在 `core/version.py`；`SeisToolbox.spec` 把这些字段写入 exe 的 Windows 版本资源。当前版本 `1.0.0`，版权字段为“版权所有 小虫”。

`--collect-data tzdata` 不能漏：时区换算用 zoneinfo，Windows 上没有系统时区库，全靠这个包里的 IANA 数据；漏了打包后选时区会报“未知时区”。

`--add-data "tools/coord_mini/data;..."` 也不能漏：坐标工具要读基准表 `datums.csv` 与投影/基准库两张 CSV；漏了打包后坐标页基准下拉为空、库对话框无数据。

六个手册条目（四份 `使用说明.md` + 四张同名预览图）也不能漏：每页右上角「使用说明」按钮直接读 `_internal/tools/<模块>/使用说明.md`，并在窗口内渲染表格和截图；漏了按钮会提示手册缺失。

图标由 `assets/make_icon.py` 生成（方形直角底板 + 白色工具箱：箱体、盖缝、开口 U 提手、锁扣，全部直角，无圆角），
128 px 及以上在箱体下腹加一条三角波负形，小尺寸自动省略；16/24/32/48/64/128/256 各尺寸单独渲染，改设计后重跑该脚本即可。

四个工具的导航简笔画由 `assets/make_tool_icons.py` 生成，输出透明底、亮蓝 `#0067C0` 线稿：
`tool_segy.png`（裁切角 + 地震道波形）、`tool_coord.png`（图钉 + 坐标十字）、`tool_gps.png`（时钟 + GPS 信号弧）、
`tool_progress.png`（图幅边框 + 测线网格 + 进度折线）。
改图后重跑脚本，再把四张 PNG 一起打包。

产物在 `dist\SeisToolbox\`（159.85 MiB，含 Qt、matplotlib、openpyxl、numpy 与坐标库表），整个目录可拷到无 Python 的机器直接运行。

打包后自检（会在冻结环境里走一遍读取、绘图、裁切、单道）：

```powershell
.\dist\SeisToolbox\SeisToolbox.exe --selftest tests\fixtures\ieee_2ms.segy selftest.log
```

### Inno 安装包

```powershell
.\release.ps1
```

`release.ps1` 依次完成：PyInstaller 重建 onedir 包 → 用本机 `CN=小虫` 证书对主程序做 SHA256 Authenticode 签名 → 调用 `SeisToolbox.iss` 生成安装包 → 再对安装包签名并校验。只想复用现有 `dist\SeisToolbox` 出安装包时，加 `-SkipBuild`。

安装包输出到 `dist\installer\SeisToolbox-1.0.0-setup.exe`（47.95 MiB）；开始菜单快捷方式默认创建，桌面快捷方式为可选项，卸载入口在系统“应用”列表中。签名证书为本机个人证书，未装入外部机器受信根目录时，对方系统可能提示未知发布者。

## 测试

```powershell
# 1. 生成用例表
python tests\gen_cases.py

# 2. 用 MATLAB 旧版生成参考值（需要本机 MATLAB）
cd tests; matlab -batch "make_reference"; cd ..

# 3. 逐字段比对
python tests\compare_gps.py

# 4. 界面冒烟 + 截图（截图需真实平台，会闪一下窗口）
python tests\smoke_ui.py
python tests\shot_real.py

# 5. SEG-Y 读取层（M2）验收：需先用 MATLAB 造 fixture
cd tests; matlab -batch "make_segy_fixtures"; cd ..
python tests\test_segy_io.py
python tests\bench_segy.py

# 6. SEG-Y 裁切（M3）验收：需先造裁切参考
cd tests; matlab -batch "make_crop_refs"; cd ..
python tests\test_crop.py

# 7. 显示层与面板（M4）：色标裁剪、抽道抽点、参数校验、缩放交互
python tests\test_render.py
python tests\test_ui_params.py
python tests\test_canvas_zoom.py
python tests\test_panel_zoom_keep.py
python tests\test_regress.py       # 复查修复项：uint16 打包、长记录裁切、时区 UTC、内存预算
python tests\test_zones.py         # 时区：夏令时、固定偏移、跨日、闰秒、批量列
python tests\test_threads.py       # 后台任务：连跑 40 条回调齐全、异常走失败回调、退出无线程残留
python tests\test_coord.py         # 坐标内核：投影对拍、基准转换、文件读取、KML/KMZ、库表（额外需要 pyproj）
python tests\test_coord_ui.py      # 坐标面板：载入/换算/互换/CSV/KMZ/SHP/库套用/基准下拉
python tests\test_stress_light.py  # 轻量数据反复测试：SEG-Y 12 轮 / GPS 14 轮 / 批量 3 轮 / 坐标 4 轮，756 项检查
python tests\test_progress.py      # 进度图：设计/日报导入、幂等、匹配归类、快照、统计表、300→150dpi 出图
python tests\test_legend_ui.py     # 图例文字：长名自适应列宽、行高随字号、关闭设计线、空图提示、本期/当日独立键、底图分键、工序增删联动、恢复默认清 dirty
python tests\shot_segy.py          # 三视图 + 缩放截图（真实平台）
python tests\shot_coord.py         # 坐标页+样例数据截图（真实平台）
python tests\shot_progress.py      # 进度图页截图（真实平台；WPM_SAMPLE 指向样例工程）
```

当前状态：版本 1.0.0，版权署名“版权所有 小虫”；四个工具（含生产进度图）在同一主窗内；全量 18 项测试连续两遍 0 失败，进度图回归 14/14、图例文字 21/21；GPS 24/24 用例与 MATLAB 旧版一致；SEG-Y 4 个 fixture 的读取层与 `readSEGY` 全等；15 个裁切用例与 `cropSEGY` 输出逐字节全等；显示层色标口径 8/8、面板参数 14/14、回归 11/11、时区 16/16、后台任务 3/3、坐标内核 155/155、坐标面板 69/69、轻量反复测试 756 项全过；`compileall` 与 `pyflakes` 均无告警；onedir 包与安装包均通过 SHA256 Authenticode 签名，冻结自检 `SELFTEST OK`，静默安装/卸载复核通过。
2026-11-24 总体审查见 `代码审查报告.md`：修掉 GPS 闰秒 `leap_seconds` 双向不一致、补 `tzdata` 依赖、加固裁切写出、修正坐标圆锥投影比例因子/列序/投影 SHP 判读、补齐 Albers WKT 标准纬线、修 SEG-Y 面板状态覆盖；重建包 157.43 MB，冻结自检通过。

## 目录

| 路径 | 作用 |
|---|---|
| `main.py` | 入口，注册工具列表 `TOOLS` |
| `core/` | 上下文（设置/日志/后台任务）、资源路径、工具契约、`version.py` 版本与版权元数据 |
| `SeisToolbox.spec` | PyInstaller onedir 打包配置，写入 exe 版本资源 |
| `SeisToolbox.iss` | Inno Setup 安装包脚本 |
| `release.ps1` | 重建、签名主程序与安装包的发布脚本 |
| `ui/` | 主窗、样式 `app.qss`、小部件工厂 |
| `assets/` | 程序图标 `app.ico`、图标生成脚本、四个工具导航线稿 |
| `tools/gps_time/` | `logic.py` 纯算法；`panel.py` 纯界面；`使用说明.md` |
| `tools/segy_crop/` | 读取/裁切/渲染/面板；`使用说明.md` |
| `tools/coord_mini/` | `coord_core.py` 坐标/投影/文件/KMZ 算法；`panel.py` 界面；`data/` 基准表与库表；`使用说明.md` |
| `tools/progress_map/` | `core/` 进度图算法（SPS、日报、线对应、快照、渲染、导出）；`ui/` 面板与对话框；`crashlog.py` 模块日志；`使用说明.md` + `preview_progress.png` |
| `tests/` | 用例表、MATLAB 参考值生成、比对与截图脚本 |

## 加新工具

1. 在 `tools/` 下建目录，写 `logic.py` 与 `panel.py`；
2. `panel.py` 末尾定义 `SPEC = ToolSpec(...)`，`icon` 填 `assets/` 下的图标文件名（没有就留空）；
3. 在 `main.py` 的 `TOOLS` 里加一行；
4. 有导航图标的，把 PNG 一并加进 `SeisToolbox.spec` 的 `datas`。

主窗不改。
