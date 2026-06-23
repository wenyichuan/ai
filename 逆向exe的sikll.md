# 修改 PyInstaller 打包 EXE 中 Python 常量的技能文档

## 概述

本文档记录了如何修改 PyInstaller 打包的 Python 可执行文件（.exe）中的常量值，而不丢失源代码的情况下进行二进制补丁。

**适用场景：**
- Python 源代码丢失，只有打包后的 .exe
- 需要修改代码中的某个常量（如限制值、配置参数等）
- 使用 PyInstaller 3.x 打包的 Python 3.6 应用

---

## 技术背景

### PyInstaller 归档结构

```
┌─────────────────────────────────┐
│  PE 可执行文件头 (MZ...)        │
├─────────────────────────────────┤
│  PyInstaller 引导程序           │
├─────────────────────────────────┤
│  Archive Start (归档开始)       │
│  ┌───────────────────────────┐  │
│  │ Entry 0 Data              │  │
│  │ Entry 1 Data              │  │
│  │ ...                       │  │
│  │ Entry N Data              │  │
│  ├───────────────────────────┤  │
│  │ TOC (目录表)               │  │
│  ├───────────────────────────┤  │
│  │ Cookie (88 bytes)         │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

### Cookie 结构 (88 字节)

| 偏移 | 大小 | 说明 |
|------|------|------|
| 0 | 8 | 魔数: `MEI\x0c\x0b\x0a\x0b\x0e` |
| 8 | 4 | pkg_len: 归档总长度 |
| 12 | 4 | toc_offset: TOC 相对于归档开始的偏移 |
| 16 | 4 | toc_len: TOC 长度 |
| 20 | 4 | pyvers: Python 版本号 (36 = Python 3.6) |
| 24 | 64 | pylibname: Python 库名 (如 `python36.dll`) |

### TOC 条目结构

每个条目的格式：
```
┌────────────────────────────────────┐
│ entry_size (4 bytes, big-endian)   │  # 条目总大小（含填充）
│ offset (4 bytes, big-endian)       │  # 数据在归档中的偏移
│ data_len (4 bytes, big-endian)     │  # 压缩后大小
│ uncompress_len (4 bytes, big-endian)│ # 压缩前大小
│ marker (1 byte, 0x01)              │  # 标记字节 ⚠️ 重要！
│ type_char (1 byte)                 │  # 类型: m=模块, s=脚本, z=PYZ
│ name (null-terminated string)      │  # 名称
│ padding (填充到 entry_size)        │
└────────────────────────────────────┘
```

### Python 3.6 Marshal 整数格式

```
TYPE_INT = 0x69 ('i')  →  4 字节小端整数
示例: 18000 = 0x4650  →  字节码: 69 50 46 00 00
```

---

## 工具准备

需要 Python 环境（推荐 Python 3.14 或其他版本）：

```bash
# 检查 Python
python --version
# 或
python3 --version
```

---

## 完整流程

### 第一步：分析 EXE 结构

```python
import struct

with open('query.exe', 'rb') as f:
    data = f.read()

# 查找 Cookie（在文件末尾 88 字节）
cookie_pos = len(data) - 88
pos = cookie_pos + 8  # 跳过魔数

pkg_len = struct.unpack('!I', data[pos:pos+4])[0]
toc_offset = struct.unpack('!I', data[pos+4:pos+8])[0]
toc_len = struct.unpack('!I', data[pos+8:pos+12])[0]
pyvers = struct.unpack('!i', data[pos+12:pos+16])[0]
pylibname = data[pos+16:pos+80].split(b'\0')[0]

archive_start = cookie_pos + 88 - pkg_len
toc_start = archive_start + toc_offset

print(f"archive_start: 0x{archive_start:x}")
print(f"toc_start: 0x{toc_start:x}")
print(f"pyvers: {pyvers}")
print(f"pylibname: {pylibname}")
```

### 第二步：解析 TOC 条目

```python
toc_data = data[toc_start:toc_start + toc_len]

