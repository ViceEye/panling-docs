# 盘灵古域 — 技术策划案

> 覆盖范围：任务系统、战斗系统、物品系统
> 目标读者：技术策划 / 程序员
> 版本：v1.0 | 2026-05-20

---

## 一、任务系统

### 1.1 系统架构

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│  QuestRegistry  │────▶│  QuestStateMachine │────▶│  RewardDispatcher │
└─────────────┘     └──────────────┘     └─────────────┘
        │                    │                      │
        ▼                    ▼                      ▼
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│  QuestConfig    │     │  PlayerQuestData   │     │  InventoryService │
│  (YAML/JSON)    │     │  (per-player)      │     │                   │
└─────────────┘     └──────────────┘     └─────────────┘
```

### 1.2 任务类型枚举

```java
enum QuestType {
    TUTORIAL,       // 新手引导
    MAIN_STORY,     // 主线剧情
    KNOWLEDGE_TRIAL,// 知识试炼（青龙试炼）
    FACTION_WAR     // 阵营战役
}
```

### 1.3 任务状态机

```
LOCKED → AVAILABLE → ACCEPTED → IN_PROGRESS → COMPLETED → REWARDED
                                      │
                                      └──→ FAILED (试炼答错)
```

| 状态 | 触发条件 | 持久化字段 |
|------|----------|-----------|
| LOCKED | 前置任务未完成 | — |
| AVAILABLE | 前置条件满足 | unlock_time |
| ACCEPTED | 玩家消耗对话泡泡与NPC交互 | accept_time |
| IN_PROGRESS | 接取后自动进入 | progress_data (JSON) |
| COMPLETED | 目标条件全部达成 | complete_time |
| REWARDED | 领取奖励 | reward_time |
| FAILED | 试炼答错 / 超时 | fail_count |

### 1.4 任务数据模型

```yaml
# quest_main_001.yml
id: "main_001"
type: MAIN_STORY
name: "找到引路人"
phase: 1  # 第一阶段：种族出生地
race_filter: ALL  # 或指定种族
prerequisites: []
objectives:
  - type: INTERACT_NPC
    npc_id: "guide_${player.race}"  # 动态解析种族引路人
    consume_item: "dialogue_bubble"
    count: 1
rewards:
  - type: ITEM
    item_id: "recommendation_letter"
    amount: 1
next_quest: "main_002"
dialogue_key: "guide_intro_${player.race}"
```

### 1.5 主线任务流程（数据驱动）

| 阶段 | quest_id | 目标类型 | 关键参数 |
|------|----------|----------|----------|
| 1 | main_001 | INTERACT_NPC | 引路人，消耗种族证明→获得推荐书 |
| 1 | main_002 | INTERACT_NPC | 族长，消耗推荐书→获得装备兑换券 |
| 1 | main_003 | INTERACT_NPC | 铁匠铺兑换员→获得职业武器+初心者套装 |
| 1 | main_004 | INTERACT_NPC | 丹药铺掌柜→获得新手疗愈丹 |
| 2 | main_005 | REACH_LOCATION | 皇城传送点区域 |
| 3 | main_006+ | MAIN_STORY | 皇城后续剧情链 |

### 1.6 NPC对话泡泡机制

```java
class DialogueBubbleService {
    // 每次NPC交互前调用
    boolean tryConsume(Player player, String npcId) {
        String bubbleType = npcRegistry.getRequiredBubble(npcId);
        if (player.inventory.has(bubbleType, 1)) {
            player.inventory.remove(bubbleType, 1);
            return true;
        }
        // 提示玩家从菜单书补充
        player.sendMessage("对话泡泡不足，请从菜单书中领取");
        return false;
    }

    // 菜单书补充（无限领取，每次固定数量）
    void replenishFromMenu(Player player) {
        player.inventory.add("dialogue_bubble", REPLENISH_AMOUNT);
        player.inventory.add("race_certificate", 1);
    }
}
```

### 1.7 剧情书系统

| 字段 | 说明 |
|------|------|
| book_id | 唯一标识 |
| race | 所属种族（各族独立内容） |
| pages | 书页内容列表 |
| unlock_quest | 解锁此书的前置任务 |
| complete_action | 阅读完毕后触发（解锁下一任务） |

实现要点：
- 使用 Minecraft 原生书本物品（Written Book）
- 监听 `PlayerInteractEvent` 检测书本关闭
- 记录 `player_book_progress[book_id] = last_page_read`
- 当 `last_page_read >= total_pages` 时标记为已读，触发 `complete_action`

### 1.8 青龙试炼（知识问答副本）

#### 副本实例化

```java
class DragonTrialInstance {
    String instanceId;
    List<Player> party;          // 组队成员
    int currentQuestion = 1;     // 当前题号（1-20，跳过8）
    int[] questionOrder = {1,2,3,4,5,6,7,9,10,11,12,13,14,15,16,17,18,19,20};
    Location startPoint;         // 起点坐标
    Location bossRoom;           // 通关区域

