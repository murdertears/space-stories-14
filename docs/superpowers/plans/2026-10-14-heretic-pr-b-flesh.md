# Еретик, подпроект 2, PR (б): путь Плоти. План реализации

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Путь Плоти целиком: 9 знаний от «Принципа Голода» до «Последнего гимна жреца», гули Хватки и Несовершенного ритуала, призывы Сырого пророка и Сталкера, «Сшить плоть», Вознесение во Владыку Ночи. Плюс видимые метки на целях для всех путей и долги PR (а).

**Architecture:** Плоть строится на общем слое PR (а) и повторяет шаблон пути Пепла:
- знания и ритуалы — прототипы `STHeretic*` с `StoriesHereticKnowledge` / `StoriesHereticRitual`;
- эффекты Хватки, клинка и метки — серверные компоненты на еретике, которые знания выдают через `addComponents` (`StoriesHereticGraspHitEvent`, `StoriesHereticBladeHitEvent`, `StoriesHereticMarkTriggeredEvent`);
- гули — `StoriesHereticGhoulSystem.TryRaise` (PR а);
- призывы — эффект ритуала `Summon` и база `STHereticSummonBase` (PR а);
- Владыка Ночи и облик Сталкера — `PolymorphSystem`.

Серверный код Плоти лежит в `Content.Server/_Stories/Heretic/Flesh/`.

**Tech Stack:** C# (RobustToolbox ECS), YAML-прототипы, Fluent, NUnit `GameTest`.

**Spec:** `docs/superpowers/specs/2026-10-14-heretic-flesh-void-blade-design.md` — §6 (Плоть), §4 (призывы и гули), §5.2 (расчленение), §11 (цель призывов), §15 (тесты). Основа — `docs/superpowers/specs/2026-10-13-heretic-core-design.md`.

## Global Constraints

- **Язык.** Комментарии в коде, коммиты и тексты для игрока на русском. Английские `name`/`description` в YAML и перевод в `Resources/Locale/ru-RU/ss14-ru/prototypes/_stories/heretic/entities.ftl` (`ent-<ID> = …`, `.desc = …`).
- **Коммиты** заканчиваются строкой `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Индексировать только изменённые файлы (никаких `git add -A`).
- **Стандарты.** `coding-standards.md` обязательны:
  - `[Dependency]` на приватных полях;
  - `Entity<T>` в обработчиках;
  - `ProtoId`/`EntProtoId` вместо строк, `static readonly` для констант-ID;
  - `[Dependency]` в компонентах не бывает;
  - поиск сущностей через сохранённый `HashSet`;
  - числа тюнинга — `DataField` в компонентах или событиях действий (из YAML), а не константы в коде.
- **Имена.** Код в `Content.{Shared,Server,Client}/_Stories/Heretic/` (код Плоти — в подпапке `Flesh/`). Типы с префиксом `StoriesHeretic`, прототипы — `STHeretic*`, действия — `STActionHeretic*`, тег пути `STHereticPathFlesh`.
- **Доступ к `StoriesHereticComponent`** ограничен `[Access(typeof(StoriesHereticSystem))]`: поля и методы коллекций меняет только `StoriesHereticSystem` (RA0002). Тесты начинаются с `#pragma warning disable RA0002`.
- **Клиентский код** не использует выражения коллекций в `Span<T>`. После клиентских изменений запускать Release-клиент и проверять песочницу.
- **Серверные компоненты в YAML** допустимы. Урон в YAML — только `types:`.
- **Спрайты и звуки tg** брать с коммита `ca79910d79497cf1ab56ebcf8db52a7289a3d412` и конвертировать `Tools/_Stories/dmi_to_rsi.py`. Скачанное класть во временную папку вне репо. Python запускать с `-I`, если не хватает Pillow — через venv в папке scratchpad. В `meta.json` писать `"license": "CC-BY-SA-3.0"` и `"copyright": "Taken from tgstation at https://github.com/tgstation/tgstation/commit/ca79910d79497cf1ab56ebcf8db52a7289a3d412"`. Звуки — запись в `Resources/Audio/_Stories/Heretic/attributions.yml`.
- **Стиль одобрен пользователем:** живые эффекты, частицы не по сетке, без тайловых спрайтов.
- **Цены Paradise:** всё по 1 очку; метка, улучшение клинка и Вознесение — по 2. Знания с `pathLocked: true`: Хватка, Метка, улучшение клинка.
- **Ступени Плоти** (`stage`): 1 Принцип Голода, 2 Хватка, 3 Несовершенный ритуал, 4 Метка, 5 Сшивание плоти, 6 Сырой ритуал, 7 Кровоточащая сталь, 8 Одинокий ритуал, 9 Вознесение.
- **Команды** (из корня репо). Перед сборкой остановить запущенные `Content.Server`/`Content.Client`: они держат `bin`.
  - сборка: `dotnet build -c Release`;
  - тесты еретика: `dotnet test Content.IntegrationTests/Content.IntegrationTests.csproj -c Release --no-build --filter "FullyQualifiedName~Tests._Stories.Heretic"`;
  - линтер: `dotnet run --project Content.YAMLLinter -c Release --no-build` (если ругается на неизвестный компонент — пересобрать линтер).

## Готовый API из PR (а) — использовать, не переписывать

