---
title: CS2 常用指令
published: 2026-06-22
description: "Counter-Strike 2 (CS2) 常用控制台指令与键位绑定，方便本地练习和作弊模式配置。"
tags: [CS2, 游戏, 控制台指令]
category: 游戏
draft: false
---

### 核心作弊与练习指令

- `sv_cheats 1` - 允许作弊模式
- `sv_infinite_ammo 1` - 开启无限子弹/投掷物模式
- `sv_infinite_ammo 2` - 开启无限弹匣/投掷物模式

### 投掷物练习

- `sv_grenade_trajectory_prac_pipreview 1` - 开启投掷物轨迹预览
- `sv_grenade_trajectory_prac_trailtime 4` - 设置投掷物轨迹显示时间为4秒
- `bind "F8" sv_rethrow_last_grenade` - 绑定重现上一个投掷物到"F8"键

### 机器人 (Bot) 控制

- `bot_kick` - 清除bot
- `bot_add` - 添加一个bot
- `bot_add_t` - 在T阵营添加一名bot
- `bot_add_ct` - 在CT阵营添加一名bot
- `bot_stop 1` - 禁止bot行动

### 游戏/回合管理

- `mp_restartgame 1` - 1秒后重新开始游戏
- `mp_freezetime 1` - 设置每回合开始准备时间为1秒
- `mp_ignore_round_win_conditions 1` - 忽略回合胜利条件
- `mp_warmup_end` - 结束热身时间
- `mp_roundtime 60` - 休闲模式/死亡竞赛/军备竞赛60min回合时间
- `mp_roundtime_defuse 60` - 竞技/搭档模式60min回合时间
- `mp_roundtime_hostage 60` - 人质模式60min回合时间

### 经济与购买

- `mp_buy_anywhere 1` - 允许在任何地方购买武器和装备
- `mp_buytime 99999` - 购买时间改成99999秒
- `mp_maxmoney 99999` - 设置最大金钱为99999
- `mp_startmoney 99999` - 设置初始所有玩家的金钱为99999
- `mp_afterroundmoney 99999` - 设置回合结束后所有玩家的金钱为99999
- `mp_weapons_allow_typecount -1` - 武器无限数量购买

### 玩家与服务器规则

- `mp_autoteambalance 0` - 禁用自动平衡队伍人数
- `mp_limitteams 10` - 设置队伍最大人数差为10
- `mp_solid_teammates 0` - 使队友变为非实体状态
- `mp_death_drop_gun 1` - 死亡竞赛/军备竞赛玩家死亡时掉落武器
- `mp_damage_headshot_only 1` - 设置只有命中头部伤害有效
- `mp_friendlyfire 0` - 关闭友军伤害
- `mp_respawn_on_death_t 1` - 设置T阵营死亡后立即复生
- `mp_respawn_on_death_ct 1` - 设置CT阵营死亡后立即复生

### 物品

- `give weapon_healthshot` - 获得治疗针

### 键位绑定

- `bind "F5" thirdperson` - 绑定第三人称到"F5"键
- `bind "F6" firstperson` - 绑定第一人称到"F6"键
- `bind "F7" noclip` - 绑定飞行穿墙到"F7"键
- `bind "F9" toggle host_timescale 1 20` - 绑定20倍游戏速度到"F9"键

### 界面与视觉

- `hud_showtargetid 0` - 瞄准时不显示玩家名称
- `cl_viewmodel_fieldofview` - 调整持枪视角 (您没有提供数值，例如 `cl_viewmodel_fieldofview 68`)

### 声音

- `snd_musicvolume 0.5` - 调整背景音乐大小为50%

### 连接与状态

- `status` - 查看steamid
- `connect {steamid}` - 使用steamid连接
- `connect {ip}` - 使用ip连接