    void onAnswer(Player player, int questionIndex, String answer) {
        if (isCorrect(questionIndex, answer)) {
            advanceToNext();
        } else {
            // 全队传送回起点
            party.forEach(p -> p.teleport(startPoint));
            currentQuestion = 1;  // 重置进度
        }
    }
}
```

#### 题库数据结构

```yaml
# dragon_trial_questions.yml
questions:
  - id: 1
    text: "总共有几种种族生活在地上"
    answer: "3种"
    accept_patterns: ["3", "三", "3种", "三种"]
  - id: 2
    text: "青龙所镇守的森林"
    answer: "龙鳞之森"
    accept_patterns: ["龙鳞之森", "龙鳞"]
  # ... 共19题（跳过第8题）
  - id: 20
    text: "前面哪题被跳过"
    answer: "第8题"
    accept_patterns: ["8", "第8题", "第八题"]
```

#### 判定逻辑
- 答案匹配：模糊匹配 `accept_patterns` 列表
- 错误处理：任意一人答错 → 全队 `teleport(startPoint)` + 重置 `currentQuestion`
- 通关奖励：青龙的祝福（Buff效果）

### 1.9 阵营战役任务

```java
class FactionWarQuest {
    String warId;           // e.g. "reverse_heaven"
    String attackerRace;    // "zhan" (战神族)
    String defenderZone;    // "imperial_city"
    Location attackerNpc;   // 3267 20 -138
    Location defenderNpc;   // 124 59 -144
    int minPlayers;         // 最低参与人数
    WarState state;         // RECRUITING → ACTIVE → ENDED

    void tryStart(Player initiator) {
        if (initiator.race != attackerRace) return;
        if (getSignedUpCount() < minPlayers) return;
        state = WarState.ACTIVE;
        notifyAllParticipants();
    }
}
```

### 1.10 接口清单

| 接口 | 输入 | 输出 | 说明 |
|------|------|------|------|
| `GET /quest/available` | player_id | List<Quest> | 当前可接取任务 |
| `POST /quest/accept` | player_id, quest_id | QuestState | 接取任务 |
| `POST /quest/progress` | player_id, quest_id, event | QuestState | 上报进度 |
| `POST /quest/complete` | player_id, quest_id | Rewards | 完成并领奖 |
| `POST /trial/answer` | instance_id, player_id, answer | TrialResult | 试炼答题 |
| `POST /war/signup` | player_id, war_id, side | SignupResult | 报名战役 |


---

## 二、战斗系统

### 2.1 系统架构

```
┌────────────┐    ┌─────────────┐    ┌──────────────┐
│ DamageEngine   │───▶│ AggroManager    │───▶│ CombatEventBus   │
└────────────┘    └─────────────┘    └──────────────┘
       │                  │                     │
       ▼                  ▼                     ▼
┌────────────┐    ┌─────────────┐    ┌──────────────┐
│ WuxingCalc     │    │ MobAI           │    │ InstanceManager  │
│ (五行公式)     │    │ (仇恨表)        │    │ (副本管理)       │
└────────────┘    └─────────────┘    └──────────────┘
```

### 2.2 伤害计算公式

#### 基础伤害

```
BaseDamage = WeaponDamage × ClassMultiplier × LevelScaling
```

#### 五行相克加成

```
FinalDamage = BaseDamage × WuxingModifier

WuxingModifier:
  相克（攻克防）: 1.3  （+30% 伤害）
  相生（攻生防）: 0.85 （-15% 伤害，防御方受益）
  中性:          1.0
  被克（防克攻）: 0.7  （-30% 伤害）
```

#### 五行相克关系表（程序用）

```java
// attacker_element → Set<被克元素>
Map<Element, Element> OVERCOME = Map.of(
    METAL, WOOD,   // 金克木
    WOOD,  EARTH,  // 木克土
    EARTH, WATER,  // 土克水
    WATER, FIRE,   // 水克火
    FIRE,  METAL   // 火克金
);
```

#### 种族祝福减免

```
// 战神族：抗火 → 火属性伤害 ×0（免疫）
// 人族：抗性提升Ⅰ → 所有伤害 ×0.95
// 神族：生命提升Ⅲ → MaxHP ×1.3（间接减免）
```

### 2.3 仇恨系统（Aggro）

```java
class AggroTable {
    Map<UUID, double> threatMap;  // playerId → threat value
    UUID currentTarget;

    void addThreat(UUID playerId, double amount) {
        threatMap.merge(playerId, amount, Double::sum);
        recalculateTarget();
    }

    void recalculateTarget() {
        currentTarget = threatMap.entrySet().stream()
            .max(Map.Entry.comparingByValue())
            .map(Map.Entry::getKey)
            .orElse(null);
    }

    // 仇恨衰减（脱离范围后）
    void decay(UUID playerId, double rate) {
        threatMap.computeIfPresent(playerId, (k, v) -> v * rate);
    }
}
```

仇恨产生规则：

| 行为 | 仇恨值 | 说明 |
|------|--------|------|
| 造成伤害 | damage × 1.0 | 基础 |
| 治疗队友 | healAmount × 0.5 | 炼丹师治疗产生仇恨 |
| 金钟横扫 | damage × 1.5 | 坦克额外仇恨加成 |
| 首次攻击 | +100 固定值 | 开怪奖励 |

### 2.4 职业战斗实现

#### 2.4.1 战士 — 金钟分支

```java
class SweepAttack {
    double range = 3.0;          // 横扫半径（格）
    double damageRatio = 0.6;    // 对周围目标伤害为主目标的60%
    double threatMultiplier = 1.5;