| API | Где |
|---|---|
| `StoriesHereticGhoulSystem.TryRaise(EntityUid master, EntityUid corpse, ProtoId<ConversionPrototype> conversion, int limit, out string? reason)`, `CountGhouls(master, conversion)` | `Content.Server/_Stories/Heretic/Ghouls/` |
| Прототип обращения `STHereticGhoul` (25/50) — образец для новых | `Resources/Prototypes/_Stories/Heretic/ghouls.yml` |
| `StoriesHereticSystem.MakeServant(uid, master)`, `IsServant(uid, master?)`, `RegisterSummon(ent, NetUserId)` | `Content.Shared/_Stories/Heretic/StoriesHereticSystem*.cs` |
| Эффект ритуала `Summon` + `StoriesHereticSummonRitual { mob }`, база мобов `STHereticSummonBase` (GhostRole, роль `STMindRoleHereticMonster`) | `Content.Server/_Stories/Heretic/Summons/`, `summons.yml` |
| Ингредиент-лужа `reagents: [...]`, `reagentAmount` | `StoriesHereticIngredient` |
| Знание: `pathLocked`, `objectives: [STHereticObjectiveSummon]`, `requiredSummons` | `StoriesHereticKnowledgeComponent` |
| `StoriesHereticDismemberSystem.Dismember/TryDismember` | `Content.Server/_Stories/Heretic/StoriesHereticDismemberSystem.cs` |
| Шаблон пути: Пепел | `knowledge_ash.yml`, `rituals_ash.yml`, `actions_ash.yml`, `entities_ash.yml`, `ash_misc.yml`, `Content.Server/_Stories/Heretic/Ash/*` |

---

### Task 1: Долги PR (а) и видимые метки

**Files:**
- Modify: `Content.Shared/_Stories/Heretic/Cloak/StoriesHereticShadowCloakSystem.cs` (порядок подписки)
- Modify: `Content.Server/_Stories/Heretic/Organs/StoriesHereticCorruptedOrganSystem.Effects.cs` (обход копии набора)
- Modify: `Content.Server/_Stories/Heretic/Summons/StoriesHereticSummonSystem.cs`, `Resources/Prototypes/_Stories/Heretic/summons.yml` (вид призыва в ожидании)
- Modify: `Resources/Prototypes/_Stories/Heretic/antag.yml`, `objectives.yml` (антаг-прототип слуг, цель «Помогать хозяину»)
- Create: `Content.Client/_Stories/Heretic/StoriesHereticMarkVisualsSystem.cs`
- Modify: `Content.Shared/_Stories/Heretic/Components/StoriesHereticMarkComponent.cs` (если не сетевой — сделать сетевым)
- Create/Modify: `Resources/Textures/_Stories/Heretic/marks.rsi` (печати tg `emark1..emark8`)
- Test: `HereticCloakTest.cs`, `HereticSummonTest.cs`, новый `HereticMarkVisualsTest.cs`

**Interfaces:**
- Produces:
  - клиентская система рисует под помеченной целью печать пути по `StoriesHereticMarkComponent.Path`;
  - таблица «путь → state» лежит в клиенте рядом с палитрой (`StoriesHereticPalette` или `StoriesHereticMarkVisualsSystem`);
  - антаг-прототип `STHereticServant` (имя «Слуга Мансуса») для ролей `STMindRoleHereticMonster` и `STMindRoleHereticGhoul`;
  - объектив `STHereticObjectiveServeMaster` («Помогать хозяину», всегда выполнен).

Шаги:

- [ ] **Step 1: Плащ.** В `Initialize` убрать мёртвый ключ `typeof(InventorySystem)` из `after:` у подписки `SeeIdentityAttemptEvent`: тип абстрактный, порядок по нему не работает. Оставить `typeof(IdentitySystem)` и комментарий о том, на чём держится порядок. В `HereticCloakTest.CloakHidesAndBreaksOnDamage` добавить случай с маской и ID: надеть `ClothingMaskGas` (или другую маску с `IdentityBlocker`) вместе с ID-картой. Имя под плащом должно быть «Неизвестный».
- [ ] **Step 2: Органы.** В обработчике взрыва органа (`Effects.cs`, цикл по `_burstTargets` с `TryChangeDamage`) обходить копию набора (`foreach (var t in _burstTargets.ToArray())`) либо собрать цели в локальный список до нанесения урона. Админ-лог писать до цикла.
- [ ] **Step 3: Призыв в ожидании игрока** (спецификация §4.1: «полупрозрачный и неподвижный»).
  - При спавне в `OnRitualEffect` добавлять неподвижность: компонент `StoriesHereticSummonWaitingComponent` с подписками `UpdateCanMoveEvent` → `Cancel`, `AttackAttemptEvent` → `Cancel`, `InteractionAttemptEvent` → `Cancelled = true`. Сверить имена событий по `Content.Shared/ActionBlocker/`.
  - Прозрачность: сделать компонент сетевым, а на клиенте ставить `sprite.Color = Color.White.WithAlpha(0.5f)` при старте и возвращать при снятии. Клиентскую систему положить в `Content.Client/_Stories/Heretic/`.
  - Снимать компонент в `OnMindAdded`.
  - Тест в `HereticSummonTest`: у невостребованного призыва есть компонент ожидания, после `TransferTo` разума его нет.
- [ ] **Step 4: Антаг-прототип слуг и цель.**
  - В `antag.yml` добавить:

```yaml
- type: antag
  id: STHereticServant
  name: stories-roles-antag-heretic-servant-name
  antagonist: true
  setPreference: false
  objective: stories-roles-antag-heretic-servant-objective
```

  - У `STMindRoleHereticMonster` и `STMindRoleHereticGhoul` поменять `antagPrototype: STHeretic` на `STHereticServant`.
  - Цель `STHereticObjectiveServeMaster`: сущность-объектив с компонентом, у которого прогресс всегда 1. Сверить, как это сделано в апстриме (например, `DieCondition` или «Escape» с `Progress = 1`); если готового нет — свой `StoriesHereticServeMasterConditionComponent` в `StoriesHereticObjectiveSystem` с `args.Progress = 1f`.
  - Выдавать цель слугам: в `StoriesHereticSummonSystem.OnMindAdded` и в `StoriesHereticGhoulSystem.TryRaise` после обращения — `_mind.TryAddObjective(mindId, mind, "STHereticObjectiveServeMaster")` через `static readonly EntProtoId`.
  - Локаль (ru/en) в `Resources/Locale/{ru-RU,en-US}/_stories/heretic.ftl`:
    - `stories-roles-antag-heretic-servant-name = Слуга Мансуса`;
    - `stories-roles-antag-heretic-servant-objective = Служите еретику, который вас призвал.`;
    - заголовок цели `stories-heretic-objective-serve-master = Помогать хозяину.`
  - Добавить `ent-`-строки.
  - Тест: после захвата призыва в целях разума есть `STHereticObjectiveServeMaster`.
