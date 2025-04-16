---
abbrlink: ''
categories:
- - Web3
date: '2025-04-16T16:40:21.592642+08:00'
tags:
- Web3
title: 0x2 Mint & Burn
updated: '2025-04-16T17:55:02.308+08:00'
---
# 0x2 Mint & Burn

## Publish Contract

```Move
module task2::my_coin;
    use sui::coin::{Self, Coin, TreasuryCap};
    use std::debug;
    use std::ascii::string;
//
    public struct MY_COIN has drop {}

    fun init(witness: MY_COIN, ctx: &mut TxContext) {
        let (treasury, metadata) = coin::create_currency(
            witness,
            6,
            b"DUDE",
            b"DUDE_COIN",
            b"This is DudeGuuud Coin.",
            option::none(),
            ctx);
        debug::print(&string(b"init MY_COIN"));
        transfer::public_freeze_object(metadata);
        transfer::public_transfer(treasury, ctx.sender())
    }

    public entry fun mint(
        treasury_cap: &mut TreasuryCap<MY_COIN>,
        amount: u64,
        recipient: address,
        ctx: &mut TxContext,
    ) {
        debug::print(&string(b"my_coin mint"));
        let coin = coin::mint(treasury_cap, amount, ctx);
        transfer::public_transfer(coin, recipient)
    }

    public entry fun burn(
        treasury_cap: &mut TreasuryCap<MY_COIN>,
        coin: Coin<MY_COIN>
    ) {
        debug::print(&string(b"burn"));
        coin::burn(treasury_cap, coin);
    }
}
```

其主要内容是mint自定义代币和销毁，因为treasury_cap属于私有，只有所属钱包（第一次publish见证者账户）

* 在当前代码中，**TreasuryCap** 在 **init** 函数中通过 **transfer::public\_transfer** 转移到 **ctx.sender()**（发布者的地址）。
* 这意味着只有发布者的地址（见证者账户）持有 **TreasuryCap**，可以调用 **mint** 和 **burn**。
* 其他账户无法直接操作，除非发布者显式转移 **TreasuryCap**（例如通过 **transfer::public\_transfer**）

同时如果需要共享TreasuryCap的权限

```
module task2::my_coin {
    use sui::coin::{Self, Coin, TreasuryCap};
    use sui::transfer;
    use sui::tx_context::{Self, TxContext};
    use sui::object::{Self, UID};
    use sui::event;
    use std::ascii::string;
    use std::option;

    // 代币结构体
    public struct MY_COIN has drop {}

    // 共享对象，用于存储 TreasuryCap
    public struct SharedTreasury has key {
        id: UID,
        treasury: TreasuryCap<MY_COIN>,
    }

    // 铸造事件
    public struct MintEvent has copy, drop {
        amount: u64,
        recipient: address,
    }

    // 销毁事件
    public struct BurnEvent has copy, drop {
        amount: u64,
    }

    // 初始化函数：创建代币并将 TreasuryCap 放入共享对象
    fun init(witness: MY_COIN, ctx: &mut TxContext) {
        let (treasury, metadata) = coin::create_currency(
            witness,
            6,
            b"DUDE",
            b"DUDE_COIN",
            b"This is DudeGuuud Coin.",
            option::none(),
            ctx
        );
        // 冻结元数据
        transfer::public_freeze_object(metadata);
        // 创建共享对象并存储 TreasuryCap
        let shared_treasury = SharedTreasury {
            id: object::new(ctx),
            treasury,
        };
        transfer::share_object(shared_treasury);
    }

    // 公共铸造函数：任何人可调用
    public entry fun mint(
        shared: &mut SharedTreasury,
        amount: u64,
        recipient: address,
        ctx: &mut TxContext,
    ) {
        let coin = coin::mint(&mut shared.treasury, amount, ctx);
        event::emit(MintEvent { amount, recipient });
        transfer::public_transfer(coin, recipient);
    }

    // 公共销毁函数：任何人可调用
    public entry fun burn(
        shared: &mut SharedTreasury,
        coin: Coin<
```

如果将 **TreasuryCap** 存储在一个共享对象（**SharedTreasury**）中，允许任何人调用。
提供公共入口函数（**public entry fun**），通过共享对象间接访问 **TreasuryCap**，因为共享了所有权，**Call** 就可以了，要求对所有权表示敬畏。