    void execute(Player attacker, Entity mainTarget) {
        double mainDmg = calculateDamage(attacker, mainTarget);
        dealDamage(mainTarget, mainDmg);
        aggroManager.addThreat(mainTarget, mainDmg * threatMultiplier);

        // 横扫周围
        getNearbyEnemies(mainTarget.location, range).stream()
            .filter(e -> e != mainTarget)
            .forEach(e -> {
                double sweepDmg = mainDmg * damageRatio;
                dealDamage(e, sweepDmg);
                aggroManager.addThreat(e, sweepDmg * threatMultiplier);
            });
    }
}
```

#### 2.4.2 战士 — 破军分支

```java
class PoJunWeapon {
    double speedBonus = 0.15;       // +15% 移速
    double attackBonus = 1.25;      // 125% 攻击力
    boolean hasSweep = false;       // 无横扫

    // 破军优势：高单体DPS + 高机动性
    // 适合追击落单目标
}
```

#### 2.4.3 弓箭手 — 游侠分支

```java
class RangerTripleShot {
    int arrowCount = 3;
    double spreadAngle = 15.0;     // 扇形角度（度）
    double perArrowDamage = 0.45;  // 每箭伤害为基础的45%（3箭共135%）
    double attackSpeedBonus = 1.3;  // 攻速 ×1.3
    double moveSpeedBonus = 0.1;    // +10% 移速

    void shoot(Player archer, Location target) {
        double baseAngle = getAngle(archer.location, target);
        for (int i = 0; i < arrowCount; i++) {
            double angle = baseAngle + (i - 1) * spreadAngle;
            spawnProjectile(archer, angle, perArrowDamage);
        }
    }
}
```

#### 2.4.4 弓箭手 — 狙击分支

```java
class SniperShot {
    double damageMultiplier = 2.5;  // 全职业最高单次伤害
    double attackSpeedPenalty = 0.6; // 攻速 ×0.6
    double moveSpeedPenalty = -0.1;  // -10% 移速
    double maxRange = 48.0;          // 最大射程（格）

    // 弹道：直线飞行，无扇形扩散
}
```

#### 2.4.5 炼丹师 — 即时元素炼化

```java
class InstantAlchemy {
    // 操作方式：主手元素 + 副手炼丹炉（或反之），右键触发
    void onRightClick(Player alchemist) {
        ItemStack mainHand = alchemist.getMainHand();
        ItemStack offHand = alchemist.getOffHand();

        Element element = getElement(mainHand, offHand);
        ItemStack furnace = getFurnace(mainHand, offHand);
        if (element == null || furnace == null) return;

        double potency = furnace.quality.getPotencyMultiplier();
        executeSkill(alchemist, element, potency);
        consumeElement(alchemist, element, 1);
    }

    void executeSkill(Player caster, Element element, double potency) {
        switch (element) {
            case METAL -> metalSkill(caster, potency);  // 蚀骨刺：随机单体高伤
            case WOOD  -> woodSkill(caster, potency);   // 若木吟：自身+附近队友治疗
            case WATER -> waterSkill(caster, potency);  // 活水泉：祛毒
            case FIRE  -> fireSkill(caster, potency);   // 引火种：抗灼烧护盾
            case EARTH -> earthSkill(caster, potency);  // 厚土御：伤害吸收护盾
        }
    }
}
```

即时炼化技能参数表：

| 技能 | 元素 | 范围 | 目标 | 效果 | 持续时间 |
|------|------|------|------|------|----------|
| 蚀骨刺 | 金×1 | 5格 | 随机1敌人 | 高额伤害 | 瞬发 |
| 若木吟 | 木×1 | 8格 | 自身+队友 | 大量治疗 | 瞬发 |
| 活水泉 | 水×1 | 8格 | 自身+队友 | 清除毒素 | 瞬发 |
| 引火种 | 火×1 | 8格 | 自身+队友 | 抗灼烧 | 10s |
| 厚土御 | 土×1 | 8格 | 自身+队友 | 护盾 | 15s |

### 2.5 副本系统

#### 2.5.1 副本实例化

```java
class DungeonManager {
    Map<String, DungeonTemplate> templates;  // 副本模板
    Map<String, DungeonInstance> active;      // 活跃实例

    DungeonInstance create(String templateId, List<Player> party) {
        DungeonTemplate tmpl = templates.get(templateId);
        String instanceId = UUID.randomUUID().toString();
        // 复制区域到独立世界/区块
        World instanceWorld = copyRegion(tmpl.schematic, instanceId);
        DungeonInstance inst = new DungeonInstance(instanceId, party, instanceWorld);
        active.put(instanceId, inst);
        return inst;
    }
}
```

#### 2.5.2 副本列表

| 副本ID | 名称 | 入口坐标 | 实例坐标 | 类型 | 队伍要求 |
|--------|------|----------|----------|------|----------|
| chiyou_trial | 蚩尤试炼 | 3140 45 -197 | 2871 25 -184 | 战斗 | 完整团队 |
| dragon_trial | 青龙试炼 | 青龙神庙 | 2087 53 3 | 问答 | 组队 |
| tiger_dungeon | 白虎副本 | 白虎神庙 | 1828 11 -812 | 战斗 | 组队 |
| phoenix_dungeon | 朱雀副本 | 朱雀神庙 | 2471 33 63 | 战斗 | 组队 |
| turtle_dungeon | 玄武副本 | 玄武神庙 | 948 38 -1219 | 战斗 | 组队 |
| emperor_tomb | 始皇陵 | 695 33 -140 | 2890 20 -486 | 战斗 | 组队 |

#### 2.5.3 四圣兽祭坛传送

```java
class AltarTeleport {
    // 祭坛 → 神庙的传送逻辑
    static final Map<String, TeleportPair> ALTARS = Map.of(
        "dragon", new TeleportPair(loc(550,34,394), loc(1696,104,878)),
        "tiger",  new TeleportPair(loc(-693,153,-267), loc(2206,86,-899)),
        "phoenix",new TeleportPair(loc(240,63,732), loc(3198,148,-800)),
        "turtle", new TeleportPair(loc(-84,34,-521), loc(2180,88,958))
    );