- [ ] **Step 5: Видимые метки.**
  - Скачать tg `icons/effects/eldritch.dmi` (сверить путь по дереву коммита; ищите states `emark1`…). Сконвертировать `emark1`–`emark8` в `Resources/Textures/_Stories/Heretic/marks.rsi`.
  - Сделать `StoriesHereticMarkComponent` сетевым, если он ещё не сетевой (`[NetworkedComponent, AutoGenerateComponentState]`, `Path` — `[AutoNetworkedField]`).
  - Клиентская система на `ComponentStartup`/`AfterAutoHandleStateEvent`/`ComponentShutdown` добавляет и убирает слой спрайта `STHereticMark` с RSI `marks.rsi`. State по пути: Ash → `emark2`, Flesh → `emark1`, Void → `emark4`, Blade → `emark5`, Rust → `emark3`, Cosmos → `emark6`, Lock → `emark7`, Moon → `emark8` (сверить номера по описаниям state в tg `eldritch.dmi`; если номера не совпадают — выбрать по виду и записать в отчёт).
  - Слой рисовать под мобом: `sprite.LayerSetOffset` не нужен, `DrawDepth` слоя через отдельную сущность-оверлей не делать — достаточно слоя с индексом 0 через `sprite.AddLayer(..., 0)`. Сверить, как это сделано в апстримных визуализаторах статусов (например, `Content.Client/Stunnable/` или `SmokingVisualizer`).
  - Тест `HereticMarkVisualsTest` (клиентский): RSI `marks.rsi` содержит все 8 states.
- [ ] **Step 6: Проверка.** Сборка, тесты еретика, линтер, Release-клиент (песочница).
- [ ] **Step 7: Коммиты** — по смыслу, например «долги PR (а)» и «видимые метки».

---

### Task 2: Ресурсы Плоти из tg

**Files:**
- Create: `Resources/Textures/_Stories/Heretic/flesh_blade.rsi`, `flesh_blade_inhand.rsi`, `raw_prophet.rsi`, `stalker.rsi`, `lord_of_night.rsi`
- Modify: `Resources/Textures/_Stories/Heretic/knowledge.rsi` (+ `grasp_flesh`, `blade_upgrade_flesh`, `ghoul_voiceless`, иконки знаний призывов)
- Modify: `Resources/Textures/_Stories/Heretic/actions.rsi` (+ `mad_touch`, иконки действий пророка, Сталкера и Владыки Ночи)
- Create: `Resources/Audio/_Stories/Heretic/ascend_flesh.ogg` и звуки «Сшить плоть»; `attributions.yml`
- Test: `HereticKnowledgeUiTest` (иконки знаний проверяются автоматически) и `EntityTest`

- [ ] **Step 1: Найти пути** в дереве коммита tg: `gh api "repos/tgstation/tgstation/git/trees/ca79910d79497cf1ab56ebcf8db52a7289a3d412?recursive=1" --jq '.tree[].path' | grep -Ei "khopesh|64x64_(left|right)hand|eldritch_mobs|actions_ecult|ui_icons/antags/heretic/knowledge|ascend_flesh|staff_healing|dismember|slice"`.
- [ ] **Step 2: Скачать и сконвертировать** во временную папку вне репо:
  - `khopesh.dmi:flesh_blade` → `flesh_blade.rsi/icon` (анимация 11 кадров);
  - `mob/inhands/64x64_{left,right}hand.dmi:flesh_blade` → `flesh_blade_inhand.rsi` (`inhand-left`/`inhand-right`, 64×64). Сверить имена state с `ash_blade_inhand.rsi`;
  - `knowledge.dmi: grasp_flesh, blade_upgrade_flesh, ghoul_voiceless` → в `knowledge.rsi`;
  - `eldritch_mobs.dmi: raw_prophet` → `raw_prophet.rsi`;
  - `eldritch_mobs.dmi: stalker` (4 направления) → `stalker.rsi`;
  - `eldritch_mobs.dmi: armsy_start` (4 направления) → `lord_of_night.rsi` (state `icon`);
  - `actions_ecult.dmi`: `mad_touch`, `worm_ascend` и иконки пророка и Сталкера (`jaunt`/`ash_shift`, `blind`, `eye`, `manse_link` и др. — по списку states в dmi; взять ближайшие по смыслу и записать в отчёт) → в `actions.rsi`;
  - звуки: `sound/music/antag/heretic/ascend_flesh.ogg`, `sound/items/weapons/slice.ogg` (или где лежит), `sound/effects/dismember.ogg`, `sound/effects/magic/staff_healing.ogg` → `Resources/Audio/_Stories/Heretic/`.
- [ ] **Step 3: Атрибуция** во всех новых и изменённых `meta.json` и в `attributions.yml`.
- [ ] **Step 4: Проверка:** сборка, `HereticKnowledgeUiTest`, линтер.
- [ ] **Step 5: Коммит** «feat(heretic): спрайты и звуки пути Плоти из tg».

---

### Task 3: Принцип Голода и Кровавый клинок

**Files:**
- Modify: `Resources/Prototypes/_Stories/Heretic/tags.yml` (`STHereticPathFlesh`)
- Create: `Resources/Prototypes/_Stories/Heretic/knowledge_flesh.yml`, `rituals_flesh.yml`, `entities_flesh.yml`
- Modify: `Resources/Locale/{ru-RU,en-US}/_stories/heretic.ftl` (`stories-heretic-path-STHereticPathFlesh = Плоть` / `Flesh`)
- Create: `Resources/Locale/{ru-RU,en-US}/_stories/heretic_flesh.ftl`
- Modify: `entities.ftl` (ru)
- Test: `Content.IntegrationTests/Tests/_Stories/Heretic/HereticFleshTest.cs`

