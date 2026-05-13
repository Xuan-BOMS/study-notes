# Astravein 角色系统与数值架构学习笔记

本文基于 `D:\Astravein` 主工作区，以及 `D:\Astravein_project` 下相关工作树整理。

目标不是只列文件，而是帮你看懂：现在项目里“角色”由哪些类组成，数值系统已经做到哪里，为什么还不能算完整，以及后续如果要做玩家、NPC、敌人共用的一套系统，应该补什么。

---

## 1. 当前结论

Astravein 现在已经有一套可运行的角色原型骨架。

它能支撑：

- 玩家移动
- 基础实体身高缩放
- 碰撞与交互范围
- 简单 NPC 对话
- 装备槽与背包原型
- 生命、魔力、施法
- 简单命中、暴击、伤害、抗性、状态效果 tick

但它还不是完整的统一角色系统。

现在最大的问题是：角色数据、组件数据、战斗结算数据还没有完全汇总到一个统一入口。玩家、普通 NPC、敌人目前不是同一种完整 Actor，只是共享了一部分基础类和组件。

如果后续目标是“玩家、NPC、敌人都能对话、装备、携带物品、战斗”，应该先补统一 Actor 层，而不是继续零散添加字段。

---

## 2. 工作树情况

当前相关工作树：

```text
D:\Astravein                          主目录，当前默认调试入口是 sandbox
D:\Astravein_project\Astravein_start  学院起始 worktree
D:\Astravein_project\Astravein_magic  施法 worktree
D:\Astravein_project\npc_text         NPC 对话实验 worktree
```

核心角色与数值文件在这些 worktree 中基本一致。

已发现的差异：

- `Astravein_magic` 的 `MagicCasterComponent` 多了测试用闲置回蓝字段。
- `Astravein_magic` 的 `CombatService` 多了 `invulnerable` 免伤判定。
- `npc_text` 额外有 `DialogueNPC`，用于接外部 `DialogueRunner`。

这说明主线结构已经比较稳定，但对话和施法分支里各自有少量实验性扩展。

---

## 3. 当前角色相关基类和类

### 3.1 BaseEntity

路径：

```text
res://Entities/BaseEntity/BaseEntity.gd
```

`BaseEntity` 是当前所有主要角色实体的根基类，继承自 `CharacterBody3D`。

它负责：

- 身高与参考身高
- 视觉节点缩放
- 碰撞体缩放
- 交互范围适配
- 角色名称、简介、使命
- 生命、精神、压力、心能
- 等级、经验
- 风格属性
- 能力、装备、关系挂点

当前 `BaseEntity` 上已有这些重要字段：

```text
height
reference_height
entity_name
summary
mission
stress
mind
max_mind
mental_power
level
experience
next_experience
style
abilities
equipment
relationships
max_health
health
```

需要注意：`BaseEntity` 已经有 `health/max_health`，但项目里还有 `HealthComponent` 也保存生命。这会导致后期出现“双写生命值”的风险。

### 3.2 StyleStats

路径：

```text
res://Entities/BaseEntity/StyleStats.gd
```

`StyleStats` 是一个 `Resource`，用于保存角色风格倾向。

当前字段：

```text
cautious     谨慎
clever       机智
flamboyant   张扬
dominant     强势
swift        迅捷
stealthy     隐匿
```

这类属性更像 CRPG 的性格、行为风格、对话倾向或施法风格，不适合直接和 `attack/defense` 混在一起理解。

### 3.3 PlayableCharacter

路径：

```text
res://Entities/Characters/Playable/PlayableCharacter.gd
```

`PlayableCharacter` 继承 `BaseEntity`，是可操作角色基类。

它新增：

```text
move_speed
gravity_scale
```

并负责读取输入：

```text
move_left
move_right
move_forward
move_backward
```

然后设置 `CharacterBody3D.velocity`，最后调用 `move_and_slide()`。

它还会把自己加入：

```text
controllable
```

这个 group 目前被 NPC 交互检测、相机跟随等系统使用。

### 3.4 Player

路径：

```text
res://Entities/Characters/Playable/Player/Player.gd
```

当前 `Player` 只是一个空子类：

```gdscript
extends "res://Entities/Characters/Playable/PlayableCharacter.gd"
class_name Player
```

这说明玩家目前没有自己的独立逻辑。它只是一个明确命名的可操作角色入口。

这种做法在早期是合理的：先保留扩展点，等玩家专属逻辑变多时再往里放。