    void onPlayerStep(Player player, Location altarLoc) {
        ALTARS.values().stream()
            .filter(pair -> pair.altar.distanceTo(altarLoc) < 3)
            .findFirst()
            .ifPresent(pair -> player.teleport(pair.temple));
    }
}
```

### 2.6 Boss与怪物刷新

```java
class BossSpawner {
    String bossId;
    Location spawnPoint;
    int respawnSeconds;       // 刷新间隔
    LootTable lootTable;
    boolean isAlive;

    void onDeath(Entity boss, Player killer) {
        isAlive = false;
        distributeLoot(killer, lootTable);
        scheduleRespawn(respawnSeconds);
    }
}
```

Boss刷新配置：

| Boss | 坐标 | 刷新间隔(建议) | 掉落 |
|------|------|---------------|------|
| 蜘蛛女王 | 666 47 40 | 300s | 三阶装备原核、赤铜锭、重生石 |
| 蜘蛛女王 | 555 39 345 | 300s | 三阶装备原核、赤铜锭、重生石 |
| 火焰魔王 | -280 48 787 | 600s | TBD |
| 水牛精 | -848 134 -137 | 600s | TBD |
| 神木妖精 | 637 50 378 | 600s | TBD |

### 2.7 阵营战（大规模PvP）

```java
class FactionWar {
    String warId;
    FactionWarState state;  // IDLE → RECRUITING → COUNTDOWN → ACTIVE → ENDED
    Set<UUID> attackers;
    Set<UUID> defenders;
    int duration = 1200;    // 战斗持续时间（秒）
    ScoreBoard scoreBoard;

    void tick() {
        if (state == ACTIVE && getElapsedSeconds() >= duration) {
            end();
            determineWinner();
            distributeRewards();
        }
    }
}
```

战役配置表：

| 战役 | 进攻方 | 进攻NPC坐标 | 防守方 | 防守NPC坐标 |
|------|--------|-------------|--------|-------------|
| 逆天计划 | 战神族 | 3267 20 -138 | 皇城 | 124 59 -144 |
| 斩妖除魔 | 人族 | 1661 181 185 | 大陆东部 | — |
| 进攻镇妖塔 | 妖族 | 2747 80 872 | 大陆西部 | -171 62 -180 |

### 2.8 网络同步要点

| 场景 | 同步策略 | tick rate |
|------|----------|-----------|
| 野外战斗 | 服务端权威，客户端预测 | 20 TPS (Minecraft默认) |
| 副本战斗 | 同上，副本实例隔离 | 20 TPS |
| 阵营战 | 服务端权威，AOI裁剪（视距内同步） | 20 TPS |
| 弹道同步 | 服务端计算命中，客户端播放特效 | — |


---

## 三、物品系统

### 3.1 系统架构

```
┌─────────────┐    ┌──────────────┐    ┌─────────────────┐
│ ItemRegistry    │───▶│ InventoryManager │───▶│ CraftingEngine      │
│ (物品定义)      │    │ (背包管理)       │    │ (锻造/炼丹)         │
└─────────────┘    └──────────────┘    └─────────────────┘
       │                   │                      │
       ▼                   ▼                      ▼
┌─────────────┐    ┌──────────────┐    ┌─────────────────┐
│ LootTable       │    │ EquipmentSlots   │    │ RecipeValidator     │
│ (掉落表)        │    │ (装备栏位)       │    │ (配方校验)          │
└─────────────┘    └──────────────┘    └─────────────────┘
```

### 3.2 物品分类体系

```java
enum ItemCategory {
    WEAPON,         // 武器
    ARMOR,          // 防具
    CONSUMABLE,     // 消耗品（丹药、疗愈丹）
    MATERIAL,       // 材料（元素、药引、锻核）
    CURRENCY,       // 货币（铜钱、银元、赤铜锭）
    FUNCTIONAL,     // 功能道具（对话泡泡、重生石、菜单书）
    QUEST_ITEM      // 任务道具（推荐书、种族证明、剧情书）
}