**Interfaces:**
- Produces: знание `STHereticKnowledgeFleshRoot`, ритуал `STHereticRitualFleshBlade`, клинок `STHereticBladeFlesh`.

- [ ] **Step 1: Тест** `HereticFleshTest.FleshRootGrantsBladeAndSummonObjective`:
  1. Еретику с разумом (как в `HereticProgressTest.KnowledgeAddsSummonObjective`) изучить `STHereticKnowledgeFleshRoot`. Проверить: `Path == STHereticPathFlesh`, `RequiredSummons == 2`, в целях разума есть `STHereticObjectiveSummon`, в ритуалах есть `STHereticRitualFleshBlade`.
  2. На руне нож и лужа крови 10 ед. (`TrySpillAt(..., new Solution("Blood", 10))`) → `TryPerform` true, `CountLimitedResults == 1`.
  3. Повторять до 4 клинков — 5-й ритуал отказывает.
- [ ] **Step 2: Запустить и убедиться, что тест падает.**
- [ ] **Step 3: YAML.** Знание:

```yaml
- type: entity
  parent: STHereticKnowledgeBase
  id: STHereticKnowledgeFleshRoot
  name: Principle of Hunger
  description: Step onto the Path of Flesh. Learn to forge bloody blades from a knife and a pool of blood. You will also be tasked with summoning monsters.
  components:
  - type: StoriesHereticKnowledge
    icon:
      sprite: _Stories/Heretic/flesh_blade.rsi
      state: icon
    path: STHereticPathFlesh
    choosesPath: true
    stage: 1
    requiredSummons: 2
    objectives:
    - STHereticObjectiveSummon
    rituals:
    - STHereticRitualFleshBlade
```

  Ритуал (лимит Paradise — 4):

```yaml
- type: entity
  parent: STHereticRitualBase
  id: STHereticRitualFleshBlade
  name: Bloody Blade
  description: A knife and a pool of blood.
  components:
  - type: StoriesHereticRitual
    limit: 4
    ingredients:
    - name: stories-heretic-ingredient-knife
      whitelist:
        tags: [ Knife ]
    - name: stories-heretic-ingredient-blood
      reagents: [ Blood, InsectBlood, CopperBlood, AmmoniaBlood, Slime ]
    results:
    - STHereticBladeFlesh
```

  Состав крови сверить с реагентами в `Resources/Prototypes/Reagents` (оставить существующие).

  Клинок: как `STHereticBladeAsh`, но спрайты Плоти, `Slash: 17`. Описание из tg: «Серповидный клинок, рождённый из изуродованного плотью существа…». Кровавые следы клинка (tg `blood_walk`) не делаем: в SS14 следы крови даёт криминалистика испачканных предметов. Сообщение при ломке «Маршал слышит твой зов…» — проверить, есть ли в `StoriesHereticBladeComponent` поле сообщения; если нет — пропустить и записать в отчёт.
- [ ] **Step 4: Локаль.** Тексты знаний — переводы tg из спецификации §6:
  - gain_text «Сотни из нас голодали, но не я... Я нашёл силу в своей жадности.»;
  - название пути «Плоть».
- [ ] **Step 5: Тесты еретика (включая UI-тест иконок), линтер.**
- [ ] **Step 6: Коммит** «feat(heretic): Принцип Голода и Кровавый клинок».

---

### Task 4: Хватка Плоти — гуль из трупа

**Files:**
- Create: `Content.Server/_Stories/Heretic/Flesh/StoriesHereticGraspGhoulComponent.cs`
- Create: `Content.Server/_Stories/Heretic/Flesh/StoriesHereticFleshSystem.cs` (partial; сюда же пойдут Метка и Сталь)
- Modify: `knowledge_flesh.yml`, `heretic_flesh.ftl`
- Test: `HereticFleshTest.cs`

**Interfaces:**
- Produces:
  - `StoriesHereticGraspGhoulComponent { ProtoId<ConversionPrototype> Conversion = "STHereticGhoul"; int Limit = 3; int AscendedBonus = 3 }`;
  - знание `STHereticKnowledgeFleshGrasp`.

- [ ] **Step 1: Тест** `GraspOfFleshRaisesGhoul`. Еретик Плоти с `STHereticKnowledgeFleshGrasp` бьёт Хваткой мёртвого гуманоида с разумом: поднять событие `StoriesHereticGraspHitEvent(heretic, corpse)` на еретике (`SEntMan.EventBus.RaiseLocalEvent(heretic, ref ev)`). Ожидание:
  - труп жив, `IsServant(corpse, heretic)`;
  - 4-й гуль при лимите 3 — отказ;
  - живой гуманоид не обращается.
- [ ] **Step 2: Запустить и убедиться, что тест падает.**
- [ ] **Step 3: Код.** Подписка `SubscribeLocalEvent<StoriesHereticGraspGhoulComponent, StoriesHereticGraspHitEvent>(OnGraspGhoul)`:

```csharp
    private void OnGraspGhoul(Entity<StoriesHereticGraspGhoulComponent> ent, ref StoriesHereticGraspHitEvent args)
    {
        // По живым Хватка работает как обычно, гулем становится только труп.
        if (!_mobState.IsDead(args.Target))
            return;

        var limit = ent.Comp.Limit + (TryComp<StoriesHereticComponent>(ent, out var heretic) && heretic.Ascended ? ent.Comp.AscendedBonus : 0);
        if (_ghouls.TryRaise(ent, args.Target, ent.Comp.Conversion, limit, out var reason))
            _popup.PopupEntity(Loc.GetString("stories-heretic-flesh-ghoul-raised", ("target", args.Target)), args.Target, ent);
        else
            _popup.PopupEntity(reason, args.Target, ent);
    }
```

  Знание (stage 2, cost 1, `pathLocked: true`, `requires: [ STHereticKnowledgeFleshRoot ]`, `addComponents: [ { type: StoriesHereticGraspGhoul } ]`, icon `knowledge.rsi:grasp_flesh`). Описание Paradise: «Хватка Мансуса поднимает мёртвых гулями — один раз на труп. Гули иссохшие и хрупкие (25 здоровья)».