### 3.5 InteractableNPC

路径：

```text
res://Entities/Characters/NPC/interactable_npc.gd
```

`InteractableNPC` 继承 `BaseEntity`，是当前 NPC 交互基类。

它负责：

- 检测玩家是否接近
- 显示交互提示
- 监听 `interact` 输入
- 显示对白气泡
- 施法时隐藏提示
- 避免多个 NPC 同时进入对话

关键字段：

```text
player_group
interaction_component_path
interact_prompt_path
dialogue_label_path
dialogue_display_seconds
interaction_approach_padding
prompt_vertical_padding
dialogue_vertical_padding
```

当前对话还是非常轻量的“气泡台词”，不是完整对话树。

### 3.6 HumanoidNPC

路径：

```text
res://Entities/Characters/NPC/humanoid_npc.gd
```

`HumanoidNPC` 继承 `InteractableNPC`。

它新增：

```text
dialogue_1
dialogue_2
dialogue_3
idle_animation
talk_animation
```

它的行为是三段对白轮转：

```text
第 1 次交互 -> dialogue_1
第 2 次交互 -> dialogue_2
第 3 次交互 -> dialogue_3
第 4 次交互 -> 回到 dialogue_1
```

这个类适合作为“普通人形 NPC 模板”，例如学生、路人、学院角色。

### 3.7 MagicDummy

路径：

```text
res://Entities/Characters/NPC/magic_dummy/magic_dummy.gd
```

`MagicDummy` 继承 `BaseEntity`，是当前施法与战斗测试用靶子。

它是目前最接近“敌人/战斗目标”的类。

它新增战斗字段：

```text
attack
defense
accuracy
dodge
crit_chance
crit_bonus
resist_fire
```

它还实现了：

```gdscript
func get_combat_stats() -> Dictionary
```

这点很重要。`CombatService` 会优先调用目标的 `get_combat_stats()`。如果一个角色没有这个方法，`CombatService` 才会尝试直接从节点属性上读取 `attack/defense/...`。

目前普通玩家和普通 NPC 没有统一实现 `get_combat_stats()`，这是后续需要补的核心缺口。

### 3.8 DialogueNPC

路径：

```text
D:\Astravein_project\npc_text\Entities\Characters\NPC\dialogue_npc.gd
```

这个类目前只在 `npc_text` 工作树中存在。

它继承 `InteractableNPC`，新增：

```text
dialogue_id
dialogue_runner_path
speaker_name
speaker_title
```

它的 `start_interaction()` 不再只显示简单气泡，而是调用外部 `DialogueRunner`：

```gdscript
start_dialogue(self, dialogue_id)
```

这代表项目已经试过更正式的对话入口，但还没有合入主目录。

---

## 4. 当前组件

### 4.1 HealthComponent

路径：

```text
res://Components/health_component.gd
```

字段：

```text
max_health
health
max_shield
shield
```

这是生命组件原型。

问题是 `BaseEntity` 里也有：

```text
max_health
health
```

后续必须决定生命值到底以哪个为准。

推荐：让生命当前值走组件，基础最大值走 stats，最终最大生命由 stats 聚合后同步给 HealthComponent。

### 4.2 MagicCasterComponent

路径：

```text
res://Components/magic_caster_component.gd
```

字段：

```text
max_mana
mana
base_element_purity
active_session
last_spell_blueprint
equipped_blueprints
```

能力：

- 判断魔力是否足够
- 消耗魔力
- 退还魔力
- 开始手绘施法 session
- 开始快捷施法 session
- 保存上一次法术蓝图

这是施法系统的角色侧入口。

### 4.3 EquipmentComponent

路径：

```text
res://Components/equipment_component.gd
```

它基于 Gloot 插件创建：

```text
Bag
WeaponSlot
ArmorSlot
AccessorySlotA
AccessorySlotB
```

当前启动物品：

```text
weapons/slime_focus
armor/slime_membrane
consumables/mana_droplet
```

它已经能初始化背包、创建槽位、自动装备默认武器和护甲。

但它还没有完整把装备上的 `EquipmentData.stat_flat_modifiers` 和 `stat_percent_modifiers` 合并到角色最终属性中。

### 4.4 InteractableComponent

路径：

```text
res://Components/interactable_component.gd
```

字段很少：

```text
enabled
```

信号：

```text
interacted(by: Node)
```

目前它更像交互区域标记，还不是完整交互系统。

