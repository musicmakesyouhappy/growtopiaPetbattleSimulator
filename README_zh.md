# 宠物对战模拟器 - 说明文档

这是宠物对战模拟器的完整文档，合并成了一个文件：

1. **[核心机制](#part-1-core-mechanics)** ， 逐个系统讲解战斗引擎是怎么运作的，附带一些不容易看出来的小知识。
2. **[宠物技能说明](#part-2-pet-ability-documentation)** ， 每只宠物的技能具体在游戏逻辑里是怎么实现的，按属性分类。
3. **[开发者/函数参考](#part-3-developer--function-reference)** ， 各种东西在代码里放在哪儿，以及怎么添加新宠物或新技能。

想搞懂游戏怎么玩，从第一部分看起。想查某只具体的宠物，直接看第二部分。要改代码，直接看第三部分。

---

# 第一部分：核心机制
---

### 1. 队伍、分组与生命值

- 每一方要带**2组**宠物，每组**3只**，一方总共6只。你要在组队界面把两组都配齐，"开始对战"按钮才会出现（两边都把6个位置填满才会出现）。
- 同一时间只有一组是**出战**状态；另一组待在替补席，直到你（或者AI）切换过去。
- 每一组都有自己独立的血量池，和搭档那一组完全分开计算。
- **基础最大生命值是150**每组。如果一组里带有拥有`passive_hp_boost`效果的宠物（Mini Mammoth Leash），那一组的最大生命值就会变成**195**（+30%），只有实际带着那只宠物的那一组能吃到这个加成，搭档那一组不会跟着涨。
- 一组的血量归零就算"阵亡"。只有当一方**两组同时**阵亡，这一方才算输。
- 理论上是可以"骗过死亡"的：如果一组在被打死的那一刻正好也在同一个游戏刻收到了治疗，回血是能赶在扣血前生效的，让它以回复后的血量活下来。
- 宠物之间偶尔会在两场对战之间自动回满血，根本不需要绷带（这是个已知的小毛病，不是特意设计的机制）。

---

### 2. 游戏刻时钟

- 整场战斗跑在**1秒一刻**的节奏上（`gameTick`函数，通过`setTimeout(gameTick, 1000)`调用），每一刻都会让全局的`battle.tick`计数器加一。
- 每一刻按这个顺序处理：天气跳一次、切换冷却和技能冷却往下走、持续伤害/持续回血跳一次、检查厄运倒计时、增益/减益倒计时并到期清除、延迟伤害和回血队列往前推进、陷阱计时器往前走、被动效果（比如替补回血）重新检查一遍、**AI每隔一刻行动一次**（`battle.tick % 2 === 0`），最后检查胜负条件。
- 这也就意味着敌方AI实际的行动节奏只有原始冷却刻度的一半，技能冷却和持续伤害每一刻都在更新，但AI只有在偶数刻才有机会出手。

---

### 3. 技能冷却与释放

**初始冷却。** 战斗开始时，所有非被动的宠物冷却都会被设成`floor(基础冷却 / 2)`，上限12秒，下限1秒。如果这一组里带有Windspeed类的宠物（`passive_cd_reduce`），初始冷却还会再减2秒（同样下限是1秒）。所以一只冷却20秒的宠物开局是10秒（有Windspeed的话是8秒），而一只冷却3秒的宠物不管怎样开局都是1秒。

**释放技能**要求这只宠物：
- 是当前*正被选中*的出战宠物（光待在出战组里还不够，得是选中的那只），
- 冷却≤0，
- 不是被动宠物（被动技能永远不能手动释放），
- 没有被眩晕/冰冻/睡眠/跳舞任何一种状态禁用，并且
- 所在的这一组没有被标记`canAct: false`（同样是跳舞状态会导致的）。

**口误判定。** 如果这一组身上带着`mess_up`家族的负面效果，释放技能*在做任何其他事情之前*会先掷一次这个概率判定。判定失败的话：技能照样正常进入冷却，日志里会记录一条"技能失败！"，本该发生的效果全都不会出来，**但有两个例外**：任何天气类技能（火焰风暴、毒云、紫雾、数羊、舒缓迷雾）就算施法被搞砸了也照样会触发，Swoop的自我闪避也是一样，照样会触发。

**成功释放技能之后：**
1. 这只宠物的冷却重置为`max(1, 基础冷却 - windspeed减免)`。
2. 如果这一组身上挂着已经蓄势待发的`extend_cd`负面效果，会把它的加成秒数叠加到刚刚重置好的冷却上，然后这个负面效果自己消失。
3. 同一组里其他所有已经冷却完毕（并且不是被动）的宠物，都会被强制打上2秒冷却，你没法在同一组里毫无限制地连续放技能；用了一只，剩下的会被锁个几秒。**这个锁定不管有没有Windspeed都照样会触发**，Windspeed在这里并不能阻止它（Windspeed那个「不锁定」的特殊效果只在换组*切入*时生效，用技能时不生效，详见第4节）。
4. 技能的伤害/治疗/效果最后才会真正结算。
- Windspeed自己的技能文案写的是"最低1秒"，但在一种特定情况下这个下限会被完全绕过：当带有Windspeed的那一组被切换*上场*时，它身上所有已经冷却完毕的宠物根本不会受到通常那个+2秒锁定的影响（见第5节），不是锁定被压到1秒，而是这个锁定完全被跳过了。
- 宠物面板上写的冷却时间（比如"10秒"），其实只有在它*第一次*施法之后才真正生效，战斗开局的冷却永远是上面那个减半（并封顶）之后的数值。

---

### 4. 切换分组

- 每一方都有一个切换按钮/动作，能把`activeDeck`在第1组和第2组之间来回切（同时会把切进来那一组的`activePet`重置回第0个位置）。
- 出现以下情况会没法切换：当前这组的切换冷却还没转完、当前这组被冰冻了、或者这一组被标记了`canSwap: false`（来自`anti_swap`/放逐类的负面效果）。
- 切换成功后：这一组自己身上的**厄运**诅咒（如果有的话）会立刻被解除，这是官方认可的甩掉厄运的办法。刚换上场的这一组会进入**3秒的切换冷却**，之后才能再次切换。
- 就像放技能会把这一组里其他已就绪的宠物锁上2秒冷却一样，切*进*一组同样会把它身上所有已经就绪、非被动的宠物打上2秒冷却，**唯一的例外**是如果这一组里带有Windspeed宠物，这个锁定会被完全跳过，那些已就绪的宠物照样保持就绪状态。
- 如果对面当前对你布下了**切换陷阱**（见第9节），你主动切换离场就会立刻触发陷阱，在切换生效之前先吃一下额外伤害。
- 强制切换类效果（践踏、放逐、强推搭档、灵魂互换）用的是和主动切换一样的"清掉离场那组身上的厄运"规则，只有当被诅咒的那一组*没法*切换（比如被冰冻或被锁住）撑到倒计时结束时，厄运才会真正生效。
- 冰冻是唯一一个会彻底锁死切换的常见控制效果；眩晕、睡眠、跳舞都只会禁用*行动*，不会阻止你撤退到备战组。

---

### 5. 属性与克制关系

- 一共四种属性：**火 → 地 → 风 → 水 → 火**（循环相克，每个都克制下一个）。
- 占属性优势的一方打出的伤害**+25%**；处于劣势的一方伤害**-25%**。属性中立（不在克制循环里）的对局没有加成。
- 这里用来判断克制关系的属性，并不是单纯"当前出战宠物自己的属性"，而是**这一组的主属性**：代码会数一下整组（全部3个位置，不只是出战那只）里各种属性各有几只，取数量最多的那种属性。如果打平，就退回去用当前出战宠物自己的属性来判断。
- 这个属性加成会自动套用在`dealDamage`里（几乎）每一次攻击上，这和某些恐龙系技能自带的"elemental_bonus"额外加成是分开算的，那种加成是在自己属性克制对方时叠加在这个基础加成之上的。
- 有些效果是明确被排除在属性加成之外的，最典型的就是Toasties（`summon`类召唤伤害），不管双方是什么属性，它永远按面板上写的固定伤害打；另外任何绕过`dealDamage`直接扣血的伤害（比如延迟的孢子爆炸、天气跳的伤害、反噬伤害等）也都不受属性加成影响。

---

### 6. 伤害结算顺序（`dealDamage`函数）

结算一次攻击的时候，模拟器会严格按下面这个顺序依次套用各种加成：

1. **紫雾天气判定**，如果天气生效中，火属性攻击者+25%（或者天气自己存的那个加成数值），其他所有属性一律-25%，这一步在其他所有步骤之前就先算。
2. **属性克制加成**（±25%），根据双方各自的主属性判定，少数几个特殊技能（比如Toasties）会完全跳过这一步，被明确排除在外。
3. **闪避判定**，除非这次攻击本身无视闪避（`anti_dodge`、`anti_dodge_block`、`mess_up_undodgeable`，或者是打在非"翱翔"闪避上的精准/暴击技能），否则一个临时的闪避状态或者被动闪避概率判定都可能直接把这一下打成0。
4. **格挡判定**，道理和闪避一样，除非这次攻击无视格挡。
5. **灵体**，如果防守方带有生效中的灵体状态，会对剩下的伤害做一次固定比例的减免。
6. **双倍伤害概率**和**下一击加成**，针对*攻击方*自己身上的增益做判定/结算（某些"无视自己增益"的技能比如辐射光束会跳过这一步）。
7. **攻击方的持续性"造成伤害"加成**（把攻击方身上所有影响造成伤害的增益/减益加起来，比如怒吼、霉运、甩巴掌）。
8. **防守方的"承受伤害"加成**（比如霉运、`dmg_taken`类负面效果）。
9. **固定25%减伤**，如果防守方当前出战的宠物带有`passive_dmg_reduce`效果（只在那只宠物正被选中出战时生效）。
10. **播放命中音效。**
11. **伤害下限**，经过上面所有加成之后，伤害会被封底到**最低1点**（光靠叠加减伤类加成，永远没法把一次命中打到0，只有闪避/格挡/护盾才能把它真正打成0）。
12. **护盾判定**，如果防守方当前带有护盾，不管前面算出来是多少，这一整下攻击都会被完全抵消，转而给防守方回复同等数额的血量（如果身上有创伤则回血会被挡住），这个函数会直接返回0，完全不会扣血。
13. **正式扣血。**
14. **静电充能叠层**，防守方那一组里任何带有`stack_burst`效果的宠物，只要这一组挨了打就会获得1点电荷，不管实际挨打的是不是它自己。
15. **反击/荆棘反伤**，如果防守方带着生效中的反击增益，攻击方那一组会在这一下命中之后紧接着承受存好的那个固定反弹伤害。
- 因为伤害下限（第11步）发生在护盾判定（第12步）*之前*，所以护盾抵消的时候，就算这一下被其他加成削得几乎为0，护盾抵消的也是"经过下限封底之后"的那个数值，回血也是照这个数值来的，至少会回1点血。
- 少数几个"无视一切"的特殊攻击（Toasties，也就是`summon`类）完全不走`dealDamage`这套流程，它们是直接把固定伤害从目标血量里扣掉的。这意味着上面流程里的每一步检查（属性、闪避、格挡、灵体、增减益，以及**紫雾**）都会被跳过，不是跳过一部分，尽管wiki上说紫雾是唯一的例外（这个不一致的地方详见第9节）。

---

### 7. 状态效果（限时，按组记录）

| 状态 | 禁止行动？ | 锁死切换？ | 实际效果 |
|---|---|---|---|
| **眩晕（Stun）** | 是 | 否 | 没法使用技能；依然可以切走。 |
| **冰冻（Freeze）** | 是 | **是** | 既没法行动，也没法切换，唯一会锁死撤退的控制效果。 |
| **睡眠（Sleep，天气类）** | 是 | 否 | 和眩晕一样，但是由「数羊」这个天气效果引起的；只要还没睡着，天气每跳一次，倒计时就会刷新成"剩余天气时间+1秒"。 |
| **跳舞（Dance）** | 是（`canAct: false`） | 否 | 被强制跳舞；有些让人跳舞的技能还会顺带给受害者一段同样时长的闪避状态。 |
| **闪避（Dodge）** | - | - | 临时的必定闪避概率，会叠加在被动闪避之上。 |
| **格挡（Block）** | - | - | 临时的、对非穿透类攻击的必定格挡。 |
| **创伤（Trauma）** | - | - | 完全不会禁止行动，它是一个纯粹的**禁止治疗标记**。游戏里所有的治疗来源（基础回血、持续回血、天气治疗、生命汲取、护盾抵消伤害时的回血、复活）都会检查目标身上有没有创伤，有的话就跳过这次治疗；至于这一下攻击的其他部分（造成的伤害、护盾抵消、复活本身）依然照常进行。 |
| **持续伤害（DoT）** | - | - | 持续时间内每秒固定扣一次血，直接从生命值里扣（不走`dealDamage`流程）。 |
| **持续回血（Hot）** | - | - | 计时方式和持续伤害一样，只是回血而不是扣血（会被创伤挡住）。 |
| **灵体（Ethereal）** | - | - | 生效期间对所受到的一切伤害做固定比例减免（不是闪避概率）。 |
| **护盾（Absorb）** | - | - | 抵消受到的伤害，并把这部分转化成给防守方的回血（除非身上有创伤，这种情况下只会抵消伤害，不会回血）。 |
| **厄运（Doom）** | - | - | 加在*目标*那一组身上的一个倒计时；如果倒计时归零时它还挂在身上，那一组的血量会瞬间清零。只要那一组被主动或被强制切换离场，就会立刻解除。 |
- **wiki与代码不一致之处：** wiki上说天气也算一种"负面效果"，理论上应该能触发「后背鸭子」（`passive_resist_negative`）的抵抗，或者被「吐饼干」（`cleanse`）净化掉。但代码里并没有这么实现，天气完全独立存放在全局的`battle.weather`对象里，不属于任何一组的`statusEffects`或`modifiers.debuffs`数组，`tryResist`和净化的处理逻辑也从来不会去检查`battle.weather`。在这个版本里，天气既不能被抵抗，也不能被净化。
- "最后一秒不再触发一次"这个规则**只针对天气**，不是所有限时效果的通用规则。普通的持续伤害（和持续回血）在剩余时间刚好倒数到0的那一跳，其实依然会正常结算，然后才被移除，所以一个每秒X点、持续Y秒的持续伤害，总伤害就是老老实实的`X × Y`。只有天气系统才有那套额外的计数逻辑（`tickCounter`和`remaining`分开算），会在最后一跳跳过一次结算。
- 影响回复速度的加成/减益（减速/加速类）除了影响冷却回复快慢之外，同样也会拉长或缩短受影响那一组身上眩晕/冰冻/睡眠状态的持续时间。

---

### 8. 增益与减益（作用于整组，和状态效果是两码事）

这些都存放在一组的`modifiers.buffs` / `modifiers.debuffs`数组里，每一刻都会重新汇总成一个固定总值（`recomputeModifiers`）：

**增益**
- `buff_damage` - 固定百分比提升造成的伤害。
- `dmg_speed` - 同时提升造成的伤害百分比，并让冷却回复速度翻倍。
- `double_dmg` - 每一次命中都有一定概率触发伤害翻倍。
- `next_dmg` - 只对未来那一次命中生效的百分比加成，生效后自动消失。
- `fast_cd` - 冷却回复速度翻倍（技能冷却走得快一倍）。
- `counter_reflect` - 把固定数额的伤害反弹给接下来命中自己的攻击方。

**减益**
- `dmg_taken` - 固定百分比提升承受的伤害。
- `debuff_damage` - 固定百分比削减造成的伤害。
- `bad_luck` - 上面两个效果同时生效，共用一个数值。
- `slow_cd` - 冷却回复速度减半。
- `mess_up` - 让这一组下一次施法有一定概率直接失败。
- `no_swap` - 完全禁止切换。
- `extend_cd` - 一次性效果，给这一组*下一次*完成的施法额外加几秒冷却，用完就消失。

**叠加规则。** 大多数情况下，同一只宠物重复施加同一种增益/减益，只会把持续时间**刷新**成新的数值，而不会再叠加一份。只有少数几只特别标记过的宠物会真正**叠加**：由*同一个*来源重新施加时，数值会累加（一般上限是伤害类加成400%、承受伤害类200%），持续时间也会取两者中较长的那个，而不是直接重置。不同宠物/不同来源的效果，不管叠不叠加规则如何，永远是各自独立共存的。
- 因为`buff_damage`/`dmg_speed`/`debuff_damage`/`bad_luck`/`dmg_taken`最终都汇总进同一套总值里，所以一组宠物完全可能同时被增益又被减益（分别来自不同来源），这些数值是代数相加的，不是谁覆盖谁。

---

### 9. 天气（全局生效，对双方影响相同）

和按组记录的状态效果、增益不同，**天气是一个全局唯一的槽位**，整个战场同一时间只能有一种天气效果生效，放一个新的天气会直接覆盖掉之前那个。每种天气都按自己独立的内部节奏跳，和主时钟的1秒一刻不是一回事：

| 天气 | 触发来源 | 跳动频率 | 效果 |
|---|---|---|---|
| 火焰风暴 | `isDot` + 8点伤害 | 每2秒 | 双方**出战**那一组都会承受固定的、无法减免的伤害。 |
| 毒云 | `isDot` + 4点伤害 | 每5秒 | 和火焰风暴类似，只是每跳数值更小。 |
| 紫雾 | 固定`effectValue`，非持续伤害类型 | 每1秒 | 全局改变属性伤害：火属性攻击者+25%，其他所有属性一律-25%，持续整个效果窗口，这一步在`dealDamage`里是最先判断的，排在正常的属性循环加成之前。 |
| 舒缓迷雾 | `weather_hot` | 每5秒 | 双方出战那一组都会回血（各自被己方的创伤挡住）。 |
| 数羊 | `sleep` | 每1秒 | 双方出战那一组都会进入睡眠状态，只要还没睡着，每跳都会刷新成"剩余天气时间+1秒"。 |
- 天气类技能就算施法者自己这次施法被`mess_up`类负面效果搞砸了，也照样会触发，这是口误判定里两个硬编码的例外之一（另一个是Swoop自带的必定闪避）。
- 造成伤害/回血类的天气效果，会在天气结束前的最后一跳跳过一次结算，所以实际的总量总是会比"单跳数值×跳数"这样直接算出来的理论值要少一点。
- **wiki与代码不一致之处：** wiki上说紫雾是"Toasties无视一切伤害加成"这条规则的唯一例外（也就是说Toasties的伤害本应该还是会被紫雾的火/非火加成调整）。但目前的代码并没有这么实现，Toasties的`summon`处理逻辑完全绕过了`dealDamage`（见第6节），而紫雾的加成只会在`dealDamage`内部生效，所以在这个版本里，不管wiki怎么说，紫雾对Toasties的伤害其实**完全没有影响**。

---

### 10. 几个特殊的一次性机制

- **延迟伤害**，有些技能不是立刻命中的，而是先排一个未来爆发的队列，等过了N个游戏刻之后才真正结算，而且结算时完全不带施法宠物本身的信息（所以不会受到那只宠物自己的穿透特性影响，只看命中那一刻目标本身处于什么状态）。
- **延迟回血**，和延迟伤害相反：一次命中造成的伤害会被安排在后续一段时间内大致平均地还给目标当作回血（和其他治疗一样会被创伤挡住）。
- **切换陷阱**，在敌方身上布下一个已激活的隐形倒计时；如果对方在陷阱生效期间主动切换离场，就会触发额外伤害，然后陷阱自我消耗。陷阱本身的*布置*动画是能看到的，但一旦布下之后，界面上不会有任何提示它还存在。
- **击杀连锁**，如果一次命中直接打死了目标当前出战那一组，紧接着会立刻对它的备战组（如果还活着）补上一次一模一样的攻击；如果第一下没能打死出战那一组，就不会有连锁。
- **强制切换系**（践踏 / 放逐 / 强推搭档 / 灵魂互换），都是在敌方备战组还活着、能切换、没被冰冻的前提下，把敌方强行推到那一组去；它们的区别只在于各自叠加的额外效果（放逐额外会把换上来的那一组锁一段时间没法再换回去；灵魂互换额外会把出战那一组身上所有的状态/增益/减益一并拖到换上来的那一组身上）。
- **护盾类叠加**，有几个具体的护盾类技能被算作"同一个技能类别"：在一个还没到期之前再放另一个，会**延长**剩余时间，而不是重置。

---

### 11. 阵亡与复活

- **Living Dead Remote**（`passive_revive`）只有当它待在**替补**（不出战）那一组的时候才会起作用，如果它自己所在的那一组正是出战组，这个效果完全不生效。一旦*另一组*（也就是它所在这一组的搭档）血量归零，只要装着Remote的这组自己还活着，就会立刻把刚阵亡的那一组原地复活到最大生命值的一定百分比，不发生切换，复活的那一组直接继续当出战组用。这个效果**每一方每场战斗只能触发一次**：一旦触发过，Remote在这场剩下的时间里就报废了（被动图标上会显示"已使用"状态），不管你上场了几只Remote都一样。
- 如果一组阵亡的时候**没有**任何还活着、待在替补席的Remote能救它（没带这只宠物、这场已经用过了、或者Remote自己那一组也刚好挂了），游戏就会回退到普通规则：下一个游戏刻，出战组会自动切换到还活着的搭档那一组（如果那一组还活着的话）。
- 另外还有一种主动**复活**技能，能把已经阵亡的*备战组*（不是当前出战的那组）复活到最大生命值的一定百分比，只要备战组当前正好是0血就能用，如果备战组还活着，放这个技能就完全没有效果。这个效果和Living Dead Remote是相互独立的，也不受"每场一次"这个限制。
- 一旦某一方两组同时阵亡，战斗立刻结束（这个判定紧接在上面的阵亡处理逻辑之后进行，确保救援机会已经用过），界面上会弹出胜负横幅，游戏刻的循环也会停止运行。
- 因为Living Dead Remote是靠"搭档那一组血量刚好归零"这个瞬间触发的，所以如果*两组*在同一个游戏刻被同时清零（比如某个效果一次性同时命中两组），Remote是救不回来的，触发判定的那一刻，Remote自己所在的那一组必须还站着，所以同刻双杀会被正常判定为战败。
- 创伤**不会**阻止复活/复生类效果生效，创伤只挡被动性质的*回血*，不会挡完整的复活。

---

### 12. 敌方AI行为

每隔一个游戏刻，AI会依次判断：
1. 如果它当前出战的宠物冷却已经转完、可以使用，并且这一组没有被禁用行动，就直接放技能。
2. 否则，如果同一组里还有*别的*宠物冷却已转完、可以使用，就把选中切换成那只宠物（这一刻就只做切换这一件事）。
3. 否则，如果它的切换冷却已经转完、备战组还活着、切换没有被锁住，并且备战组里确实*有*可以用的宠物，就切换分组。
4. 如果上面条件都不满足，这一刻就什么都不做，等着冷却继续往下转。

AI从来不会主动排队"等一下"，它每一次行动基本都是"用当前能用的最好选项"，每隔一刻都会重新评估一次。

---

### 13. 胜负判定

只要某一方*两组*同时血量为0，就立刻判负，这个判定发生在每一刻的最后，在Living Dead已经有过一次介入机会之后。如果双方碰巧在同一刻都归零了，判定顺序依然是先判"你这一方"，所以真正意义上的同刻双杀，会因为判定顺序而被判成*敌方*获胜（先检查你这边是否阵亡，如果是，除非你的复活刚好在同一轮触发，否则战斗就已经按敌方获胜结束了，来不及等你这边的Living Dead状态翻盘）。


---

# 第二部分：宠物技能说明

### 目录
- [风系 Air](#air)（49只）
- [地系 Earth](#earth)（48只）
- [火系 Fire](#fire)（47只）
- [水系 Water](#water)（49只）

---

### 风系 (Air)

1. Eye Of Growganoth - 风 | Gift of Growganoth | 被动
`effect: passive_heal_inactive, effectValue: 7, effectDuration: 5`
**回血：** 7
> *wiki原文 - Gift of Growganoth:* Passive - Heal 7 health every 5s when not active.

一直生效。每过5个游戏刻（判断条件是`battle.tick % 5 === 0`），只要这只宠物所在的那组当前是替补（不是出战组里正在上场的宠物）、这组还活着、身上也没有创伤，就回7点血。全程安静自动完成，不需要施法，没有冷却，除了回血本身也不会额外弹出日志。

2. Ladybug Leaf - 风 | Exoskeleton | 被动
`effect: passive_shorten_debuff, effectValue: 2`
> *wiki原文 - Exoskeleton:* Negative effects on you have 2s shorter duration (minimum 1s).

一直生效（内部叫`exoReduction`）。这组宠物从对面那里受到的任何带持续时间的负面效果，不管是持续伤害、眩晕/冰冻/创伤，还是通过`applyModifiers`/`applyStatusEffect`加上来的普通负面效果，在生效那一刻持续时间都会被砍掉2秒（最低砍到0）。自己给自己上的负面效果不会被砍，因为这个减免只在`!isSelf`（不是自己招惹自己）的情况下才生效。

3. Summer Kite - 风 | Windspeed | 被动
`effect: passive_cd_reduce, effectValue: 2`
> *wiki原文 - Windspeed:* Passive - all cooldowns are 2s shorter (minimum 1s).

一直生效。一个被动同时干三件事：（1）战斗开始时，这一组里每只宠物的初始冷却都会减少2秒；（2）这一组里任意一只宠物放完技能后，它刚刷新出来的冷却也会再减2秒；（3）如果你切换进了带有这只宠物的这一组，通常会有的「+2秒锁定」惩罚会被直接跳过，完全不生效。整个效果是让全组受益，而不只是它自己。

4. Yeonnalligi - 风 | Soaring | 被动
`effect: passive_dodge, effectValue: 25`
> *wiki原文 - Soaring:* Passive - 25% chance to dodge attacks.

一直生效，没有冷却。只要这只宠物是当前正被选中出战的那一只（不只是待在这一组里，而是正处在出战位置），每一次受到的攻击都有25%概率被完全躲开（伤害归零）。`dealDamage`里对它有个特殊照顾：一般来说「精准」系恐龙技能（`anti_dodge_block_conditional`）能穿透闪避，但如果这次闪避恰好是这只宠物的被动触发的，「精准」就不能穿透，被动优先生效，这一下照常被闪避掉。这和wiki里提到的「翱翔」克制「精准/暴击」的小知识对得上。

5. Betty Bluetooth Doll - 风 | Static Charge | 主动（冷却3秒）
`effect: stack_burst, effectValue: 10`
**伤害：** 5 （风） | **冷却时间：** 3秒
> *wiki原文 - Static Charge:* Each time you take damage, gain 1 static charge (up to 10). This skill launches them all at once, for 5 damage each.

被动加主动的混合技能：被动部分是，只要这只宠物所在的这组受到伤害，不管来源是什么，也不管这只宠物本身是不是在出战，就会累积1点电荷，上限10点。它自己还有一个主动技能，冷却3秒，放出去会把攒下的所有电荷一次性打出去，总伤害是`5 × 电荷数`，打完电荷清零。如果攒了0点电荷就放技能，那就是白放一次，技能照样进入冷却，但什么都打不出去。

6. Playful Wind Sprite - 风 | Playful Wind | 主动（冷却4秒）
`effect: buff_chance_double_damage, effectValue: 15, effectDuration: 8`
**伤害：** 6 （风） | **冷却时间：** 4秒
> *wiki原文 - Playful Wind:* Grants you a 15% chance to do double damage for 8 seconds. Inflicts 6 Air damage.

主动技能，冷却时间4秒。造成6点伤害，然后给施法者自己这组宠物加上一个`double_dmg`增益，持续8秒，期间每次攻击有15%概率触发。增益生效时，这组宠物之后每一次攻击在`dealDamage`结算时都会掷一次`doubleDmgChance`判定，一旦命中，那一下的伤害就翻倍。在增益消失前再次释放，只是把倒计时刷新回8秒（不同来源的增益各自独立叠加，不会互相覆盖）。

7. Chick Leash - 风 | Chirp | 主动（冷却5秒）
`effect: anti_block`
**伤害：** 9 （风） | **冷却时间：** 5秒
> *wiki原文 - Chirp:* Cheep adorably. Inflicts 9 Air damage. Can't be blocked.

主动技能，冷却时间5秒。造成9点伤害。在`dealDamage`里设置`ignoresBlock`，目标身上的「格挡」状态对这一下完全不起作用，但闪避不受影响，该触发还是照常触发。

8. Flying Bell - 风 | Sound Blast! | 主动（冷却5秒）
`effect: anti_dodge`
**伤害：** 9 （风） | **冷却时间：** 5秒
> *wiki原文 - Sound Blast!:* Blast sound waves to deal 9 Air damage! Inflicts 9 Air damage. Can't be dodged.

主动技能，冷却时间5秒。造成9点伤害。在`dealDamage`里设置`ignoresDodge`，不管是临时的「闪避」状态还是被动闪避概率（比如「翱翔」），都没法帮目标躲开这一下，但格挡不受影响，还是能挡住。

9. Grey Pet Pteranodon - 风 | Precision Strike A | 主动（冷却5秒）
`effect: anti_dodge_block_conditional, effectValue: 25`
**伤害：** 7 （风） | **冷却时间：** 5秒
> *wiki原文 - Precision Strike A:* If the opponent attempts to dodge or block, hits for 25% more damage. Inflicts 7 Air damage. Can't be dodged or blocked.

主动技能，冷却时间5秒，属于「精准/暴击」这一系恐龙技能。造成7点伤害。在`dealDamage`里，这个技能会无条件把`ignoresDodge`和`ignoresBlock`都设成true，不管对方是闪避还是格挡状态，这一下都会实打实命中满伤害。需要注意：wiki上写的是「如果对方尝试闪避或格挡，则造成25%额外伤害」，但实际代码从来没读取过这个技能的`effectValue`字段，它只是单纯无视闪避和格挡，照面板上的固定伤害打，并没有额外加成。唯一的例外：如果对方的闪避恰好是来自某个带`passive_dodge`被动的宠物（比如Yeonnalligi的「翱翔」被动），穿透效果会被关掉，这一下还是会被正常闪避掉，这跟wiki里「翱翔」克制「精准/暴击」的小知识对得上。

10. Leashed Silkworm - Purple - 风 | Face Slap | 主动（冷却5秒）
`effect: debuff_damage_dealt, effectValue: 25, effectDuration: 10`
**伤害：** 12 （风） | **冷却时间：** 5秒
> *wiki原文 - Face Slap:* Smack the target in the face, enraging them to inflict 25% more damage for 10s. The rage stacks up to 20 times. Inflicts 12 Air damage.

主动技能，冷却时间5秒（「甩巴掌」系技能）。造成12点伤害，然后给目标（不是施法者自己）附加持续10秒的`debuff_damage`负面效果：目标在这段时间里造成的伤害-25%。根据wiki的小知识，这个效果的文案写的是「激怒目标」，但实际上是加在敌人身上的负面效果，不是施法者自己的增益。如果这只宠物属于会叠加的那几只（Leashed Silkworm - Purple、Pet Present Goblin），连续释放会让减伤幅度叠加（上限400%），而不是刷新重置。

11. Leashed Silkworm - White - 风 | Psyblade | 主动（冷却5秒）
`effect: self_damage, effectValue: 4`
**伤害：** 12 （风） | **冷却时间：** 5秒
> *wiki原文 - Psyblade:* Shoot a mind blade, causing psychic backlash to you for 4 damage. Inflicts 12 Air damage.

主动技能，冷却时间5秒（「灵刃冲击」）。通过普通伤害流程对敌方造成12点伤害，与此同时，施法者自己这组还会额外承受4点固定的、无法减免的反噬伤害（直接从血量里扣，完全绕过`dealDamage`，所以施法者自己的闪避/格挡/灵体等效果对这部分反噬毫无作用）。

12. Sonar Bracelet - 风 | Sonic Burst | 主动（冷却5秒）
`effect: anti_dodge`
**伤害：** 9 （风） | **冷却时间：** 5秒
> *wiki原文 - Sonic Burst:* Blast sonic waves. Inflicts 9 Air damage. Can't be dodged.

主动技能，冷却时间5秒。造成9点伤害。在`dealDamage`里设置`ignoresDodge`，不管是临时的「闪避」状态还是被动闪避概率（比如「翱翔」），都没法帮目标躲开这一下，但格挡不受影响，还是能挡住。

13. Grey Pet Apatodon - 风 | Dino Dive A | 主动（冷却6秒）
`effect: elemental_bonus, effectValue: 50`
**伤害：** 10 （风） | **冷却时间：** 6秒
> *wiki原文 - Dino Dive A:* Against a Water opponent, inflicts +50% damage (on top of normal bonus.) Inflicts 10 Air damage.

主动技能，冷却时间6秒，属于「恐龙冲撞/突袭/俯冲/撕咬」这一系技能。造成10点风系基础伤害。结算这次攻击之前，代码会先判断这只宠物的属性是否克制对方那组宠物的主属性（通过`ELEMENT_BEATS`表）；如果克制，伤害会先乘以`1 + 50/100`（也就是+50%），再交给`dealDamage`处理，而`dealDamage`自己还会照常再叠加一次±25%的属性克制加成，所以属性有利时两个加成是叠在一起生效的。如果属性不占优，这只宠物就只老老实实打出10点固定伤害，没有任何加成。

14. Grey Pet Apatoceratops - 风 | Dino Charge A | 主动（冷却8秒）
`effect: elemental_bonus, effectValue: 60`
**伤害：** 14 （风） | **冷却时间：** 8秒
> *wiki原文 - Dino Charge A:* Against a Water opponent, inflicts +60% damage (on top of normal bonus). Inflicts 14 Air damage.

主动技能，冷却时间8秒，属于「恐龙冲撞/突袭/俯冲/撕咬」这一系技能。造成14点风系基础伤害。结算这次攻击之前，代码会先判断这只宠物的属性是否克制对方那组宠物的主属性（通过`ELEMENT_BEATS`表）；如果克制，伤害会先乘以`1 + 60/100`（也就是+60%），再交给`dealDamage`处理，而`dealDamage`自己还会照常再叠加一次±25%的属性克制加成，所以属性有利时两个加成是叠在一起生效的。如果属性不占优，这只宠物就只老老实实打出14点固定伤害，没有任何加成。

15. Rainbow Kite - 风 | Taste The Rainbow | 主动（冷却8秒）
`effect: random_element`
**伤害：** 16 （风） | **冷却时间：** 8秒
> *wiki原文 - Taste The Rainbow:* Blast a rainbow beam that inflicts damage of a random element. Inflicts 16 Air damage.

主动技能，冷却时间8秒（「彩虹之息」）。仅限这一次施法，这只宠物的属性会在伤害结算前被随机换成火/地/风/水中的一个（会影响属性克制加成和「紫雾」的火属性判定），打完之后立刻恢复成它原本的属性。造成16点基础伤害，具体倍率取决于这次随机到的属性。

16. Grey Pet Pteratops - 风 | Precision Attack A | 主动（冷却9秒）
`effect: anti_dodge_block_conditional, effectValue: 17`
**伤害：** 11 （风） | **冷却时间：** 9秒
> *wiki原文 - Precision Attack A:* If the opponent attempts to dodge or block, hits for 17% more damage. Inflicts 11 Air damage. Can't be dodged or blocked.

主动技能，冷却时间9秒，属于「精准/暴击」这一系恐龙技能。造成11点伤害。在`dealDamage`里，这个技能会无条件把`ignoresDodge`和`ignoresBlock`都设成true，不管对方是闪避还是格挡状态，这一下都会实打实命中满伤害。需要注意：wiki上写的是「如果对方尝试闪避或格挡，则造成17%额外伤害」，但实际代码从来没读取过这个技能的`effectValue`字段，它只是单纯无视闪避和格挡，照面板上的固定伤害打，并没有额外加成。唯一的例外：如果对方的闪避恰好是来自某个带`passive_dodge`被动的宠物（比如Yeonnalligi的「翱翔」被动），穿透效果会被关掉，这一下还是会被正常闪避掉，这跟wiki里「翱翔」克制「精准/暴击」的小知识对得上。

17. Pineapple Kite - 风 | Pineapple Bomb | 主动（冷却9秒）
`effect: none`
**伤害：** 18 （风） | **冷却时间：** 9秒
> *wiki原文 - Pineapple Bomb:* Lob a heavy pineapple. Inflicts 18 Air damage.

主动技能，冷却时间9秒。单纯造成18点固定伤害，不附带任何额外效果，就是纯粹打一下。

18. Red Floaty Balloon - 风 | Floaty Bomb | 主动（冷却9秒）
`effect: none`
**伤害：** 18 （风） | **冷却时间：** 9秒
> *wiki原文 - Floaty Bomb:* Lob a heavy balloon at your foes. Inflicts 18 Air damage.

主动技能，冷却时间9秒。单纯造成18点固定伤害，不附带任何额外效果，就是纯粹打一下。

19. Butterfly Leash - 风 | Acrobatics | 主动（冷却10秒）
`effect: dodge, effectDuration: 5`
**冷却时间：** 10秒
> *wiki原文 - Acrobatics:* Dodge all attacks for 5s.

主动技能，冷却时间10秒。给施法者自己这组宠物加上持续5秒的「闪避」状态。闪避生效期间，只要来的这一下攻击不是明确无视闪避的类型（不带`anti_dodge`/`anti_dodge_block`/`mess_up_undodgeable`标记，也不是能穿透闪避的「精准」系技能），`dealDamage`就会在伤害下限生效之前把它拦成0。根据wiki的小知识，至少有一只用这个效果的宠物（Swoop）就算施法被「口误」判定搞砸了，也照样能拿到这段闪避窗口。

20. Electro Magnifying Glass - 风 | Shock Blast | 主动（冷却10秒）
`effect: stun, effectDuration: 1`
**伤害：** 18 （风） | **冷却时间：** 10秒
> *wiki原文 - Shock Blast:* Electrocute for a 1s stun. Inflicts 18 Air damage.

主动技能，冷却时间10秒。造成18点伤害，然后让目标进入1秒的「眩晕」状态，没法用技能，但（和冰冻不同）不会锁死切换按钮，所以被眩晕的一组宠物随时都能撤回备战组。

21. Grey Pet Apatos Rex - 风 | Dino Chomp A | 主动（冷却10秒）
`effect: elemental_bonus, effectValue: 25`
**伤害：** 18 （风） | **冷却时间：** 10秒
> *wiki原文 - Dino Chomp A:* Against a Water opponent, inflicts +25% damage (on top of normal bonus). Inflicts 18 Air damage.

主动技能，冷却时间10秒，属于「恐龙冲撞/突袭/俯冲/撕咬」这一系技能。造成18点风系基础伤害。结算这次攻击之前，代码会先判断这只宠物的属性是否克制对方那组宠物的主属性（通过`ELEMENT_BEATS`表）；如果克制，伤害会先乘以`1 + 25/100`（也就是+25%），再交给`dealDamage`处理，而`dealDamage`自己还会照常再叠加一次±25%的属性克制加成，所以属性有利时两个加成是叠在一起生效的。如果属性不占优，这只宠物就只老老实实打出18点固定伤害，没有任何加成。

22. Grey Pet Apatosaurus - 风 | Dino Slam A | 主动（冷却10秒）
`effect: elemental_bonus, effectValue: 100`
**伤害：** 15 （风） | **冷却时间：** 10秒
> *wiki原文 - Dino Slam A:* Against a Water opponent, inflicts +100% damage (on top of normal bonus). Inflicts 15 Air damage.

主动技能，冷却时间10秒，属于「恐龙冲撞/突袭/俯冲/撕咬」这一系技能。造成15点风系基础伤害。结算这次攻击之前，代码会先判断这只宠物的属性是否克制对方那组宠物的主属性（通过`ELEMENT_BEATS`表）；如果克制，伤害会先乘以`1 + 100/100`（也就是+100%），再交给`dealDamage`处理，而`dealDamage`自己还会照常再叠加一次±25%的属性克制加成，所以属性有利时两个加成是叠在一起生效的。如果属性不占优，这只宠物就只老老实实打出15点固定伤害，没有任何加成。

23. Grey Pet Triceradon - 风 | Defensive Flurry A | 主动（冷却10秒）
`effect: block, effectDuration: 2`
**伤害：** 12 （风） | **冷却时间：** 10秒
> *wiki原文 - Defensive Flurry A:* Strike and then block attacks for 2s. Inflicts 12 Air damage.

主动技能，冷却时间10秒。对敌方造成12点伤害，然后让施法者自己这组宠物获得持续2秒的「格挡」状态（代码里是给它设置`applyStatusEffect`，类型为`block`）。格挡生效期间，只要这一下攻击不是明确无视格挡的类型（也就是不带`anti_block`/`anti_dodge_block`标记，也不是能穿透格挡的「精准」系技能），`dealDamage`就会在伤害下限生效之前直接把这次伤害拦成0。

24. Grey Pet Tyranodon - 风 | Piercing Jaws A | 主动（冷却10秒）
`effect: trauma, effectDuration: 4`
**伤害：** 15 （风） | **冷却时间：** 10秒
> *wiki原文 - Piercing Jaws A:* Causes trauma, preventing the target from healing for 4s. Inflicts 15 Air damage.

主动技能，冷却时间10秒。造成15点伤害，并给目标附加持续4秒的「创伤」状态。创伤本身不造成伤害，它只是一个纯粹的「禁止治疗」标记：游戏里所有的治疗来源（宠物自带回血、持续回血效果、天气治疗比如「舒缓迷雾」、生命汲取、护盾抵消伤害时顺带的回血、复活）在生效前都会检查目标身上有没有创伤，有的话就直接跳过这次治疗。根据wiki的小知识，创伤并不会阻止「复生/复活」类效果把一组宠物救回来，也不会阻止「护盾」抵消伤害本身，它只会挡住护盾抵消伤害时顺带的回血部分。

25. Haunted Pants - 风 | Phantom Pain | 主动（冷却10秒）
`effect: heal_back, effectDuration: 10`
**伤害：** 30 （风） | **冷却时间：** 10秒
> *wiki原文 - Phantom Pain:* Inflicts phantasmal damage, which heals back over 10s. Inflict 30 Air damage.

主动技能，冷却时间10秒（「幻痛」）。立刻造成30点伤害，然后把这30点伤害安排在接下来10秒内慢慢原样还给目标（通过`battle.healBacks`大致按每一跳平均分摊回血，和其他治疗一样会被创伤挡住）。只要目标没有在回血还完之前死掉或者中创伤，整个窗口结束后血量其实是打平的。

26. Haunted Synthoid - 风 | Draining Beam | 主动（冷却10秒）
`effect: extend_cooldown, effectValue: 3`
**伤害：** 5 （风） | **冷却时间：** 10秒
> *wiki原文 - Draining Beam:* Drain the enemy. Any skills they have on cooldown will take 3s longer to recharge. Inflicts 5 Air damage.

主动技能，冷却时间10秒。根据目标当前状态有两种不同表现：如果目标当前出战的宠物正在冷却中，直接给它剩余冷却加3秒。如果目标出战的宠物已经冷却完毕（可以放技能），则给那一组挂上一个持续时间为0秒的`extend_cd`负面效果，等到那一组的出战宠物下一次放完任意技能，会立刻额外加3秒冷却，然后这个效果自行消失。

27. Lucky Pendant - 风 | Lucky Strike | 主动（冷却10秒）
`effect: random_damage, effectValue: 40`
**伤害：** 1 （风） | **冷却时间：** 10秒
> *wiki原文 - Lucky Strike:* Attack enemy with a chance to inflict 1 - 40 Air damage.

主动技能，冷却时间10秒（「幸运一击」）。在`min(damage, effectValue)`到`max(damage, effectValue)`之间（也就是1到40之间）随机取一个整数当作这一下的伤害。这只宠物面板上写的伤害数值（1）其实只是随机区间的下限，并不是打出去的保底伤害。

28. Mid-Pacific Owl - 风 | Swoop | 主动（冷却10秒）
`effect: dodge, effectDuration: 2`
**伤害：** 15 （风） | **冷却时间：** 10秒
> *wiki原文 - Swoop:* Attack, and dodge all attacks for 2s. Inflicts 15 Air Damage.

主动技能，冷却时间10秒。造成15点伤害，然后给施法者自己这组加上持续2秒的「闪避」状态。闪避生效期间，只要来的这一下攻击不是明确无视闪避的类型（不带`anti_dodge`/`anti_dodge_block`/`mess_up_undodgeable`标记，也不是能穿透闪避的「精准」系技能），`dealDamage`就会在伤害下限生效之前把它拦成0。根据wiki的小知识，至少有一只用这个效果的宠物（Swoop）就算施法被「口误」判定搞砸了，也照样能拿到这段闪避窗口。

29. Pet Hatchley - 风 | Musically Stunned! | 主动（冷却10秒）
`effect: stun, effectDuration: 8`
**伤害：** 10 （风） | **冷却时间：** 10秒
> *wiki原文 - Musically Stunned!:* Sings to deal 10 Air Damage and stuns the target for 8s, making them unable to act.

主动技能，冷却时间10秒。造成10点伤害，然后让目标进入8秒的「眩晕」状态，没法用技能，但（和冰冻不同）不会锁死切换按钮，所以被眩晕的一组宠物随时都能撤回备战组。

30. Zorbnik Leash - 风 | Stun Ray | 主动（冷却10秒）
`effect: stun, effectDuration: 3`
**伤害：** 10 （风） | **冷却时间：** 10秒
> *wiki原文 - Stun Ray:* Stuns the target for 3s, making them unable to act. Inflicts 10 Air damage.

主动技能，冷却时间10秒。造成10点伤害，然后让目标进入3秒的「眩晕」状态，没法用技能，但（和冰冻不同）不会锁死切换按钮，所以被眩晕的一组宠物随时都能撤回备战组。

31. Blue Floaty Balloon - 风 | Bouncy Barrier | 主动（冷却11秒）
`effect: counter, effectDuration: 4`
**伤害：** 25 （风） | **冷却时间：** 11秒
> *wiki原文 - Bouncy Barrier:* Bounce attacks back for 25 damage if hit within 4s.

主动技能，冷却时间11秒，释放时不直接造成伤害。给施法者这组宠物挂上一个数值为25的`counter_reflect`增益（用这只宠物面板上的`damage`字段当反弹伤害数值），持续4秒。增益生效期间，只要这组宠物挨了一下实打实命中的攻击（不管第几次），`dealDamage`都会额外把这25点固定伤害反弹回攻击者那一组，这是在施法者自己受到的伤害之外另外算的。重新释放会直接替换掉之前的反击增益，不会叠加。

32. Grey Pet Triceratops - 风 | Defensive Gore A | 主动（冷却13秒）
`effect: block, effectDuration: 4`
**伤害：** 16 （风） | **冷却时间：** 13秒
> *wiki原文 - Defensive Gore A:* Strike and then block attacks for 4s. Inflicts 16 Air damage.

主动技能，冷却时间13秒。对敌方造成16点伤害，然后让施法者自己这组宠物获得持续4秒的「格挡」状态（代码里是给它设置`applyStatusEffect`，类型为`block`）。格挡生效期间，只要这一下攻击不是明确无视格挡的类型（也就是不带`anti_block`/`anti_dodge_block`标记，也不是能穿透格挡的「精准」系技能），`dealDamage`就会在伤害下限生效之前直接把这次伤害拦成0。

33. Grey Pet Tyranotops - 风 | Crushing Beak A | 主动（冷却13秒）
`effect: trauma, effectDuration: 7`
**伤害：** 19 （风） | **冷却时间：** 13秒
> *wiki原文 - Crushing Beak A:* Causes Trauma, preventing the target from healing for 7 seconds. Inflicts 19 Air damage.

主动技能，冷却时间13秒。造成19点伤害，并给目标附加持续7秒的「创伤」状态。创伤本身不造成伤害，它只是一个纯粹的「禁止治疗」标记：游戏里所有的治疗来源（宠物自带回血、持续回血效果、天气治疗比如「舒缓迷雾」、生命汲取、护盾抵消伤害时顺带的回血、复活）在生效前都会检查目标身上有没有创伤，有的话就直接跳过这次治疗。根据wiki的小知识，创伤并不会阻止「复生/复活」类效果把一组宠物救回来，也不会阻止「护盾」抵消伤害本身，它只会挡住护盾抵消伤害时顺带的回血部分。

34. Grey Pet Pterosaurus - 风 | Critical Strike A | 主动（冷却15秒）
`effect: anti_dodge_block_conditional, effectValue: 50`
**伤害：** 17 （风） | **冷却时间：** 15秒
> *wiki原文 - Critical Strike A:* If the opponent attempts to dodge or block, hits for 50% more damage. Inflicts 17 Air damage. Can't be dodged or blocked.

主动技能，冷却时间15秒，属于「精准/暴击」这一系恐龙技能。造成17点伤害。在`dealDamage`里，这个技能会无条件把`ignoresDodge`和`ignoresBlock`都设成true，不管对方是闪避还是格挡状态，这一下都会实打实命中满伤害。需要注意：wiki上写的是「如果对方尝试闪避或格挡，则造成50%额外伤害」，但实际代码从来没读取过这个技能的`effectValue`字段，它只是单纯无视闪避和格挡，照面板上的固定伤害打，并没有额外加成。唯一的例外：如果对方的闪避恰好是来自某个带`passive_dodge`被动的宠物（比如Yeonnalligi的「翱翔」被动），穿透效果会被关掉，这一下还是会被正常闪避掉，这跟wiki里「翱翔」克制「精准/暴击」的小知识对得上。

35. Pineapple Blump Pet - 风 | Spikey Shield | 主动（冷却15秒）
`effect: counter, effectDuration: 8`
**伤害：** 20 （风） | **冷却时间：** 15秒
> *wiki原文 - Spikey Shield:* Strike back for 20 Air Damage if hit within 8s.

主动技能，冷却时间15秒，释放时不直接造成伤害。给施法者这组宠物挂上一个数值为20的`counter_reflect`增益（用这只宠物面板上的`damage`字段当反弹伤害数值），持续8秒。增益生效期间，只要这组宠物挨了一下实打实命中的攻击（不管第几次），`dealDamage`都会额外把这20点固定伤害反弹回攻击者那一组，这是在施法者自己受到的伤害之外另外算的。重新释放会直接替换掉之前的反击增益，不会叠加。

36. Grey Pet Tyrannosaurus - 风 | Crushing Jaws A | 主动（冷却16秒）
`effect: trauma, effectDuration: 3`
**伤害：** 29 （风） | **冷却时间：** 16秒
> *wiki原文 - Crushing Jaws A:* Causes trauma, preventing the target for 3s. Inflicts 29 Air damage.

主动技能，冷却时间16秒。造成29点伤害，并给目标附加持续3秒的「创伤」状态。创伤本身不造成伤害，它只是一个纯粹的「禁止治疗」标记：游戏里所有的治疗来源（宠物自带回血、持续回血效果、天气治疗比如「舒缓迷雾」、生命汲取、护盾抵消伤害时顺带的回血、复活）在生效前都会检查目标身上有没有创伤，有的话就直接跳过这次治疗。根据wiki的小知识，创伤并不会阻止「复生/复活」类效果把一组宠物救回来，也不会阻止「护盾」抵消伤害本身，它只会挡住护盾抵消伤害时顺带的回血部分。

37. Grey Pet Tricerus Rex - 风 | Defensive Bite A | 主动（冷却18秒）
`effect: block, effectDuration: 2`
**伤害：** 22 （风） | **冷却时间：** 18秒
> *wiki原文 - Defensive Bite A:* Strike and then block attacks for 2s. Inflicts 22 Air damage.

主动技能，冷却时间18秒。对敌方造成22点伤害，然后让施法者自己这组宠物获得持续2秒的「格挡」状态（代码里是给它设置`applyStatusEffect`，类型为`block`）。格挡生效期间，只要这一下攻击不是明确无视格挡的类型（也就是不带`anti_block`/`anti_dodge_block`标记，也不是能穿透格挡的「精准」系技能），`dealDamage`就会在伤害下限生效之前直接把这次伤害拦成0。

38. Grey Pet Pteranus Rex - 风 | Precision Crush A | 主动（冷却19秒）
`effect: anti_dodge_block_conditional, effectValue: 10`
**伤害：** 25 （风） | **冷却时间：** 19秒
> *wiki原文 - Precision Crush A:* If the opponent attempts to dodge or block, hits for 10% more damage. Inflicts 25 Air damage. Can't be dodged or blocked.

主动技能，冷却时间19秒，属于「精准/暴击」这一系恐龙技能。造成25点伤害。在`dealDamage`里，这个技能会无条件把`ignoresDodge`和`ignoresBlock`都设成true，不管对方是闪避还是格挡状态，这一下都会实打实命中满伤害。需要注意：wiki上写的是「如果对方尝试闪避或格挡，则造成10%额外伤害」，但实际代码从来没读取过这个技能的`effectValue`字段，它只是单纯无视闪避和格挡，照面板上的固定伤害打，并没有额外加成。唯一的例外：如果对方的闪避恰好是来自某个带`passive_dodge`被动的宠物（比如Yeonnalligi的「翱翔」被动），穿透效果会被关掉，这一下还是会被正常闪避掉，这跟wiki里「翱翔」克制「精准/暴击」的小知识对得上。

39. Cloud Rabbit - 风 | Self Sacrifice | 主动（冷却20秒）
`effect: dodge_swap, effectDuration: 4`
**冷却时间：** 20秒
> *wiki原文 - Self Sacrifice:* If struck in the next 4 seconds, avoid it completely and swap out instead (if able to swap). The smoke left behind helps your teammate dodge all attacks for 4 seconds. Isn't visible to your enemy.

主动技能，冷却时间20秒（「舍身」/「金蝉脱壳」系技能）。如果施法者还有一组活着的备战宠物、并且当前允许切换，就立刻切换过去（同时清掉正在离场那组身上的「厄运」诅咒），并给刚换上场的那一组加上持续4秒的「闪避」状态，是换上来的这组拿到闪避窗口，而不是撤下去的那组（因为撤下去那组已经不在场上，本来就打不到）。根据wiki的小知识，这招理论上也能被这只宠物其他自我指向的技能连带触发（比如石化、杂技、跳跃等），但目前的模拟器只实现了这只宠物自己主动释放这一招的情况。

40. Cosmic Unicorn Bracelet - 风 | Cosmic Barrage | 主动（冷却20秒）
`effect: anti_dodge`
**伤害：** 36 （风） | **冷却时间：** 20秒
> *wiki原文 - Cosmic Barrage:* Crush the target with a comet from the heavens. Inflicts 36 Air damage. Can't be dodged.

主动技能，冷却时间20秒。造成36点伤害。在`dealDamage`里设置`ignoresDodge`，不管是临时的「闪避」状态还是被动闪避概率（比如「翱翔」），都没法帮目标躲开这一下，但格挡不受影响，还是能挡住。

41. Discoball Companion - 风 | Just Dance! | 主动（冷却20秒）
`effect: force_dance, effectDuration: 4, dotDamage: 5, dotDuration: 4`
**伤害：** 5 （风） | **冷却时间：** 20秒 | **持续伤害：** 每秒5点，持续4秒
> *wiki原文 - Just Dance!:* Infect the target to dance for 4s, unable to do anything other than swap out, and suffering for 5 Air Damage per second.

主动技能，冷却时间20秒（「魅惑」/「迪斯科热潮」/「劲舞」系技能）。造成5点伤害（如果有的话），并让目标进入4秒的「跳舞」状态，期间`canAct`被设为false，目标完全没法用任何技能，不过依然能正常切换（「跳舞」不在锁定切换的状态列表里）。如果具体是「迪斯科热潮」这一招，还会额外给目标附加同样时长的「闪避」状态，根据wiki的小知识，这就是「跳舞」类负面效果会顺带给受害者一段闪避窗口的原因，而不是给施法者自己。

42. Green Floaty Balloon - 风 | Pumped Up | 主动（冷却20秒）
`effect: none`
**回血：** 30 | **冷却时间：** 20秒
> *wiki原文 - Pumped Up:* Pump yourself up to regenerate health. Heals 30 life.

主动技能，冷却时间20秒。不造成伤害也没有其他附加效果，单纯给施法者自己这组宠物回30点固定生命值，仅此而已。

43. Grey Pet Tripatosaurus - 风 | Defensive Bash A | 主动（冷却20秒）
`effect: block, effectDuration: 7`
**伤害：** 10 （风） | **冷却时间：** 20秒
> *wiki原文 - Defensive Bash A:* Strike and then block attacks for 7s. Inflicts 10 Air damage

主动技能，冷却时间20秒。对敌方造成10点伤害，然后让施法者自己这组宠物获得持续7秒的「格挡」状态（代码里是给它设置`applyStatusEffect`，类型为`block`）。格挡生效期间，只要这一下攻击不是明确无视格挡的类型（也就是不带`anti_block`/`anti_dodge_block`标记，也不是能穿透格挡的「精准」系技能），`dealDamage`就会在伤害下限生效之前直接把这次伤害拦成0。

44. Grey Pet Tyranopatos - 风 | Grinding Jaws A | 主动（冷却20秒）
`effect: trauma, effectDuration: 30`
**伤害：** 3 （风） | **冷却时间：** 20秒
> *wiki原文 - Grinding Jaws A:* Causes Trauma, preventing the target from healing for 30s. Inflicts 3 Air damage.

主动技能，冷却时间20秒。造成3点伤害，并给目标附加持续30秒的「创伤」状态。创伤本身不造成伤害，它只是一个纯粹的「禁止治疗」标记：游戏里所有的治疗来源（宠物自带回血、持续回血效果、天气治疗比如「舒缓迷雾」、生命汲取、护盾抵消伤害时顺带的回血、复活）在生效前都会检查目标身上有没有创伤，有的话就直接跳过这次治疗。根据wiki的小知识，创伤并不会阻止「复生/复活」类效果把一组宠物救回来，也不会阻止「护盾」抵消伤害本身，它只会挡住护盾抵消伤害时顺带的回血部分。

45. Lovebird Pendant - 风 | Enthrall | 主动（冷却20秒）
`effect: force_dance, effectDuration: 6`
**伤害：** 10 （风） | **冷却时间：** 20秒
> *wiki原文 - Enthrall:* Mesmerize your enemy, so it can do nothing but swap out for 6s. Inflicts 10 Air Damage.

主动技能，冷却时间20秒（「魅惑」/「迪斯科热潮」/「劲舞」系技能）。造成10点伤害（如果有的话），并让目标进入6秒的「跳舞」状态，期间`canAct`被设为false，目标完全没法用任何技能，不过依然能正常切换（「跳舞」不在锁定切换的状态列表里）。如果具体是「迪斯科热潮」这一招，还会额外给目标附加同样时长的「闪避」状态，根据wiki的小知识，这就是「跳舞」类负面效果会顺带给受害者一段闪避窗口的原因，而不是给施法者自己。

46. Spiritual Resonator - 风 | Possession | 主动（冷却20秒）
`effect: random_skill_wrong_target`
**冷却时间：** 20秒
> *wiki原文 - Possession:* Possesses the target into using a random skill on the wrong target.

主动技能，冷却时间20秒（「附身」）。这只宠物自己不直接造成伤害。它会从敌方出战那一组里随机挑一只带有攻击技能的宠物，把那只宠物的攻击「反过来」打在敌方自己身上，伤害就是那只宠物面板上写的固定伤害。如果敌方当前出战的宠物里根本没有能被附身的攻击型宠物，这招就会打空，什么都不会发生。（源码里注明这是一个「尽力而为」的实现，wiki没有明确说明具体的选取/目标规则。）

47. Death's Scarf - 风 | Doom | 主动（冷却30秒）
`effect: doom, effectDuration: 20`
**冷却时间：** 30秒
> *wiki原文 - Doom:* Mark the target for death in 20s. The effect can be erased by simply swapping pets.

主动技能，冷却时间30秒。不造成伤害，转而给目标那组诅咒上持续20秒的「厄运」。如果倒计时归零时厄运还挂在那一组身上，`gameTick`会立刻把它的血量直接清零。关键在于：只要中了厄运的那一组主动切换离场，厄运就会立刻被清掉（`swapDeck`在切换时会顺手清掉任何`doom`状态），这正好对应wiki里说的「只要切换就能解除」，但如果它被什么效果锁住没法切换，厄运就会一直挂着直到生效。

48. Thinking Cap - 风 | Mind Swap | 主动（冷却30秒）
`effect: mind_swap`
**冷却时间：** 30秒
> *wiki原文 - Mind Swap:* Take all the positive effects on your opponent, and give all your negative effects to them.

主动技能，冷却时间30秒（「思考帽」）。不直接造成伤害。按wiki的说法：「把对手身上所有正面效果拿过来，再把自己身上所有负面效果丢给对手」。具体到代码里：敌方那组身上所有的增益会被整体搬到施法者这组身上（敌方的增益列表被清空），同时施法者这组身上所有的减益会被整体搬到敌方那组身上（施法者的减益列表被清空），相当于增益和减益朝相反方向做了一次整体互换。

49. Onisim's Genie - 风 | Wish | 主动（冷却999秒）
`effect: revive, effectValue: 30`
**回血：** 30 | **冷却时间：** 999秒
> *wiki原文 - Wish:* Bring your partner back to life with 30% health!

主动技能，冷却时间999秒，因为冷却长得离谱，实际能出手的窗口几乎可以忽略不计。如果施法者自己的备战组当前血量是0，会把它复活到最大生命值的30%。如果备战组还活着，放这个技能就完全没有效果（不回血也没有其他作用），它只是纯粹的复活手段，不是治疗技能。

---

### 地系 (Earth)

50. Mini Mammoth Leash - 地 | Mammoth Heart | 被动
`effect: passive_hp_boost, effectValue: 30`
> *wiki原文 - Mammoth Heart:* Passive - Maximum life is 30% higher.

一直生效。`getMaxHp`会检查出战的那一组里有没有带这个效果的宠物；如果有，那一组的最大生命值就会从正常的150变成195（也就是固定+30%，跟wiki一致）。只有拿着这只宠物的那一组能吃到这个加成，搭档那一组除非自己也带了一份，否则依旧是150上限。

51. Pet Burrito - 地 | Stubborn | 被动
`effect: passive_dmg_reduce, effectValue: 25`
> *wiki原文 - Stubborn:* Passive - Suffers 25% less damage from all attacks.

一直生效，但只在这只宠物正是当前出战宠物的时候才起作用（`dealDamage`里会去核对`activePet`）。它要挨的每一下攻击都会先乘以0.75（也就是固定减伤25%），这个减免发生在属性加成、闪避/格挡判定、灵体减伤之后，但在伤害下限（最低1点）生效之前。根据wiki的小知识，这只宠物一旦被换下场，这个减伤马上就失效。

52. Baby Bunny - 地 | Hop | 主动（冷却3秒）
`effect: none`
**冷却时间：** 3秒
> *wiki原文 - Hop:* Hop. (This ability does not do anything beside making the current pet jump)

主动技能，冷却时间3秒。单纯造成0点固定伤害，不附带任何额外效果，就是纯粹打一下。

53. Puppy Leash - 地 | Pounce | 主动（冷却4秒）
`effect: none`
**伤害：** 9 （地） | **冷却时间：** 4秒
> *wiki原文 - Pounce:* Leap to strike! Inflicts 9 Earth damage.

主动技能，冷却时间4秒。单纯造成9点固定伤害，不附带任何额外效果，就是纯粹打一下。

54. Brown Pet Pteranodon - 地 | Precision Strike E | 主动（冷却5秒）
`effect: anti_dodge_block_conditional, effectValue: 25`
**伤害：** 7 （地） | **冷却时间：** 5秒
> *wiki原文 - Precision Strike E:* If the opponent attempts to dodge or block, hits for 25% more damage. Inflicts 7 Earth damage. Can't be dodged or blocked.

主动技能，冷却时间5秒，属于「精准/暴击」这一系恐龙技能。造成7点伤害。在`dealDamage`里，这个技能会无条件把`ignoresDodge`和`ignoresBlock`都设成true，不管对方是闪避还是格挡状态，这一下都会实打实命中满伤害。需要注意：wiki上写的是「如果对方尝试闪避或格挡，则造成25%额外伤害」，但实际代码从来没读取过这个技能的`effectValue`字段，它只是单纯无视闪避和格挡，照面板上的固定伤害打，并没有额外加成。唯一的例外：如果对方的闪避恰好是来自某个带`passive_dodge`被动的宠物（比如Yeonnalligi的「翱翔」被动），穿透效果会被关掉，这一下还是会被正常闪避掉，这跟wiki里「翱翔」克制「精准/暴击」的小知识对得上。

55. Cinder Sprites - 地 | Multiply! | 主动（冷却5秒）
`effect: revive, effectValue: 5, effectDuration: 5`
**回血：** 5 | **冷却时间：** 5秒
> *wiki原文 - Multiply!:* Summon a friend for 5s. If beaten, it takes the hit instead, reviving you with 5% life.

主动技能，冷却时间5秒，虽然冷却不长，但实际能等到的出手窗口也就是这么长。如果施法者自己的备战组当前血量是0，会把它复活到最大生命值的5%。如果备战组还活着，放这个技能就完全没有效果（不回血也没有其他作用），它只是纯粹的复活手段，不是治疗技能。

56. Puddy Leash - 地 | Hairball | 主动（冷却5秒）
`effect: none`
**伤害：** 10 （地） | **冷却时间：** 5秒
> *wiki原文 - Hairball:* Launch a hairball. Gross. Inflitcts 10 Earth damage.

主动技能，冷却时间5秒。单纯造成10点固定伤害，不附带任何额外效果，就是纯粹打一下。

57. Brown Pet Apatodon - 地 | Dino Dive E | 主动（冷却6秒）
`effect: elemental_bonus, effectValue: 50`
**伤害：** 10 （地） | **冷却时间：** 6秒
> *wiki原文 - Dino Dive E:* Against an Air opponent, inflicts +50% damage (on top of normal bonus.) Inflicts 10 Earth damage.

主动技能，冷却时间6秒，属于「恐龙冲撞/突袭/俯冲/撕咬」这一系技能。造成10点地系基础伤害。结算这次攻击之前，代码会先判断这只宠物的属性是否克制对方那组宠物的主属性（通过`ELEMENT_BEATS`表）；如果克制，伤害会先乘以`1 + 50/100`（也就是+50%），再交给`dealDamage`处理，而`dealDamage`自己还会照常再叠加一次±25%的属性克制加成，所以属性有利时两个加成是叠在一起生效的。如果属性不占优，这只宠物就只老老实实打出10点固定伤害，没有任何加成。

58. Mini Growtopian - 地 | Punch, Build, Grow | 主动（冷却6秒）
`effect: stacking_build, effectValue: 4`
**伤害：** 8 （地） | **冷却时间：** 6秒
> *wiki原文 - Punch, Build, Grow:* Punch hits for 8 Earth damage. Build builds a wall that blocks attacks for 1s. Grow adds 4 damage to Punch and 1s to Build, stacking up to 5 times.

主动技能，冷却时间6秒（「迷你格罗托比安」）。连续释放会在三个招式之间固定轮换，先是「重拳」，再是「建造」，再是「生长」，然后又回到「重拳」：重拳造成`8 + 生长层数 × 4`点伤害；建造给自己上持续`1 + 生长层数`秒的格挡；生长会永久性（本场战斗内，最多叠5层）增加一层，同时提升另外两招的效果。轮换到第几招、叠了几层，都会跟着这只宠物记一整场战斗。

59. Rhino Horn - 地 | Charge | 主动（冷却6秒）
`effect: self_stun, effectDuration: 3`
**伤害：** 16 （地） | **冷却时间：** 6秒
> *wiki原文 - Charge:* Ram the enemy, stunning yourself for 3s in the process. Inflicts 16 Earth damage.

主动技能，冷却时间6秒（「犀牛角」）。通过普通流程对敌方造成16点伤害，然后作为反噬效果让施法者自己这组眩晕3秒，没法行动，但按眩晕的一般规则，因为眩晕不在锁定切换的状态列表里（只有冰冻才会锁切换），所以还是能正常切换离场。

60. Unicorn Garland - 地 | Rainbow Beam | 主动（冷却6秒）
`effect: anti_block`
**伤害：** 11 （地） | **冷却时间：** 6秒
> *wiki原文 - Rainbow Beam:* Fire a blast of mythical colors. Inflicts 11 Earth Damage. Can't be blocked.

主动技能，冷却时间6秒。造成11点伤害。在`dealDamage`里设置`ignoresBlock`，目标身上的「格挡」状态对这一下完全不起作用，但闪避不受影响，该触发还是照常触发。

61. Calf Leash - 地 | Trample | 主动（冷却7秒）
`effect: force_swap`
**伤害：** 10 （地） | **冷却时间：** 7秒
> *wiki原文 - Trample:* Force opponent to swap pets. Inflicts 10 Earth Damage.

主动技能，冷却时间7秒（「践踏」）。造成10点伤害，然后，如果敌方还有一组活着、可以切换、没被冰冻的备战宠物，强行把敌方换到那一组去，同时清掉正在离场那一组身上的厄运。和「放逐」不同的是，这招不会锁住敌方换上来的那一组，让它之后没法再换回去。

62. Brown Pet Apatoceratops - 地 | Dino Charge E | 主动（冷却8秒）
`effect: elemental_bonus, effectValue: 60`
**伤害：** 14 （地） | **冷却时间：** 8秒
> *wiki原文 - Dino Charge E:* Against an Air opponent, inflicts +60% damage (on top of normal bonus). Inflicts 14 Earth damage.

主动技能，冷却时间8秒，属于「恐龙冲撞/突袭/俯冲/撕咬」这一系技能。造成14点地系基础伤害。结算这次攻击之前，代码会先判断这只宠物的属性是否克制对方那组宠物的主属性（通过`ELEMENT_BEATS`表）；如果克制，伤害会先乘以`1 + 60/100`（也就是+60%），再交给`dealDamage`处理，而`dealDamage`自己还会照常再叠加一次±25%的属性克制加成，所以属性有利时两个加成是叠在一起生效的。如果属性不占优，这只宠物就只老老实实打出14点固定伤害，没有任何加成。

63. Brown Pet Pteratops - 地 | Precision Attack E | 主动（冷却9秒）
`effect: anti_dodge_block_conditional, effectValue: 17`
**伤害：** 11 （地） | **冷却时间：** 9秒
> *wiki原文 - Precision Attack E:* If the opponent attempts to dodge or block, hits for 17% more damage. Inflicts 11 Earth damage. Can't be dodged or blocked.

主动技能，冷却时间9秒，属于「精准/暴击」这一系恐龙技能。造成11点伤害。在`dealDamage`里，这个技能会无条件把`ignoresDodge`和`ignoresBlock`都设成true，不管对方是闪避还是格挡状态，这一下都会实打实命中满伤害。需要注意：wiki上写的是「如果对方尝试闪避或格挡，则造成17%额外伤害」，但实际代码从来没读取过这个技能的`effectValue`字段，它只是单纯无视闪避和格挡，照面板上的固定伤害打，并没有额外加成。唯一的例外：如果对方的闪避恰好是来自某个带`passive_dodge`被动的宠物（比如Yeonnalligi的「翱翔」被动），穿透效果会被关掉，这一下还是会被正常闪避掉，这跟wiki里「翱翔」克制「精准/暴击」的小知识对得上。

64. Brown Pet Apatos Rex - 地 | Dino Chomp E | 主动（冷却10秒）
`effect: elemental_bonus, effectValue: 25`
**伤害：** 18 （地） | **冷却时间：** 10秒
> *wiki原文 - Dino Chomp E:* Against an Air opponent, inflicts +25% damage (on top of normal bonus). Inflicts 18 Earth damage.

主动技能，冷却时间10秒，属于「恐龙冲撞/突袭/俯冲/撕咬」这一系技能。造成18点地系基础伤害。结算这次攻击之前，代码会先判断这只宠物的属性是否克制对方那组宠物的主属性（通过`ELEMENT_BEATS`表）；如果克制，伤害会先乘以`1 + 25/100`（也就是+25%），再交给`dealDamage`处理，而`dealDamage`自己还会照常再叠加一次±25%的属性克制加成，所以属性有利时两个加成是叠在一起生效的。如果属性不占优，这只宠物就只老老实实打出18点固定伤害，没有任何加成。

65. Brown Pet Apatosaurus - 地 | Dino Slam E | 主动（冷却10秒）
`effect: elemental_bonus, effectValue: 100`
**伤害：** 15 （地） | **冷却时间：** 10秒
> *wiki原文 - Dino Slam E:* Against an Air opponent, inflicts +100% damage (on top of normal bonus). Inflicts 15 Earth damage.

主动技能，冷却时间10秒，属于「恐龙冲撞/突袭/俯冲/撕咬」这一系技能。造成15点地系基础伤害。结算这次攻击之前，代码会先判断这只宠物的属性是否克制对方那组宠物的主属性（通过`ELEMENT_BEATS`表）；如果克制，伤害会先乘以`1 + 100/100`（也就是+100%），再交给`dealDamage`处理，而`dealDamage`自己还会照常再叠加一次±25%的属性克制加成，所以属性有利时两个加成是叠在一起生效的。如果属性不占优，这只宠物就只老老实实打出15点固定伤害，没有任何加成。

66. Brown Pet Triceradon - 地 | Defensive Flurry E | 主动（冷却10秒）
`effect: block, effectDuration: 2`
**伤害：** 12 （地） | **冷却时间：** 10秒
> *wiki原文 - Defensive Flurry E:* Strike and then block attacks for 2s. Inflicts 12 Earth damage.

主动技能，冷却时间10秒。对敌方造成12点伤害，然后让施法者自己这组宠物获得持续2秒的「格挡」状态（代码里是给它设置`applyStatusEffect`，类型为`block`）。格挡生效期间，只要这一下攻击不是明确无视格挡的类型（也就是不带`anti_block`/`anti_dodge_block`标记，也不是能穿透格挡的「精准」系技能），`dealDamage`就会在伤害下限生效之前直接把这次伤害拦成0。

67. Brown Pet Tyranodon - 地 | Piercing Jaws E | 主动（冷却10秒）
`effect: trauma, effectDuration: 4`
**伤害：** 15 （地） | **冷却时间：** 10秒
> *wiki原文 - Piercing Jaws E:* Causes trauma, preventing the target from healing for 4s. Inflicts 15 Earth damage.

主动技能，冷却时间10秒。造成15点伤害，并给目标附加持续4秒的「创伤」状态。创伤本身不造成伤害，它只是一个纯粹的「禁止治疗」标记：游戏里所有的治疗来源（宠物自带回血、持续回血效果、天气治疗比如「舒缓迷雾」、生命汲取、护盾抵消伤害时顺带的回血、复活）在生效前都会检查目标身上有没有创伤，有的话就直接跳过这次治疗。根据wiki的小知识，创伤并不会阻止「复生/复活」类效果把一组宠物救回来，也不会阻止「护盾」抵消伤害本身，它只会挡住护盾抵消伤害时顺带的回血部分。

68. Leashed Silkworm - Grey - 地 | Stoneform | 主动（冷却10秒）
`effect: block, effectDuration: 5`
**冷却时间：** 10秒
> *wiki原文 - Stoneform:* Turn to stone, blocking attacks for 5s.

主动技能，冷却时间10秒。对敌方造成0点伤害，然后让施法者自己这组宠物获得持续5秒的「格挡」状态（代码里是给它设置`applyStatusEffect`，类型为`block`）。格挡生效期间，只要这一下攻击不是明确无视格挡的类型（也就是不带`anti_block`/`anti_dodge_block`标记，也不是能穿透格挡的「精准」系技能），`dealDamage`就会在伤害下限生效之前直接把这次伤害拦成0。

69. Piglet Leash - 地 | Wallow | 主动（冷却10秒）
`effect: none`
**回血：** 20 | **冷却时间：** 10秒
> *wiki原文 - Wallow:* Wallow in the mud, healing yourself. Heals 20 life.

主动技能，冷却时间10秒。不造成伤害也没有其他附加效果，单纯给施法者自己这组宠物回20点固定生命值，仅此而已。

70. Tumbleweed Attractor - 地 | Needles | 主动（冷却10秒）
`effect: thorns, effectDuration: 15`
**伤害：** 5 （地） | **冷却时间：** 10秒
> *wiki原文 - Needles:* For 15s, inflict 5 Earth damage when you are hit.

主动技能，冷却时间10秒，释放时不直接造成伤害。给施法者这组宠物挂上一个数值为5的`counter_reflect`增益（用这只宠物面板上的`damage`字段当反弹伤害数值），持续15秒。增益生效期间，只要这组宠物挨了一下实打实命中的攻击（不管第几次），`dealDamage`都会额外把这5点固定伤害反弹回攻击者那一组，这是在施法者自己受到的伤害之外另外算的。重新释放会直接替换掉之前的反击增益，不会叠加。

71. Unearthly Synthoid - 地 | Unearthly Beam | 主动（冷却10秒）
`effect: bonus_on_debuff, effectValue: 15`
**伤害：** 10 （地） | **冷却时间：** 10秒
> *wiki原文 - Unearthly Beam:* Shoot a beam. If the enemy has any negative effects, one is vaporized for an extra 15 damage. Inflicts 10 Earth Damage.

主动技能，冷却时间10秒。基础伤害先按普通流程打出去；然后代码会检查目标身上有没有带任何负面效果，先看有没有减益类负面效果（`targetDeck.modifiers.debuffs`），没有的话再看有没有属于`['stun','freeze','trauma','dot','dance','sleep']`这几种的状态效果。只要找到一个，就会把它从目标身上摘掉，同时这只宠物额外多打15点伤害。如果目标身上什么负面效果都没有，就不会有额外伤害，也不会摘掉任何东西。

72. Zombie Hound - 地 | Bursting Spores | 主动（冷却10秒）
`effect: delayed_damage, effectValue: 15, effectDuration: 6`
**伤害：** 10 （地） | **冷却时间：** 10秒
> *wiki原文 - Bursting Spores:* Bite, infecting with spores that burst after 6s for 15 damage. Inflicts 10 Earth damage.

主动技能，冷却时间10秒（「僵尸猎犬」）。基础伤害先按普通流程打出去，然后会另外安排一次延迟爆发：6秒后（每个游戏刻倒数一次，在`gameTick`里处理），目标会通过`dealDamage`额外承受15点固定伤害，而且这次伤害不带任何施法宠物的信息，所以不会受到这只宠物自身的闪避/格挡穿透特性影响，只看孢子爆开那一刻目标本身的状态。

73. 14DEViL's Friendly Doggy Leash - 地 | Lockjaw! | 主动（冷却12秒）
`effect: trap_swap, effectValue: 40`
**伤害：** 5 （地） | **冷却时间：** 12秒
> *wiki原文 - Lockjaw!:* Bite its opponent for 5 Earth Damage holding on. If your opponent retreats, it lashes out for 40 Earth Damage. Inflicts 5 Earth Damage. Isn't visible to your enemy.

主动技能，冷却时间12秒。在敌方身上布下一个看不见的陷阱，根据wiki的小知识，这个陷阱本身不会显示给敌方看（不过施法动画能看到，所以对面能猜到你放了什么），持续窗口是4秒。如果陷阱布下期间敌方主动切换，就会触发，额外造成40点伤害；如果具体是那只技能名叫「Surprise!」的宠物放的，触发时会同时打中敌方两组宠物，而不只是换上场的那一组。

74. Flowersaurus Rex - 地 | Pollen Barrage | 主动（冷却12秒）
`effect: consume_buff, effectValue: 15`
**伤害：** 15 （地） | **冷却时间：** 12秒
> *wiki原文 - Pollen Barrage:* Spew pollen. Consumes your positive effects to add 15 damage per effect. Inflicts 15 Earth damage.

主动技能，冷却时间12秒（「辐射光束」/「花粉轰炸」系技能）。在结算伤害之前，会先告诉`dealDamage`这一下要无视施法者自己身上的增益（`isConsumeBuff`），结算完之后还会直接删掉施法者这组身上最早加上的那个增益。总的效果就是：这一下攻击完全吃不到施法者自己的伤害增益（比如「怒吼」「幸运加持」之类都会被跳过），而且还要搭上一个增益当作释放代价，这正好对应wiki里说的「伤害增益不生效是因为被这招消耗掉了」的小知识。

75. Moose Cap - 地 | Antler Bash | 主动（冷却12秒）
`effect: anti_block`
**伤害：** 22 （地） | **冷却时间：** 12秒
> *wiki原文 - Antler Bash:* Smash them with your mighty antlers! Inflicts 22 Earth damage. Can't be blocked.

主动技能，冷却时间12秒。造成22点伤害。在`dealDamage`里设置`ignoresBlock`，目标身上的「格挡」状态对这一下完全不起作用，但闪避不受影响，该触发还是照常触发。

76. Brown Pet Triceratops - 地 | Defensive Gore E | 主动（冷却13秒）
`effect: block, effectDuration: 4`
**伤害：** 16 （地） | **冷却时间：** 13秒
> *wiki原文 - Defensive Gore E:* Strike and then block attacks for 4s. Inflicts 16 Earth damage.

主动技能，冷却时间13秒。对敌方造成16点伤害，然后让施法者自己这组宠物获得持续4秒的「格挡」状态（代码里是给它设置`applyStatusEffect`，类型为`block`）。格挡生效期间，只要这一下攻击不是明确无视格挡的类型（也就是不带`anti_block`/`anti_dodge_block`标记，也不是能穿透格挡的「精准」系技能），`dealDamage`就会在伤害下限生效之前直接把这次伤害拦成0。

77. Brown Pet Tyranotops - 地 | Crushing Beak E | 主动（冷却13秒）
`effect: trauma, effectDuration: 7`
**伤害：** 19 （地） | **冷却时间：** 13秒
> *wiki原文 - Crushing Beak E:* Causes Trauma, preventing the target from healing for 7 seconds. Inflicts 19 Earth damage.

主动技能，冷却时间13秒。造成19点伤害，并给目标附加持续7秒的「创伤」状态。创伤本身不造成伤害，它只是一个纯粹的「禁止治疗」标记：游戏里所有的治疗来源（宠物自带回血、持续回血效果、天气治疗比如「舒缓迷雾」、生命汲取、护盾抵消伤害时顺带的回血、复活）在生效前都会检查目标身上有没有创伤，有的话就直接跳过这次治疗。根据wiki的小知识，创伤并不会阻止「复生/复活」类效果把一组宠物救回来，也不会阻止「护盾」抵消伤害本身，它只会挡住护盾抵消伤害时顺带的回血部分。

78. Brown Pet Pterosaurus - 地 | Critical Strike E | 主动（冷却15秒）
`effect: anti_dodge_block_conditional, effectValue: 50`
**伤害：** 17 （地） | **冷却时间：** 15秒
> *wiki原文 - Critical Strike E:* If the opponent attempts to dodge or block, hits for 50% more damage. Inflicts 17 Earth damage. Can't be dodged or blocked.

主动技能，冷却时间15秒，属于「精准/暴击」这一系恐龙技能。造成17点伤害。在`dealDamage`里，这个技能会无条件把`ignoresDodge`和`ignoresBlock`都设成true，不管对方是闪避还是格挡状态，这一下都会实打实命中满伤害。需要注意：wiki上写的是「如果对方尝试闪避或格挡，则造成50%额外伤害」，但实际代码从来没读取过这个技能的`effectValue`字段，它只是单纯无视闪避和格挡，照面板上的固定伤害打，并没有额外加成。唯一的例外：如果对方的闪避恰好是来自某个带`passive_dodge`被动的宠物（比如Yeonnalligi的「翱翔」被动），穿透效果会被关掉，这一下还是会被正常闪避掉，这跟wiki里「翱翔」克制「精准/暴击」的小知识对得上。

79. Cool Capybara - 地 | Fortunate Fame! | 主动（冷却15秒）
`effect: dodge, effectDuration: 10`
**冷却时间：** 15秒
> *wiki原文 - Fortunate Fame!:* Dodge all attacks for 10s.

主动技能，冷却时间15秒。给施法者自己这组宠物加上持续10秒的「闪避」状态。闪避生效期间，只要来的这一下攻击不是明确无视闪避的类型（不带`anti_dodge`/`anti_dodge_block`/`mess_up_undodgeable`标记，也不是能穿透闪避的「精准」系技能），`dealDamage`就会在伤害下限生效之前把它拦成0。根据wiki的小知识，至少有一只用这个效果的宠物（Swoop）就算施法被「口误」判定搞砸了，也照样能拿到这段闪避窗口。

80. MickeyMay Leash - 地 | Cower | 主动（冷却15秒）
`effect: block, effectDuration: 7`
**冷却时间：** 15秒
> *wiki原文 - Cower:* Blocks all attacks for 7s.

主动技能，冷却时间15秒。对敌方造成0点伤害，然后让施法者自己这组宠物获得持续7秒的「格挡」状态（代码里是给它设置`applyStatusEffect`，类型为`block`）。格挡生效期间，只要这一下攻击不是明确无视格挡的类型（也就是不带`anti_block`/`anti_dodge_block`标记，也不是能穿透格挡的「精准」系技能），`dealDamage`就会在伤害下限生效之前直接把这次伤害拦成0。

81. Pet Bunny - 地 | Big Sharp Pointy Teeth | 主动（冷却15秒）
`effect: none`
**伤害：** 30 （地） | **冷却时间：** 15秒
> *wiki原文 - Big Sharp Pointy Teeth:* Murder violently. Inflicts 30 Earth damage.

主动技能，冷却时间15秒。单纯造成30点固定伤害，不附带任何额外效果，就是纯粹打一下。

82. Playful Wood Sprite - 地 | Playtime | 主动（冷却15秒）
`effect: buff_chance_double_damage, effectValue: 30, effectDuration: 4`
**伤害：** 5 （地） | **冷却时间：** 15秒
> *wiki原文 - Playtime:* Grants you a 30% chance to do double damage for 4 seconds. Inflicts 5 Earth damage.

主动技能，冷却时间15秒。造成5点伤害，然后给施法者自己这组宠物加上一个`double_dmg`增益，持续4秒，期间每次攻击有30%概率触发。增益生效时，这组宠物之后每一次攻击在`dealDamage`结算时都会掷一次`doubleDmgChance`判定，一旦命中，那一下的伤害就翻倍。在增益消失前再次释放，只是把倒计时刷新回4秒（不同来源的增益各自独立叠加，不会互相覆盖）。

83. Prickly Pet - 地 | Spikey Shield | 主动（冷却15秒）
`effect: counter, effectDuration: 8`
**伤害：** 20 （地） | **冷却时间：** 15秒
> *wiki原文 - Spikey Shield:* Strike back for 20 Earth Damage if hit within 8s.

主动技能，冷却时间15秒，释放时不直接造成伤害。给施法者这组宠物挂上一个数值为20的`counter_reflect`增益（用这只宠物面板上的`damage`字段当反弹伤害数值），持续8秒。增益生效期间，只要这组宠物挨了一下实打实命中的攻击（不管第几次），`dealDamage`都会额外把这20点固定伤害反弹回攻击者那一组，这是在施法者自己受到的伤害之外另外算的。重新释放会直接替换掉之前的反击增益，不会叠加。

84. Brown Pet Tyrannosaurus - 地 | Crushing Jaws E | 主动（冷却16秒）
`effect: trauma, effectDuration: 3`
**伤害：** 29 （地） | **冷却时间：** 16秒
> *wiki原文 - Crushing Jaws E:* Causes trauma, preventing the target for 3s. Inflicts 29 Earth damage.

主动技能，冷却时间16秒。造成29点伤害，并给目标附加持续3秒的「创伤」状态。创伤本身不造成伤害，它只是一个纯粹的「禁止治疗」标记：游戏里所有的治疗来源（宠物自带回血、持续回血效果、天气治疗比如「舒缓迷雾」、生命汲取、护盾抵消伤害时顺带的回血、复活）在生效前都会检查目标身上有没有创伤，有的话就直接跳过这次治疗。根据wiki的小知识，创伤并不会阻止「复生/复活」类效果把一组宠物救回来，也不会阻止「护盾」抵消伤害本身，它只会挡住护盾抵消伤害时顺带的回血部分。

85. Brown Pet Tricerus Rex - 地 | Defensive Bite E | 主动（冷却18秒）
`effect: block, effectDuration: 2`
**伤害：** 22 （地） | **冷却时间：** 18秒
> *wiki原文 - Defensive Bite E:* Strike and then block attacks for 2s. Inflicts 22 Earth damage.

主动技能，冷却时间18秒。对敌方造成22点伤害，然后让施法者自己这组宠物获得持续2秒的「格挡」状态（代码里是给它设置`applyStatusEffect`，类型为`block`）。格挡生效期间，只要这一下攻击不是明确无视格挡的类型（也就是不带`anti_block`/`anti_dodge_block`标记，也不是能穿透格挡的「精准」系技能），`dealDamage`就会在伤害下限生效之前直接把这次伤害拦成0。

86. Brown Pet Pteranus Rex - 地 | Precision Crush E | 主动（冷却19秒）
`effect: anti_dodge_block_conditional, effectValue: 10`
**伤害：** 25 （地） | **冷却时间：** 19秒
> *wiki原文 - Precision Crush E:* If the opponent attempts to dodge or block, hits for 10% more damage. Inflicts 25 Earth damage. Can't be dodged or blocked.

主动技能，冷却时间19秒，属于「精准/暴击」这一系恐龙技能。造成25点伤害。在`dealDamage`里，这个技能会无条件把`ignoresDodge`和`ignoresBlock`都设成true，不管对方是闪避还是格挡状态，这一下都会实打实命中满伤害。需要注意：wiki上写的是「如果对方尝试闪避或格挡，则造成10%额外伤害」，但实际代码从来没读取过这个技能的`effectValue`字段，它只是单纯无视闪避和格挡，照面板上的固定伤害打，并没有额外加成。唯一的例外：如果对方的闪避恰好是来自某个带`passive_dodge`被动的宠物（比如Yeonnalligi的「翱翔」被动），穿透效果会被关掉，这一下还是会被正常闪避掉，这跟wiki里「翱翔」克制「精准/暴击」的小知识对得上。

87. Brown Pet Tripatosaurus - 地 | Defensive Bash E | 主动（冷却20秒）
`effect: block, effectDuration: 7`
**伤害：** 10 （地） | **冷却时间：** 20秒
> *wiki原文 - Defensive Bash E:* Strike and then block attacks for 7s. Inflicts 10 Earth damage

主动技能，冷却时间20秒。对敌方造成10点伤害，然后让施法者自己这组宠物获得持续7秒的「格挡」状态（代码里是给它设置`applyStatusEffect`，类型为`block`）。格挡生效期间，只要这一下攻击不是明确无视格挡的类型（也就是不带`anti_block`/`anti_dodge_block`标记，也不是能穿透格挡的「精准」系技能），`dealDamage`就会在伤害下限生效之前直接把这次伤害拦成0。

88. Brown Pet Tyranopatos - 地 | Grinding Jaws E | 主动（冷却20秒）
`effect: trauma, effectDuration: 30`
**伤害：** 3 （地） | **冷却时间：** 20秒
> *wiki原文 - Grinding Jaws E:* Causes Trauma, preventing the target from healing for 30s. Inflicts 3 Earth damage.

主动技能，冷却时间20秒。造成3点伤害，并给目标附加持续30秒的「创伤」状态。创伤本身不造成伤害，它只是一个纯粹的「禁止治疗」标记：游戏里所有的治疗来源（宠物自带回血、持续回血效果、天气治疗比如「舒缓迷雾」、生命汲取、护盾抵消伤害时顺带的回血、复活）在生效前都会检查目标身上有没有创伤，有的话就直接跳过这次治疗。根据wiki的小知识，创伤并不会阻止「复生/复活」类效果把一组宠物救回来，也不会阻止「护盾」抵消伤害本身，它只会挡住护盾抵消伤害时顺带的回血部分。

89. Pet Leprechaun - 地 | Get Lucky | 主动（冷却20秒）
`effect: buff_chance_double_damage, effectValue: 25, effectDuration: 10`
**冷却时间：** 20秒
> *wiki原文 - Get Lucky:* Grants 25% chance of hitting for double damage for 10s.

主动技能，冷却时间20秒。造成0点伤害，然后给施法者自己这组宠物加上一个`double_dmg`增益，持续10秒，期间每次攻击有25%概率触发。增益生效时，这组宠物之后每一次攻击在`dealDamage`结算时都会掷一次`doubleDmgChance`判定，一旦命中，那一下的伤害就翻倍。在增益消失前再次释放，只是把倒计时刷新回10秒（不同来源的增益各自独立叠加，不会互相覆盖）。

90. Pineapple Finger Ring - 地 | Sticky Pineapple Web | 主动（冷却20秒）
`effect: anti_swap, effectDuration: 5, dotDamage: 4, dotDuration: 5`
**伤害：** 4 （地） | **冷却时间：** 20秒 | **持续伤害：** 每秒4点，持续5秒
> *wiki原文 - Sticky Pineapple Web:* Prevent opponent from swapping out for 5s, dealing 4 Earth damage every second for 5s.

主动技能，冷却时间20秒，不直接造成伤害。给目标那组附加持续5秒的`no_swap`（禁止切换）负面效果（走`applyModifiers`）。这个效果生效期间，`recomputeModifiers`会把那组的`canSwap`设为false，切换按钮会被锁住（界面上显示`🔒 LOCKED`），`swapDeck()`在这段时间里也会拒绝执行，目标就算想跑也跑不掉，只能困在当前这组宠物里。

91. Pet Clover - 地 | Get Lucky! | 主动（冷却25秒）
`effect: buff_chance_double_damage, effectValue: 25, effectDuration: 10`
**冷却时间：** 25秒
> *wiki原文 - Get Lucky!:* Grants 25% chance of hitting double damage for 10s.

主动技能，冷却时间25秒。造成0点伤害，然后给施法者自己这组宠物加上一个`double_dmg`增益，持续10秒，期间每次攻击有25%概率触发。增益生效时，这组宠物之后每一次攻击在`dealDamage`结算时都会掷一次`doubleDmgChance`判定，一旦命中，那一下的伤害就翻倍。在增益消失前再次释放，只是把倒计时刷新回10秒（不同来源的增益各自独立叠加，不会互相覆盖）。

92. Dryad - 地 | Draining Roots | 主动（冷却30秒）
`effect: life_drain`
**伤害：** 30 （地） | **回血：** 30 | **冷却时间：** 30秒
> *wiki原文 - Draining Roots:* Drain life from the target, healing you for the same amount. Inflicts 30 Earth damage.

主动技能，冷却时间30秒。对敌方造成30点伤害，然后，只要这一下确实命中了（没被闪避/格挡/护盾抵消），并且施法者自己没有中创伤，就给施法者回血，回血量正好等于实际打出去的伤害（是减免之后的最终数值，而不是面板上写的固定30点）。

93. Wolf Tamer's Glove - 地 | Howl | 主动（冷却30秒）
`effect: buff_damage_dealt, effectValue: 25, effectDuration: 30`
**冷却时间：** 30秒
> *wiki原文 - Howl:* Boost your damage by 25% for 30s.

主动技能，冷却时间30秒。给施法者自己这组加一个`buff_damage`自增益：之后30秒内的所有攻击伤害+25%。如果这只宠物属于几只特殊的可叠加来源之一（草莓史莱姆、Leashed Silkworm - Purple、Pet Present Goblin），在增益还没到期前再放一次会额外叠一层（上限400%），持续时间也会取两者中较长的那个，而不是简单地刷新重置。

94. Growmoji's Little Partner - 地 | Adorable Eyes... and Claw | 主动（冷却40秒）
`effect: force_partner_forward`
**伤害：** 35 （地） | **冷却时间：** 40秒
> *wiki原文 - Adorable Eyes... and Claw:* Fool your foe with your adorableness scratching them harshly and bringing their partner forward. Inflicts 35 Earth Damage.

主动技能，冷却时间40秒。造成35点伤害，然后，根据wiki小知识澄清，「他们的搭档」指的是敌方的搭档，强制把敌方换到他们自己的备战组（前提是那一组还活着、能切换、没被冰冻），把那一组拉上场。这是强加在对手身上的强制切换，不是施法者自己主动换人。

95. Lil' Sheepers - 地 | Count Sheep | 主动（冷却40秒）
`effect: sleep, effectDuration: 8`
**冷却时间：** 40秒
> *wiki原文 - Count Sheep:* Weather Effect. Put everybody to sleep for 8s. Replaces any other active weather effect.

主动技能，冷却时间40秒（「数羊」），属于天气类技能。把全场天气设置成持续8秒的「睡眠」效果；此后每一跳，双方当前出战的两组都会被扣上「睡眠」状态（禁止行动，和眩晕/冰冻/跳舞是同一个列表），只要还没睡着，就会不断刷新成「剩余天气时间+1秒」。根据wiki的小知识，这个睡眠倒计时本身也会受到那一组身上的回复速度类效果影响：减速类负面效果（吐泥/泥团）会让那一组睡得比天气本身结束得还久，而加速类增益（姜汁冲击/糖分狂飙）则会让它提前醒过来，在天气结束前就能重新行动。

96. Muddy Pants - 地 | Mud Glob | 主动（冷却40秒）
`effect: slow_recharge, effectValue: 50, effectDuration: 10`
**伤害：** 20 （地） | **冷却时间：** 40秒
> *wiki原文 - Mud Glob:* For 10s, the target's abilities recharge half as fast. Inflicts 20 Earth damage.

主动技能，冷却时间40秒。造成20点伤害，然后给目标的出战组**以及**它替补的那一组同时附加持续10秒的`slow_cd`负面效果，让技能冷却回复速度减半（`speedMult *= 0.5`），同时那一组的眩晕/冰冻/睡眠等状态的倒计时也会变慢，根据wiki的小知识，这种「连替补组一起影响」的范围wiki文案里没写明，但代码里确实是这么实现的，而且不只影响技能冷却，控制类状态的持续时间也会一起变慢。

97. Leashed Silkworm - Green - 地 | Toxic Cloud (Weather Effect) | 主动（冷却60秒）
`effect: weather, effectValue: 4, effectDuration: 60, dotDamage: 4, dotDuration: 60`
**冷却时间：** 60秒 | **持续伤害：** 每秒4点，持续60秒
> *wiki原文 - Toxic Cloud (Weather Effect):* Spread poison over the whole battle, hurting everyone for 4 Earth damage every 5s, lasting 60s. Replaces any other active Weather Effect.

主动技能，冷却时间60秒（「毒云」）。把全场天气设置为持续60秒的效果：此后每5跳，双方当前出战的两组各承受4点固定的、无法减免的伤害。根据wiki的小知识，和「火焰风暴」一样，这种天气在到期前的最后一跳不会再触发伤害。

---

### 火系 (Fire)

98. Hotpants - 火 | Pepper | 主动（冷却3秒）
`effect: none`
**伤害：** 6 （火） | **冷却时间：** 3秒
> *wiki原文 - Pepper:* Pepper your foes with this rapid-fire attack. Inflicts 6 Fire damage.

主动技能，冷却时间3秒。单纯造成6点固定伤害，不附带任何额外效果，就是纯粹打一下。

99. Party Fowl - 火 | Party Foul | 主动（冷却5秒）
`effect: auto_attack_inactive`
**伤害：** 5 （火） | **冷却时间：** 5秒
> *wiki原文 - Party Foul:* Attack when inactive! 5 Fire damage attack is used automatically when inactive.

属于「类被动」效果（数据库里`is_passive`标的是false，所以它同时也有自己能主动释放的技能），但除此之外还额外附带一条：只要这只宠物所在的这组当前不是出战组，不管是因为同组的队友在出战，还是整组都在替补，它就会每5秒自动啄敌方一下，造成5点固定伤害（判断条件是`battle.tick % cooldown === 0`），这和它自己技能的冷却完全独立、互不影响。

100. Red Pet Pteranodon - 火 | Dino Dive F | 主动（冷却5秒）
`effect: anti_dodge_block_conditional, effectValue: 25`
**伤害：** 7 （火） | **冷却时间：** 5秒
> *wiki原文 - Dino Dive F:* If the opponent attempts to dodge or block, hits for 25% more damage. Inflicts 7 Fire damage. Can't be dodged or blocked.

主动技能，冷却时间5秒，属于「精准/暴击」这一系恐龙技能。造成7点伤害。在`dealDamage`里，这个技能会无条件把`ignoresDodge`和`ignoresBlock`都设成true，不管对方是闪避还是格挡状态，这一下都会实打实命中满伤害。需要注意：wiki上写的是「如果对方尝试闪避或格挡，则造成25%额外伤害」，但实际代码从来没读取过这个技能的`effectValue`字段，它只是单纯无视闪避和格挡，照面板上的固定伤害打，并没有额外加成。唯一的例外：如果对方的闪避恰好是来自某个带`passive_dodge`被动的宠物（比如Yeonnalligi的「翱翔」被动），穿透效果会被关掉，这一下还是会被正常闪避掉，这跟wiki里「翱翔」克制「精准/暴击」的小知识对得上。

101. Dragon Hand - 火 | Fire Breath | 主动（冷却6秒）
`effect: dot, effectValue: 3, effectDuration: 4, dotDamage: 3, dotDuration: 4`
**伤害：** 3 （火） | **冷却时间：** 6秒 | **持续伤害：** 每秒3点，持续4秒
> *wiki原文 - Fire Breath:* Ignite the target for 3 Fire damage plus 3 more per second for 4 seconds. Inflicts 3 Fire damage.

主动技能，冷却时间6秒。给目标附加一个持续伤害状态，每一跳造成3点伤害，持续4秒（和其他负面效果一样，会受到passive_resist_negative抵抗判定和passive_shorten_debuff持续时间缩减的影响）。根据wiki的小知识，持续伤害（和所有限时效果一样）不会在倒计时的最后一秒再跳一次，所以实际造成的总伤害会比直接拿「3 × 4」算出来的数字少一跳。

102. Phlogiston - 火 | Toasties | 主动（冷却6秒）
`effect: summon, dotDamage: 2, dotDuration: 3`
**伤害：** 2 （火） | **冷却时间：** 6秒 | **持续伤害：** 每秒2点，持续3秒
> *wiki原文 - Toasties:* Summons a toasty friend who inflicts 2 Fire damage every 3s.

主动技能，冷却时间6秒（「小烤饼」）。造成2点完全绕过`dealDamage`的伤害，直接从目标血量里扣，所以属性克制、紫雾、闪避/格挡、灵体，以及游戏里所有的伤害增减益效果对它都不起作用（正好对应wiki小知识里说的「小烤饼无视一切伤害修正效果」）。如果这只宠物还带有持续伤害的部分（`dotDuration > 0`），还会额外附加一个每秒2点、持续3秒的燃烧效果，而这部分是正常的持续伤害，每一跳都会照常受各种加减益影响。

103. Red Pet Apatodon - 火 | Precision Strike F | 主动（冷却6秒）
`effect: elemental_bonus, effectValue: 50`
**伤害：** 10 （火） | **冷却时间：** 6秒
> *wiki原文 - Precision Strike F:* Against an Earth opponent, inflicts +50% damage (on top of normal bonus.) Inflicts 10 Fire damage.

主动技能，冷却时间6秒，属于「恐龙冲撞/突袭/俯冲/撕咬」这一系技能。造成10点火系基础伤害。结算这次攻击之前，代码会先判断这只宠物的属性是否克制对方那组宠物的主属性（通过`ELEMENT_BEATS`表）；如果克制，伤害会先乘以`1 + 50/100`（也就是+50%），再交给`dealDamage`处理，而`dealDamage`自己还会照常再叠加一次±25%的属性克制加成，所以属性有利时两个加成是叠在一起生效的。如果属性不占优，这只宠物就只老老实实打出10点固定伤害，没有任何加成。

104. Lantern's ChronoReaper - 火 | To the Shadow Realm! | 主动（冷却7秒）
`effect: banish, effectDuration: 10`
**伤害：** 10 （火） | **冷却时间：** 7秒
> *wiki原文 - To the Shadow Realm!:* Open a portal dealing 10 Fire Damage to the enemy banishing them for 10s!

主动技能，冷却时间7秒。造成10点伤害，然后，如果敌方还有一组活着的备战宠物，强行把敌方换到那一组去（同时清掉离场那一组身上的厄运），并给它们换上来的那一组挂上持续10秒的`no_swap`负面效果，锁死`canSwap = false`，让敌方在这段时间里没法再换回去。如果敌方没有活着的备战组可以被推过去，这招就会打空、不会触发强制切换（但直接的伤害部分依然照常命中）。

105. Pet Fox - 火 | Desperate Bite | 主动（冷却7秒）
`effect: desperate`
**伤害：** 10 （火） | **冷却时间：** 7秒
> *wiki原文 - Desperate Bite:* Inflicts 1% more damage for every 1% of life you are missing. Deals 10 Fire damage.

主动技能，冷却时间7秒（「垂死一击」，Pet Fox专属）。伤害会根据施法者自己这组缺了多少血来浮动：代码算出`缺血比例 = 1 - 当前血量/最大血量`，然后打出`10 × (1 + 缺血比例)`点伤害，满血时就是老老实实的10点，血量见底时大概能翻到接近两倍。这只宠物的具体数值wiki没给出确切的浮动公式（数据库里`effectValue`是0），所以这条曲线是模拟器自己给的一个合理默认值，并不是wiki确认过的官方数字。

106. Playful Fire Sprite - 火 | Playful Fire | 主动（冷却8秒）
`effect: buff_chance_double_damage, effectValue: 10, effectDuration: 5`
**伤害：** 15 （火） | **冷却时间：** 8秒
> *wiki原文 - Playful Fire:* Grants you a 10% chance to do double damage for 5 seconds. Inflicts 15 Fire damage.

主动技能，冷却时间8秒。造成15点伤害，然后给施法者自己这组宠物加上一个`double_dmg`增益，持续5秒，期间每次攻击有10%概率触发。增益生效时，这组宠物之后每一次攻击在`dealDamage`结算时都会掷一次`doubleDmgChance`判定，一旦命中，那一下的伤害就翻倍。在增益消失前再次释放，只是把倒计时刷新回5秒（不同来源的增益各自独立叠加，不会互相覆盖）。

107. Red Pet Apatoceratops - 火 | Dino Charge F | 主动（冷却8秒）
`effect: elemental_bonus, effectValue: 60`
**伤害：** 14 （火） | **冷却时间：** 8秒
> *wiki原文 - Dino Charge F:* Against an Earth opponent, inflicts +60% damage (on top of normal bonus). Inflicts 14 Fire damage.

主动技能，冷却时间8秒，属于「恐龙冲撞/突袭/俯冲/撕咬」这一系技能。造成14点火系基础伤害。结算这次攻击之前，代码会先判断这只宠物的属性是否克制对方那组宠物的主属性（通过`ELEMENT_BEATS`表）；如果克制，伤害会先乘以`1 + 60/100`（也就是+60%），再交给`dealDamage`处理，而`dealDamage`自己还会照常再叠加一次±25%的属性克制加成，所以属性有利时两个加成是叠在一起生效的。如果属性不占优，这只宠物就只老老实实打出14点固定伤害，没有任何加成。

108. Red Pet Pteratops - 火 | Precision Attack F | 主动（冷却9秒）
`effect: anti_dodge_block_conditional, effectValue: 17`
**伤害：** 11 （火） | **冷却时间：** 9秒
> *wiki原文 - Precision Attack F:* If the opponent attempts to dodge or block, hits for 17% more damage. Inflicts 11 Fire damage. Can't be dodged or blocked.

主动技能，冷却时间9秒，属于「精准/暴击」这一系恐龙技能。造成11点伤害。在`dealDamage`里，这个技能会无条件把`ignoresDodge`和`ignoresBlock`都设成true，不管对方是闪避还是格挡状态，这一下都会实打实命中满伤害。需要注意：wiki上写的是「如果对方尝试闪避或格挡，则造成17%额外伤害」，但实际代码从来没读取过这个技能的`effectValue`字段，它只是单纯无视闪避和格挡，照面板上的固定伤害打，并没有额外加成。唯一的例外：如果对方的闪避恰好是来自某个带`passive_dodge`被动的宠物（比如Yeonnalligi的「翱翔」被动），穿透效果会被关掉，这一下还是会被正常闪避掉，这跟wiki里「翱翔」克制「精准/暴击」的小知识对得上。

109. Fiesta Dragon - 火 | Party Breath | 主动（冷却10秒）
`effect: hit_both`
**伤害：** 10 （火） | **冷却时间：** 10秒
> *wiki原文 - Party Breath:* Attack, hitting both enemy pets at once. Inflicts 10 Fire damage to each one.

主动技能，冷却时间10秒。基础的10点伤害先按普通流程打在敌方出战那一组身上；如果敌方的备战组也还活着，这只宠物会紧接着把同样的10点伤害再直接打一次到那个备战组身上，一次施法同时命中敌方两组宠物。

110. Red Pet Apatos Rex - 火 | Dino Chomp F | 主动（冷却10秒）
`effect: elemental_bonus, effectValue: 25`
**伤害：** 18 （火） | **冷却时间：** 10秒
> *wiki原文 - Dino Chomp F:* Against an Earth opponent, inflicts +25% damage (on top of normal bonus). Inflicts 18 Fire damage.

主动技能，冷却时间10秒，属于「恐龙冲撞/突袭/俯冲/撕咬」这一系技能。造成18点火系基础伤害。结算这次攻击之前，代码会先判断这只宠物的属性是否克制对方那组宠物的主属性（通过`ELEMENT_BEATS`表）；如果克制，伤害会先乘以`1 + 25/100`（也就是+25%），再交给`dealDamage`处理，而`dealDamage`自己还会照常再叠加一次±25%的属性克制加成，所以属性有利时两个加成是叠在一起生效的。如果属性不占优，这只宠物就只老老实实打出18点固定伤害，没有任何加成。

111. Red Pet Apatosaurus - 火 | Dino Slam F | 主动（冷却10秒）
`effect: elemental_bonus, effectValue: 100`
**伤害：** 15 （火） | **冷却时间：** 10秒
> *wiki原文 - Dino Slam F:* Against an Earth opponent, inflicts +100% damage (on top of normal bonus). Inflicts 15 Fire damage.

主动技能，冷却时间10秒，属于「恐龙冲撞/突袭/俯冲/撕咬」这一系技能。造成15点火系基础伤害。结算这次攻击之前，代码会先判断这只宠物的属性是否克制对方那组宠物的主属性（通过`ELEMENT_BEATS`表）；如果克制，伤害会先乘以`1 + 100/100`（也就是+100%），再交给`dealDamage`处理，而`dealDamage`自己还会照常再叠加一次±25%的属性克制加成，所以属性有利时两个加成是叠在一起生效的。如果属性不占优，这只宠物就只老老实实打出15点固定伤害，没有任何加成。

112. Red Pet Triceradon - 火 | Defensive Flurry F | 主动（冷却10秒）
`effect: block, effectDuration: 2`
**伤害：** 12 （火） | **冷却时间：** 10秒
> *wiki原文 - Defensive Flurry F:* Strike and then block attacks for 2s. Inflicts 12 Fire damage.

主动技能，冷却时间10秒。对敌方造成12点伤害，然后让施法者自己这组宠物获得持续2秒的「格挡」状态（代码里是给它设置`applyStatusEffect`，类型为`block`）。格挡生效期间，只要这一下攻击不是明确无视格挡的类型（也就是不带`anti_block`/`anti_dodge_block`标记，也不是能穿透格挡的「精准」系技能），`dealDamage`就会在伤害下限生效之前直接把这次伤害拦成0。

113. Red Pet Tyranodon - 火 | Piercing Jaws F | 主动（冷却10秒）
`effect: trauma, effectDuration: 4`
**伤害：** 15 （火） | **冷却时间：** 10秒
> *wiki原文 - Piercing Jaws F:* Causes trauma, preventing the target from healing for 4s. Inflicts 15 Fire damage.

主动技能，冷却时间10秒。造成15点伤害，并给目标附加持续4秒的「创伤」状态。创伤本身不造成伤害，它只是一个纯粹的「禁止治疗」标记：游戏里所有的治疗来源（宠物自带回血、持续回血效果、天气治疗比如「舒缓迷雾」、生命汲取、护盾抵消伤害时顺带的回血、复活）在生效前都会检查目标身上有没有创伤，有的话就直接跳过这次治疗。根据wiki的小知识，创伤并不会阻止「复生/复活」类效果把一组宠物救回来，也不会阻止「护盾」抵消伤害本身，它只会挡住护盾抵消伤害时顺带的回血部分。

114. Skeletal Dragon Claw - 火 | Deathfire | 主动（冷却10秒）
`effect: life_drain`
**伤害：** 10 （火） | **回血：** 10 | **冷却时间：** 10秒
> *wiki原文 - Deathfire:* Drain life from the target, healing you for the same amount. Deals 10 Fire damage.

主动技能，冷却时间10秒。对敌方造成10点伤害，然后，只要这一下确实命中了（没被闪避/格挡/护盾抵消），并且施法者自己没有中创伤，就给施法者回血，回血量正好等于实际打出去的伤害（是减免之后的最终数值，而不是面板上写的固定10点）。

115. Toy Lock-Bot - 火 | Lockdown | 主动（冷却10秒）
`effect: anti_swap, effectDuration: 4`
**伤害：** 17 （火） | **冷却时间：** 10秒
> *wiki原文 - Lockdown:* Smack the target with a lock, keeping them from swapping out for 4s. Deals 17 Fire damage.

主动技能，冷却时间10秒，不直接造成伤害。给目标那组附加持续4秒的`no_swap`（禁止切换）负面效果（走`applyModifiers`）。这个效果生效期间，`recomputeModifiers`会把那组的`canSwap`设为false，切换按钮会被锁住（界面上显示`🔒 LOCKED`），`swapDeck()`在这段时间里也会拒绝执行，目标就算想跑也跑不掉，只能困在当前这组宠物里。

116. Magic Reindeer Bell - 火 | Glowing Nose | 主动（冷却12秒）
`effect: dot, effectValue: 2, effectDuration: 10, dotDamage: 2, dotDuration: 10`
**伤害：** 10 （火） | **冷却时间：** 12秒 | **持续伤害：** 每秒2点，持续10秒
> *wiki原文 - Glowing Nose:* Ignite them with rays from your nose, doing 2 Fire damage per second for 10s. inflicts 10 Fire damage.

主动技能，冷却时间12秒。给目标附加一个持续伤害状态，每一跳造成2点伤害，持续10秒（和其他负面效果一样，会受到passive_resist_negative抵抗判定和passive_shorten_debuff持续时间缩减的影响）。根据wiki的小知识，持续伤害（和所有限时效果一样）不会在倒计时的最后一秒再跳一次，所以实际造成的总伤害会比直接拿「2 × 10」算出来的数字少一跳。

117. Leashed Silkworm - Red - 火 | Death Ray | 主动（冷却13秒）
`effect: chain_on_kill`
**伤害：** 24 （火） | **冷却时间：** 13秒
> *wiki原文 - Death Ray:* If this beat destroys its target, it also hits the target's partner. Inflicts 24 Fire damage.

主动技能，冷却时间13秒（「死亡射线」）。基础的24点伤害先按普通流程打在敌方出战那一组身上；如果这一下正好把目标那一组的血量打到0或以下，这只宠物会立刻把同样的24点伤害再打向敌方另一组（备战组），前提是那一组还活着。如果第一下没能打死出战那一组，就不会有连锁，这纯粹是个「终结奖励」，而不是保证触发的第二下。

118. Mechasaurus Rex - 火 | Ratatata! | 主动（冷却13秒）
`effect: none`
**伤害：** 30 （火） | **冷却时间：** 13秒
> *wiki原文 - Ratatata!:* Fires several shots dealing 30 Fire Damage! Inflicts 30 Fire damage.

主动技能，冷却时间13秒。单纯造成30点固定伤害，不附带任何额外效果，就是纯粹打一下。

119. Radioactive Synthoid - 火 | Radiation Beam | 主动（冷却13秒）
`effect: consume_buff`
**伤害：** 30 （火） | **冷却时间：** 13秒
> *wiki原文 - Radiation Beam:* Fizzles unless you have a positive effect on you. Consumes the effect to fire a beam. Inflicts 30 Fire damage.

主动技能，冷却时间13秒（「辐射光束」/「花粉轰炸」系技能）。在结算伤害之前，会先告诉`dealDamage`这一下要无视施法者自己身上的增益（`isConsumeBuff`），结算完之后还会直接删掉施法者这组身上最早加上的那个增益。总的效果就是：这一下攻击完全吃不到施法者自己的伤害增益（比如「怒吼」「幸运加持」之类都会被跳过），而且还要搭上一个增益当作释放代价，这正好对应wiki里说的「伤害增益不生效是因为被这招消耗掉了」的小知识。

120. Red Pet Triceratops - 火 | Defensive Gore F | 主动（冷却13秒）
`effect: block, effectDuration: 4`
**伤害：** 16 （火） | **冷却时间：** 13秒
> *wiki原文 - Defensive Gore F:* Strike and then block attacks for 4s. Inflicts 16 Fire damage.

主动技能，冷却时间13秒。对敌方造成16点伤害，然后让施法者自己这组宠物获得持续4秒的「格挡」状态（代码里是给它设置`applyStatusEffect`，类型为`block`）。格挡生效期间，只要这一下攻击不是明确无视格挡的类型（也就是不带`anti_block`/`anti_dodge_block`标记，也不是能穿透格挡的「精准」系技能），`dealDamage`就会在伤害下限生效之前直接把这次伤害拦成0。

121. Red Pet Tyranotops - 火 | Crushing Beak F | 主动（冷却13秒）
`effect: trauma, effectDuration: 7`
**伤害：** 19 （火） | **冷却时间：** 13秒
> *wiki原文 - Crushing Beak F:* Causes Trauma, preventing the target from healing for 7 seconds. Inflicts 19 Fire damage.

主动技能，冷却时间13秒。造成19点伤害，并给目标附加持续7秒的「创伤」状态。创伤本身不造成伤害，它只是一个纯粹的「禁止治疗」标记：游戏里所有的治疗来源（宠物自带回血、持续回血效果、天气治疗比如「舒缓迷雾」、生命汲取、护盾抵消伤害时顺带的回血、复活）在生效前都会检查目标身上有没有创伤，有的话就直接跳过这次治疗。根据wiki的小知识，创伤并不会阻止「复生/复活」类效果把一组宠物救回来，也不会阻止「护盾」抵消伤害本身，它只会挡住护盾抵消伤害时顺带的回血部分。

122. Demon Control Cap - 火 | Flaming Tongue | 主动（冷却15秒）
`effect: self_dot, effectValue: 5, effectDuration: 7, dotDamage: 5, dotDuration: 7`
**回血：** 40 | **冷却时间：** 15秒 | **持续伤害：** 每秒5点，持续7秒
> *wiki原文 - Flaming Tongue:* Lick your wounds, but then you burn 5 Fire damage per second for 7s. Heal 40 life.

主动技能，冷却时间15秒（「炽热之舌」）。虽然数据库里这招被标记成`isDot: true`（所以走的是普通持续伤害的计时机制），但灼烧其实是加在施法者自己身上，而不是敌人身上：施法者这组每秒承受5点伤害，持续7秒。这个特殊判断会在普通持续伤害分支之前就被处理掉，专门是为了避免被误判成打在目标身上的伤害。

123. Leashed Silkworm - Black - 火 | Absorb | 主动（冷却15秒）
`effect: absorb, effectDuration: 2`
**冷却时间：** 15秒
> *wiki原文 - Absorb:* Absorb incoming energy for 2s - any attacks that would hurt you will heal you instead.

主动技能，冷却时间15秒，不直接造成伤害。给施法者自己这组加上持续2秒的「护盾」状态。护盾在场期间，这组接下来要挨的那一下（或几下）攻击会在`dealDamage`里被完全抵消（伤害归零），并且，只要这组身上没有创伤，还会按原本会造成的伤害数值原样回血。如果身上有创伤，伤害依然会被抵消，但回血部分会被挡掉。根据wiki的小知识，这个技能和另一个护盾类技能算是同一个「技能类别」：在一个还没到期前再放另一个，会延长剩余时间而不是重置；其中一个版本用的施法动画固定是2秒，即便它实际的护盾时长更长，所以光看动画是看不出剩余时间的。

124. Lion Taming Whip - 火 | Claws Out | 主动（冷却15秒）
`effect: counter, effectDuration: 4`
**伤害：** 35 （火） | **冷却时间：** 15秒
> *wiki原文 - Claws Out:* Strike back for 35 Fire damage if hit within 4s.

主动技能，冷却时间15秒，释放时不直接造成伤害。给施法者这组宠物挂上一个数值为35的`counter_reflect`增益（用这只宠物面板上的`damage`字段当反弹伤害数值），持续4秒。增益生效期间，只要这组宠物挨了一下实打实命中的攻击（不管第几次），`dealDamage`都会额外把这35点固定伤害反弹回攻击者那一组，这是在施法者自己受到的伤害之外另外算的。重新释放会直接替换掉之前的反击增益，不会叠加。

125. Red Pet Pterosaurus - 火 | Critical Strike F | 主动（冷却15秒）
`effect: anti_dodge_block_conditional, effectValue: 50`
**伤害：** 17 （火） | **冷却时间：** 15秒
> *wiki原文 - Critical Strike F:* If the opponent attempts to dodge or block, hits for 50% more damage. Inflicts 17 Fire damage. Can't be dodged or blocked.

主动技能，冷却时间15秒，属于「精准/暴击」这一系恐龙技能。造成17点伤害。在`dealDamage`里，这个技能会无条件把`ignoresDodge`和`ignoresBlock`都设成true，不管对方是闪避还是格挡状态，这一下都会实打实命中满伤害。需要注意：wiki上写的是「如果对方尝试闪避或格挡，则造成50%额外伤害」，但实际代码从来没读取过这个技能的`effectValue`字段，它只是单纯无视闪避和格挡，照面板上的固定伤害打，并没有额外加成。唯一的例外：如果对方的闪避恰好是来自某个带`passive_dodge`被动的宠物（比如Yeonnalligi的「翱翔」被动），穿透效果会被关掉，这一下还是会被正常闪避掉，这跟wiki里「翱翔」克制「精准/暴击」的小知识对得上。

126. Red Pet Tyrannosaurus - 火 | Crushing Jaws F | 主动（冷却16秒）
`effect: trauma, effectDuration: 3`
**伤害：** 29 （火） | **冷却时间：** 16秒
> *wiki原文 - Crushing Jaws F:* Causes trauma, preventing the target for 3s. Inflicts 29 Fire damage.

主动技能，冷却时间16秒。造成29点伤害，并给目标附加持续3秒的「创伤」状态。创伤本身不造成伤害，它只是一个纯粹的「禁止治疗」标记：游戏里所有的治疗来源（宠物自带回血、持续回血效果、天气治疗比如「舒缓迷雾」、生命汲取、护盾抵消伤害时顺带的回血、复活）在生效前都会检查目标身上有没有创伤，有的话就直接跳过这次治疗。根据wiki的小知识，创伤并不会阻止「复生/复活」类效果把一组宠物救回来，也不会阻止「护盾」抵消伤害本身，它只会挡住护盾抵消伤害时顺带的回血部分。

127. Red Pet Tricerus Rex - 火 | Defensive Bite F | 主动（冷却18秒）
`effect: block, effectDuration: 2`
**伤害：** 22 （火） | **冷却时间：** 18秒
> *wiki原文 - Defensive Bite F:* Strike and then block attacks for 2s. Inflicts 22 Fire damage.

主动技能，冷却时间18秒。对敌方造成22点伤害，然后让施法者自己这组宠物获得持续2秒的「格挡」状态（代码里是给它设置`applyStatusEffect`，类型为`block`）。格挡生效期间，只要这一下攻击不是明确无视格挡的类型（也就是不带`anti_block`/`anti_dodge_block`标记，也不是能穿透格挡的「精准」系技能），`dealDamage`就会在伤害下限生效之前直接把这次伤害拦成0。

128. Red Pet Pteranus Rex - 火 | Precision Crush F | 主动（冷却19秒）
`effect: anti_dodge_block_conditional, effectValue: 10`
**伤害：** 25 （火） | **冷却时间：** 19秒
> *wiki原文 - Precision Crush F:* If the opponent attempts to dodge or block, hits for 10% more damage. Inflicts 25 Fire damage. Can't be dodged or blocked.

主动技能，冷却时间19秒，属于「精准/暴击」这一系恐龙技能。造成25点伤害。在`dealDamage`里，这个技能会无条件把`ignoresDodge`和`ignoresBlock`都设成true，不管对方是闪避还是格挡状态，这一下都会实打实命中满伤害。需要注意：wiki上写的是「如果对方尝试闪避或格挡，则造成10%额外伤害」，但实际代码从来没读取过这个技能的`effectValue`字段，它只是单纯无视闪避和格挡，照面板上的固定伤害打，并没有额外加成。唯一的例外：如果对方的闪避恰好是来自某个带`passive_dodge`被动的宠物（比如Yeonnalligi的「翱翔」被动），穿透效果会被关掉，这一下还是会被正常闪避掉，这跟wiki里「翱翔」克制「精准/暴击」的小知识对得上。

129. Battle Mutant Fish - 火 | Acid Spit | 主动（冷却20秒）
`effect: dot, effectValue: 10, effectDuration: 5, dotDamage: 10, dotDuration: 5`
**冷却时间：** 20秒 | **持续伤害：** 每秒10点，持续5秒
> *wiki原文 - Acid Spit:* Spray acid that burns the target for 10 Fire damage per second lasting for 5 seconds.

主动技能，冷却时间20秒。给目标附加一个持续伤害状态，每一跳造成10点伤害，持续5秒（和其他负面效果一样，会受到passive_resist_negative抵抗判定和passive_shorten_debuff持续时间缩减的影响）。根据wiki的小知识，持续伤害（和所有限时效果一样）不会在倒计时的最后一秒再跳一次，所以实际造成的总伤害会比直接拿「10 × 5」算出来的数字少一跳。

130. Leashed Silkworm - Yellow - 火 | Liquify | 主动（冷却20秒）
`effect: debuff_damage_taken, effectValue: 25, effectDuration: 5`
**伤害：** 15 （火） | **冷却时间：** 20秒
> *wiki原文 - Liquify:* Melt down the enemy so they take 25% more damage for 5s. Inflicts 15 Fire Damage.

主动技能，冷却时间20秒。造成15点伤害，然后给目标附加持续5秒的`dmg_taken`负面效果：这段时间里目标承受的所有伤害+25%。如果是由指定的几只可叠加宠物重复释放，会按来源分别叠加而不是直接覆盖重置，上限是+200%。

131. Nightmare Magnifying Glass - 火 | Nightmares | 主动（冷却20秒）
`effect: mess_up, effectValue: 50, effectDuration: 6`
**伤害：** 10 （火） | **冷却时间：** 20秒
> *wiki原文 - Nightmares:* Terrify the target into messing up their skills 50% of the time for 6s. Inflicts 10 Fire damage.

主动技能，冷却时间20秒。造成10点伤害，然后给目标附加持续6秒、触发概率50%的`mess_up`（口误）负面效果。这个效果生效期间，中招的那一组每次想放技能，都会先掷一次这个概率判定，一旦判定失败技能就直接打空（`useAbility`会在其他任何流程之前先掷这个判定），技能照样进入冷却，就跟正常放了一样，但原本该有的伤害/治疗/效果统统不会发生。根据wiki的小知识，天气类技能和Swoop自带的闪避，即便施法被口误判定搞砸了，依然会照常触发，这是两个明确写死的例外。

132. Red Pet Tripatosaurus - 火 | Defensive Bash F | 主动（冷却20秒）
`effect: block, effectDuration: 7`
**伤害：** 10 （火） | **冷却时间：** 20秒
> *wiki原文 - Defensive Bash F:* Strike and then block attacks for 7s. Inflicts 10 Fire damage

主动技能，冷却时间20秒。对敌方造成10点伤害，然后让施法者自己这组宠物获得持续7秒的「格挡」状态（代码里是给它设置`applyStatusEffect`，类型为`block`）。格挡生效期间，只要这一下攻击不是明确无视格挡的类型（也就是不带`anti_block`/`anti_dodge_block`标记，也不是能穿透格挡的「精准」系技能），`dealDamage`就会在伤害下限生效之前直接把这次伤害拦成0。

133. Red Pet Tyranopatos - 火 | Grinding Jaws F | 主动（冷却20秒）
`effect: trauma, effectDuration: 30`
**伤害：** 3 （火） | **冷却时间：** 20秒
> *wiki原文 - Grinding Jaws F:* Causes Trauma, preventing the target from healing for 30s. Inflicts 3 Fire damage.

主动技能，冷却时间20秒。造成3点伤害，并给目标附加持续30秒的「创伤」状态。创伤本身不造成伤害，它只是一个纯粹的「禁止治疗」标记：游戏里所有的治疗来源（宠物自带回血、持续回血效果、天气治疗比如「舒缓迷雾」、生命汲取、护盾抵消伤害时顺带的回血、复活）在生效前都会检查目标身上有没有创伤，有的话就直接跳过这次治疗。根据wiki的小知识，创伤并不会阻止「复生/复活」类效果把一组宠物救回来，也不会阻止「护盾」抵消伤害本身，它只会挡住护盾抵消伤害时顺带的回血部分。

134. Retro Magnifying Glass - 火 | Disco Fever | 主动（冷却20秒）
`effect: force_dance, effectDuration: 4, dotDamage: 5, dotDuration: 4`
**伤害：** 5 （火） | **冷却时间：** 20秒 | **持续伤害：** 每秒5点，持续4秒
> *wiki原文 - Disco Fever:* Infect the target to dance for 4s, unable to do anything other than swap out, and suffering 5 Fire damage per second.

主动技能，冷却时间20秒（「魅惑」/「迪斯科热潮」/「劲舞」系技能）。造成5点伤害（如果有的话），并让目标进入4秒的「跳舞」状态，期间`canAct`被设为false，目标完全没法用任何技能，不过依然能正常切换（「跳舞」不在锁定切换的状态列表里）。如果具体是「迪斯科热潮」这一招，还会额外给目标附加同样时长的「闪避」状态，根据wiki的小知识，这就是「跳舞」类负面效果会顺带给受害者一段闪避窗口的原因，而不是给施法者自己。

135. Royal Captain Bark - 火 | Supercharge | 主动（冷却20秒）
`effect: buff_next_damage, effectValue: 80`
**冷却时间：** 20秒
> *wiki原文 - Supercharge:* Charge up your next skill with +80% Damage.

主动技能，冷却时间20秒，不直接造成伤害。给施法者自己这组加上一个+80%的`next_dmg`增益，这个增益不会因为倒计时而自然消失（内部设定的持续时间是99秒），而是在施法者这组打出下一次命中的瞬间被消耗掉：`dealDamage`结算完加成之后马上把这个增益摘掉，所以不管你等多久才用出下一击，它都只会作用在那一次攻击上。

136. Spiritfire Mask - 火 | Spirit Swap | 主动（冷却20秒）
`effect: force_swap_transfer`
**伤害：** 10 （火） | **冷却时间：** 20秒
> *wiki原文 - Spirit Swap:* Force opponent to swap and transfer all status effects to the new enemy. Inflicts 10 Fire damage.

主动技能，冷却时间20秒（「灵魂互换」）。造成10点伤害，然后，如果能强制让敌方切换，会先把敌方出战那一组身上当前所有的状态效果和增减益，整体转移到即将被换上来的那一组身上（出战那一组的状态全部清空），转移完成之后才真正执行强制切换。相当于敌方等于是把身上一堆麻烦全都拖到了备战组身上，而不是甩在原地。

137. Pinata Pal - 火 | Surprise! | 主动（冷却22秒）
`effect: trap_swap, effectDuration: 4`
**伤害：** 25 （火） | **冷却时间：** 22秒
> *wiki原文 - Surprise!:* Prepare a surprise: If the opponent swaps out within 4s, hit both enemies for 25 Fire damage. Isn't visible to your enemy.

主动技能，冷却时间22秒。在敌方身上布下一个看不见的陷阱，根据wiki的小知识，这个陷阱本身不会显示给敌方看（不过施法动画能看到，所以对面能猜到你放了什么），持续窗口是4秒。如果陷阱布下期间敌方主动切换，就会触发，额外造成25点伤害；如果具体是那只技能名叫「Surprise!」的宠物放的，触发时会同时打中敌方两组宠物，而不只是换上场的那一组。

138. Captain Bark - 火 | Supercharge | 主动（冷却30秒）
`effect: buff_next_damage, effectValue: 50`
**冷却时间：** 30秒
> *wiki原文 - Supercharge:* Charge up your next skill with +50% Damage.

主动技能，冷却时间30秒，不直接造成伤害。给施法者自己这组加上一个+50%的`next_dmg`增益，这个增益不会因为倒计时而自然消失（内部设定的持续时间是99秒），而是在施法者这组打出下一次命中的瞬间被消耗掉：`dealDamage`结算完加成之后马上把这个增益摘掉，所以不管你等多久才用出下一击，它都只会作用在那一次攻击上。

139. Gingerbread Man - 火 | Ginger Blast | 主动（冷却30秒）
`effect: buff_damage_and_speed, effectValue: 25, effectDuration: 5`
**冷却时间：** 30秒
> *wiki原文 - Ginger Blast:* Deals 25% more damage. Cooldowns go twice as fast.

主动技能，冷却时间30秒，不直接造成伤害（「姜汁冲击」/「糖分狂飙」系技能）。给施法者自己这组以及它替补的那一组同时加上持续5秒的`dmg_speed`增益：伤害+25%，同时冷却回复速度翻倍（`speedMult *= 2`），这个加速对每只宠物的技能冷却、以及那一组身上眩晕/冰冻/睡眠的剩余时间都有效，根据wiki的小知识，这个增益会连带影响替补那一组，尽管技能文案里没有明说。

140. Maidmare - 火 | Slime Kiss | 主动（冷却30秒）
`effect: mess_up, effectValue: 100, effectDuration: 10`
**冷却时间：** 30秒
> *wiki原文 - Slime Kiss:* Terrify their target into messing up their skills 100% of the time for 10s.

主动技能，冷却时间30秒。不造成伤害，转而给目标附加持续10秒、触发概率100%的`mess_up`（口误）负面效果。这个效果生效期间，中招的那一组每次想放技能，都会先掷一次这个概率判定，一旦判定失败技能就直接打空（`useAbility`会在其他任何流程之前先掷这个判定），技能照样进入冷却，就跟正常放了一样，但原本该有的伤害/治疗/效果统统不会发生。根据wiki的小知识，天气类技能和Swoop自带的闪避，即便施法被口误判定搞砸了，依然会照常触发，这是两个明确写死的例外。

141. Violet Protodrake Leash - 火 | Purple Haze | 主动（冷却30秒）
`effect: weather, effectValue: 25, effectDuration: 10`
**冷却时间：** 30秒
> *wiki原文 - Purple Haze:* Weather Effect. Cover the world in burning haze for 10s, increasing all Fire damage by 25%, and reducing all other damage by 25%. Replaces any other active Weather Effect.

主动技能，冷却时间30秒（「紫雾」）。把全场天气设置成持续10秒、本身不直接造成伤害的效果。生效期间，`dealDamage`里结算的每一下攻击都会按攻击方的属性来调整：火属性攻击者伤害+25%，其他所有属性一律-25%，这个调整发生在闪避/格挡/灵体判定之前，并且会和正常的属性克制加成叠加。根据wiki的小知识，「紫雾」是唯一一个被明确排除在「小烤饼无视一切伤害修正」规则之外的天气效果，小烤饼的伤害依然会被紫雾调整。

142. Dragon of Legend - 火 | BURNINATE! | 主动（冷却40秒）
`effect: anti_dodge_block`
**伤害：** 40 （火） | **冷却时间：** 40秒
> *wiki原文 - BURNINATE!:* Vaporize the target with a massive stream of flame. Inflicts 40 Fire damage. Can't be dodged or blocked.

主动技能，冷却时间40秒。造成40点伤害。直接在`dealDamage`里把`ignoresDodge`和`ignoresBlock`都设成true（不是走「精准」那种有条件的判断路径），所以这一下会无条件穿透临时的闪避状态和格挡状态。和「精准/暴击」这一系恐龙技能不同的是，这招连「翱翔」被动也会一并无视，这条分支里没有给`passive_dodge`留任何特殊照顾。

143. Marshmallow Basket - 火 | Sugar Rush | 主动（冷却60秒）
`effect: fast_recharge, effectValue: 100, effectDuration: 10`
**冷却时间：** 60秒
> *wiki原文 - Sugar Rush:* For 10s, your abilities recharge twice as fast for both your pets.

主动技能，冷却时间60秒（「糖分狂飙」）。给施法者自己这组以及它替补的那一组同时加上持续10秒的`fast_cd`增益，让技能冷却回复速度翻倍（`speedMult *= 2`），受影响的这组身上眩晕/冰冻/睡眠的倒计时同样会加快，根据wiki的小知识，这个好处也会延伸到替补组身上，尽管技能文案没有明说。

144. Phoenix Pacifier - 火 | Firestorm | 主动（冷却60秒）
`effect: weather, effectValue: 8, effectDuration: 10, dotDamage: 8, dotDuration: 10`
**冷却时间：** 60秒 | **持续伤害：** 每秒8点，持续10秒
> *wiki原文 - Firestorm:* Weather Effect. Ignite the planet! Everybody burns for 8 Fire damage every 2s, lasting 10s. Replaces any other active Weather Effect.

主动技能，冷却时间60秒（「火焰风暴」）。把全场天气设置成持续10秒的效果：此后每2跳，双方当前出战的两组各承受8点固定的、无法减免的伤害（直接从血量里扣，绕过`dealDamage`）。根据wiki的小知识，天气效果在到期前的最后一跳不会再触发伤害，所以按`8 × 10`算出来的理论总伤害实际上会少一点（按wiki自己举的例子，一场持续10秒的「火焰风暴」实际会少打8点伤害）。

---

### 水系 (Water)

145. Mini Mallard - 水 | Duck's Back | 被动
`effect: passive_resist_negative, effectDuration: 30`
> *wiki原文 - Passive - Automatically resist a negative effect completely. This effect recharges every 30s.
|:* -'''

一直生效，内部自带30秒冷却（`duckBackCd`，如果没设置就默认30秒）。这组接下来第一个要承受的负面效果，不管是通过`applyStatusEffect`来的眩晕/冰冻/创伤，还是通过`applyModifiers`来的各种减益，又或者是持续伤害，都会被完全挡下来（`tryResist`），这个被动随即进入冷却。因为根据wiki小知识，天气效果也算负面效果的一种，所以理论上它也能被这个被动抵抗（同理，也能被「清除」类技能净化）。

146. Living Dead Remote - 水 | Dawn of the Dead | 被动
`effect: passive_revive, effectValue: 30`
**回血：** 30
> *wiki原文 - Dawn of the Dead:* Passive - Revive dead partner with 30% health on death.

这是一个只有待在替补席时才起作用的、每场战斗仅一次的保命效果。`applyPassives`/`gameTick`在它是出战宠物的时候完全不会去检查它，它必须待在*另一*组（没出战的那一组）里才行。一旦那一组的搭档血量归零（这是在死亡那一刻通过`tryLivingDeadRevive`检查的，不是等到后面"两组都死了"才检查），并且Remote自己所在的替补组当时还活着，它就会把刚阵亡的那一组原地复活到最大生命值的30%，不会发生切换，复活的那一组就直接继续当出战组用。一旦触发过一次，`livingDeadUsed`就会把它锁死，这场战斗剩下的时间里不管上场了几只这个宠物都不会再触发，被动图标上也会显示"已使用"的状态。

147. Pet Present Goblin - 水 | Face Slap | 主动（冷却4秒）
`effect: debuff_damage_dealt, effectValue: 20, effectDuration: 15`
**伤害：** 10 （水） | **冷却时间：** 4秒
> *wiki原文 - Face Slap:* Smack the target in the face with a tennis ball, enraging them to inflict 20% more Damage for 15s. The rage stacks up to 20 times. Inflicts 10 Water damage.

主动技能，冷却时间4秒（「甩巴掌」系技能）。造成10点伤害，然后给目标（不是施法者自己）附加持续15秒的`debuff_damage`负面效果：目标在这段时间里造成的伤害-20%。根据wiki的小知识，这个效果的文案写的是「激怒目标」，但实际上是加在敌人身上的负面效果，不是施法者自己的增益。如果这只宠物属于会叠加的那几只（Leashed Silkworm - Purple、Pet Present Goblin），连续释放会让减伤幅度叠加（上限400%），而不是刷新重置。

148. Strawberry Slime - 水 | Corrosion | 主动（冷却4秒）
`effect: buff_damage_dealt, effectValue: 10, effectDuration: 10`
**伤害：** 6 （水） | **冷却时间：** 4秒
> *wiki原文 - Corrosion:* Spray acid that increases damage given by 10% for 10s. Stacks up to 20 times. Inflicts 6 Water damage.

主动技能，冷却时间4秒。给施法者自己这组加一个`buff_damage`自增益：之后10秒内的所有攻击伤害+10%。如果这只宠物属于几只特殊的可叠加来源之一（草莓史莱姆、Leashed Silkworm - Purple、Pet Present Goblin），在增益还没到期前再放一次会额外叠一层（上限400%），持续时间也会取两者中较长的那个，而不是简单地刷新重置。

149. Green Pet Pteranodon - 水 | Precision Strike W | 主动（冷却5秒）
`effect: anti_dodge_block_conditional, effectValue: 25`
**伤害：** 7 （水） | **冷却时间：** 5秒
> *wiki原文 - Precision Strike W:* If the opponent attempts to dodge or block, hits for 25% more damage. Inflicts 7 Water damage. Can't be dodged or blocked.

主动技能，冷却时间5秒，属于「精准/暴击」这一系恐龙技能。造成7点伤害。在`dealDamage`里，这个技能会无条件把`ignoresDodge`和`ignoresBlock`都设成true，不管对方是闪避还是格挡状态，这一下都会实打实命中满伤害。需要注意：wiki上写的是「如果对方尝试闪避或格挡，则造成25%额外伤害」，但实际代码从来没读取过这个技能的`effectValue`字段，它只是单纯无视闪避和格挡，照面板上的固定伤害打，并没有额外加成。唯一的例外：如果对方的闪避恰好是来自某个带`passive_dodge`被动的宠物（比如Yeonnalligi的「翱翔」被动），穿透效果会被关掉，这一下还是会被正常闪避掉，这跟wiki里「翱翔」克制「精准/暴击」的小知识对得上。

150. Penguin Leash - 水 | Frosty Slide | 主动（冷却5秒）
`effect: extend_cooldown, effectValue: 2, effectDuration: 5`
**伤害：** 9 （水） | **冷却时间：** 5秒
> *wiki原文 - Frosty Slide:* Chill the target, making its abilities take 2s longer to recharge if used in the next 5s. Inflicts 9 Water damage.

主动技能，冷却时间5秒。根据目标当前状态有两种不同表现：如果目标当前出战的宠物正在冷却中，直接给它剩余冷却加2秒。如果目标出战的宠物已经冷却完毕（可以放技能），则给那一组挂上一个持续5秒的`extend_cd`负面效果，等到那一组的出战宠物下一次放完任意技能，会立刻额外加2秒冷却，然后这个效果自行消失。

151. Green Pet Apatodon - 水 | Dino Dive W | 主动（冷却6秒）
`effect: elemental_bonus, effectValue: 50`
**伤害：** 10 （水） | **冷却时间：** 6秒
> *wiki原文 - Dino Dive W:* Against a Fire opponent, inflicts +50% damage (on top of normal bonus.) Inflicts 10 Water damage.

主动技能，冷却时间6秒，属于「恐龙冲撞/突袭/俯冲/撕咬」这一系技能。造成10点水系基础伤害。结算这次攻击之前，代码会先判断这只宠物的属性是否克制对方那组宠物的主属性（通过`ELEMENT_BEATS`表）；如果克制，伤害会先乘以`1 + 50/100`（也就是+50%），再交给`dealDamage`处理，而`dealDamage`自己还会照常再叠加一次±25%的属性克制加成，所以属性有利时两个加成是叠在一起生效的。如果属性不占优，这只宠物就只老老实实打出10点固定伤害，没有任何加成。

152. Playful Water Sprite - 水 | Playful Water | 主动（冷却6秒）
`effect: buff_chance_double_damage, effectValue: 20, effectDuration: 4`
**伤害：** 10 （水） | **冷却时间：** 6秒
> *wiki原文 - Playful Water:* Grants you a 20% chance to do double damage for 4 seconds. Inflicts 10 Water damage.

主动技能，冷却时间6秒。造成10点伤害，然后给施法者自己这组宠物加上一个`double_dmg`增益，持续4秒，期间每次攻击有20%概率触发。增益生效时，这组宠物之后每一次攻击在`dealDamage`结算时都会掷一次`doubleDmgChance`判定，一旦命中，那一下的伤害就翻倍。在增益消失前再次释放，只是把倒计时刷新回4秒（不同来源的增益各自独立叠加，不会互相覆盖）。

153. Polar Bear Leash - 水 | Maul | 主动（冷却7秒）
`effect: none`
**伤害：** 14 （水） | **冷却时间：** 7秒
> *wiki原文 - Maul:* Smash and slash! Inflicts 14 Water damage.

主动技能，冷却时间7秒。单纯造成14点固定伤害，不附带任何额外效果，就是纯粹打一下。

154. Black Crystal Dragon - 水 | Crystal Spikes | 主动（冷却8秒）
`effect: anti_block`
**伤害：** 15 （水） | **冷却时间：** 8秒
> *wiki原文 - Crystal Spikes:* A blast of razor-sharp crystals. Inflicts 15 Water damage. Can't be blocked.

主动技能，冷却时间8秒。造成15点伤害。在`dealDamage`里设置`ignoresBlock`，目标身上的「格挡」状态对这一下完全不起作用，但闪避不受影响，该触发还是照常触发。

155. Green Pet Apatoceratops - 水 | Dino Charge W | 主动（冷却8秒）
`effect: elemental_bonus, effectValue: 60`
**伤害：** 14 （水） | **冷却时间：** 8秒
> *wiki原文 - Dino Charge W:* Against a Fire opponent, inflicts +60% damage (on top of normal bonus). Inflicts 14 Water damage.

主动技能，冷却时间8秒，属于「恐龙冲撞/突袭/俯冲/撕咬」这一系技能。造成14点水系基础伤害。结算这次攻击之前，代码会先判断这只宠物的属性是否克制对方那组宠物的主属性（通过`ELEMENT_BEATS`表）；如果克制，伤害会先乘以`1 + 60/100`（也就是+60%），再交给`dealDamage`处理，而`dealDamage`自己还会照常再叠加一次±25%的属性克制加成，所以属性有利时两个加成是叠在一起生效的。如果属性不占优，这只宠物就只老老实实打出14点固定伤害，没有任何加成。

156. Green Pet Pteratops - 水 | Precision Attack W | 主动（冷却9秒）
`effect: anti_dodge_block_conditional, effectValue: 17`
**伤害：** 11 （水） | **冷却时间：** 9秒
> *wiki原文 - Precision Attack W:* If the opponent attempts to dodge or block, hits for 17% more damage. Inflicts 11 Water damage. Can't be dodged or blocked.

主动技能，冷却时间9秒，属于「精准/暴击」这一系恐龙技能。造成11点伤害。在`dealDamage`里，这个技能会无条件把`ignoresDodge`和`ignoresBlock`都设成true，不管对方是闪避还是格挡状态，这一下都会实打实命中满伤害。需要注意：wiki上写的是「如果对方尝试闪避或格挡，则造成17%额外伤害」，但实际代码从来没读取过这个技能的`effectValue`字段，它只是单纯无视闪避和格挡，照面板上的固定伤害打，并没有额外加成。唯一的例外：如果对方的闪避恰好是来自某个带`passive_dodge`被动的宠物（比如Yeonnalligi的「翱翔」被动），穿透效果会被关掉，这一下还是会被正常闪避掉，这跟wiki里「翱翔」克制「精准/暴击」的小知识对得上。

157. Battle Gar - 水 | Snappy Jaws | 主动（冷却10秒）
`effect: trauma, effectDuration: 10`
**伤害：** 15 （水） | **冷却时间：** 10秒
> *wiki原文 - Snappy Jaws:* Causes trauma and prevents the opponent from healing for 10s. Inflicts 15 Water damage.

主动技能，冷却时间10秒。造成15点伤害，并给目标附加持续10秒的「创伤」状态。创伤本身不造成伤害，它只是一个纯粹的「禁止治疗」标记：游戏里所有的治疗来源（宠物自带回血、持续回血效果、天气治疗比如「舒缓迷雾」、生命汲取、护盾抵消伤害时顺带的回血、复活）在生效前都会检查目标身上有没有创伤，有的话就直接跳过这次治疗。根据wiki的小知识，创伤并不会阻止「复生/复活」类效果把一组宠物救回来，也不会阻止「护盾」抵消伤害本身，它只会挡住护盾抵消伤害时顺带的回血部分。

158. Familiar Leash - 水 | Black Magic | 主动（冷却10秒）
`effect: mess_up_undodgeable, effectValue: 10, effectDuration: 20`
**伤害：** 10 （水） | **冷却时间：** 10秒
> *wiki原文 - Black Magic:* Makes your target's abilities fail 10% of the time for the next 20s. Inflicts 10 Water Damage that can't be dodged.

主动技能，冷却时间10秒。造成10点无法被闪避的伤害（这个技能在`dealDamage`里被列进了`ignoresDodge`名单，不过格挡对它依然有效），然后给目标附加和前面同类效果一样的`mess_up`（口误）负面效果，10%概率放技能直接失败，持续20秒。和普通「口误」系宠物唯一的区别是，这只宠物自己打出的伤害没法靠闪避躲开。

159. Green Pet Apatos Rex - 水 | Dino Chomp W | 主动（冷却10秒）
`effect: elemental_bonus, effectValue: 25`
**伤害：** 18 （水） | **冷却时间：** 10秒
> *wiki原文 - Dino Chomp W:* Against a Fire opponent, inflicts +25% damage (on top of normal bonus). Inflicts 18 Water damage.

主动技能，冷却时间10秒，属于「恐龙冲撞/突袭/俯冲/撕咬」这一系技能。造成18点水系基础伤害。结算这次攻击之前，代码会先判断这只宠物的属性是否克制对方那组宠物的主属性（通过`ELEMENT_BEATS`表）；如果克制，伤害会先乘以`1 + 25/100`（也就是+25%），再交给`dealDamage`处理，而`dealDamage`自己还会照常再叠加一次±25%的属性克制加成，所以属性有利时两个加成是叠在一起生效的。如果属性不占优，这只宠物就只老老实实打出18点固定伤害，没有任何加成。

160. Green Pet Apatosaurus - 水 | Dino Slam W | 主动（冷却10秒）
`effect: elemental_bonus, effectValue: 100`
**伤害：** 15 （水） | **冷却时间：** 10秒
> *wiki原文 - Dino Slam W:* Against a Fire opponent, inflicts +100% damage (on top of normal bonus). Inflicts 15 Water damage.

主动技能，冷却时间10秒，属于「恐龙冲撞/突袭/俯冲/撕咬」这一系技能。造成15点水系基础伤害。结算这次攻击之前，代码会先判断这只宠物的属性是否克制对方那组宠物的主属性（通过`ELEMENT_BEATS`表）；如果克制，伤害会先乘以`1 + 100/100`（也就是+100%），再交给`dealDamage`处理，而`dealDamage`自己还会照常再叠加一次±25%的属性克制加成，所以属性有利时两个加成是叠在一起生效的。如果属性不占优，这只宠物就只老老实实打出15点固定伤害，没有任何加成。

161. Green Pet Triceradon - 水 | Defensive Flurry W | 主动（冷却10秒）
`effect: block, effectDuration: 2`
**伤害：** 12 （水） | **冷却时间：** 10秒
> *wiki原文 - Defensive Flurry W:* Strike and then block attacks for 2s. Inflicts 12 Water damage.

主动技能，冷却时间10秒。对敌方造成12点伤害，然后让施法者自己这组宠物获得持续2秒的「格挡」状态（代码里是给它设置`applyStatusEffect`，类型为`block`）。格挡生效期间，只要这一下攻击不是明确无视格挡的类型（也就是不带`anti_block`/`anti_dodge_block`标记，也不是能穿透格挡的「精准」系技能），`dealDamage`就会在伤害下限生效之前直接把这次伤害拦成0。

162. Green Pet Tyranodon - 水 | Piercing Jaws W | 主动（冷却10秒）
`effect: trauma, effectDuration: 4`
**伤害：** 15 （水） | **冷却时间：** 10秒
> *wiki原文 - Piercing Jaws W:* Causes trauma, preventing the target from healing for 4s. Inflicts 15 Water damage.

主动技能，冷却时间10秒。造成15点伤害，并给目标附加持续4秒的「创伤」状态。创伤本身不造成伤害，它只是一个纯粹的「禁止治疗」标记：游戏里所有的治疗来源（宠物自带回血、持续回血效果、天气治疗比如「舒缓迷雾」、生命汲取、护盾抵消伤害时顺带的回血、复活）在生效前都会检查目标身上有没有创伤，有的话就直接跳过这次治疗。根据wiki的小知识，创伤并不会阻止「复生/复活」类效果把一组宠物救回来，也不会阻止「护盾」抵消伤害本身，它只会挡住护盾抵消伤害时顺带的回血部分。

163. Ice Dragon Hand - 水 | Frost Breath | 主动（冷却10秒）
`effect: freeze, effectDuration: 2`
**伤害：** 10 （水） | **冷却时间：** 10秒
> *wiki原文 - Frost Breath:* Freeze the target for 2s. Frozen targets can't even swap out. Inflicts 10 Water damage.

主动技能，冷却时间10秒。造成10点伤害，然后把目标冰冻2秒。冰冻和眩晕一样会让对方没法使用技能（都在stun/freeze/sleep/dance这个禁止行动的列表里），但还会额外把切换按钮也锁死，只要`freeze`状态还剩时间，`swapDeck`就会直接拒绝执行，所以被冰冻的一组连撤退到备战组都做不到。

164. Reanimator Remote - 水 | Supercharge | 主动（冷却10秒）
`effect: buff_next_damage, effectValue: 50`
**冷却时间：** 10秒
> *wiki原文 - Supercharge:* Charge up your next skill with +50% damage.

主动技能，冷却时间10秒，不直接造成伤害。给施法者自己这组加上一个+50%的`next_dmg`增益，这个增益不会因为倒计时而自然消失（内部设定的持续时间是99秒），而是在施法者这组打出下一次命中的瞬间被消耗掉：`dealDamage`结算完加成之后马上把这个增益摘掉，所以不管你等多久才用出下一击，它都只会作用在那一次攻击上。

165. Pet Frog - 水 | Spit Mud | 主动（冷却12秒）
`effect: slow_recharge, effectValue: 50, effectDuration: 5`
**伤害：** 20 （水） | **冷却时间：** 12秒
> *wiki原文 - Spit Mud:* For 5s, the target's abilities recharge half as fast. Inflicts 20 Water damage.

主动技能，冷却时间12秒。造成20点伤害，然后给目标的出战组**以及**它替补的那一组同时附加持续5秒的`slow_cd`负面效果，让技能冷却回复速度减半（`speedMult *= 0.5`），同时那一组的眩晕/冰冻/睡眠等状态的倒计时也会变慢，根据wiki的小知识，这种「连替补组一起影响」的范围wiki文案里没写明，但代码里确实是这么实现的，而且不只影响技能冷却，控制类状态的持续时间也会一起变慢。

166. Green Pet Triceratops - 水 | Defensive Gore W | 主动（冷却13秒）
`effect: block, effectDuration: 4`
**伤害：** 16 （水） | **冷却时间：** 13秒
> *wiki原文 - Defensive Gore W:* Strike and then block attacks for 4s. Inflicts 16 Water damage.

主动技能，冷却时间13秒。对敌方造成16点伤害，然后让施法者自己这组宠物获得持续4秒的「格挡」状态（代码里是给它设置`applyStatusEffect`，类型为`block`）。格挡生效期间，只要这一下攻击不是明确无视格挡的类型（也就是不带`anti_block`/`anti_dodge_block`标记，也不是能穿透格挡的「精准」系技能），`dealDamage`就会在伤害下限生效之前直接把这次伤害拦成0。

167. Green Pet Tyranotops - 水 | Crushing Beak W | 主动（冷却13秒）
`effect: trauma, effectDuration: 7`
**伤害：** 19 （水） | **冷却时间：** 13秒
> *wiki原文 - Crushing Beak W:* Causes Trauma, preventing the target from healing for 7 seconds. Inflicts 19 Water damage.

主动技能，冷却时间13秒。造成19点伤害，并给目标附加持续7秒的「创伤」状态。创伤本身不造成伤害，它只是一个纯粹的「禁止治疗」标记：游戏里所有的治疗来源（宠物自带回血、持续回血效果、天气治疗比如「舒缓迷雾」、生命汲取、护盾抵消伤害时顺带的回血、复活）在生效前都会检查目标身上有没有创伤，有的话就直接跳过这次治疗。根据wiki的小知识，创伤并不会阻止「复生/复活」类效果把一组宠物救回来，也不会阻止「护盾」抵消伤害本身，它只会挡住护盾抵消伤害时顺带的回血部分。

168. Robo Shark - 水 | Laserbeam! | 主动（冷却13秒）
`effect: none`
**伤害：** 30 （水） | **冷却时间：** 13秒
> *wiki原文 - Laserbeam!:* Fires a laserbeam that deals Water Damage! Inflicts 30 Water Damage.

主动技能，冷却时间13秒。单纯造成30点固定伤害，不附带任何额外效果，就是纯粹打一下。

169. Green Pet Pterosaurus - 水 | Critical Strike W | 主动（冷却15秒）
`effect: anti_dodge_block_conditional, effectValue: 50`
**伤害：** 17 （水） | **冷却时间：** 15秒
> *wiki原文 - Critical Strike W:* If the opponent attempts to dodge or block, hits for 50% more damage. Inflicts 17 Water damage. Can't be dodged or blocked.

主动技能，冷却时间15秒，属于「精准/暴击」这一系恐龙技能。造成17点伤害。在`dealDamage`里，这个技能会无条件把`ignoresDodge`和`ignoresBlock`都设成true，不管对方是闪避还是格挡状态，这一下都会实打实命中满伤害。需要注意：wiki上写的是「如果对方尝试闪避或格挡，则造成50%额外伤害」，但实际代码从来没读取过这个技能的`effectValue`字段，它只是单纯无视闪避和格挡，照面板上的固定伤害打，并没有额外加成。唯一的例外：如果对方的闪避恰好是来自某个带`passive_dodge`被动的宠物（比如Yeonnalligi的「翱翔」被动），穿透效果会被关掉，这一下还是会被正常闪避掉，这跟wiki里「翱翔」克制「精准/暴击」的小知识对得上。

170. Mini-Peng - 水 | Trick Escape | 主动（冷却15秒）
`effect: dodge_swap, effectDuration: 4`
**冷却时间：** 15秒
> *wiki原文 - Trick Escape:* If struck in the next 4s, avoid it completely and swap out if able to). Teammate gets ability to dodge for 4s. Is not visible to enemy.

主动技能，冷却时间15秒（「舍身」/「金蝉脱壳」系技能）。如果施法者还有一组活着的备战宠物、并且当前允许切换，就立刻切换过去（同时清掉正在离场那组身上的「厄运」诅咒），并给刚换上场的那一组加上持续4秒的「闪避」状态，是换上来的这组拿到闪避窗口，而不是撤下去的那组（因为撤下去那组已经不在场上，本来就打不到）。根据wiki的小知识，这招理论上也能被这只宠物其他自我指向的技能连带触发（比如石化、杂技、跳跃等），但目前的模拟器只实现了这只宠物自己主动释放这一招的情况。

171. Muni's Caspiro - 水 | Luck Be A Kitty | 主动（冷却15秒）
`effect: none`
**回血：** 30 | **冷却时间：** 15秒
> *wiki原文 - Luck Be A Kitty:* Good luck heals thy teammate. Heals 30 life.

主动技能，冷却时间15秒。不造成伤害也没有其他附加效果，单纯给施法者自己这组宠物回30点固定生命值，仅此而已。

172. Mysterious Synthoid - 水 | Mysterious Beam | 主动（冷却15秒）
`effect: ethereal, effectValue: 50, effectDuration: 4`
**伤害：** 10 （水） | **冷却时间：** 15秒
> *wiki原文 - Mysterious Beam:* Drain your molecules to shoot a beam. This leaves you ethereal for 4s, taking 50% less damage. Inflicts 10 Water damage.

主动技能，冷却时间15秒（「灵体化」/「迷雾光束」系技能）。给施法者自己这组宠物加上持续4秒的「灵体」状态。这明确不是闪避概率，`dealDamage`会把它算成固定50%的减伤，在闪避/格挡判定之后、其他伤害增减益结算之前，把受到的伤害乘以`1 - 50/100`。重新释放会直接替换掉现有的灵体状态，而不是叠加。

173. Reindeer Bell - 水 | Winter Gift | 主动（冷却15秒）
`effect: none`
**回血：** 30 | **冷却时间：** 15秒
> *wiki原文 - Winter Gift:* Give a gift to your teammate, healing them. Heals 30 life.

主动技能，冷却时间15秒。不造成伤害也没有其他附加效果，单纯给施法者自己这组宠物回30点固定生命值，仅此而已。

174. Winter Flu Vaccine - 水 | Influenza | 主动（冷却15秒）
`effect: dot, effectValue: 2, effectDuration: 25, dotDamage: 2, dotDuration: 25`
**冷却时间：** 15秒 | **持续伤害：** 每秒2点，持续25秒
> *wiki原文 - Influenza:* Infect the target for 2 Water damage per second over 25s.

主动技能，冷却时间15秒。给目标附加一个持续伤害状态，每一跳造成2点伤害，持续25秒（和其他负面效果一样，会受到passive_resist_negative抵抗判定和passive_shorten_debuff持续时间缩减的影响）。根据wiki的小知识，持续伤害（和所有限时效果一样）不会在倒计时的最后一秒再跳一次，所以实际造成的总伤害会比直接拿「2 × 25」算出来的数字少一跳。

175. Green Pet Tyrannosaurus - 水 | Crushing Jaws W | 主动（冷却16秒）
`effect: trauma, effectDuration: 3`
**伤害：** 29 （水） | **冷却时间：** 16秒
> *wiki原文 - Crushing Jaws W:* Causes trauma, preventing the target for 3s. Inflicts 29 Water damage.

主动技能，冷却时间16秒。造成29点伤害，并给目标附加持续3秒的「创伤」状态。创伤本身不造成伤害，它只是一个纯粹的「禁止治疗」标记：游戏里所有的治疗来源（宠物自带回血、持续回血效果、天气治疗比如「舒缓迷雾」、生命汲取、护盾抵消伤害时顺带的回血、复活）在生效前都会检查目标身上有没有创伤，有的话就直接跳过这次治疗。根据wiki的小知识，创伤并不会阻止「复生/复活」类效果把一组宠物救回来，也不会阻止「护盾」抵消伤害本身，它只会挡住护盾抵消伤害时顺带的回血部分。

176. Green Pet Tricerus Rex - 水 | Defensive Bite W | 主动（冷却18秒）
`effect: block, effectDuration: 2`
**伤害：** 22 （水） | **冷却时间：** 18秒
> *wiki原文 - Defensive Bite W:* Strike and then block attacks for 2s. Inflicts 22 Water damage.

主动技能，冷却时间18秒。对敌方造成22点伤害，然后让施法者自己这组宠物获得持续2秒的「格挡」状态（代码里是给它设置`applyStatusEffect`，类型为`block`）。格挡生效期间，只要这一下攻击不是明确无视格挡的类型（也就是不带`anti_block`/`anti_dodge_block`标记，也不是能穿透格挡的「精准」系技能），`dealDamage`就会在伤害下限生效之前直接把这次伤害拦成0。

177. Green Pet Pteranus Rex - 水 | Precision Crush W | 主动（冷却19秒）
`effect: anti_dodge_block_conditional, effectValue: 10`
**伤害：** 25 （水） | **冷却时间：** 19秒
> *wiki原文 - Precision Crush W:* If the opponent attempts to dodge or block, hits for 10% more damage. Inflicts 25 Water damage. Can't be dodged or blocked.

主动技能，冷却时间19秒，属于「精准/暴击」这一系恐龙技能。造成25点伤害。在`dealDamage`里，这个技能会无条件把`ignoresDodge`和`ignoresBlock`都设成true，不管对方是闪避还是格挡状态，这一下都会实打实命中满伤害。需要注意：wiki上写的是「如果对方尝试闪避或格挡，则造成10%额外伤害」，但实际代码从来没读取过这个技能的`effectValue`字段，它只是单纯无视闪避和格挡，照面板上的固定伤害打，并没有额外加成。唯一的例外：如果对方的闪避恰好是来自某个带`passive_dodge`被动的宠物（比如Yeonnalligi的「翱翔」被动），穿透效果会被关掉，这一下还是会被正常闪避掉，这跟wiki里「翱翔」克制「精准/暴击」的小知识对得上。

178. Bride Of Reanimator Remote - 水 | Reanimate | 主动（冷却20秒）
`effect: revive, effectValue: 50, effectDuration: 5`
**回血：** 50 | **冷却时间：** 20秒
> *wiki原文 - Reanimate:* If beaten within 5s, resurrect with 50% life.

主动技能，冷却时间20秒，考虑到冷却这么长，实际能等到的出手窗口大概也就5秒左右。如果施法者自己的备战组当前血量是0，会把它复活到最大生命值的50%。如果备战组还活着，放这个技能就完全没有效果（不回血也没有其他作用），它只是纯粹的复活手段，不是治疗技能。

179. Eldritch Spawn - 水 | Cosmic Fear | 主动（冷却20秒）
`effect: mess_up_undodgeable, effectValue: 20, effectDuration: 10`
**伤害：** 20 （水） | **冷却时间：** 20秒
> *wiki原文 - Cosmic Fear:* Curse the target's skills to fail 20% of the time for 10s. Inflicts 20 Water Damage. Can't be dodged.

主动技能，冷却时间20秒。造成20点无法被闪避的伤害（这个技能在`dealDamage`里被列进了`ignoresDodge`名单，不过格挡对它依然有效），然后给目标附加和前面同类效果一样的`mess_up`（口误）负面效果，20%概率放技能直接失败，持续10秒。和普通「口误」系宠物唯一的区别是，这只宠物自己打出的伤害没法靠闪避躲开。

180. Green Pet Tripatosaurus - 水 | Defensive Bash W | 主动（冷却20秒）
`effect: block, effectDuration: 7`
**伤害：** 10 （水） | **冷却时间：** 20秒
> *wiki原文 - Defensive Bash W:* Strike and then block attacks for 7s. Inflicts 10 Water damage

主动技能，冷却时间20秒。对敌方造成10点伤害，然后让施法者自己这组宠物获得持续7秒的「格挡」状态（代码里是给它设置`applyStatusEffect`，类型为`block`）。格挡生效期间，只要这一下攻击不是明确无视格挡的类型（也就是不带`anti_block`/`anti_dodge_block`标记，也不是能穿透格挡的「精准」系技能），`dealDamage`就会在伤害下限生效之前直接把这次伤害拦成0。

181. Green Pet Tyranopatos - 水 | Grinding Jaws W | 主动（冷却20秒）
`effect: trauma, effectDuration: 30`
**伤害：** 3 （水） | **冷却时间：** 20秒
> *wiki原文 - Grinding Jaws W:* Causes Trauma, preventing the target from healing for 30s. Inflicts 3 Water damage.

主动技能，冷却时间20秒。造成3点伤害，并给目标附加持续30秒的「创伤」状态。创伤本身不造成伤害，它只是一个纯粹的「禁止治疗」标记：游戏里所有的治疗来源（宠物自带回血、持续回血效果、天气治疗比如「舒缓迷雾」、生命汲取、护盾抵消伤害时顺带的回血、复活）在生效前都会检查目标身上有没有创伤，有的话就直接跳过这次治疗。根据wiki的小知识，创伤并不会阻止「复生/复活」类效果把一组宠物救回来，也不会阻止「护盾」抵消伤害本身，它只会挡住护盾抵消伤害时顺带的回血部分。

182. Leashed Silkworm - Blue - 水 | Toss Cookies | 主动（冷却20秒）
`effect: cleanse`
**伤害：** 10 （水） | **冷却时间：** 20秒
> *wiki原文 - Toss Cookies:* Purge yourself of harmful effects, inflicting 10 Water damage per effect removed.

主动技能，冷却时间20秒（「吐饼干」）。清除施法者自己这组身上当前所有的负面状态（眩晕/冰冻/创伤/持续伤害/跳舞/睡眠）和所有的减益类负面效果，同时数一下总共清掉了几个。每清掉一个，敌方就会承受10点伤害（所以清掉3个负面效果就是`10 × 3`点伤害）；如果一个都没清到，就完全不造成伤害。根据wiki的小知识，这个净化即便技能本身该造成的伤害被闪避/格挡掉了、甚至基础攻击根本没命中，净化部分照样会触发；但如果施法本身被「口误」类效果搞砸了，那就完全不会触发。

183. Magnifying Glass - 水 | Infect | 主动（冷却20秒）
`effect: dot, effectValue: 6, effectDuration: 7, dotDamage: 6, dotDuration: 7`
**冷却时间：** 20秒 | **持续伤害：** 每秒6点，持续7秒
> *wiki原文 - Infect:* Infect the target for 6 Water damage per second lasting 7s.

主动技能，冷却时间20秒。给目标附加一个持续伤害状态，每一跳造成6点伤害，持续7秒（和其他负面效果一样，会受到passive_resist_negative抵抗判定和passive_shorten_debuff持续时间缩减的影响）。根据wiki的小知识，持续伤害（和所有限时效果一样）不会在倒计时的最后一秒再跳一次，所以实际造成的总伤害会比直接拿「6 × 7」算出来的数字少一跳。

184. Pet Slime - 水 | Regrowth | 主动（冷却20秒）
`effect: hot, effectDuration: 10, dotDamage: 4, dotDuration: 10`
**回血：** 4 | **冷却时间：** 20秒 | **持续伤害：** 每秒4点，持续10秒
> *wiki原文 - Regrowth:* Heal 4 health every second for 10s.

主动技能，冷却时间20秒（「新生」）。不对任何人直接造成伤害。给施法者自己这组加上一个每秒回复4点血、持续10秒的持续回血状态。虽然数据库里这招和真正的持续伤害效果共用同一个`isDot: true`标记（所以走的是同一套计时机制），但处理逻辑会把它导向给施法者自己回血，而不是对目标造成伤害，这个判断会在普通持续伤害分支之前就被处理掉，专门是为了避免被误判成打在敌方身上的伤害。和其他治疗一样，会被施法者身上的创伤挡住。

185. Rayman's Fist - 水 | Super Slap | 主动（冷却20秒）
`effect: stun, effectDuration: 5`
**伤害：** 20 （水） | **冷却时间：** 20秒
> *wiki原文 - Super Slap:* Stuns the target for 5s, making them unable to act. Inflicts 20 Water damage.

主动技能，冷却时间20秒。造成20点伤害，然后让目标进入5秒的「眩晕」状态，没法用技能，但（和冰冻不同）不会锁死切换按钮，所以被眩晕的一组宠物随时都能撤回备战组。

186. Passionate Painter Paintbrush Pet - 水 | Hit by Inspiration | 主动（冷却25秒）
`effect: absorb, effectDuration: 4`
**冷却时间：** 25秒
> *wiki原文 - Hit by Inspiration:* For 4 seconds, you take in your surroundings to paint a picture converting any incoming damage to health.

主动技能，冷却时间25秒，不直接造成伤害。给施法者自己这组加上持续4秒的「护盾」状态。护盾在场期间，这组接下来要挨的那一下（或几下）攻击会在`dealDamage`里被完全抵消（伤害归零），并且，只要这组身上没有创伤，还会按原本会造成的伤害数值原样回血。如果身上有创伤，伤害依然会被抵消，但回血部分会被挡掉。根据wiki的小知识，这个技能和另一个护盾类技能算是同一个「技能类别」：在一个还没到期前再放另一个，会延长剩余时间而不是重置；其中一个版本用的施法动画固定是2秒，即便它实际的护盾时长更长，所以光看动画是看不出剩余时间的。

187. Black Cat Leash - 水 | Bad Luck | 主动（冷却30秒）
`effect: debuff_damage_both, effectValue: 25, effectDuration: 8`
**冷却时间：** 30秒
> *wiki原文 - Bad Luck:* Cross the target's path, making them do -25% damage, and suffer +25% damage, for 8s.

主动技能，冷却时间30秒（「霉运」），不直接造成伤害。给目标附加持续8秒的`bad_luck`负面效果，这个效果一次性坑对方两次：造成的伤害-25%，同时承受的伤害+25%，两个数值用的是同一个`effectValue`。

188. Diamond Dragon - 水 | Diamond Block | 主动（冷却30秒）
`effect: anti_swap, effectDuration: 10`
**冷却时间：** 30秒
> *wiki原文 - Diamond Block:* Prevent the target from swapping out for 10s.

主动技能，冷却时间30秒，不直接造成伤害。给目标那组附加持续10秒的`no_swap`（禁止切换）负面效果（走`applyModifiers`）。这个效果生效期间，`recomputeModifiers`会把那组的`canSwap`设为false，切换按钮会被锁住（界面上显示` LOCKED`），`swapDeck()`在这段时间里也会拒绝执行，目标就算想跑也跑不掉，只能困在当前这组宠物里。

189. Ghost Wolf Monocle - 水 | Mist Form | 主动（冷却30秒）
`effect: ethereal, effectValue: 50, effectDuration: 10`
**冷却时间：** 30秒
> *wiki原文 - Mist Form:* Become ethereal for 10s, reducing all damage taken by 50%.

主动技能，冷却时间30秒（「灵体化」/「迷雾光束」系技能）。给施法者自己这组宠物加上持续10秒的「灵体」状态。这明确不是闪避概率，`dealDamage`会把它算成固定50%的减伤，在闪避/格挡判定之后、其他伤害增减益结算之前，把受到的伤害乘以`1 - 50/100`。重新释放会直接替换掉现有的灵体状态，而不是叠加。

190. Ice Calf Leash - 水 | Ice Laser | 主动（冷却30秒）
`effect: freeze, effectDuration: 5`
**冷却时间：** 30秒
> *wiki原文 - Ice Laser:* Freeze the target for 5s. Frozen targets can't even swap out.

主动技能，冷却时间30秒。造成0点伤害，然后把目标冰冻5秒。冰冻和眩晕一样会让对方没法使用技能（都在stun/freeze/sleep/dance这个禁止行动的列表里），但还会额外把切换按钮也锁死，只要`freeze`状态还剩时间，`swapDeck`就会直接拒绝执行，所以被冰冻的一组连撤退到备战组都做不到。

191. Magical Carrot - 水 | That's Cold | 主动（冷却30秒）
`effect: throw_teammate`
**冷却时间：** 30秒
> *wiki原文 - 30s:* 

主动技能，冷却时间30秒（「透心凉」）。如果施法者自己的备战组还活着，会直接把它的血量清零，同时把那一组「阵亡前」的血量取一半（向下取整），分别打在敌方两组身上（出战那一组走正常伤害流程，备战组如果还活着也会挨这一下）。如果这次施法本身被「口误」类效果搞砸了，整个流程就完全不会执行，备战组会被放过，**除非**敌方那一刻恰好处于闪避或格挡状态，这种情况下备战组依然会被直接清零，就算施法本身失败了也不例外（这是wiki明确记载的、「口误放过搭档」规则的一个特殊例外）。

192. Royal Eldritch Spawn - 水 | Cosmic Fear | 主动（冷却30秒）
`effect: mess_up_undodgeable, effectValue: 20, effectDuration: 15`
**伤害：** 30 （水） | **冷却时间：** 30秒
> *wiki原文 - Cosmic Fear:* Curse the target's skills to fail 20% of the time for 15s. Inflicts 30 Water Damage. Can't be dodged.

主动技能，冷却时间30秒。造成30点无法被闪避的伤害（这个技能在`dealDamage`里被列进了`ignoresDodge`名单，不过格挡对它依然有效），然后给目标附加和前面同类效果一样的`mess_up`（口误）负面效果，20%概率放技能直接失败，持续15秒。和普通「口误」系宠物唯一的区别是，这只宠物自己打出的伤害没法靠闪避躲开。

193. Leashed Silkworm - Aqua - 水 | Soothing Mist | 主动（冷却60秒）
`effect: weather_hot, effectDuration: 30, dotDamage: 6, dotDuration: 30`
**冷却时间：** 60秒 | **持续伤害：** 每秒6点，持续30秒
> *wiki原文 - Soothing Mist:* Weather Effect. Heal everyone for 6 life every 5s, lasting 30s. Replaces any other active weather effect.

主动技能，冷却时间60秒（「舒缓迷雾」）。把全场天气设置成持续30秒的效果：此后每5跳，双方当前出战的两组各回复6点血（如果哪一组身上有创伤，这部分回血会被挡住）。根据wiki的小知识，和伤害类天气一样，这个效果在到期前的最后一跳也不会再触发，所以实际回血总量会比按整个窗口直接算出来的理论值（比如wiki举例里提到的36点）要少一点，最后落地大概是30点左右。

---


---

# 第三部分：开发者 / 函数参考
---

### 1. 文件结构

`simulator.html`是一个自包含的单文件：HTML body里有一个`<style>`块，一个`#team-builder`界面和一个空的`#battle-screen`容器，然后是一大块`<script>`。没有构建步骤，没有模块系统，也没有用任何框架，全是普普通通的全局函数和全局变量。

```
<style>...</style>
<body>
  #team-builder   （组队界面，一开始就能看到）
  #battle-screen  （点"开始对战"之前是空的，之后由JS填充）
  <script>
    // 1. 组队界面的状态与函数
    // 2. 战斗引擎（状态、伤害流程、效果分发、游戏主循环）
    // 3. 界面渲染函数
    // 4. "开始对战"按钮的点击处理（把上面这些串起来）
  </script>
</body>
```

宠物数据本身**不是**写死在这个文件里的，它是在页面加载时从`Data/pet_battle_database.json`读取进来的，存到全局的`allPets`数组里。**要添加新宠物，改的就是这个文件**（详见第7节）。`simulator.html`本身只包含负责解读宠物`effect`字段的那套*引擎*代码。

---

### 2. 数据模型

#### 2.1 宠物在数据库里的原始记录（`pet_battle_database.json`里的一条）

```js
{
  name: "Yeonnalligi",
  element: "Air",               // Fire | Earth | Air | Water
  ability: "Soaring",
  is_passive: true,             // true = always-on
  cooldown: 0,                  // 秒; ignored if is_passive
  damage: 0, damageType: "",
  heal: 0,
  effect: "passive_dodge",
  effectValue: 25,
  effectDuration: 0,
  isDot: false, dotDamage: 0, dotDuration: 0,
  img: "....png", hitSound: "....mp3", powerSound: "....mp3"
}
```

组队界面里每一张宠物卡片，都是这些记录里某一条的浅拷贝（`{ ...pet }`）。引擎里绝大多数逻辑都不会针对某只具体宠物的名字做特殊处理，真正决定走哪套代码逻辑的是`effect`字段，**唯一的例外**是下面这一小份、故意保留的按名字特判清单，专门用来处理那些光靠`effect`字段没法表达的行为：

| 按**名字/技能名**特判，而不是靠`effect` | 位置 | 原因 |
|---|---|---|
| `sourcePet.data.ability === 'Soaring'` | `dealDamage` | 精准/暴击这一系恐龙技能明确不会穿透这只宠物的闪避 |
| `sourcePet.data.ability === 'Disco Fever'` | `applyModifiers`（`force_dance`） | 只有这一招让人跳舞的技能会额外附带闪避 |
| `pet.data.name === 'Pinata Pal'` | `handlePetEffect`（`trap_swap`） | 只有这只宠物的陷阱能同时打中敌方两组 |
| `['Strawberry Slime', 'Leashed Silkworm - Purple', 'Pet Present Goblin']` | `applyModifiers`（`stackingPets`） | 只有这几个来源的增益/减益是叠加而不是刷新 |
| `attackerPet.data.ability === 'Toasties'` | `dealDamage`（`isToasties`） | 标记这一下攻击几乎跳过所有加成 |

如果你要加的新宠物需要用到这类特殊交互，就应该来这张表里加一条，不要为了特判一只宠物的名字而专门发明一个新的`effect`键。

#### 2.2 `battle`对象（所有运行时状态都存在这里）

```js
battle = {
  running: true, tick: 0,
  delayedHits: [], healBacks: [], swapTraps: [],   // 队列，详见第6.7节
  weather: { type, remaining, damage, tickRate, tickCounter, heal, fireDmgMod, otherDmgMod, sourceName, sourceImg },
  your:   { activeDeck: 1, activePet: 0, livingDeadUsed: false, decks: { 1: <deck>, 2: <deck> } },
  enemy:  { ...结构一样... }
}
```

**玩家**对象（`battle.your` / `battle.enemy`）记录着当前选中的是哪一组/哪只宠物，以及"Living Dead是否已经用过"这个标记。**一组（deck）**对象长这样：

```js
deck = {
  hp: 150, swapCd: 0, duckBackCd: 0,
  pets: [ {data, cd, charge?, growStacks?, buildPhase?}, ... 最多3个 ],
  statusEffects: [ {type, duration, remaining, sourceName, sourceImg, ...额外字段} ],
  modifiers: {
    damageDealt, damageTaken, nextDamage, doubleDmgChance, rechargeSpeed,
    messUpChance, canSwap, canAct, buffs: [], debuffs: []
  }
}
```

`pet.data`是静态的数据库记录；`pet`上其他的字段（`cd`、`charge`、`growStacks`、`buildPhase`）都是挂在这个宠物实例上的、随战斗过程会变化的状态。

`statusEffects`存放的是**带时限、按类型打标签的条目**（眩晕/冰冻/创伤/闪避/格挡/持续伤害/持续回血/灵体/护盾/厄运/跳舞/睡眠），凡是带`remaining`倒计时、又不是百分比加成的东西都放在这里。`modifiers.buffs`/`.debuffs`存放的是**百分比/固定数值的加成**（伤害百分比、冷却回复速度、口误概率等等），这些会被`recomputeModifiers`汇总进上面那些固定字段里。这么拆分是因为状态效果控制的是*能不能行动*（这一组现在能不能行动/切换？），而加成影响的是*数值大小*（伤害多少、冷却转多快），具体哪个属于哪一类，详见第5节。

---

### 3. 组队界面相关函数

这些函数只在战斗开始之前运行；它们都不会碰`battle`对象（因为战斗开始前这个对象还不存在）。

| 函数 | 作用 |
|---|---|
| `filterPets(query, containerId)` | 在搜索框里打字时，实时过滤宠物卡片网格；如果某个属性分类下一个匹配都没有，就把整个分类隐藏掉。 |
| `buildTeamBuilder()` | `Data/pet_battle_database.json`加载完成后的入口函数。负责搭建双方的宠物列表，以及两组各自的空位行。 |
| `buildPetList(cid, owner)` | 渲染某一方可滚动的宠物网格，按风/地/火/水分区，每只宠物一张`.pet-card`卡片，点击事件绑定到`addPet`。 |
| `buildSlots(cid, deck, owner, dnum)` | 渲染某一组的3个空位占位符；每个空位的点击事件是`removePet`。 |
| `countInTeam(team, petName)` | 数一下某个名字的宠物已经在这一方两组里放了几份（用来限制最多2份）。 |
| `updateCardState(owner, petName)` | 增/删一份宠物之后，更新对应卡片的视觉状态（`in-team`/`partial-team`样式和"1/2"/"2/2"角标）。 |
| `addPet(pet, owner, card)` | 宠物卡片的点击处理函数。如果这一方已经放了2份就直接拒绝。否则先扫描第1组再扫描第2组，**跳过任何已经带了同名宠物的那一组**，把这一份放进第一个"既没有这只宠物、又还有空位"的那一组的第一个空位，真正执行"每组最多一份、两组总共最多两份"这条规则的就是这一步，不只是最外层那个2份上限的判断。 |
| `removePet(owner, dnum, slot)` | 已占用位置的点击处理函数，把它清空。 |
| `refreshAll()` / `refreshSlots(cid, deck)` | 每次增删之后，根据`yourTeam`/`enemyTeam`的状态重新渲染全部4行分组位置。 |
| `updateBtn()` | 控制"⚔️ 开始对战"按钮的显示/隐藏，只有双方6个位置都填满了才会显示。 |
| `playSound(soundPath)` | 带缓存的`Audio`播放辅助函数（战斗中的命中/放技能音效也用这个）；如果被浏览器的自动播放限制挡住了，会安静地忽略错误。 |

**要添加新宠物：** 完全不需要动这些函数，它们都是直接读`allPets`的，而这个数组本身就是从JSON文件里来的。详见第9节。

---

### 4. 战斗初始化

| 函数 | 作用 |
|---|---|
| `initBattle()` | 根据`yourTeam`/`enemyTeam`搭建全新的`battle`对象。对每一组：计算每只宠物的初始冷却（`floor(冷却/2)`，上限12，下限1，如果这一组带了Windspeed再多减2），并调用`resetModifiers`。 |
| `resetModifiers(deck)` | 把某一组的`modifiers`对象重置为全零/中立的默认值。战斗开始时每组调用一次，**不是**一个通用的"清空所有增益"函数；战斗中途要清效果（比如`cleanse`净化）走的是直接操作`.buffs`/`.debuffs`数组、再调用`recomputeModifiers`这条路。 |
| `getDeck(player)` | `player.decks[player.activeDeck]`，当前出战的那一组。 |
| `getPet(player)` | 那一组里当前选中的宠物。 |
| `inactiveDeck(player)` | 另外那个（替补）组。 |
| `getMaxHp(player, deckNum?)` | 如果那一组带有`passive_hp_boost`宠物（Mammoth），返回195，否则返回150。不传`deckNum`时默认取出战组。 |
| `getDeckElement(deck, activeIdx)` | 用来判断属性克制关系的这一组"主属性"：3个位置里数量最多的那种属性，打平时以`activeIdx`那个位置的宠物属性为准。 |

---

### 5. 伤害与效果处理流程

这是你最常会动到的部分。在往里加任何东西之前，先搞懂这里的*执行顺序*非常关键。

#### 5.1 `dealDamage(player, amount, attackerPet)`

几乎所有伤害都会经过的这一个总入口（例外：Toasties的固定命中、天气跳的伤害、反噬类的自伤，这些都是直接扣血，绕过这个函数，具体可以看第6节效果表里"绕过`dealDamage`"那一列）。**严格按这个顺序**依次套用：紫雾天气加成 → 属性±25% → 闪避判定 → 格挡判定 → 灵体百分比减伤 → 攻击方的双倍伤害判定/下一击加成 → 攻击方的持续性造成伤害加成 → 防守方的承受伤害加成 → `passive_dmg_reduce`固定-25% → 播放命中音效 → **封底到最低1点** → 护盾抵消并回血的判定 → 正式扣血 → 静电充能叠层 → 反击/荆棘反伤。返回值是这一下实际造成的伤害数字（被闪避/格挡/护盾抵消则是0），**记录日志时永远用这个返回值**，而不是宠物面板上原始的`damage`字段，因为返回值才反映了这一下实际发生了什么。

#### 5.2 "基础命中"和"附加效果"是分开的两步

放一个技能是**两个独立的步骤**，不是一个函数搞定：
1. 点击处理函数/`aiAct`先按通用流程打出这只宠物基础的`damage`/`heal`（这一步会检查`SPECIAL_DAMAGE_EFFECTS`，见下文）。
2. 然后调用`handlePetEffect(pet, sourceDeck, targetDeck, owner)`，去套用这只宠物具体的*附加*效果（眩晕、增益、持续伤害、强制切换等等）。

**`SPECIAL_DAMAGE_EFFECTS`**是一份需要跳过第1步的效果键列表，因为这些效果的伤害是由`handlePetEffect`自己用非通用的方式算出来并打出去的（随机取值、先吸血再回血、打中第二组、根本不立刻命中等等）。如果你新加的效果需要自定义的伤害算法，而不是简单打出`pet.data.damage`这个固定值，**一定要把它的键加进`SPECIAL_DAMAGE_EFFECTS`**，不然通用的第1步会在你自己的处理逻辑跑之前，先多打一次重复的普通伤害。

#### 5.3 `handlePetEffect`，效果分发表

这个函数是一长串从上到下依次判断的if/return（**不是**switch语句），第一个匹配上的分支处理完效果就直接返回。有几个互相有重叠的情况，顺序很重要（源码里有注释），最重要的一点是`self_dot`和`hot`是在通用的`isDot`分支**之前**判断的，因为这两者在数据库里都被标记成`isDot: true`，但实际上需要给*施法者自己*回血/加燃烧，而不是打在目标身上。

大致按判断顺序排列：

1. **天气类效果**（`WEATHER_EFFECTS`）→ `applyWeather`
2. `force_dance`（可能还带一个持续伤害的部分）
3. `self_dot` - 烧的是施法者自己，不是目标
4. `hot` - 随时间给施法者自己回血
5. 通用的`isDot` - 烧的是目标
6. `ENEMY_STATUS_EFFECTS`（眩晕/冰冻/创伤）→ 对目标调用`applyStatusEffect`
7. `SELF_STATUS_EFFECTS`（闪避/格挡）→ 对施法者自己调用`applyStatusEffect`
8. 接下来是针对剩余每个独立效果键各自一个`if`块：`revive`、`cleanse`、`slow_recharge`、`extend_cooldown`、`debuff_damage_dealt`、`SELF_BUFFS`（buff_damage_dealt / buff_chance_double_damage / buff_next_damage / buff_damage_and_speed / fast_recharge）、`random_damage`、`bonus_on_debuff`、`desperate`、`random_element`、`life_drain`、`self_damage`、`self_stun`、`delayed_damage`、`counter`/`thorns`、`doom`、`banish`、`absorb`、`ethereal`、`hit_both`、`mind_swap`、`force_partner_forward`、`consume_buff`、`force_swap`、`chain_on_kill`、`heal_back`、`elemental_bonus`、`summon`、`throw_teammate`、`dodge_swap`、`stack_burst`、`stacking_build`、`force_swap_transfer`、`random_skill_wrong_target`、`trap_swap`、`mess_up_undodgeable`。
9. **兜底分支：** 上面都没匹配上的，最后都会落到最后一行，`applyModifiers(targetDeck, pet.data, false, pet, ...)`，也就是一个通用的、打在敌方身上的减益。这也是为什么有些效果（比如`anti_swap`）在`handlePetEffect`里根本没有专门的`if`分支：它们足够简单，靠这个通用兜底就能处理，具体行为写在`applyModifiers`的switch语句里。

#### 5.4 `applyModifiers(deck, effectData, isSelf, sourcePet, owner)` 和 `applyStatusEffect(deck, effectData, sourcePet, owner)`

对于那些更简单、更统一的效果类型（不管是状态效果类还是加成类），这两个函数才是**真正干活的地方**。如果你要加的新效果是一个直白的"持续Y秒、+X%"的增益/减益，或者单纯的眩晕/冰冻/闪避/格挡，你应该把对应的case加在这两个函数的`switch`语句里，而不是加进`handlePetEffect`，`handlePetEffect`应该只处理那些真的需要自定义逻辑的特殊情况（同时打两组、随机数、条件分支等等）。

这两个函数都会自动记录日志（见第5.6节），凡是走这两个函数的效果，你在`handlePetEffect`那一层不需要再单独调用`addLog`。

**`owner`参数的约定（很重要，也很容易搞反）：** 在这两个函数里，`owner`指的永远是被修改的那个`deck`参数所属的一方，**不一定**是施法者。对于减益（`isSelf`/`isDebuff`为false），施法者其实是*另一方*；对于自增益或自身状态（闪避/格挡），施法者才等于`owner`。这两个函数开头都会专门算出一个本地的`casterOwner`/`casterType`变量，就是为了让日志的归属显示正确，建议照抄这个模式，不要想当然地认为`owner`就是"施法者"。

#### 5.5 `recomputeModifiers(deck)`

每次改动增益/减益数组之后都会调用这个函数。把这一组身上固定的数值字段先清零，然后把`modifiers.buffs`/`.debuffs`里每一条加成累加进`damageDealt`、`damageTaken`、`doubleDmgChance`、`nextDamage`、`messUpChance`、`rechargeSpeed`（乘法叠加，每有一个加速来源乘2倍，每有一个减速来源乘0.5倍，两者同时存在则连乘）、`canSwap`、`canAct`这些字段里。**如果你新加了一种增益/减益的`type`字符串，必须在这里加一个对应的`case`**，不然它就只是安静地待在数组里，什么效果都不会有。

#### 5.6 `tryResist` / `upsertModifierEntry`

- `tryResist(deck, owner)` - 「后背鸭子」（`passive_resist_negative`）唯一的实现位置：如果这一组带有这个被动、并且它没在冷却中，就完全挡掉即将到来的负面效果，同时让这个被动进入冷却。这个函数会在`applyStatusEffect`（针对眩晕/冰冻/创伤）的开头被调用，另外还在几处持续伤害/Toasties相关的分支里被直接调用，**没有**接入天气或`cleanse`净化的流程（如果你想把这个和wiki的说法对上，可以看[第一部分](#part-1-core-mechanics)第7/9节里提到的那处不一致）。
- `upsertModifierEntry(list, type, entry, isStackingPool)` - 增益/减益数组共用的"添加或刷新"逻辑。同一来源重复施加会刷新持续时间（对于会叠加的宠物，则是数值累加、持续时间取较长的那个）；不同来源则各自作为独立的数组条目共存。正是这段逻辑保证了多个不同的减益来源不会互相覆盖。

---

### 6. 技能释放、切换、天气、被动、AI

| 函数 | 作用 |
|---|---|
| `useAbility(player)` | 施法的把关函数：检查冷却/是否被禁用/是否被动，掷一次口误判定（失败的话记录"技能失败！"日志，同时依然套用那两条口误例外，Swoop的闪避和天气照样触发），然后重置冷却（会考虑Windspeed和已经蓄势待发的`extend_cd`负面效果），并把这一组其他已就绪的宠物锁上2秒。返回`true`/`false`；调用方只有在返回`true`时才会继续结算伤害/调用`handlePetEffect`。 |
| `swapDeck(player)` | 切换的把关函数：检查切换冷却/是否被冰冻/`canSwap`，先触发任何已经布在这个玩家身上的切换陷阱，清掉正在离场那一组身上的厄运，把`activeDeck`翻转过去，给新出战那一组设置3秒切换冷却，并锁住它已就绪的宠物2秒（如果那一组带Windspeed则跳过锁定）。 |
| `selectPet(player, index)` | 只是把`activePet`换到同一出战组里另一个（非空的）位置，不消耗冷却，也不触发任何效果结算。 |
| `applyWeather(effectData, sourcePet, owner)` | 根据施法宠物的字段，把全局唯一的`battle.weather`槽位设成火焰风暴/毒云/紫雾/舒缓迷雾/数羊中的一种，并记录施法日志。会覆盖掉当前已经生效的任何天气。 |
| `applyPassives(player)` | 每一刻都会运行，处理那些一直生效、和当前选中哪只宠物无关的被动：替补回血（`passive_heal_inactive`，每5刻一次）和替补自动攻击（`auto_attack_inactive`，每`pet.data.cooldown`刻一次）。`passive_dodge`/`passive_dmg_reduce`等**不在**这里处理，这些是在`dealDamage`内部针对当前出战的宠物直接判断的，因为它们只在命中那一刻才有意义。 |
| `getPassiveDesc(pet, deck)` | 被动图标的悬浮提示文案生成器，每个被动的`effect`键对应一个`case`。如果你加了新的被动效果，也要在这里加一条描述，不然它的提示文字就只会退回去显示技能名字。 |
| `aiAct()` | 每隔一刻（`battle.tick % 2 === 0`）为`battle.enemy`运行一次。优先级：出战宠物冷却好了就放技能 → 否则切换到同一组里另一只已就绪的宠物 → 否则如果备战组有能用的宠物就切换分组 → 否则什么都不做。 |
| `tryLivingDeadRevive(player, deadDeckNum)` | Living Dead Remote的复活判定（完整行为详见[第一部分第11节](#11-deck-death--revival)）。要求Remote待在刚阵亡那一组的*另一组*（还活着的那一组）；靠`player.livingDeadUsed`保证每场战斗只触发一次。 |
| `gameTick()` | 每秒一次的心跳函数，具体顺序见下面的分步说明。 |

#### 6.7 延迟效果队列

`battle`上有三个数组，专门存放那些施放时不会立刻结算的效果，每次`gameTick`都会往前推进一次：
- `delayedHits` - Zombie Hound的孢子爆炸；`ticksLeft`归零时打出固定伤害。
- `healBacks` - Phantom Pain的返还回血；把总量大致平均分摊到剩余的每一跳上。
- `swapTraps` - Lockjaw!/Surprise!的隐形陷阱；在*被瞄准*的那个玩家调用`swapDeck`的那一刻触发（造成额外伤害），而不是按自己的计时器触发（计时器只控制陷阱如果一直没被触发，能维持多久）。

如果你要加一个"稍后才发生"的新效果，尽量套用这三种模式里的一种，除非真的有必要，不要再发明第四种队列结构。

#### 6.8 `gameTick()`分步说明（顺序很重要）

1. `battle.tick`加一。
2. 天气跳一次（伤害/回血/进入睡眠，用的是天气自己内部的`tickRate`，和1秒一次的主时钟是分开的）。
3. 对双方的每一组：切换冷却、后背鸭子冷却、每只宠物的技能冷却都往下走一格（会被`rechargeSpeed`缩放）；每个状态效果的`remaining`倒计时往下走一格（眩晕/冰冻/睡眠会被`rechargeSpeed`缩放，其他一律固定减1）；结算持续伤害/持续回血的跳动（注意：这些是在`remaining`刚好倒数到0的那一跳触发的，比真正被移除早一拍，具体可参考[第一部分第7节](#7-status-effects-time-limited-tracked-per-deck)里关于"最后一跳"的小知识）；处理刚好倒数到0的厄运；清理已经到期的状态效果和增益/减益；调用`recomputeModifiers`。
4. 推进`delayedHits`、`healBacks`、`swapTraps`这三个队列。
5. 对双方都调用一次`applyPassives`。
6. 偶数刻调用`aiAct()`。
7. 各自检查阵亡：如果出战组血量为0，先尝试`tryLivingDeadRevive`；如果救不回来，且还有活着的备战组，就自动切换过去。
8. 计算`yDead`/`eDead`（两组是否都为0），只要有一方成立就结束战斗。
9. 调用`updateBattleUI()`。
10. 如果战斗还在继续，通过`setTimeout(gameTick, 1000)`重新安排自己下一次执行。

---

### 7. 日志系统

| 函数 | 作用 |
|---|---|
| `addLog(msg, type, owner)` | 在对应一方出战组区域下方弹出一条短暂的浮动提示（2.2秒后自动消失），同时转发给`addDetailedLog`。`type`可以是`'your-dmg'`（绿色）、`'enemy-dmg'`（红色），或者其他任意值（默认走蓝色/"回血"样式）。`owner`决定弹在哪一方（'your'/'enemy'），如果传`null`或其他值会默认落到敌方那一栏，所以一定要显式传一个明确的key。 |
| `addDetailedLog(msg, type, owner)` | 往可滚动的侧边日志面板里追加一条永久记录（最多保留最近100条），前面带上当前的游戏刻数和一个🟢/🔴标记（如果`owner`既不是`'your'`也不是`'enemy'`就留空，用于像天气跳这种不属于任何一方的全局事件）。 |
| `toggleDetailLog()` | 面板标题栏的点击处理函数，展开/收起详细日志的正文。 |

**约定：** 记录日志时永远用当时那个函数开头算出来的`casterOwner`/`casterType`（见第5.4节），不要直接拿原始的`owner`参数去做归属判断，因为它的含义会随着isSelf/isDebuff而反过来。日志文案里一定要带上实际的数值（伤害/回血/持续时间/百分比），具体可以参照贯穿第一部分的记录惯例；过去有不少bug都是因为漏了这一步，导致某个效果完全"悄无声息"地生效了却没人发现。

---

### 8. 界面渲染

下面这些函数都只是从`battle`里读数据、然后重新构建DOM，都不会修改战斗状态本身。

| 函数 | 作用 |
|---|---|
| `updatePassiveIndicators()` | 渲染每一方的被动图标行。大多数被动只在它所在的宠物待在*出战*组时才显示；`passive_heal_inactive`和`passive_revive`是例外，它们只在**待在替补**时才显示，正好对应它们实际的触发条件（另外`passive_revive`一旦`livingDeadUsed`变成true，还会额外显示一个"已使用"的角标）。 |
| `updateDebuffIndicators()` / `updateBuffIndicators()` | 根据`statusEffects`加上`modifiers.debuffs`/`.buffs`渲染减益/增益图标行，另外还有两个专属计数器（`stack_burst`的`charge`、`stacking_build`的`growStacks`）。如果你加了新的状态/加成`type`，要在这里的`names`/`typeNames`映射表里加上对应的显示名字，不然虽然效果照常生效，但不会显示图标（等于是"隐形"生效）。 |
| `updateWeatherUI()` | 根据`battle.weather`渲染顶部中央的天气角标（图标、名字、剩余秒数）。 |
| `updateBattleUI()` | 最核心的一个，重新构建双方的分组标签、状态/增益/减益标签行、宠物行（带点击事件，人类玩家实际调用`useAbility`/`handlePetEffect`就是通过它，见下文）、血条、备战组侧栏，以及切换按钮；然后再依次调用上面那四个函数。每次`gameTick()`之后、以及玩家每做一次手动操作之后都会调用这个函数。 |

**人类玩家的点击处理逻辑**就直接写在`updateBattleUI`渲染宠物行那部分代码里（大概在`pr.appendChild(el)`那个循环附近），它基本上是`aiAct()`施法逻辑的镜像，只是触发方式从AI的自动决策换成了DOM点击事件，同样会走第5.2节说的"通用命中，除非在`SPECIAL_DAMAGE_EFFECTS`里"加`handlePetEffect`这两步流程。

---

### 9. 教程：怎么添加一只新宠物

1. 按第2.1节的格式，往`Data/pet_battle_database.json`里加一条新记录。如果这只宠物的技能和某个已有机制完全一致（只是数值不同），直接复用那个已有的`effect`键就行，这种情况完全不需要改代码。
2. 把它的立绘放进`Sprites/`目录，文件名要对上`img`字段；音效同理放到`hitSound`/`powerSound`指定的位置（找不到图片会通过`onerror`自动退回`Sprites/0.png`；找不到音效则会被`playSound`的`.catch(() => {})`安静地忽略掉）。
3. 如果它需要一套全新的机制，请转去看第10节。
4. 就这样，组队界面、宠物列表、网格展示都是动态读取`allPets`的；不需要再改别的地方来让游戏"知道"这只宠物的存在。

### 10. 教程：怎么添加一个新的技能效果

1. **先找一个和你需求最接近、改动量最小的现成模式**，先去看`handlePetEffect`、`applyModifiers`或`applyStatusEffect`里对应的那部分代码，大多数新技能其实都是在某个已有效果上做变化（换个减益百分比、换个目标、多加一次附加伤害），而不是真正从零开始的全新东西。
2. **想清楚它该放在哪里：**
   - 简单的限时状态（禁止行动，或者必定闪避/格挡）→ 在`applyStatusEffect`的switch里加一个case，并把它的type加进`ENEMY_STATUS_EFFECTS`或`SELF_STATUS_EFFECTS`，让`handlePetEffect`能正确路由过去。
   - 简单的百分比/固定数值增益或减益 → 在`applyModifiers`的switch里加一个case（如果是自增益，还要加进`SELF_BUFFS`让`handlePetEffect`能路由过去），**并且**在`recomputeModifiers`里加上对应的case，不然存进去的数值不会真正产生任何效果。
   - 任何需要自定义伤害算法、同时打多组、随机数，或者条件分支的东西 → 直接在`handlePetEffect`里加一个新的`if (pet.data.effect === 'your_new_effect') { ...; return; }`代码块，照着现有的代码块抄模板就行。
3. **如果它的伤害不是走通用流程算出来的**（随机数值、打第二个目标、不会立刻命中、先吸血再回血等等），一定要把它的效果键加进`SPECIAL_DAMAGE_EFFECTS`，不然通用的点击/AI处理逻辑会在你的自定义代码跑之前，额外用`pet.data.damage`多打一次重复的固定伤害。
4. **如果它完全绕开了正常的伤害规则**（比如Toasties无视属性/闪避/格挡/增减益），就直接扣`deck.hp`来造成伤害，而不是调用`dealDamage`，除非未来会有多个效果都需要同样的绕过逻辑，否则不要为了这一个效果就往`dealDamage`里塞新的绕过标记。
5. **记得写日志。** 每个分支结尾都应该有一句`addLog(...)`，把实际涉及的数值（造成的伤害、生效的百分比、持续时间）写清楚，具体约定见第7节。这是加新效果时最常被漏掉的一步，没有之一。
6. **如果它是状态效果或者加成类型，记得加一个界面显示标签**，不要只写完`if`代码块就完事：
   - `updateDebuffIndicators`/`updateBuffIndicators`里的`names`/`typeNames`映射表（图标的悬浮提示）
   - 如果是被动，还要加进`getPassiveDesc`
7. **不要只测试正常情况，交互也要测到：** 如果它涉及回血，需不需要考虑创伤？如果是减益，需不需要考虑后背鸭子/`tryResist`？如果是限时效果，需不需要考虑`exoReduction`？行动前需不需要检查`canAct`/`canSwap`？它会不会和某些已有宠物的被动（Windspeed、Stubborn的`passive_dmg_reduce`、Soaring的闪避）产生wiki里特别提到的那种交互？记得对照[第二部分](#part-2-pet-ability-documentation)和[第一部分](#part-1-core-mechanics)，看看有没有和你这个新效果所属类别相关的已记录小知识/边界情况。

### 11. 速查：常量列表

| 常量 | 内容 | 用途 |
|---|---|---|
| `SELF_BUFFS` | `buff_damage_dealt, buff_chance_double_damage, buff_next_damage, buff_damage_and_speed, fast_recharge` | 在`handlePetEffect`里把这些效果键路由到`applyModifiers(sourceDeck, ..., isSelf=true, ...)` |
| `ENEMY_STATUS_EFFECTS` | `stun, freeze, trauma` | 路由到`applyStatusEffect(targetDeck, ...)` |
| `SELF_STATUS_EFFECTS` | `dodge, block` | 路由到`applyStatusEffect(sourceDeck, ...)` |
| `WEATHER_EFFECTS` | `weather, weather_hot, sleep` | 路由到`applyWeather(...)` |
| `SPECIAL_DAMAGE_EFFECTS` | `cleanse, random_damage, life_drain, counter, thorns, desperate, random_element, elemental_bonus, summon, stack_burst, stacking_build` | 让点击/AI处理逻辑跳过通用的固定伤害命中，好让`handlePetEffect`能用自己的方式结算伤害，避免重复打一次 |
| `ELEMENT_BEATS` | `{Fire: Earth, Earth: Air, Air: Water, Water: Fire}` | `dealDamage`里的属性克制查询表，基于`getDeckElement`的各种判断，以及`elemental_bonus`用到的表 |

---

### 12. 已知的一些粗糙之处（在"修复"它们之前最好先了解一下）

- `desperate`（垂死一击）和`random_skill_wrong_target`（附身）在源码里明确注释说是尽力而为的实现，wiki并没有给出这两者精确的数值/目标选取规则。如果你拿到了权威的准确数值，记得同时更新代码**和**[第二部分](#part-2-pet-ability-documentation)里对应宠物的条目。
- 天气目前**没法**被抵抗（`tryResist`）或者被净化（`cleanse`），尽管wiki暗示它应该可以，`battle.weather`是一个完全独立的全局对象，这两个函数都不会去检查它。如果想让它符合wiki的说法，需要在`tryResist`的调用处（加一个能识别天气的分支）和`cleanse`的清除逻辑里都做相应修改。
- Toasties（`summon`）完全绕过了`dealDamage`，所以也就跳过了紫雾的加成，但wiki上说紫雾是"Toasties无视一切"这条规则的唯一例外。如果想让代码符合这个说法，就需要在`handlePetEffect`的`summon`分支里手动套用紫雾的`fireDmgMod`/`otherDmgMod`，而不能指望`dealDamage`来处理。