- [ ] **Step 4: Тесты и коммит** «feat(heretic): Хватка Плоти поднимает гулей».

---

### Task 5: Несовершенный ритуал — Безгласный мертвец

**Files:**
- Modify: `Content.Shared/_Stories/Heretic/Components/StoriesHereticRitualComponent.cs` (эффект `Ghoul`)
- Create: `Content.Server/_Stories/Heretic/Ghouls/StoriesHereticGhoulRitualComponent.cs`
- Modify: `Content.Server/_Stories/Heretic/Ghouls/StoriesHereticGhoulSystem.cs` (обработчик эффекта, призрачная роль на бездушном трупе)
- Modify: `Resources/Prototypes/_Stories/Heretic/ghouls.yml` (`STHereticGhoulVoiceless`), `rituals_flesh.yml`, `knowledge_flesh.yml`
- Test: `HereticFleshTest.cs`

**Interfaces:**
- Produces:
  - эффект `StoriesHereticRitualEffect.Ghoul`;
  - `StoriesHereticGhoulRitualComponent { ProtoId<ConversionPrototype> Conversion; int Limit; int AscendedBonus; TimeSpan OfferTime = 5 мин }`;
  - обращение `STHereticGhoulVoiceless` (50/100, немой);
  - знание `STHereticKnowledgeFleshImperfect`.
  - Расколотый ритуал в PR (г) использует тот же эффект с другим обращением.

Поведение (спецификация §6.3):
- Ритуал ищет на руне мёртвого гуманоида без `StoriesHereticRaisedComponent`.
- **С душой** — `TryRaise` сразу.
- **Без души** (нет разума с `UserId`):
  - на труп ставится призрачная роль (`GhostRole` с `name`/`description`/`rules` из локали, `GhostTakeoverAvailable`) и маркер `StoriesHereticGhoulOfferComponent { Master, Conversion, Limit, ExpireTime }`;
  - на `MindAddedMessage` трупа — `TryRaise`, маркер и роль снимаются;
  - через 5 мин без игрока маркер и роль снимаются, мак не возвращается (это запись «труп остаётся трупом» в спецификации; поправить в спецификации §6.3: «мак возвращается» → «мак тратится», с пометкой в отчёте).
- Мак — ингредиент `prototypes: [ FoodPoppy ]` (сверить ID в `Resources/Prototypes/Entities/Objects/Consumable/Food/` или `Hydroponics`). Его поглощает ядро.
- Труп не поглощается (ингредиент с `consume: false` не нужен: тело берёт обработчик эффекта, не через `ingredients`).
- `STHereticGhoulVoiceless`: как `STHereticGhoul`, но `critThreshold: 50`, `deadThreshold: 100` и компонент немоты. Сверить, как в SS14 делают постоянную немоту: `Muted` (`MutedComponent`) или бессрочный статус `StatusEffectMuted`. Компонентом удобнее, его снимет обращение.
- Лимит 2, на Вознесении +4.

- [ ] **Step 1: Тесты.**
  - `ImperfectRitualRaisesVoiceless`: труп с душой и мак → гуль немой, пороги 50/100.
  - `ImperfectRitualOffersSoullessCorpse`: труп без разума → на нём `GhostRole`. После `TransferTo` нового разума с UserId тестового игрока (см. `HereticGhoulTest.RaiseReturnsGhostedPlayer` про UserId) → гуль.
  - Лимит 2.
- [ ] **Step 2–4:** код, проверка, коммит «feat(heretic): Несовершенный ритуал — Безгласные мертвецы».

---

### Task 6: Метка Плоти и Кровоточащая сталь

**Files:**
- Create: `Content.Server/_Stories/Heretic/Flesh/StoriesHereticFleshMarkComponent.cs`, `StoriesHereticBladeBleedComponent.cs`
- Modify: `Content.Server/_Stories/Heretic/Flesh/StoriesHereticFleshSystem.cs`, `knowledge_flesh.yml`
- Test: `HereticFleshTest.cs`

**Interfaces:**
- Produces:
  - `StoriesHereticFleshMarkComponent { float Bleed = 8; DamageSpecifier Damage (Slash 10) }`;
  - `StoriesHereticBladeBleedComponent { float Bleed = 4 }`;
  - знания `STHereticKnowledgeFleshMark` (stage 4, cost 2, pathLocked, `addComponents: [StoriesHereticGraspMark { path: STHereticPathFlesh }, StoriesHereticFleshMark]`) и `STHereticKnowledgeFleshBleedingSteel` (stage 7, cost 2, pathLocked, icon `blade_upgrade_flesh`).

- [ ] **Step 1: Тест.**
  - Метка: поднять `StoriesHereticMarkTriggeredEvent(heretic, target, STHereticPathFlesh, 0)` broadcast. У цели выросло кровотечение (`BloodstreamComponent.BleedAmount`) на 8, есть Slash.
  - Сталь: поднять `StoriesHereticBladeHitEvent(heretic, target, blade)` на еретике. Кровотечение +4, по мёртвым — без эффекта.
- [ ] **Step 2–4:** код по образцу `StoriesHereticAshSystem.OnMarkTriggered` и `OnBladeFireHit`. Использовать `BloodstreamSystem.TryModifyBleedAmount` (`Content.Shared/Body/Systems/BloodstreamSystem.cs:513`) и `DamageableSystem.TryChangeDamage`. Коммит «feat(heretic): Метка Плоти и Кровоточащая сталь».

---

### Task 7: Сшивание плоти — «Сшить плоть»

**Files:**
- Create: `Content.Server/_Stories/Heretic/Flesh/StoriesHereticKnitFleshComponent.cs`, `StoriesHereticFleshSystem.Knit.cs`
- Modify: `Content.Shared/_Stories/Heretic/Events/StoriesHereticActionEvents.cs` (`StoriesHereticKnitFleshEvent : EntityTargetActionEvent`), `StoriesHereticEvents.cs` (DoAfter-событие извлечения)
- Modify: `actions.yml` → новый `actions_flesh.yml`, `knowledge_flesh.yml`, `heretic_flesh.ftl`
- Test: `HereticFleshTest.cs`