---

## 5. 当前数值系统

### 5.1 StatBlock

路径：

```text
res://Systems/Stats/stat_block.gd
```

`StatBlock` 是通用数值字典。

核心字段：

```gdscript
@export var values: Dictionary = {}
```

它支持：

- 判断是否有某个属性
- 获取属性
- 设置属性
- 增加 flat 值
- 百分比乘法
- 克隆所有属性

它本身不规定属性有哪些。

优点是灵活。

缺点是没有 schema 时容易乱：有人写 `crit`，有人写 `critical`，有人写 `crit_chance`，最终系统读不到。

### 5.2 StatAggregator

路径：

```text
res://Systems/Stats/stat_aggregator.gd
```

`StatAggregator` 是属性聚合器。

当前计算顺序：

```text
base_stats
  -> flat_layers
  -> percent_layers
  -> clamp
```

也就是：

```text
基础值 + 固定加成，再乘百分比加成，最后限制最小最大值
```

例子：

```text
attack 基础 10
装备固定 +5
状态百分比 +20%

最终 attack = (10 + 5) * 1.2 = 18
```

这个顺序是合理的，也已经有测试覆盖。

当前缺口不是聚合器，而是“谁来收集这些层，并把结果挂到角色上”。

---

## 6. 当前战斗系统

### 6.1 CombatService

路径：

```text
res://Systems/Combat/combat_service.gd
```

`CombatService` 负责：

- 进入战斗
- 退出战斗
- 保存参与者
- 解析技能使用
- 调用命中计算
- 调用伤害计算
- 扣目标生命

它当前读取这些战斗属性：

```text
attack
defense
accuracy
dodge
crit_chance
crit_bonus
intelligence
mental_power
resist_fire
resist_ice
resist_lightning
```

这里可以看出问题：`CombatService` 已经假设这些属性存在，但普通 `BaseEntity` 并没有完整定义它们。

### 6.2 HitResolver

路径：

```text
res://Systems/Combat/hit_resolver.gd
```

命中公式：

```text
hit_rate = accuracy + hit_bias - dodge
```

然后限制在：

```text
0.05 到 1.0
```

暴击公式：

```text
total_crit_rate = crit_chance + crit_bias
crit_multiplier = 1.0 + crit_bonus
```

这是一套很轻量的原型公式。

### 6.3 DamageModel

路径：

```text
res://Systems/Combat/damage_model.gd
```

伤害计算大致是：

```text
raw = attack + base_damage + scaling_value * scaling_ratio
raw *= crit_multiplier

reduced = raw - defense
reduced *= 1.0 - resist_damage_type
damage = max(reduced, min_damage)
```

例如火焰伤害会读：

```text
resist_fire
```

冰霜伤害会读：

```text
resist_ice
```

这说明后续新增伤害类型时，属性命名要保持：

```text
resist_<type>
```

### 6.4 StatusEffectRunner

路径：

```text
res://Systems/Combat/status_effect_runner.gd
```

它能：

- 给目标挂状态
- 记录剩余时间
- 按 tick 间隔触发 payload
- 到期移除状态

当前它只是发信号，并不直接把状态 modifier 合并进角色最终属性。

也就是说，`StatusEffectData` 已经有：

```text
stat_flat_modifiers
stat_percent_modifiers
```

但状态效果对最终 stats 的影响链路还没有闭合。

---

## 7. 物品、装备、技能、状态数据

当前数据基类：

```text
ItemData
EquipmentData
ConsumableData
SkillData
StatusEffectData
```

### 7.1 ItemData

通用物品字段：

```text
item_id
display_name
description
rarity
tags
icon
max_stack
base_value
metadata
```

### 7.2 EquipmentData

装备字段：

```text
slot_type
stat_flat_modifiers
stat_percent_modifiers
durability
max_durability
requirements
```

已有装备示例：

```text
slime_focus:
  mental_power +4
  mind +8

slime_membrane:
  defense +3
  health +12

arcane_ring:
  intelligence +2
  mental_power +8%
```

### 7.3 ConsumableData

消耗品字段：

```text
cooldown_seconds
use_time_seconds
use_effects
can_use_outside_combat
```

已有消耗品：

```text
mana_droplet:
  restore_mana 15

healing_potion_minor:
  restore_health 25
```

### 7.4 SkillData

技能字段：

```text
skill_id
display_name
description
mana_cost
cooldown_seconds
cast_time_seconds
tags
base_scaling
metadata
```