enum ItemRarity {
    COMMON,     // 普通
    UNCOMMON,   // 优良
    RARE,       // 稀有
    EPIC,       // 史诗
    LEGENDARY   // 传说
}
```

### 3.3 装备数据模型

```yaml
# item_weapon_jinzhong_t2.yml
id: "weapon_jinzhong_t2"
category: WEAPON
name: "二阶金钟剑"
tier: 2
class_restriction: WARRIOR
branch_restriction: JINZHONG
equip_slot: MAIN_HAND
stats:
  attack: 18
  defense_bonus: 8
  sweep_range: 3.0
  sweep_ratio: 0.6
  threat_multiplier: 1.5
wuxing_element: null  # 可附魔
set_id: "jinzhong_t2_set"
lore: "金钟一脉的标准制式武器"
```

### 3.4 装备阶级系统

| 阶级 | 获取方式 | 等级要求 | 材料需求 |
|------|----------|----------|----------|
| 一阶 | 新手兑换券 | 1 | 无 |
| 二阶 | 锻造台制作 | 15 | 精炼元素+雏形/锻核 |
| 三阶 | Boss掉落原核+锻造 | 25(建议) | 三阶原核+高级材料 |
| 四阶 | 高级副本 | 35(建议) | TBD |
| 五阶 | 顶级挑战 | 45(建议) | TBD |

### 3.5 装备套装系统

```java
class SetBonus {
    String setId;
    Map<Integer, List<StatModifier>> bonuses;  // 件数 → 加成列表

    // 规则：
    // 2件：激活基础属性加成
    // 4件同属性：激活职业专属 + 传说增幅
    List<StatModifier> getActiveBonus(Player player) {
        int equippedCount = countEquippedSetPieces(player, setId);
        List<StatModifier> result = new ArrayList<>();
        bonuses.forEach((threshold, mods) -> {
            if (equippedCount >= threshold) result.addAll(mods);
        });
        return result;
    }
}
```

推荐套装路线（五阶前）：土木套装 或 水木套装

### 3.6 锻造系统

#### 3.6.1 锻造台交互

```java
class ForgingStation {
    int slots = 5;  // 锻造台格子数（与炼丹炉相同）
    Location stationLocation;

    ForgingResult tryForge(Player player, ItemStack[] materials) {
        Recipe recipe = recipeRegistry.match(materials);
        if (recipe == null) {
            // 顺序错误或配方不存在 → 不消耗材料
            return ForgingResult.INVALID_RECIPE;
        }
        if (player.level < recipe.levelRequirement) {
            return ForgingResult.LEVEL_TOO_LOW;
        }
        // 消耗材料，产出装备
        consumeMaterials(player, materials);
        giveItem(player, recipe.output);
        return ForgingResult.SUCCESS;
    }
}
```

#### 3.6.2 二阶装备锻造配方

**前置条件**：等级 ≥ 15，拥有元素×32（含木×10）+ 铜钱×32

**二阶武器**：
```yaml
recipe_weapon_t2:
  slots: [精炼木元素, 精炼木元素, 职业武器雏形]
  output: "weapon_${class}_${branch}_t2"
  level_req: 15
```

**二阶防具**：
```yaml
recipe_armor_t2:
  slots: [任意精炼元素, 防具锻核, 部位材料]
  output: "armor_${slot}_t2"
  level_req: 15
```

#### 3.6.3 材料获取链

```
野外采集/击杀 → 元素(金木水火土)
                    ↓ (铁匠铺兑换，消耗铜钱)
              精炼元素 + 武器雏形/防具锻核
                    ↓ (锻造台)
              二阶装备
```

#### 3.6.4 锻造地点注册

```java
Map<String, Location> FORGE_LOCATIONS = Map.of(
    "ren",   loc(1739, 160, 103),  // 人族铁匠铺
    "xian",  loc(3231, 67, 881),   // 仙族仙器铺
    "shen",  loc(3286, 115, 363),  // 神族资源中心
    "yao",   loc(2653, 96, 858),   // 妖族铁匠铺
    "zhan",  loc(3280, 23, -256),  // 战神族战备资源部
    "city",  loc(83, 46, 143)      // 皇城铁匠铺
);
```

### 3.7 炼丹系统（详细）

#### 3.7.1 炼丹炉交互

```java
class AlchemyFurnace {
    static final int SLOTS = 5;
    // 格子布局：[药引] [金] [木] [水] [火] [土]
    // 实际只有5格，药引占第1格，元素按顺序填入后续格
    static final Element[] ELEMENT_ORDER = {METAL, WOOD, WATER, FIRE, EARTH};