**Interfaces:**
- Produces:
  - действие `STActionHereticKnitFlesh` (касание, перезарядка 20 с, требует фокус, шёпот «CL'M M'N!» через `Speech`/`ChatSystem` как у Пепла, если там есть заклинательные крики — иначе без крика);
  - компонент на еретике `StoriesHereticKnitFleshComponent { TimeSpan CorpseTime = 3 с; LivingTime = 15 с; HealBrute = 10; HealBurn = 5 }`;
  - знание `STHereticKnowledgeFleshKnitting` (stage 5, cost 1, icon `actions.rsi:mad_touch`).

Решение контроллера: извлечение органов через глаголы, а не радиальное меню. Глаголы повторяют UX расчленения из PR (а) и не требуют нового BUI. Поправить в спецификации §6.5 «радиальное меню» → «глаголы «Извлечь: {орган}»» в этом же коммите.

Поведение:
- **Действие по своему слуге** (`IsServant(target, heretic)`, жив): лечение 10 Brute + 5 Burn (распределить по `Blunt`/`Slash`/`Piercing` и `Heat` через `DamageSpecifier` с отрицательными значениями или `DamageableSystem.HealDistributed`, сверить API), звук `staff_healing`. По другим целям действие ничего не делает и не тратит перезарядку (`args.Handled = false`).
- **Глаголы** на гуманоиде с `BodyComponent`, если у пользователя есть `StoriesHereticKnitFleshComponent` и фокус. Для каждого органа в `body_organs` категорий `Eyes`, `Tongue`, `Heart`, `Lungs`, `Liver`, `Stomach`, `Appendix`, `Kidneys`, `Ears`:
  - из живых — без `Heart` и `Lungs`;
  - мозг — никогда;
  - текст «Извлечь: {название}» через ключи `stories-heretic-knit-part-<Category>` в винительном падеже: «глаза», «язык», «сердце», «лёгкие», «печень», «желудок», «аппендикс», «почки», «уши».
- **DoAfter** 3 с по трупу и 15 с по живому, `BreakOnMove`, `DistanceThreshold`. По завершении перепроверить условия, `_containers.Remove(organ, container)`, орган кладётся на пол и в руку, если она свободна.
- **Живая цель:** при начале извлечения кричит (`ChatSystem.TryEmoteWithChat(target, "Scream")`), получает невнятную речь на 15 с (`StatusEffectSlurredSpeech` или как в `StoriesHereticGraspSystem.SlurredEffect`).
- **Испорченный орган** при извлечении может взорваться (PR а, `OnRemoved`) — после `Remove` проверять `TerminatingOrDeleted(organ)`.

- [ ] **Step 1: Тесты.**
  - Слуга с уроном 30 Blunt после действия получает −10.
  - Глагол «Извлечь: печень» на трупе есть, на живом нет «Извлечь: сердце».
  - Прямой вызов метода извлечения достаёт орган из `body_organs`.
- [ ] **Step 2–4:** код, проверка, коммит «feat(heretic): Сшить плоть — лечение слуг и извлечение органов».

---

### Task 8: Ингредиент «отделённая конечность» и Сырой ритуал

**Files:**
- Modify: `Content.Shared/_Stories/Heretic/StoriesHereticRitualSystem.cs` (`Matches`: `organCategory` совпадает и с отделённой частью тела)
- Create: `Resources/Prototypes/_Stories/Heretic/summons_flesh.yml` (моб `STHereticMobRawProphet`)
- Create: `Content.Server/_Stories/Heretic/Flesh/StoriesHereticFocusedAttackComponent.cs` (+ обработчик)
- Modify: `rituals_flesh.yml`, `knowledge_flesh.yml`, `actions_flesh.yml`, `ash_misc.yml` (длинный эфирный прыжок)
- Test: `HereticFleshTest.cs`, `HereticRitualTest.cs`

**Interfaces:**
- Produces:
  - ингредиент `organCategory: ArmLeft` засчитывает отделённую левую руку. После `DetachableOrganSystem.Detach` это сущность `DetachedBody` с органом в контейнере `body_organs` — сверить, что возвращает `Detach`, и сопоставлять сущность, у которой в `body_organs` есть орган нужной категории и нет `MobState`;
  - моб `STHereticMobRawProphet`, знание `STHereticKnowledgeFleshRaw` (stage 6, cost 1), ритуал `STHereticRitualRawProphet` (глаза + левая рука + лужа крови, эффект `Summon`, `StoriesHereticSummonRitual { mob: STHereticMobRawProphet }`).

Сырой пророк (спецификация §6.6):
- Родители `[ BaseSimpleMob, MobCombat, STHereticSummonBase ]` — по образцу `Resources/Prototypes/_Stories/Entities/Mobs/Demons/shadow_demon.yml`.
- Здоровье: `MobThresholds` 0 Alive / 65 Dead.
- Набор модификаторов урона `STHereticSummonDamageModifierSet`: Blunt/Slash/Piercing/Heat/Cold/Shock ×0.7, Poison/Asphyxiation ×0. Его используют все призывы Плоти.
- Удар `MeleeWeapon` Blunt 7.
- `StoriesHereticFocusedAttackComponent { Bonus = 2; MaxBonus = 10 }`: каждый следующий удар по той же цели +2 (до +10), смена цели сбрасывает бонус. Реализация — подписка на `GetMeleeDamageEvent` (сверить имя) или `MeleeHitEvent` с `BonusDamage`.
- Тепловое зрение — `VisionProvider` как у `STHereticAshenEyes`.
- `GhostRole` `name`/`description` из локали: «Сырой пророк» / «Мерзость, сшитая из нескольких отрубленных рук и одного распухшего осиротевшего глаза…».
- Спрайт `raw_prophet.rsi`.
- При смерти роняет ингредиенты — через ядро призывов.
- Действия (только у моба, без `StoriesHereticRequiresFocus`):
  - «Эфирный прыжок»: как `STActionHereticAshPassage`, но полиморф `STHereticAshJauntLong` (вдвое дольше, сверить `ash_misc.yml`);
  - «Ослепление»: `EntityTargetAction`, дальность 10, перезарядка 20 с, `StatusEffectBlindness` 10 с, событие `StoriesHereticEldritchBlindEvent { Duration }`, обработчик в `StoriesHereticFleshSystem`;
  - Связь Мансуса и Расширение взора — Task 9.

