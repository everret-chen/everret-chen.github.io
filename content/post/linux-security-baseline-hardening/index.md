---
title: Linux安全基线加固（一）
slug: linux-security-baseline-hardening
date: 2026-11-13T22:00:00+08:00
lastmod: 2026-11-13T22:00:00+08:00
description: 以 Kylin V10 为例，把 login.defs、pam_pwquality、pam_faillock 三件套配明白——谁管新用户、谁管老用户、谁会把你自己锁在外面。
categories:
    - 服务器运维
tags:
    - Linux
    - 安全加固
---

## 前情提要

今天无所事事，就做了个 Kylin V10 安全加固，这篇博客权当整理一下知识吧。

不过知识写起来好费时间，让我分开几份写……

1. 口令有效期 —— `/etc/login.defs`
2. 口令复杂度 —— `pam_pwquality.so`
3. 登录失败锁定 —— `pam_faillock.so`

<!--more-->

## 一、口令有效期：`/etc/login.defs`

首先要做的就是密码复杂度和有效期的事情：用户口令要定期更换，长度建议 8 位以上，必须包含大小写字母、数字和特殊字符，登录失败多次要锁定。

先了解配置文件 `/etc/login.defs`，它管的是口令有效期，主要用于两方面：**用户创建**和**密码策略**。

| 参数 | 含义 | 建议值 |
| :--- | :--- | :--- |
| `PASS_MAX_DAYS` | 口令最长有效天数，`9999` 相当于不过期 | 90 |
| `PASS_MIN_DAYS` | 两次修改口令的最短间隔 | 1 ~ 7 |
| `PASS_MIN_LEN` | 口令最短长度（仅作下限参考，实际由 PAM 把关） | 8 |
| `PASS_WARN_AGE` | 口令过期前多少天开始提醒用户 | 7 |

配置示例：

```bash
# /etc/login.defs
PASS_MAX_DAYS   90
PASS_MIN_DAYS   1
PASS_WARN_AGE   7
```

查看当前策略：

```bash
grep -En "^PASS_" /etc/login.defs
```

> [!WARNING]
> `login.defs` 只影响**新建用户**的初始属性，对已经存在的用户完全无效。老用户要靠 `chage` 单独改。

老用户的处理方式：

```bash
chage -l everret                    # 查看某用户当前生效的口令策略
chage -M 90 -m 1 -W 7 everret       # 修改最长有效期/最小间隔/提前提醒
chage -d 0 everret                  # 强制该用户下次登录立即改口令
```

## 二、口令复杂度：`pam_pwquality.so`

配完 `login.defs`，接下来是口令复杂度的要求，通过 `pam_pwquality.so` 实现。它的前身是 `pam_cracklib.so`（RHEL 7 / CentOS 7 之后被正式取代）。

它的唯一职责是：**用户设置或修改口令时，检查新口令是否符合要求**——不负责校验历史口令，也不负责登录阶段的判定。

```bash
# /etc/pam.d/system-auth 的 password 段
password requisite pam_pwquality.so try_first_pass local_users_only \
    minlen=8 dcredit=-1 lcredit=-1 ucredit=-1 ocredit=-1 enforce_for_root
```

参数解释：

| 参数 | 含义 |
| :--- | :--- |
| `password` | 这行配置只在「修改 / 设置口令」环节生效 |
| `requisite` | 一票否决，不符合要求立刻中断流程，不再往下走 |
| `minlen=8` | 口令最小长度 8 |
| `dcredit=-1` | 数字至少 1 个（若写成 `1`，则最多 1 个） |
| `lcredit=-1` | 小写字母至少 1 个 |
| `ucredit=-1` | 大写字母至少 1 个|
| `ocredit=-1` | 特殊字符至少 1 个 |
| `enforce_for_root` | root 改口令时同样要检查 |

> [!TIP]
> `*credit` 的正负号很容易记反：**负数 = 至少包含 N 个**，**正数 = 最多包含 N 个**，`0` = 不检查。基线加固一般都用负数。


## 三、登录失败锁定：`pam_faillock.so`

最后是口令失败锁定，主要分为认证 `auth` 和账户 `account` 两部分：**auth 阶段负责检查与计数，account 阶段负责判定锁定**。

```bash
# 第一部分：auth 认证阶段
## 预认证：锁定事件记录到审计日志，失败 3 次锁定，root 也一样，锁定 60 秒
auth        requisite      pam_faillock.so preauth audit deny=3 even_deny_root unlock_time=60
## 密码校验，正确就放行
auth        sufficient     pam_unix.so nullok try_first_pass
## 认证失败时执行，作用是给失败计数器 +1
auth        [default=die]  pam_faillock.so authfail audit deny=3 even_deny_root unlock_time=60

# 第二部分：account 账户阶段
## 锁定判定
account     required       pam_faillock.so
```

| 参数 | 含义 |
| :--- | :--- |
| `deny=3` | 连续失败 3 次即锁定 |
| `even_deny_root` | root 也参与锁定（不加这个，爆破 root 不受限） |
| `unlock_time=60` | 锁定时长 60 秒；`0` 表示只能人工解锁 |
| `audit` | 把锁定事件写入审计日志，便于溯源 |
| `preauth` / `authfail` | 前者在口令校验**前**查是否已锁，后者在校验失败时**累加计数** |

查看被锁定用户 `everret` 的失败记录：

```bash
faillock --user everret
```

手动将计数器清零（解锁）：

```bash
faillock --user everret --reset
```

## 小结

| 加固项 | 配置文件 / 模块 | 作用范围 |
| :--- | :--- | :--- |
| 口令有效期 | `/etc/login.defs` | 仅新建用户的初始属性 |
| 老用户有效期 | `chage` | 指定已存在用户 |
| 口令复杂度 | `pam_pwquality.so` | 修改 / 设置口令时检查 |
| 失败锁定 | `pam_faillock.so` | 登录认证时计数与判定 |