pos = 0
entries = []
while pos < len(toc_data) - 16:
    fields = struct.unpack('!iiii', toc_data[pos:pos+16])
    entry_size = fields[0]
    data_offset = fields[1]
    data_len = fields[2]
    data_ulen = fields[3]

    # ⚠️ 关键：第 17 字节是标记 (0x01)，第 18 字节开始是类型+名称
    marker = toc_data[pos+16]
    name_start = pos + 17
    null_pos = toc_data.index(b'\0', name_start)
    raw_name = toc_data[name_start:null_pos+1]
    type_char = chr(raw_name[0])
    name = raw_name[1:].decode('utf-8', errors='replace')

    entries.append({
        'size': entry_size,
        'offset': data_offset,
        'dlen': data_len,
        'ulen': data_ulen,
        'marker': marker,
        'type': type_char,
        'name': name,
    })
    pos += entry_size

# 打印所有条目
for e in entries:
    print(f"[{e['type']}] {e['name']}: dlen={e['dlen']}, ulen={e['ulen']}")
```

### 第三步：查找目标常量

```python
import zlib

# 找到目标脚本条目（通常是 type='s' 的主脚本）
target_entry = None
for e in entries:
    if e['type'] == 's' and e['name'] == 'query':  # 替换为你的脚本名
        target_entry = e
        break

# 解压脚本
script_offset = archive_start + target_entry['offset']
script_compressed = data[script_offset:script_offset + target_entry['dlen']]
script = zlib.decompress(script_compressed)

# 搜索目标值（以 18000 为例）
old_value = 18000
target_bytes = b'\x69' + struct.pack('<I', old_value)  # TYPE_INT + 4字节LE

positions = []
idx = 0
while True:
    idx = script.find(target_bytes, idx)
    if idx == -1:
        break
    positions.append(idx)
    print(f"Found at offset {idx}")
    idx += 1

print(f"Total occurrences: {len(positions)}")
```

### 第四步：修改常量值

```python
new_value = 30000  # 新值
replacement = b'\x69' + struct.pack('<I', new_value)

# 替换所有出现
for pos in positions:
    script[pos:pos+len(target_bytes)] = replacement

print(f"Replaced {len(positions)} occurrences: {old_value} -> {new_value}")
```

### 第五步：重新压缩并重建归档

```python
import zlib

# 重新压缩
new_compressed = zlib.compress(bytes(script))
size_diff = len(new_compressed) - target_entry['dlen']
print(f"Size diff: {size_diff} bytes")

# 按偏移排序条目
entries_sorted = sorted(entries, key=lambda e: e['offset'])

# 重建数据段
new_data_section = bytearray()
current_offset = 0

for e in entries_sorted:
    abs_off = archive_start + e['offset']
    if e == target_entry:
        new_data_section.extend(new_compressed)
        e['new_dlen'] = len(new_compressed)
        e['new_ulen'] = len(script)
    else:
        new_data_section.extend(data[abs_off:abs_off + e['dlen']])
        e['new_dlen'] = e['dlen']
        e['new_ulen'] = e['ulen']
    e['new_offset'] = current_offset
    current_offset += e['new_dlen']

# 重建 TOC（⚠️ 注意包含 marker 字节）
new_toc = bytearray()
for e in entries_sorted:
    entry_header = struct.pack('!iiii', e['size'], e['new_offset'], e['new_dlen'], e['new_ulen'])
    # ⚠️ 关键：marker 字节 (0x01) + type_char + name + null
    marker_and_name = bytes([e['marker']]) + e['type'].encode('ascii') + e['name'].encode('utf-8') + b'\0'
    entry_data = entry_header + marker_and_name
    if len(entry_data) < e['size']:
        entry_data += b'\0' * (e['size'] - len(entry_data))
    new_toc.extend(entry_data)

