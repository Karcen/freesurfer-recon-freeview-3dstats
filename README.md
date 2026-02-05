# FreeSurfer完整流程手册：从重建到3D模型+核团统计
## 一、文档说明
本文档从**FreeSurfer基础重建（recon-all）** 开始，完整覆盖「脑表面重建→freeview可视化→核团大小统计→3D彩色模型导出」全流程，所有命令适配Mac ARM架构，可直接复制运行。

<img width="1470" height="923" alt="image" src="https://github.com/user-attachments/assets/920fc380-0957-40c7-b40a-1c1b545722ca" />


## 二、前置准备
1. **FreeSurfer环境配置**（首次运行需执行，一般需要加sudo）：
   ```bash
   # 设置FreeSurfer根目录（替换为你的安装路径）
   export FREESURFER_HOME=你的FreeSurfer安装路径
   # 激活环境
   source $FREESURFER_HOME/SetUpFreeSurfer.sh
   # 配置受试者目录（存放重建结果）
   export SUBJECTS_DIR=你的受试者目录路径
   # 创建目录（避免权限问题）
   mkdir -p $SUBJECTS_DIR
   ```
   **路径说明**：
   - `你的FreeSurfer安装路径`：例如 `/Applications/freesurfer/8.1.0`
   - `你的受试者目录路径`：例如 `/Users/用户名/Documents/freesurfer_output`

2. **数据准备**：将你的NIfTI格式脑影像（如`sub-01.nii.gz`）放入`$SUBJECTS_DIR/raw_data/`目录。

## 三、核心流程（按顺序执行）
### 3.1 步骤1：recon-all全脑表面重建（核心）
`recon-all`是FreeSurfer核心命令，自动完成脑提取、表面重建、解剖分区标注：
```bash
#!/bin/bash
# 激活FreeSurfer环境（每次新开终端都要执行）
export FREESURFER_HOME=你的FreeSurfer安装路径
source $FREESURFER_HOME/SetUpFreeSurfer.sh
export SUBJECTS_DIR=你的受试者目录路径

# 重建命令（关键参数说明）
# -subjid：受试者ID（自定义，如sub-01）
# -i：输入原始脑影像路径
# -all：执行完整重建流程（含表面、注释、分区）
recon-all -subjid sub-01 \
          -i $SUBJECTS_DIR/raw_data/sub-01.nii.gz \
          -all \
          -parallel  # 多线程加速（可选）

# 验证重建结果（出现surf/、label/、mri/目录则成功）
echo "✅ 重建完成，验证关键目录："
ls $SUBJECTS_DIR/sub-01/surf/
ls $SUBJECTS_DIR/sub-01/label/
ls $SUBJECTS_DIR/sub-01/mri/
```
**运行说明**：  
- 重建耗时：普通电脑约2-4小时，服务器约30分钟；  
- 中断后恢复：执行`recon-all -subjid sub-01 -all -continue`。

### 3.2 步骤2：freeview可视化验证重建结果
重建完成后，用`freeview`查看脑表面、解剖分区、核团标注是否正常：
```bash
#!/bin/bash
# 激活环境
export FREESURFER_HOME=你的FreeSurfer安装路径
source $FREESURFER_HOME/SetUpFreeSurfer.sh
export SUBJECTS_DIR=你的受试者目录路径

# 打开freeview并加载关键文件（可视化解剖分区+核团）
freeview \
  -v $SUBJECTS_DIR/sub-01/mri/T1.mgz \
  -v $SUBJECTS_DIR/sub-01/mri/aseg.mgz:colormap=lut:opacity=0.5 \
  -f $SUBJECTS_DIR/sub-01/surf/lh.pial:annot=aparc:color=lut \
  -f $SUBJECTS_DIR/sub-01/surf/rh.pial:annot=aparc:color=lut
```
**可视化操作指南**：
1. 打开后默认显示3D脑表面，左侧面板可切换：
   - `T1.mgz`：原始T1结构像；
   - `aseg.mgz`：自动分割的核团/脑区（彩色标注）；
   - 左右脑`pial`表面：带Desikan-Killiany解剖分区的彩色皮层表面。