已有技能主要是火系：

```text
fire_bolt
fire_missile
fire_beam
fire_cone
fire_meteor
```

这些技能已经使用：

```text
damage
mental_power_scale
intelligence_scale
min_damage_ratio
damage_type
release_mode
```

### 7.5 StatusEffectData

状态字段：

```text
effect_id
display_name
description
duration_seconds
tick_interval_seconds
stack_mode
max_stacks
stat_flat_modifiers
stat_percent_modifiers
tick_payload
```

已有状态：

```text
burn_small:
  每 1 秒造成 3 点 fire 伤害
  持续 6 秒
  最多 3 层
```

---

## 8. 当前已有属性总表

### 8.1 身体与场景属性

```text
height
reference_height
move_speed
gravity_scale
```

### 8.2 角色说明属性

```text
entity_name
summary
mission
level
experience
next_experience
relationships
```

### 8.3 资源属性

```text
health
max_health
shield
max_shield
mana
max_mana
mind
max_mind
stress
```

### 8.4 魔法属性

```text
mental_power
base_element_purity
intelligence
```

其中 `intelligence` 已经被装备和技能使用，但还没有统一挂到 `BaseEntity` 或角色 stats schema 里。

### 8.5 战斗属性

```text
attack
defense
accuracy
dodge
crit_chance
crit_bonus
```

这些目前主要存在于 `MagicDummy` 和测试 dummy 中。

### 8.6 抗性属性

```text
resist_fire
resist_ice
resist_lightning
```

实际主数据里现在主要使用 `resist_fire`。

### 8.7 风格属性

```text
cautious
clever
flamboyant
dominant
swift
stealthy
```

---

## 9. 目前缺少什么

### 9.1 缺统一 Actor 层

现在有：

```text
BaseEntity
PlayableCharacter
InteractableNPC
HumanoidNPC
MagicDummy
```

但缺一个统一承载“角色能力”的层。

可以理解成：

```text
Actor = 能被世界识别、能持有数据、能参与系统交互的角色单位
```

Actor 应该能统一提供：

```text
身份
阵营
基础属性
当前资源
装备
背包
技能
状态
关系
对话入口
战斗入口
死亡/失效逻辑
```

现在这些能力分散在多个脚本和组件里。

### 9.2 缺统一最终属性入口

建议每个可战斗角色都能回答：

```gdscript
func get_final_stats() -> Dictionary
func get_combat_stats() -> Dictionary
```

其中 `get_final_stats()` 做完整聚合：

```text
基础属性
+ 装备固定加成
+ 状态固定加成
* 装备百分比加成
* 状态百分比加成
-> clamp
```

`get_combat_stats()` 则从最终属性里挑出战斗系统需要的 key。

### 9.3 缺敌人基类

现在 `MagicDummy` 是靶子，不是正式敌人。

正式敌人至少需要：

```text
EnemyCharacter
faction
aggro_range
target
ai_state
loot_table
experience_reward
death_state
```

否则后续战斗会只能打 dummy，很难扩成可玩的 CRPG 战斗。

### 9.4 缺阵营与关系系统

`relationships` 目前只是字典挂点。

建议后续至少拆出：

```text
FactionComponent
RelationshipComponent
```

它们解决不同问题：

- faction：谁和谁敌对、友好、中立
- relationship：某个 NPC 对玩家的个人态度

阵营决定默认战斗关系，个人关系影响对话、任务、价格、支援。

### 9.5 缺物品使用链路

现在有 `ConsumableData.use_effects`，但还没有完整“使用物品 -> 检查冷却 -> 执行效果 -> 扣数量”的正式链路。

建议后续有：

```text
InventoryUseService
```

处理：

```text
restore_health
restore_mana
apply_status
remove_status
trigger_dialogue_flag
```

### 9.6 缺状态属性加成闭环

`StatusEffectData` 已经能定义 stat modifiers。

但 `StatusEffectRunner` 当前只管理 tick 和到期，没有把 active effects 作为 stat layer 交给 `StatAggregator`。

后续需要让状态系统提供：

```gdscript
func get_flat_modifier_layers(target: Node) -> Array[Dictionary]
func get_percent_modifier_layers(target: Node) -> Array[Dictionary]
```

然后角色最终属性聚合时读取它。

### 9.7 缺成长与升级规则

`BaseEntity` 有：

```text
level
experience
next_experience
```

但还没有：