- [ ] **Step 1: Тесты.**
  - `HereticRitualTest`: ритуал с `organCategory: ArmLeft` проходит с отделённой рукой (`Dismember` трупа → рука на руне).
  - `HereticFleshTest`: Сырой ритуал (глаза из трупа через `Remove`, рука, кровь) призывает пророка с `StoriesHereticSummon`; бонус фокус-атаки растёт.
- [ ] **Step 2–4:** код, проверка, коммит «feat(heretic): Сырой ритуал — Сырой пророк».

---

### Task 9: Сырой пророк — Связь Мансуса и Расширение взора

**Files:**
- Create: `Resources/Prototypes/_Stories/Heretic/radio_flesh.yml` (канал `STHereticMansusLink`) — или в `Resources/Prototypes/_Stories/radio_channels.yml`, если там все каналы форка
- Create: `Content.Server/_Stories/Heretic/Flesh/StoriesHereticMansusLinkComponent.cs`, `StoriesHereticFleshSystem.Link.cs`
- Modify: `actions_flesh.yml`, `summons_flesh.yml`, `heretic_flesh.ftl`
- Test: `HereticFleshTest.cs`

Поведение (спецификация §6.6):
- **Канал** `STHereticMansusLink`: цвет `#568b00`, ключ `:з`/`:m` — сверить свободный ключ по `radio_channels.yml`. Слышат его только участники: пророк получает канал при спавне, хозяин — при захвате пророка игроком.
- **Связь Мансуса** — `EntityTargetAction`, дальность 7, перезарядка 20 с, DoAfter 6 с.
  - По завершении цели даются `IntrinsicRadioReceiver`/`IntrinsicRadioTransmitter`/`ActiveRadio` с каналом и маркер `StoriesHereticMansusLinkComponent { Prophet }`. Так же, как каналы выдаёт `ConversionSystem.DoConvert`.
  - Майндшилд и святость (`StoriesHereticGraspSystem.IsProtected` или `HolySystem`) блокируют связь.
- **Разрыв:** смерть или удаление пророка снимает канал со всех связанных. Каждый связанный кричит и получает паралич 0,5 с.
- **Расширение взора** — `InstantAction` с переключением зума камеры ×1,5 и перезарядкой 8 с. Сверить API зума: `SharedContentEyeSystem.SetZoom` / `ContentEyeComponent.TargetZoom`.

- [ ] **Step 1: Тест.**
  - Связь даёт цели канал; смерть пророка снимает канал и даёт паралич.
  - Цель с майндшилдом (`MindShieldComponent`) не связывается.
- [ ] **Step 2–4:** код, проверка, коммит «feat(heretic): Связь Мансуса и Расширение взора Сырого пророка».

---

### Task 10: Одинокий ритуал — Сталкер

**Files:**
- Modify: `summons_flesh.yml` (моб `STHereticMobStalker`, полиморфы облика), `rituals_flesh.yml`, `knowledge_flesh.yml`, `actions_flesh.yml`
- Create: `Content.Shared/_Stories/Heretic/Flesh/StoriesHereticStalkerFormComponent.cs` + сообщения BUI
- Create: `Content.Server/_Stories/Heretic/Flesh/StoriesHereticFleshSystem.Stalker.cs`
- Create: `Content.Client/_Stories/Heretic/UI/StoriesHereticStalkerFormBoundUserInterface.cs` — по образцу `Content.Client/Changeling/UI/ChangelingTransformBoundUserInterface.cs` (`SimpleRadialMenu`)
- Test: `HereticFleshTest.cs`

Сталкер (спецификация §6.7):
- **Ритуал** (Paradise): 3 куска мяса (`FoodMeat`, сверить ID) + ручка (тег `Write`) + лист бумаги (`Paper`, сверить) → `Summon` Сталкера. Знание `STHereticKnowledgeFleshLonely` (stage 8, cost 1).
- **Моб:**
  - 150 HP, удар Slash 17, тепловое зрение, `STHereticSummonDamageModifierSet`;
  - спрайт `stalker.rsi` с 4 направлениями;
  - `GhostRole` «Сталкер» — «Мерзость, собранная из разных останков. Её облик чуть меняется всякий раз, когда вы моргаете».
- **Действие ЭМИ:** `InstantAction`, перезарядка 30 с, `SharedEmpSystem.EmpPulse(coords, 6f, energyConsumption: 50000f, duration: 15 с)` (`Content.Shared/Emp/SharedEmpSystem.cs:71`), событие `StoriesHereticEldritchEmpEvent { Range, Energy, Duration }`.
- **Эфирный прыжок** — тот же, что у пророка.
- **Смена облика:**
  - при первом использовании открывается радиальное меню из пяти вариантов: кот, корги, мышь, лиса, карп (`MobCat`, `MobCorgi`, `MobMouse`, `MobFox`, `MobCarp`);
  - выбор записывается в `StoriesHereticStalkerFormComponent.Chosen` (сетевой);
  - дальше действие превращает в этот облик через `PolymorphSystem.PolymorphEntity(uid, config)` с `Forced = false`, `TransferDamage = true`, `RevertOnDeath = true`, `AllowRepeatedMorphs = true`. Обратно — штатное действие возврата полиморфа;
  - полиморфы `STHereticStalkerFormCat` … `…Carp` в `summons_flesh.yml`.