2. 常用操作：
   - 鼠标左键：旋转视图；
   - 鼠标中键：平移视图；
   - 鼠标右键：缩放视图；
   - 快捷键`3`：切换3D视图，`1`：切换轴位视图。
3. 验证要点：皮层表面无破损、核团标注无缺失、解剖分区颜色正常。

### 3.3 步骤3：导出核团/脑区大小统计文件
生成包含核团体积、表面积、灰度值的定量统计文件（可直接用于数据分析）：
```bash
#!/bin/bash
# 激活环境
export FREESURFER_HOME=你的FreeSurfer安装路径
source $FREESURFER_HOME/SetUpFreeSurfer.sh
export SUBJECTS_DIR=你的受试者目录路径

# 配置输出路径
STAT_OUTPUT=你的统计结果保存路径
mkdir -p $STAT_OUTPUT

# 1. 生成全脑核团（aseg）体积统计（含海马、杏仁核等深部核团）
asegstats2table \
  --subjects sub-01 \
  --meas volume \
  --tablefile $STAT_OUTPUT/深部核团体积统计.tsv \
  --delimiter tab  # 制表符分隔，方便Excel打开

# 2. 生成皮层分区（aparc）表面积/厚度/体积统计（左右脑分开）
# 左脑
mris_anatomical_stats \
  -a $SUBJECTS_DIR/sub-01/label/lh.aparc.annot \
  $SUBJECTS_DIR/sub-01/surf/lh.pial \
  > $STAT_OUTPUT/左脑皮层分区统计.txt

# 右脑
mris_anatomical_stats \
  -a $SUBJECTS_DIR/sub-01/label/rh.aparc.annot \
  $SUBJECTS_DIR/sub-01/surf/rh.pial \
  > $STAT_OUTPUT/右脑皮层分区统计.txt

# 3. 合并所有统计（可选，生成汇总文件）
cat $STAT_OUTPUT/深部核团体积统计.tsv > $STAT_OUTPUT/全脑核团大小汇总.txt
echo -e "\n===== 左脑皮层分区统计 =====" >> $STAT_OUTPUT/全脑核团大小汇总.txt
cat $STAT_OUTPUT/左脑皮层分区统计.txt >> $STAT_OUTPUT/全脑核团大小汇总.txt
echo -e "\n===== 右脑皮层分区统计 =====" >> $STAT_OUTPUT/全脑核团大小汇总.txt
cat $STAT_OUTPUT/右脑皮层分区统计.txt >> $STAT_OUTPUT/全脑核团大小汇总.txt

echo "✅ 核团统计文件生成完成："
ls $STAT_OUTPUT/
```
   **路径说明**：
   - `你的统计结果保存路径`：例如 `/Users/用户名/Documents/脑区核团统计结果`

**统计文件说明**：
- `深部核团体积统计.tsv`：包含丘脑、尾状核、海马等深部核团的体积（mm³）；
- `左右脑皮层分区统计.txt`：包含额叶、颞叶、顶叶等皮层分区的表面积、平均厚度、体积；
- 可直接用Excel/SPSS/R打开进行定量分析。