```text
经验获取
升级
属性成长
技能解锁
等级需求检查
```

装备里已有 `requirements = {"level": 1}`，但缺正式检查链路。

---

## 10. 推荐的最小统一属性 schema

不要一开始做太复杂。先统一最小可用集合。

### 10.1 生存

```text
max_health
max_shield
defense
```

`health/shield` 是当前值，不建议当成装备长期加成属性。装备应该加 `max_health`，而不是加 `health`。

当前 `slime_membrane` 写的是：

```text
health +12
```

后续更推荐改成：

```text
max_health +12
```

### 10.2 魔法与精神

```text
max_mana
max_mind
mental_power
intelligence
stress_limit
```

`mind` 和 `mana` 要明确区别：

- `mana`：施法资源，消耗后可恢复
- `mind`：精神容量、意志、认知、对话/技能检定资源
- `stress`：压力、污染、精神负担
- `mental_power`：法术强度
- `intelligence`：知识、理解、学习、理性检定

### 10.3 命中与伤害

```text
attack
spell_power
accuracy
dodge
crit_chance
crit_bonus
initiative
```

如果想区分物理和法术，可以后续拆成：

```text
physical_power
spell_power
```

但当前项目已有 `mental_power`，所以短期可以先让法术继续吃 `mental_power`。

### 10.4 抗性

```text
resist_physical
resist_fire
resist_ice
resist_lightning
resist_poison
resist_mind
```

命名必须统一成：

```text
resist_<damage_type>
```

### 10.5 行动

```text
move_speed
action_speed
cast_speed
cooldown_reduction
```

现在 `move_speed` 在 `PlayableCharacter` 上。后续如果敌人和 NPC 也移动，最好纳入统一 stats。

### 10.6 社交与探索

```text
persuasion
intimidation
insight
lore
stealth
perception
reputation
```

这些会服务：

- 对话选项
- 任务分支
- 潜行
- 发现隐藏信息
- 阵营反应

---

## 11. 推荐架构

### 11.1 不要继续把所有字段塞进 BaseEntity

`BaseEntity` 应该保持“世界实体基础能力”：

```text
身高
碰撞
视觉
交互形状
基础身份
```

不要把它变成所有系统的垃圾桶。

### 11.2 增加 CharacterStatsComponent

推荐新增：

```text
res://Components/character_stats_component.gd
```

职责：

```text
保存 base_stats
读取装备加成
读取状态加成
调用 StatAggregator
缓存 final_stats
提供 get_stat()
提供 get_final_stats()
提供 get_combat_stats()
```

这样玩家、NPC、敌人都可以挂同一个 stats 组件。

### 11.3 角色节点推荐结构

```text
Player / NPC / Enemy
  AnimatedSprite3D
  CollisionShape3D
  Components
    CharacterStatsComponent
    HealthComponent
    EquipmentComponent
    MagicCasterComponent
    InventoryComponent 或 Gloot Bag
    StatusEffectComponent
    FactionComponent
    DialogueComponent
```

不同角色可以不挂某些组件。

例如：

- 路人 NPC 可以没有 `MagicCasterComponent`
- 敌人可以没有 `DialogueComponent`
- 商人可以有 `InventoryComponent` 和 `DialogueComponent`
- 法师敌人可以有 `MagicCasterComponent`

### 11.4 CombatService 不应该到处猜字段

当前 `CombatService` 会尝试：

```gdscript
target.get("attack")
target.get("defense")
...
```

这在原型期可以，正式系统最好改成：

```gdscript
if target.has_method("get_combat_stats"):
    return target.get_combat_stats()
```

或者从统一 stats 组件读取。

这样战斗系统不会关心玩家、NPC、敌人具体是什么类。

---

## 12. 学习重点：Godot 里的继承和组件

### 12.1 继承适合表达“是什么”

例如：

```text
Player 是 PlayableCharacter
PlayableCharacter 是 BaseEntity
HumanoidNPC 是 InteractableNPC
InteractableNPC 是 BaseEntity
```

这就是继承。

继承适合稳定关系：

```text
玩家一定是可控角色
人形 NPC 一定是可交互 NPC
```

### 12.2 组件适合表达“有什么能力”

例如：

```text
这个角色有生命
这个角色有装备
这个角色能施法
这个角色能对话
这个角色有阵营
```

这些不适合都塞进继承树。

因为：