    AlchemyResult brew(ItemStack[] slots) {
        // 1. 验证第一格是否为药引
        if (!isReagent(slots[0])) return AlchemyResult.INVALID;

        // 2. 验证元素顺序（跳过空格不允许）
        boolean foundEmpty = false;
        for (int i = 1; i < SLOTS; i++) {
            if (slots[i] == null) {
                foundEmpty = true;
            } else {
                if (foundEmpty) return AlchemyResult.INVALID; // 中间有空格
                if (getElement(slots[i]) != ELEMENT_ORDER[i-1]) {
                    return AlchemyResult.INVALID; // 顺序错误
                }
            }
        }

        // 3. 匹配配方
        Recipe recipe = matchRecipe(slots);
        if (recipe == null) return AlchemyResult.NO_MATCH;

        // 4. 判断双倍炼制
        int multiplier = checkDoubleBrewing(slots, recipe);

        // 5. 产出
        return new AlchemyResult(recipe.output, recipe.outputCount * multiplier);
    }
}
```

#### 3.7.2 配方数据结构

```yaml
# alchemy_recipes.yml
recipes:
  - id: "fenghou"
    name: "封喉"
    type:术丹
    double_brew: true
    reagent: ANY  # 初/中/高级药引均可
    elements:
      METAL: 5  # per_level (初等×1, 中等×2, 高等×3)
    effect:
      type: DAMAGE
      target: ENEMY_AOE
      base_value: 80
      scaling_per_tier: 40

  - id: "fengmu_huichun"
    name: "逢木回春泉"
    type: 法丹
    double_brew: true
    reagent: ANY
    elements:
      WOOD: 5
    effect:
      type: HEAL
      target: PARTY_AOE
      base_value: 60
      scaling_per_tier: 30

  - id: "hunyuan_yiqi"
    name: "混元一气丹"
    type: 药丹
    double_brew: false
    reagent: ANY
    elements:
      EARTH: 15
    effect:
      type: BUFF
      target: SELF
      buffs: [DEFENSE_UP, FIRE_RESIST]
      duration_seconds: 120

  - id: "juli_wan"
    name: "巨力丸"
    type: 药丹
    double_brew: false
    reagent: ANY
    elements:
      FIRE: 10
    effect:
      type: BUFF
      target: SELF
      buffs: [ATTACK_UP, BOW_POWER_UP]
      duration_seconds: 90

  - id: "tianshen_huti"
    name: "天神护体"
    type: 法丹
    double_brew: false
    reagent: ANY
    elements:
      WOOD: 5
      FIRE: 5
      EARTH: 5
    effect:
      type: BUFF
      target: PARTY_AOE
      buffs: [RESISTANCE, STRENGTH]
      duration_seconds: 60

  - id: "jiuzhuan_huanhun"
    name: "九转还魂香"
    type: 法丹
    double_brew: false
    reagent: ANY
    elements:
      WOOD: 5
      WATER: 5
      FIRE: 5
      EARTH: 5
    effect:
      type: BUFF_DEBUFF
      target: PARTY_AOE
      initial_debuff: [NAUSEA, WEAKNESS]
      initial_duration: 5
      delayed_buff: [MAX_HP_UP, REGEN]
      buff_duration: 180
```

#### 3.7.3 丹药等级与元素倍率

```java
enum ReagentTier {
    BASIC(1),    // 初级药引 → 元素×1
    MEDIUM(2),   // 中级药引 → 元素×2
    HIGH(3);     // 高级药引 → 元素×3

    int multiplier;
}

// 实际消耗 = recipe.elements[element] × reagent.multiplier
// 效果强度 = recipe.effect.base_value + recipe.effect.scaling_per_tier × reagent.tier
```

#### 3.7.4 双倍炼制判定

```java
int checkDoubleBrewing(ItemStack[] slots, Recipe recipe) {
    if (!recipe.doubleBrew) return 1;
    // 检查是否放入了双份元素
    for (var entry : recipe.elements.entrySet()) {
        int required = entry.getValue() * reagentTier.multiplier;
        int actual = countElement(slots, entry.getKey());
        if (actual >= required * 2) continue;
        else return 1;  // 不满足双倍条件
    }
    return 2;  // 产出×2
    // 注意：三倍以上永远失败
}
```

### 3.8 功能道具实现

| 道具ID | 名称 | 使用方式 | 逻辑 |
|--------|------|----------|------|
| dialogue_bubble | NPC对话泡泡 | 右键NPC时自动消耗 | 库存-1，触发NPC对话树 |
| respawn_stone | 重生石 | 死亡时自动消耗 | 传送至记忆重生点 |
| menu_book | 菜单书 | 右键打开 | 弹出GUI，可领取泡泡/证明 |
| race_certificate | 种族证明 | 交给引路人 | 兑换推荐书 |
| recommendation_letter | 推荐书 | 交给族长 | 推进主线 |
| equip_voucher | 新手装备兑换券 | 交给铁匠铺兑换员 | 获得武器+防具 |

### 3.9 货币系统

```java
class CurrencyManager {
    // 三种货币
    enum Currency { COPPER, SILVER, RED_COPPER_INGOT }

    // 钱庄：存取操作
    void deposit(Player player, Currency type, int amount) {
        player.wallet.remove(type, amount);
        player.bank.add(type, amount);
    }

    void withdraw(Player player, Currency type, int amount) {
        if (player.bank.get(type) < amount) throw new InsufficientFundsException();
        player.bank.remove(type, amount);
        player.wallet.add(type, amount);
    }
}
```

| 货币 | 获取途径 | 主要用途 |
|------|----------|----------|
| 铜钱 | 野外击杀 | 购买武器雏形、防具锻核 |
| 银元 | 山神庙任务奖励 | 高级物品购买 |
| 赤铜锭 | 怪物掉落（Boss） | 装备材料 |

钱庄坐标：
- 皇城：33 46 155
- 蓬莱：342 52 -677
- 各种族领地内均有

### 3.10 背包与堆叠

```java
class InventoryManager {
    static final int MAX_STACK_SIZE = 64;  // Minecraft默认
    static final Map<ItemCategory, Integer> CUSTOM_STACKS = Map.of(
        WEAPON, 1,
        ARMOR, 1,
        CONSUMABLE, 64,
        MATERIAL, 64,
        FUNCTIONAL, 16,  // 对话泡泡等
        QUEST_ITEM, 1
    );