### 3.4 步骤4：导出带解剖颜色的3D PLY模型
生成可导入Blender/MeshLab的彩色PLY模型（保留FreeSurfer原生解剖分区颜色）：
#### 4.1 第一步：Python导出脚本（`export_colored_ply.py`）
```python3
#!/usr/bin/env python3
"""
从FreeSurfer重建结果导出带解剖颜色的PLY模型
适配Mac ARM架构，解决格式兼容/颜色丢失问题
"""
import nibabel.freesurfer as fs
import numpy as np

def export_colored_ply(surf_path, annot_path, output_path):
    """
    核心功能：读取表面+注释，导出带RGB颜色的PLY文件
    :param surf_path: FreeSurfer表面文件（如lh.pial）
    :param annot_path: 解剖注释文件（如lh.aparc.annot）
    :param output_path: 输出彩色PLY路径
    """
    # 1. 读取表面几何（顶点坐标+面索引）
    coords, faces = fs.read_geometry(surf_path)
    # 2. 读取注释（标签+颜色表+脑区名称）
    labels, ctab, _ = fs.read_annot(annot_path)
    
    # 3. 映射每个顶点的解剖颜色（RGB 0-255）
    vertex_colors = np.zeros((len(coords), 3), dtype=np.uint8)
    for i, label in enumerate(labels):
        if 0 <= label < len(ctab):
            vertex_colors[i] = ctab[label, :3]  # 取原生解剖色
        else:
            vertex_colors[i] = [128, 128, 128]  # 未知区域设为灰色
    
    # 4. 写入标准PLY文件（避免Blender识别错误）
    with open(output_path, 'w') as f:
        # PLY头部（严格遵循标准格式）
        f.write("ply\n")
        f.write("format ascii 1.0\n")
        f.write(f"element vertex {len(coords)}\n")
        f.write("property float x\n")
        f.write("property float y\n")
        f.write("property float z\n")
        f.write("property uchar red\n")
        f.write("property uchar green\n")
        f.write("property uchar blue\n")
        f.write(f"element face {len(faces)}\n")
        f.write("property list uchar int vertex_indices\n")
        f.write("end_header\n")
        
        # 写入顶点+颜色
        for v, c in zip(coords, vertex_colors):
            f.write(f"{v[0]:.6f} {v[1]:.6f} {v[2]:.6f} {c[0]} {c[1]} {c[2]}\n")
        # 写入面
        for face in faces:
            f.write(f"3 {face[0]} {face[1]} {face[2]}\n")
    
    print(f"✅ 彩色PLY已生成：{output_path}")

# === 主程序 ===
if __name__ == "__main__":
    # 配置路径
    SUBJECTS_DIR = "你的受试者目录路径"
    OUTPUT_DIR = "你的PLY模型输出目录路径"

    # 导出左脑彩色PLY
    export_colored_ply(
        surf_path=f"{SUBJECTS_DIR}/surf/lh.pial",
        annot_path=f"{SUBJECTS_DIR}/label/lh.aparc.annot",
        output_path=f"{OUTPUT_DIR}/左脑_彩色.ply"
    )

    # 导出右脑彩色PLY
    export_colored_ply(
        surf_path=f"{SUBJECTS_DIR}/surf/rh.pial",
        annot_path=f"{SUBJECTS_DIR}/label/rh.aparc.annot",
        output_path=f"{OUTPUT_DIR}/右脑_彩色.ply"
    )

    print("\n🎨 所有3D彩色模型导出完成！")
```
   **路径说明**：
   - `你的受试者目录路径`：与前面设置的`SUBJECTS_DIR`一致
   - `你的PLY模型输出目录路径`：例如 `/Users/用户名/Documents/3D模型`

#### 4.2 第二步：运行脚本生成PLY文件
```bash
# 安装依赖（仅需执行一次）
pip install nibabel numpy

# 运行导出脚本
python3 export_colored_ply.py
```

#### 4.3 第三步：Blender导入验证（可选）
1. 打开Blender → 删除默认立方体；
2. `File` → `Import` → `Stanford (.ply)`，导入`左脑_彩色.ply`/`右脑_彩色.ply`；
3. 显示彩色：选中模型 → 右侧「材质属性」→ 新建材质 → 基础色选择「顶点颜色」→ 选`Col`通道。

## 四、关键文件目录总结
| 路径/文件                          | 用途                                  |
|------------------------------------|---------------------------------------|
| `freesurfer_output/sub-01/`        | recon-all重建结果根目录               |
| `sub-01/surf/lh/rh.pial`           | 左右脑皮层表面文件                    |
| `sub-01/label/lh/rh.aparc.annot`   | 解剖分区注释文件（含颜色信息）        |
| `sub-01/mri/aseg.mgz`              | 深部核团自动分割文件                  |
| `脑区核团统计结果/`                | 核团体积、皮层表面积等定量数据        |
| `左脑_彩色.ply/右脑_彩色.ply`      | 带解剖颜色的3D模型文件                |