- 有的 NPC 能对话但不能战斗
- 有的敌人能战斗但不能对话
- 有的角色能装备但不能施法
- 有的怪物能施法但没有普通背包

所以更适合组件。

### 12.3 一个实用判断

问自己一句：

```text
这个东西是角色的身份，还是角色的一种能力？
```

如果是身份，用继承。

如果是能力，用组件。

例子：

```text
Player             身份 -> 类
HumanoidNPC        身份 -> 类
Health             能力/状态 -> 组件
Equipment          能力 -> 组件
Dialogue           能力 -> 组件
Faction            能力/属性 -> 组件
```

---

## 13. 学习重点：属性结算为什么要分层

一个角色最终攻击力可能来自很多地方：

```text
基础攻击
武器固定攻击
戒指百分比攻击
状态 buff
状态 debuff
难度修正
临时场地修正
```

如果每个系统都直接改 `attack`，很快就会乱。

更好的做法是保留分层：

```text
base_stats:
  attack: 10

equipment_flat:
  attack: 5

status_percent:
  attack: 0.2
```

最终计算：

```text
(10 + 5) * 1.2 = 18
```

这样有几个好处：

- 脱装备时可以移除装备层
- buff 结束时可以移除状态层
- UI 可以显示属性来源
- 存档更清晰
- 调试更容易

---

## 14. 学习重点：当前值和最大值要分开

生命有两个概念：

```text
max_health  最大生命
health      当前生命
```

装备通常应该影响最大生命：

```text
max_health +12
```

而不是直接影响当前生命：

```text
health +12
```

原因是：如果装备加当前生命，脱装备时应该扣哪里？如果角色已经受伤怎么办？如果装备临时失效怎么办？

更稳的规则是：

```text
装备、属性、等级影响 max_health
战斗、治疗、伤害影响 health
health 永远 clamp 到 0..max_health
```

魔力、护盾、精神也一样：

```text
max_mana / mana
max_shield / shield
max_mind / mind
```

---

## 15. 学习重点：Dictionary 很灵活，但必须有命名规范

Godot 的 `Dictionary` 很适合做数据驱动属性：

```gdscript
{
    "attack": 10,
    "defense": 3,
    "mental_power": 6
}
```

但它不会帮你检查拼写。

如果一个地方写：

```text
critical
```

另一个地方读：

```text
crit_chance
```

系统不会报错，只是读不到。

所以项目必须有一份稳定属性 key 表。

建议所有属性 key 只用小写蛇形：

```text
crit_chance
crit_bonus
max_health
resist_fire
mental_power
```

不要混用：

```text
critical
critChance
fire_resist
FireResist
```

---

## 16. 建议下一步实施顺序

### 第一步：冻结属性 key 表

先写一个小文档或常量文件，规定当前认可的属性 key。

短期最小集合：

```text
max_health
max_shield
max_mana
max_mind
stress_limit
attack
defense
accuracy
dodge
crit_chance
crit_bonus
mental_power
intelligence
move_speed
resist_physical
resist_fire
resist_ice
resist_lightning
resist_poison
resist_mind
```

### 第二步：新增 CharacterStatsComponent

让玩家、NPC、敌人都从同一个 stats 组件拿最终属性。

### 第三步：改装备属性

把装备里类似：

```text
health +12
```

调整为：

```text
max_health +12
```

避免当前值和最大值混淆。

### 第四步：让 CombatService 只读统一 combat stats

不要让战斗系统直接猜节点属性。

### 第五步：补 EnemyCharacter

先做最小敌人：

```text
继承 BaseEntity
挂 CharacterStatsComponent
挂 HealthComponent
挂 FactionComponent
实现死亡
实现 get_combat_stats()
```

AI 可以后补，不要一开始做复杂。

### 第六步：把 DialogueNPC 的思路合回主线

把 `npc_text` 里的 `DialogueNPC` 思路整理成正式对话组件或 NPC 子类。

---

## 17. 最后用一句话记住

当前 Astravein 的角色系统已经有了“骨架”，但还缺“统一内脏”。

`BaseEntity` 是身体基础，`PlayableCharacter` 和 `HumanoidNPC` 是身份分支，`HealthComponent / EquipmentComponent / MagicCasterComponent` 是能力组件，`StatBlock / StatAggregator` 是数值工具。

下一步最关键的是加一个统一的 `CharacterStatsComponent`，让玩家、NPC、敌人都从同一套最终属性里读取生命上限、战斗属性、装备加成、状态加成和对话检定属性。
