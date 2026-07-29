---
title: "Linux 特殊权限详解：SUID、SGID 与 Sticky Bit"
date: 2025-01-04
categories: [知识仓库,Linux]
tags: [权限管理, Linux]
---

### Linux 特殊权限详解：SUID、SGID 与 Sticky Bit

在基础权限（rwx）之外，Linux 还有三种特殊权限。

---

#### 1. 特殊权限是什么？

| 特殊权限   | 显示符号 | 八进制值 | 主要作用 |
|------------|----------|----------|----------|
| SUID       | s        | 4000     | 执行时以文件所有者身份运行 |
| SGID       | s        | 2000     | 执行时以文件所属组身份运行 + 目录继承组 |
| Sticky Bit | t        | 1000     | 目录中文件只能由文件所有者或者目录所有者删除 |

---

#### 2. SUID（Set User ID）

**作用**：文件运行时，临时使用**文件所有者**的权限（常用于需要 root 权限的命令，如 `passwd`）。

**设置方法**：

```bash
chmod u+s 文件名          # 符号方式
chmod 4755 文件名         # 数字方式（推荐）
```

**示例**：
```bash
chown root:root myscript.sh
chmod 4755 myscript.sh
```

---

#### 3. SGID（Set Group ID）

**作用**：
- 文件：以文件所属组权限运行
- 目录：新建的文件自动继承该目录的**组**（最常用场景）

**设置方法**：

```bash
chmod g+s 目录名          # 符号方式
chmod 2755 目录名         # 数字方式
```

**实用示例**（团队共享目录）：

```bash
mkdir /project
chown root:developers /project
chmod 2775 /project        # 关键：2775 = SGID + 权限
```

以后在 `/project` 里新建的文件会自动属于 `developers` 组。

---

#### 4. Sticky Bit（粘滞位）

**作用**：即使其他人有写权限，也**只能由文件所有者或目录所有者删除文件**。

**最经典例子**：`/tmp` 目录（`drwxrwxrwt`）

**设置方法**：

```bash
chmod +t 目录名           # 符号方式
chmod 1777 目录名         # 数字方式（最常用）
```

---

#### 5. 常用组合与移除

```bash
chmod 6755 文件名         # SUID + SGID
chmod 1777 目录名         # Sticky Bit + 完全开放

# 移除特殊权限
chmod u-s 文件名          # 移除 SUID
chmod g-s 目录名          # 移除 SGID
chmod -t 目录名           # 移除 Sticky Bit
```

---

#### 6. 实用命令

```bash
# 查找系统中的特殊权限文件
find / -perm -4000 -o -perm -2000 -o -perm -1000 2>/dev/null
```

---

**注意事项**：
- 特殊权限有安全风险，谨慎使用
- 生产环境尽量少用 SUID
- 共享目录推荐使用 `3775`（SGID+sticky bit）

---