# 重建 Cookie
new_pkg_len = len(new_data_section) + len(new_toc) + 88
new_toc_offset = len(new_data_section)

new_cookie = struct.pack(
    '!8sIIii64s',
    b'MEI\x0c\x0b\x0a\x0b\x0e',
    new_pkg_len,
    new_toc_offset,
    len(new_toc),
    pyvers,
    pylibname
)

# 组装新归档
new_archive = bytes(new_data_section) + bytes(new_toc) + new_cookie
```

### 第六步：写入新 EXE

```python
# 备份原文件
import shutil
shutil.copy2('query.exe', 'query.exe.backup')

# 替换归档
old_archive_end = archive_start + pkg_len
new_exe = data[:archive_start] + new_archive + data[old_archive_end:]

with open('query.exe', 'wb') as f:
    f.write(new_exe)

print(f"New exe size: {len(new_exe)} bytes")
print("Done!")
```

### 第七步：验证修改

```python
with open('query.exe', 'rb') as f:
    verify_data = f.read()

# 重新解析归档找到目标脚本
# ... (重复第二、三步的解析过程)

# 检查常量
target_30000 = b'\x69' + struct.pack('<I', 30000)
target_18000 = b'\x69' + struct.pack('<I', 18000)

print(f"30000 count: {script.count(target_30000)}")
print(f"18000 count: {script.count(target_18000)}")

if script.count(target_30000) > 0 and script.count(target_18000) == 0:
    print("SUCCESS!")
```

---

## 常见问题

### 1. 启动报错 "Failed to unmarshal code object"

**原因：** TOC 条目重建时漏掉了 marker 字节（0x01）

**解决：** 确保 TOC 条目格式为：
```python
entry_header = struct.pack('!iiii', size, offset, dlen, ulen)
marker_and_name = bytes([0x01]) + type_char.encode() + name.encode() + b'\0'  # ← 0x01 不能漏
```

### 2. 找不到目标常量

**可能原因：**
- 常量可能在 PYZ 归档的其他模块中（type='z'）
- 值可能以不同格式存储（如字符串 "18000" 而非整数）
- 搜索时需要考虑大小端

### 3. 压缩后大小变化导致偏移错误

**解决：** 必须重建整个归档，更新所有条目的偏移量和 TOC。

---

## 完整脚本模板

```python
#!/usr/bin/env python3
"""
PyInstaller EXE 常量修改工具
用法: python patch_exe.py <exe_path> <old_value> <new_value>
"""

import struct
import zlib
import shutil
import sys