- [ ] **Step 1: Тесты.** Ритуал призывает Сталкера. ЭМИ-событие вызывает `EmpPulse` — проверить, что в радиусе `BatteryComponent` потерял заряд. Выбор формы «кот» через серверный обработчик сообщения BUI → полиморф в `MobCat`.
- [ ] **Step 2–4:** код, сборка, Release-клиент (новый BUI — песочница), коммит «feat(heretic): Одинокий ритуал — Сталкер».

---

### Task 11: Последний гимн жреца — Вознесение Плоти и Владыка Ночи

**Files:**
- Create: `Content.Server/_Stories/Heretic/Flesh/StoriesHereticFleshAscensionRitualComponent.cs`, `StoriesHereticFleshSystem.Ascension.cs`, `StoriesHereticArmEaterComponent.cs`
- Modify: `rituals_flesh.yml`, `knowledge_flesh.yml`, `actions_flesh.yml`, `summons_flesh.yml` (моб `STHereticMobLordOfNight`, полиморф `STHereticLordOfNight`), `heretic_flesh.ftl`
- Modify: `Content.Shared/_Stories/Heretic/StoriesHereticSystem.cs` — если нужен аксессор «снять лимит клинков» (`CountLimitedResults` / лимит ритуала). Предпочтительно вместо этого добавить в `StoriesHereticRitualComponent` поле `AscendedLimit` (`int?`; `null` — без лимита после Вознесения) и проверять его в `TryPerform`.
- Test: `HereticFleshTest.cs`

Поведение (спецификация §6.9):
- **Ритуал:** 4 мёртвых гуманоида на руне, цели выполнены, ещё не вознёсся. Шаблон — `StoriesHereticAshSystem.OnRitualEffect` (выбросить вещи трупов, поглотить трупы, `SetAscended`, `RemoveRitual`).
- **Эффекты Вознесения:**
  - лимиты: Хватка +3 (`AscendedBonus` в Task 4), Безгласные +4 (`AscendedBonus` в Task 5);
  - клинки без лимита: `AscendedLimit: null` у `STHereticRitualFleshBlade`;
  - выдаётся действие «Сбросить форму» (`STActionHereticShedForm`, `PolymorphActionEvent` с `STHereticLordOfNight`), перезарядка 10 с, крик «REALITY UNCOIL!»;
  - объявление «Вечно свивающийся вихрь. Реальность развернулась. РУКИ РАСПРОСТЁРТЫ — ВЛАДЫКА НОЧИ { $name } вознёсся! Бойтесь вечно извивающейся длани!» и звук `ascend_flesh.ogg` (как у Пепла: `DispatchGlobalAnnouncement` + `PlayGlobal`).
- **Полиморф** `STHereticLordOfNight`: `forced: false` (штатное действие возврата), `transferDamage: true`, `revertOnDeath: true`, `revertOnCrit: false`, `inventory: Drop`.
- **Моб** `STHereticMobLordOfNight`:
  - 1000 HP, удар Slash 40 + Structural 150 (ломает стены и шлюзы — сверить тип урона по структурам в SS14: `Structural`);
  - `MovementSpeedModifier` 3/4.5;
  - не тянется: без `Pullable` или с отменой `PullAttemptEvent`;
  - не толкается: большая масса физики, сверить `Fixtures` density;
  - спрайт `lord_of_night.rsi`, `StoriesHereticArmEater`.
- **Поедание рук:** удар Владыки по отделённой руке (сущность с органом категории `ArmLeft`/`ArmRight`/`HandLeft`/`HandRight` без `MobState`) удаляет её и лечит 50% от максимума здоровья (Brute и Burn поровну), звук `/Audio/Effects/demon_consume.ogg` (сверить путь). Подписка на `MeleeHitEvent` у `StoriesHereticArmEaterComponent`.
- **Ужас Владыки:** при превращении (событие полиморфа — сверить `PolymorphedEvent` или аналог) каждый не-еретик и не-слуга в радиусе 9 с шансом 25% получает:
  - дрожь (`Jitter`) и заикание (`Stutter`) на 60 с;
  - головокружение (`Dizzy`/`Drunk`) на 30 с;
  - сообщение «Вы видите Врата Мансуса, и ваш разум трещит».
  
  Числа — `DataField` в `StoriesHereticFleshAscensionRitualComponent`.

- [ ] **Step 1: Тесты.**
  - Вознесение с 4 трупами: `Ascended`, действие `STActionHereticShedForm` выдано, лимит клинков снят.
  - 3 трупа — отказ с подсказкой.
  - Поедание руки лечит Владыку.
  - Повтор Вознесения невозможен.
- [ ] **Step 2–4:** код, проверка, коммит «feat(heretic): Последний гимн жреца — Вознесение Плоти».

---

### Task 12: Гайдбук, итоги раунда, финальная проверка

**Files:**
- Modify: страница гайдбука еретика (`Resources/Prototypes/_Stories/Heretic/guidebook.yml` → файл `.xml` в `Resources/ServerInfo/_Stories/Guidebook/`, сверить путь) — раздел «Путь Плоти»: знания по ступеням, призывы, гули, Вознесение.
- Modify: текст итогов раунда еретика — добавить строку про призывы, если в итогах выводятся цели (сверить `StoriesHereticRuleSystem` round-end).
- Modify: `.superpowers/heretic/NOTES.md` (статус PR б).

- [ ] **Step 1:** гайдбук и итоги.
- [ ] **Step 2: Полная проверка:**
  - `dotnet build -c Release` — без ошибок и без новых CS0618 в `_Stories/Heretic`;
  - линтер;
  - `dotnet test … --filter "FullyQualifiedName~Tests._Stories.Heretic|FullyQualifiedName~EntityTest|FullyQualifiedName~AllGamePresets|FullyQualifiedName~Secret"`;
  - Release-клиент (песочница).
- [ ] **Step 3: Коммит** «docs(heretic): гайдбук пути Плоти».
- [ ] **Step 4: Push ветки** `heretic-flesh` в форк. PR открывается после слияния `heretic-shared-layer` (см. NOTES.md).