    boolean addItem(Player player, String itemId, int amount) {
        int maxStack = CUSTOM_STACKS.getOrDefault(getCategory(itemId), MAX_STACK_SIZE);
        // 先尝试合并到已有堆叠，再占用空格
        return mergeOrAddNew(player.inventory, itemId, amount, maxStack);
    }
}
```

### 3.11 装备职业绑定检查

```java
class EquipmentValidator {
    boolean canEquip(Player player, ItemStack equipment) {
        EquipmentData data = getEquipData(equipment);
        if (data.classRestriction != null && player.playerClass != data.classRestriction) {
            player.sendMessage("该装备仅限" + data.classRestriction.displayName + "使用");
            return false;
        }
        if (data.branchRestriction != null && player.branch != data.branchRestriction) {
            player.sendMessage("该装备仅限" + data.branchRestriction.displayName + "分支使用");
            return false;
        }
        if (data.tierLevelReq > player.level) {
            player.sendMessage("等级不足，需要" + data.tierLevelReq + "级");
            return false;
        }
        return true;
    }
}
```

### 3.12 掉落表系统

```yaml
# loot_table_spider_queen.yml
loot_table:
  boss_id: "spider_queen"
  guaranteed:  # 必掉
    - item_id: "red_copper_ingot"
      amount: [1, 3]  # 随机1-3个
  random:  # 随机掉落
    - item_id: "t3_equipment_core"
      weight: 30
      amount: 1
    - item_id: "respawn_stone"
      weight: 50
      amount: [1, 2]
    - item_id: "copper_coin"
      weight: 100
      amount: [5, 15]
```


---

## 四、跨系统依赖关系

### 4.1 系统交互矩阵

```
任务系统 ←→ 物品系统：任务奖励发放、道具消耗触发任务进度
任务系统 ←→ 战斗系统：副本通关触发任务完成、战役参与记录
战斗系统 ←→ 物品系统：装备属性影响伤害、丹药提供Buff、掉落物产出
```

| 调用方 | 被调用方 | 接口 | 场景 |
|--------|----------|------|------|
| QuestSystem | InventoryService | consumeItem / giveItem | 任务道具消耗与奖励 |
| QuestSystem | DungeonManager | onDungeonComplete | 副本通关触发任务完成 |
| CombatSystem | InventoryService | dropLoot | 击杀掉落 |
| CombatSystem | ItemEffectService | applyBuff | 丹药/装备效果生效 |
| CombatSystem | WuxingCalc | getModifier | 五行伤害计算 |
| ItemSystem | CombatSystem | getPlayerStats | 装备属性汇总到战斗面板 |
| ItemSystem | QuestSystem | checkQuestItem | 验证任务道具有效性 |

### 4.2 事件总线设计

```java
// 统一事件驱动，解耦三大系统
interface GameEvent {}

class MobKilledEvent implements GameEvent {
    UUID killer; Entity mob; Location location; List<ItemStack> loot;
}
class QuestProgressEvent implements GameEvent {
    UUID player; String questId; String objectiveType; Object data;
}
class ItemUsedEvent implements GameEvent {
    UUID player; String itemId; UseContext context;
}
class DungeonCompleteEvent implements GameEvent {
    String instanceId; List<UUID> party; String dungeonId; boolean success;
}
class PlayerDeathEvent implements GameEvent {
    UUID player; Location deathLoc; UUID killer; // null if PvE
}

class EventBus {
    Map<Class<? extends GameEvent>, List<Consumer<GameEvent>>> listeners;
    void publish(GameEvent event) { ... }
    void subscribe(Class<? extends GameEvent> type, Consumer<GameEvent> handler) { ... }
}
```

### 4.3 玩家数据模型

```java
class PlayerData {
    // 基础信息
    UUID playerId;
    String race;          // ren/xian/shen/yao/zhan
    String playerClass;   // warrior/archer/alchemist
    String branch;        // jinzhong/pojun/ranger/sniper/null
    int level;
    int exp;

    // 任务数据
    Map<String, QuestState> quests;
    Set<String> completedQuests;
    Map<String, Integer> bookProgress;  // book_id → last_page

    // 战斗数据
    double maxHp;
    double currentHp;
    Element wuxingAffinity;  // 当前五行亲和
    List<ActiveBuff> buffs;

    // 物品数据
    Inventory inventory;
    Equipment equipment;
    BankStorage bank;
    Map<Currency, Integer> wallet;