合并左右脑文件为：
```Python
"""
合并两个彩色 PLY 文件（左脑 + 右脑）为一个文件。
"""

import struct
import sys
import os


def parse_ply_header(filepath):
    """解析 PLY 文件头，返回 (header_lines, vertex_count, face_count, is_binary, binary_format, properties)"""
    properties = []
    vertex_count = 0
    face_count = 0
    is_binary = False
    binary_format = None
    header_lines = []

    with open(filepath, "rb") as f:
        while True:
            line = f.readline()
            if not line:
                raise ValueError(f"未找到 end_header: {filepath}")
            line_str = line.decode("ascii", errors="replace").strip()
            header_lines.append(line_str)

            if line_str.startswith("format"):
                parts = line_str.split()
                if "binary_little_endian" in line_str:
                    is_binary = True
                    binary_format = "little"
                elif "binary_big_endian" in line_str:
                    is_binary = True
                    binary_format = "big"
                # else: ascii

            elif line_str.startswith("element vertex"):
                vertex_count = int(line_str.split()[-1])
            elif line_str.startswith("element face"):
                face_count = int(line_str.split()[-1])
            elif line_str.startswith("property") and face_count == 0:
                # vertex properties (before face element)
                properties.append(line_str)

            if line_str == "end_header":
                header_end_offset = f.tell()
                break

    return header_lines, vertex_count, face_count, is_binary, binary_format, properties, header_end_offset


# PLY type -> (struct format char, byte size)
PLY_TYPE_MAP = {
    "float": ("f", 4),
    "float32": ("f", 4),
    "double": ("d", 8),
    "float64": ("d", 8),
    "int": ("i", 4),
    "int32": ("i", 4),
    "uint": ("I", 4),
    "uint32": ("I", 4),
    "short": ("h", 2),
    "int16": ("h", 2),
    "ushort": ("H", 2),
    "uint16": ("H", 2),
    "char": ("b", 1),
    "int8": ("b", 1),
    "uchar": ("B", 1),
    "uint8": ("B", 1),
}


def get_vertex_struct(properties, endian="little"):
    """根据属性列表构建 struct 格式"""
    prefix = "<" if endian == "little" else ">"
    fmt = prefix
    for prop in properties:
        parts = prop.split()
        # "property <type> <name>"
        ptype = parts[1]
        if ptype in PLY_TYPE_MAP:
            fmt += PLY_TYPE_MAP[ptype][0]
        else:
            raise ValueError(f"不支持的属性类型: {ptype}")
    return struct.Struct(fmt)


def read_ply(filepath):
    """读取 PLY 文件，返回 (vertices_data_bytes, face_lines_or_bytes, header_info)"""
    header_lines, vcount, fcount, is_binary, binary_format, properties, header_end = \
        parse_ply_header(filepath)

    print(f"  文件: {os.path.basename(filepath)}")
    print(f"  顶点数: {vcount}, 面数: {fcount}, 格式: {'binary' if is_binary else 'ascii'}")
    print(f"  属性: {len(properties)} 个")

    vertices = []
    faces = []

    if is_binary:
        vstruct = get_vertex_struct(properties, binary_format)
        with open(filepath, "rb") as f:
            f.seek(header_end)
            for _ in range(vcount):
                data = f.read(vstruct.size)
                vertices.append(vstruct.unpack(data))
            # 读取面数据（list property）
            for _ in range(fcount):
                # 通常是 uchar + N*int
                n_bytes = f.read(1)
                n = struct.unpack("B", n_bytes)[0]
                idx_fmt = "<" + "i" * n if binary_format == "little" else ">" + "i" * n
                idx_data = f.read(4 * n)
                indices = struct.unpack(idx_fmt, idx_data)
                faces.append(indices)
    else:
        with open(filepath, "r", errors="replace") as f:
            # 跳过 header
            for line in f:
                if line.strip() == "end_header":
                    break
            for _ in range(vcount):
                line = f.readline().strip()
                vertices.append(line)
            for _ in range(fcount):
                line = f.readline().strip()
                faces.append(line)

    return vertices, faces, vcount, fcount, is_binary, binary_format, properties


def merge_and_write(file1, file2, output):
    print(f"\n读取文件 1...")
    v1, f1, vc1, fc1, is_bin1, fmt1, props1 = read_ply(file1)

    print(f"\n读取文件 2...")
    v2, f2, vc2, fc2, is_bin2, fmt2, props2 = read_ply(file2)

    # 检查属性是否一致
    if props1 != props2:
        print("\n⚠️  警告: 两个文件的顶点属性不完全一致，将使用第一个文件的属性格式。")
        print(f"  文件1属性: {props1}")
        print(f"  文件2属性: {props2}")

    total_vertices = vc1 + vc2
    total_faces = fc1 + fc2
    vertex_offset = vc1  # 第二个文件的面索引需要偏移

    print(f"\n合并中...")
    print(f"  总顶点数: {total_vertices}")
    print(f"  总面数: {total_faces}")

    # 统一用 ASCII 格式输出（兼容性最好）
    with open(output, "w") as out:
        # 写 header
        out.write("ply\n")
        out.write("format ascii 1.0\n")
        out.write(f"element vertex {total_vertices}\n")
        for prop in props1:
            out.write(prop + "\n")
        if total_faces > 0:
            out.write(f"element face {total_faces}\n")
            out.write("property list uchar int vertex_indices\n")
        out.write("end_header\n")

        # 写顶点
        for v in v1:
            if isinstance(v, str):
                out.write(v + "\n")
            else:
                out.write(" ".join(str(x) for x in v) + "\n")
        for v in v2:
            if isinstance(v, str):
                out.write(v + "\n")
            else:
                out.write(" ".join(str(x) for x in v) + "\n")

        # 写面（偏移第二个文件的索引）
        for face in f1:
            if isinstance(face, str):
                out.write(face + "\n")
            else:
                out.write(f"{len(face)} " + " ".join(str(i) for i in face) + "\n")

        for face in f2:
            if isinstance(face, str):
                parts = face.split()
                n = int(parts[0])
                indices = [str(int(parts[i + 1]) + vertex_offset) for i in range(n)]
                out.write(f"{n} " + " ".join(indices) + "\n")
            else:
                shifted = [i + vertex_offset for i in face]
                out.write(f"{len(shifted)} " + " ".join(str(i) for i in shifted) + "\n")

    print(f"\n✅ 合并完成! 输出文件: {output}")


if __name__ == "__main__":
    # ===== 修改这里的路径 =====
    file1 = "你的左脑PLY文件路径"
    file2 = "你的右脑PLY文件路径"
    output = "你的合并后输出文件路径"
    # ==========================

    if not os.path.exists(file1):
        print(f"❌ 文件不存在: {file1}")
        sys.exit(1)
    if not os.path.exists(file2):
        print(f"❌ 文件不存在: {file2}")
        sys.exit(1)

    merge_and_write(file1, file2, output)
```
   **路径说明**：
   - `你的左脑PLY文件路径`：例如 `/Users/用户名/Documents/3D模型/左脑_彩色.ply`
   - `你的右脑PLY文件路径`：例如 `/Users/用户名/Documents/3D模型/右脑_彩色.ply`
   - `你的合并后输出文件路径`：例如 `/Users/用户名/Documents/3D模型/合并_大脑_彩色.ply`

## 五、常见问题解决
1. **recon-all报错“权限不足”**：执行`chmod -R 755 $SUBJECTS_DIR`赋予目录权限；
2. **freeview无法打开**：确认`DISPLAY`环境变量配置（Mac需安装XQuartz）；
3. **PLY文件Blender显示黑白**：必开「顶点颜色」显示（步骤4.3）；
4. **核团统计为空**：重建时需确保`-all`参数执行完整，检查`aseg.mgz`是否存在。

## 六、核心流程回顾
1. **重建**：`recon-all`完成脑表面+解剖分区重建；
2. **验证**：`freeview`可视化确认重建质量；
3. **统计**：`asegstats2table`/`mris_anatomical_stats`生成核团大小文件；
4. **导出**：Python脚本生成带解剖颜色的3D PLY模型。