def patch_pyinstaller_exe(exe_path, old_value, new_value):
    # 1. 备份
    backup_path = exe_path + '.backup'
    shutil.copy2(exe_path, backup_path)
    print(f"Backup created: {backup_path}")

    # 2. 读取文件
    with open(exe_path, 'rb') as f:
        data = bytearray(f.read())

    # 3. 查找 Cookie
    cookie_pos = len(data) - 88
    pos = cookie_pos + 8
    pkg_len = struct.unpack('!I', data[pos:pos+4])[0]
    toc_offset = struct.unpack('!I', data[pos+4:pos+8])[0]
    toc_len = struct.unpack('!I', data[pos+8:pos+12])[0]
    pyvers = struct.unpack('!i', data[pos+12:pos+16])[0]
    pylibname = data[pos+16:pos+80].split(b'\0')[0]
    archive_start = cookie_pos + 88 - pkg_len

    # 4. 解析 TOC
    toc_start = archive_start + toc_offset
    toc_data = data[toc_start:toc_start + toc_len]

    pos = 0
    entries = []
    while pos < len(toc_data) - 16:
        fields = struct.unpack('!iiii', toc_data[pos:pos+16])
        marker = toc_data[pos+16]
        name_start = pos + 17
        null_pos = toc_data.index(b'\0', name_start)
        raw_name = toc_data[name_start:null_pos+1]
        type_char = chr(raw_name[0])
        name = raw_name[1:].decode('utf-8', errors='replace')

        entries.append({
            'size': fields[0], 'offset': fields[1],
            'dlen': fields[2], 'ulen': fields[3],
            'marker': marker, 'type': type_char, 'name': name,
        })
        pos += fields[0]

    # 5. 查找并修改目标脚本
    target = b'\x69' + struct.pack('<I', old_value)
    replacement = b'\x69' + struct.pack('<I', new_value)
    total_patches = 0

    entries_sorted = sorted(entries, key=lambda e: e['offset'])
    new_data = bytearray()
    current_offset = 0

    for e in entries_sorted:
        abs_off = archive_start + e['offset']
        entry_data = data[abs_off:abs_off + e['dlen']]

        if e['type'] == 's':  # 脚本类型
            script = bytearray(zlib.decompress(entry_data))
            count = 0
            idx = 0
            while True:
                idx = script.find(target, idx)
                if idx == -1:
                    break
                script[idx:idx+len(target)] = replacement
                idx += len(target)
                count += 1
            if count > 0:
                print(f"  Patched {e['name']}: {count} occurrences")
                total_patches += count
                entry_data = zlib.compress(bytes(script))
                e['ulen'] = len(script)

        new_data.extend(entry_data)
        e['new_offset'] = current_offset
        e['new_dlen'] = len(entry_data)
        current_offset += len(entry_data)

    # 6. 重建 TOC
    new_toc = bytearray()
    for e in entries_sorted:
        header = struct.pack('!iiii', e['size'], e['new_offset'], e['new_dlen'], e['ulen'])
        name_bytes = bytes([e['marker']]) + e['type'].encode() + e['name'].encode() + b'\0'
        entry = header + name_bytes
        entry += b'\0' * (e['size'] - len(entry))
        new_toc.extend(entry)

    # 7. 重建 Cookie
    new_pkg_len = len(new_data) + len(new_toc) + 88
    new_cookie = struct.pack('!8sIIii64s',
        b'MEI\x0c\x0b\x0a\x0b\x0e',
        new_pkg_len, len(new_data), len(new_toc),
        pyvers, pylibname)

    # 8. 写入新文件
    new_archive = bytes(new_data) + bytes(new_toc) + new_cookie
    old_end = archive_start + pkg_len
    new_exe = data[:archive_start] + new_archive + data[old_end:]

    with open(exe_path, 'wb') as f:
        f.write(new_exe)

    print(f"Total patches: {total_patches}")
    print(f"New file size: {len(new_exe)} bytes")
    return total_patches > 0

if __name__ == '__main__':
    if len(sys.argv) != 4:
        print("Usage: python patch_exe.py <exe_path> <old_value> <new_value>")
        sys.exit(1)

    success = patch_pyinstaller_exe(sys.argv[1], int(sys.argv[2]), int(sys.argv[3]))
    sys.exit(0 if success else 1)
```

---

## 使用示例

```bash
# 修改 query.exe 中的 18000 为 30000
python patch_exe.py query.exe 18000 30000

# 输出:
# Backup created: query.exe.backup
#   Patched query: 2 occurrences
# Total patches: 2
# New file size: 6984371 bytes
```

---

## 注意事项

1. **始终先备份** - 修改前创建 .backup 文件
2. **marker 字节不能漏** - TOC 条目必须包含 0x01 标记字节
3. **压缩大小会变** - 修改后需要重建整个归档
4. **只改整数常量** - 字符串常量格式不同，需要额外处理
5. **Python 版本兼容** - 本方法适用于 Python 3.6 的 marshal 格式

---

## 相关文件

- `query.exe` - 目标可执行文件
- `query.exe.backup` - 原始备份
- `patch_pyinstaller_exe_skill.md` - 本文档

---

*文档创建时间: 2026-06-12*
*适用环境: Python 3.6 + PyInstaller 3.x*