    // 重生数据
    Location memorizedRespawn;
}
```

---

## 五、数据存储方案

### 5.1 存储选型

| 数据类型 | 存储方式 | 理由 |
|----------|----------|------|
| 玩家存档 | 文件(YAML/JSON) per player | Minecraft插件惯例，易于备份 |
| 任务配置 | YAML文件 | 策划可直接编辑，热加载 |
| 物品/配方定义 | YAML文件 | 同上 |
| Boss刷新状态 | 内存 + 定期持久化 | 重启恢复 |
| 副本实例 | 纯内存 | 临时数据，重启清除 |
| 战役记录 | 日志文件 | 事后分析 |

### 5.2 配置热加载

```java
class ConfigManager {
    // 支持 /reload 命令重新加载配置
    void reload() {
        questRegistry.reload();    // 任务配置
        itemRegistry.reload();     // 物品定义
        recipeRegistry.reload();   // 配方
        lootTableRegistry.reload();// 掉落表
        bossSpawnerRegistry.reload();
    }
}
```

---

## 六、实现优先级

### P0 — 核心循环（第一期）

| 模块 | 内容 | 预估工时 |
|------|------|----------|
| 物品-背包 | 背包管理、堆叠、职业绑定 | 3天 |
| 物品-装备 | 一阶装备定义、装备栏、属性计算 | 3天 |
| 战斗-基础 | 伤害公式、五行加成、种族祝福 | 3天 |
| 战斗-仇恨 | AggroTable、怪物AI目标切换 | 2天 |
| 任务-状态机 | 接取/完成/奖励流程 | 3天 |
| 任务-NPC交互 | 对话泡泡消耗、菜单书补充 | 2天 |
| 任务-新手引导 | 主线Phase1（出生地→皇城） | 3天 |

### P1 — 职业差异化（第二期）

| 模块 | 内容 | 预估工时 |
|------|------|----------|
| 战斗-金钟横扫 | 范围判定、额外仇恨 | 2天 |
| 战斗-弓箭手弹道 | 射击飞行、三箭扇形 | 3天 |
| 战斗-炼丹师即时炼化 | 5种元素技能实现 | 3天 |
| 物品-炼丹系统 | 炼丹炉GUI、配方验证、产出 | 4天 |
| 物品-锻造系统 | 锻造台GUI、二阶配方 | 3天 |
| 物品-货币 | 铜钱/银元/赤铜锭、钱庄 | 2天 |

### P2 — 副本与战役（第三期）

| 模块 | 内容 | 预估工时 |
|------|------|----------|
| 战斗-副本 | 实例化、四圣兽祭坛传送 | 5天 |
| 战斗-Boss | 刷新机制、掉落表、AI | 4天 |
| 任务-青龙试炼 | 问答系统、全队重置 | 3天 |
| 战斗-阵营战 | 报名、匹配、计分、奖励 | 5天 |
| 物品-套装 | 套装属性激活、推荐路线 | 2天 |
| 任务-剧情书 | 书本阅读追踪、种族分支内容 | 2天 |

---

## 七、技术风险与注意事项

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| 阵营战大量玩家同屏 | 服务器TPS下降 | AOI裁剪、实体数量上限、分阶段进入 |
| 炼丹配方顺序校验复杂 | 玩家困惑 | 配方错误不消耗材料（已设计）、UI提示 |
| 副本实例化内存占用 | OOM | 实例超时自动回收、最大并发数限制 |
| 五行伤害公式平衡 | 某属性过强 | 参数外置YAML、可热调整 |
| 对话泡泡消耗体验 | 新手卡住 | 菜单书无限补充、首次交互免费 |
| 多种族新手流程维护 | 配置膨胀 | 模板化设计，种族差异仅为参数替换 |

---

## 附录A：关键坐标速查

| 用途 | 坐标 |
|------|------|
| 皇城重生点 | 179 43 63 |
| 皇城铁匠铺 | 83 46 143 |
| 皇城丹药铺 | 260 50 30 |
| 皇城钱庄 | 33 46 155 |
| 逆天计划-进攻方 | 3267 20 -138 |
| 逆天计划-防守方 | 124 59 -144 |
| 蚩尤试炼入口 | 3140 45 -197 |
| 蚩尤试炼实例 | 2871 25 -184 |
| 青龙祭坛 | 550 34 394 |
| 白虎祭坛 | -693 153 -267 |
| 朱雀祭坛 | 240 63 732 |
| 玄武祭坛 | -84 34 -521 |
| 龙须镇（新手发育） | 552 34 42 |
| 蜘蛛女王(石制祭坛) | 666 47 40 |
| 山神庙 | 833 40 99 |

## 附录B：炼丹配方速查

| 丹药 | 类型 | 元素(每级) | 可双倍 | 核心效果 |
|------|------|-----------|--------|----------|
| 封喉 | 术丹 | 金×5 | ✓ | 高额伤害 |
| 逢木回春泉 | 法丹 | 木×5 | ✓ | 群体治疗 |
| 混元一气丹 | 药丹 | 土×15 | ✗ | 防御+抗火 |
| 洛神丹 | 药丹 | 水×10 | ✗ | 水下活动 |
| 万灵丹-速 | 药丹 | 金×5+水×5 | ✗ | 移速+视觉 |
| 万灵丹-跃 | 药丹 | 金×5+木×5 | ✗ | 跳跃+免摔 |
| 辟谷丹 | 药丹 | 木×5+土×5 | ✗ | 减饥饿+缓愈 |
| 巨力丸 | 药丹 | 火×10 | ✗ | 攻击力提升 |
| 玄甲丹 | 药丹 | 金×5+土×5 | ✗ | 高防御-移速 |
| 天神护体 | 法丹 | 木×5+火×5+土×5 | ✗ | 硬化+力量 |
| 九转还魂香 | 法丹 | 木×5+水×5+火×5+土×5 | ✗ | 先debuff后增益 |
