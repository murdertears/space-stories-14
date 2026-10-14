# Еретик, подпроект 2, PR (а): общие системы. План реализации

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Общий слой для путей Плоти, Пустоты и Клинка. В него входят:
- переходы между путями по кольцу Paradise и мосты в окне знаний;
- ингредиент-лужа;
- цели из знаний и цель «Призвать монстров»;
- призывы с призрачной ролью;
- гули;
- Плащ Тени;
- испорченные органы;
- расчленение трупа клинком.

**Architecture:** Всё расширяет ядро подпроекта 1, ничего не переписывает.
- **Данные знаний и ритуалов** получают новые поля: `PathLocked`, `Bridge`, `Objectives`, `RequiredSummons`, `Reagents`.
- **Новые эффекты ритуала** (`Summon`) обрабатываются серверными подписчиками широковещательного `StoriesHereticRitualEffectEvent`, как жертвоприношение.
- **Гули** обращаются через форковую `ConversionSystem`.
- **Органы** меняются в `StoriesHereticSacrificeSystem.Sacrifice()`.
- **Окно** раскладывает колонки по кольцу. Это чистая функция, её проверяет тест.

**Tech Stack:** C# (RobustToolbox ECS), YAML-прототипы, Fluent, NUnit `GameTest` (`Content.IntegrationTests`).

**Spec:** `docs/superpowers/specs/2026-10-14-heretic-flesh-void-blade-design.md` (разделы 2, 3, 4, 5, 10, 11, 15). Основа — `docs/superpowers/specs/2026-10-13-heretic-core-design.md`.

## Global Constraints

- **Язык.** Комментарии в коде, коммиты и тексты для игрока на русском. Английские `name`/`description` в YAML плюс перевод в `Resources/Locale/ru-RU/ss14-ru/prototypes/_stories/heretic/entities.ftl` (`ent-<ID> = …`, `.desc = …`), как у Пепла.
- **Коммиты** заканчиваются строкой `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- **Стандарты.** Правила `coding-standards.md` обязательны: `[Dependency]` на приватных полях, `Entity<T>` в обработчиках, `ProtoId`/`EntProtoId` вместо строк, никаких `[Dependency]` в компонентах.
- **Имена.** Новые типы и файлы лежат в `Content.{Shared,Server,Client}/_Stories/Heretic/`, типы называются с префиксом `StoriesHeretic`, прототипы — с префиксом `STHeretic` (действия — `STActionHeretic`).
- **Доступ к `StoriesHereticComponent`.** Компонент помечен `[Access(typeof(StoriesHereticSystem))]`. Менять его поля и вызывать методы на его коллекциях (RA0002) можно только внутри `StoriesHereticSystem`; остальные системы получают методы-аксессоры там же.
- **Клиентский код** не использует выражения коллекций в `Span<T>` (песочница Release-клиента), см. `StoriesHereticKnowledgeTreeControl.DrawThickLine`.
- **Серверные компоненты в YAML** допустимы: клиент игнорирует отсутствующие.
- **Урон в YAML** задаётся только через `types:` (не `groups:`).
- **Спрайты tg** конвертируются `Tools/_Stories/dmi_to_rsi.py`. Скачанное лежит во временной папке вне репо и запускается как `python -I`. В `meta.json` остаётся `"license": "CC-BY-SA-3.0"` и `"copyright": "Taken from tgstation at https://github.com/tgstation/tgstation/commit/ca79910d79497cf1ab56ebcf8db52a7289a3d412"`.
- **Команды проверки** (из корня репо):
  - сборка: `dotnet build -c Release`;
  - тесты еретика: `dotnet test Content.IntegrationTests/Content.IntegrationTests.csproj -c Release --no-build --filter "FullyQualifiedName~Tests._Stories.Heretic"`;
  - линтер: `dotnet run --project Content.YAMLLinter -c Release --no-build`.

---

## Карта файлов

| Файл | Что | Задача |
|---|---|---|
| `Content.Shared/_Stories/Heretic/Components/StoriesHereticKnowledgeComponent.cs` | + `PathLocked`, `Bridge`, `Objectives`, `RequiredSummons` | 1, 4 |
| `Content.Shared/_Stories/Heretic/StoriesHereticSystem.Knowledge.cs` | правило переходов, `RequiredSummons` | 1, 4 |
| `Content.Shared/_Stories/Heretic/Components/StoriesHereticComponent.cs` | + `SummonCount`, `RequiredSummons` | 4 |
| `Content.Shared/_Stories/Heretic/StoriesHereticSystem.cs` | `RegisterSummon`, `ObjectivesComplete` | 4 |
| `Content.Shared/_Stories/Heretic/Components/StoriesHereticRitualComponent.cs` | + `Reagents`, `ReagentAmount`, эффект `Summon` | 3, 5 |
| `Content.Shared/_Stories/Heretic/StoriesHereticRitualSystem.cs` | сопоставление лужи | 3 |
| `Content.Server/_Stories/Heretic/StoriesHereticObjectiveSystem.cs`, `…ObjectiveConditionComponent.cs`, `…RoleComponent.cs` | цель призывов, выдача целей из знаний | 4 |
| `Content.Server/_Stories/Heretic/Summons/*` | призывы | 5 |
| `Content.Server/_Stories/Heretic/Ghouls/*` | гули | 6 |
| `Content.Shared/_Stories/Heretic/Cloak/*` | Плащ Тени | 7 |
| `Content.Server/_Stories/Heretic/Organs/*` | испорченные органы | 8, 9, 10 |
| `Content.Server/_Stories/Heretic/StoriesHereticDismemberSystem.cs` | расчленение трупа | 11 |
| `Content.Client/_Stories/Heretic/UI/StoriesHereticRing.cs` + BUI + TreeControl | кольцо и мосты | 2 |
| `Resources/Prototypes/_Stories/Heretic/{knowledge,knowledge_ash,actions,objectives,antag,entities,summons,organs}.yml` | данные | все |
| `Resources/Prototypes/_Stories/conversions.yml` | — (гули — в `Resources/Prototypes/_Stories/Heretic/ghouls.yml`) | 6 |
| `Resources/Textures/_Stories/Heretic/{actions,effects}.rsi` | спрайты плаща, дыма, вспышки | 12 |
| `Content.IntegrationTests/Tests/_Stories/Heretic/*` | тесты | все |

---

### Task 1: Переходы между путями (`PathLocked`, `Bridge`)

**Files:**
- Modify: `Content.Shared/_Stories/Heretic/Components/StoriesHereticKnowledgeComponent.cs`
- Modify: `Content.Shared/_Stories/Heretic/StoriesHereticSystem.Knowledge.cs:101-102`
- Modify: `Resources/Prototypes/_Stories/Heretic/knowledge_ash.yml` (Хватка, Метка, Огненный клинок)
- Test: `Content.IntegrationTests/Tests/_Stories/Heretic/HereticKnowledgeTest.cs`

**Interfaces:**
- Produces:
  - `StoriesHereticKnowledgeComponent.PathLocked` (`bool`);
  - `StoriesHereticKnowledgeComponent.Bridge` (`List<ProtoId<TagPrototype>>`);
  - правило: знание с `Path != own` можно изучить, только если оно не `PathLocked`, не `ChoosesPath` и не `Ascension`.

- [ ] **Step 1: Тест.** Добавить в `[TestPrototypes]` файла `HereticKnowledgeTest.cs` прототипы ниже. Существующие `STTestHereticPathA/B`, `RootA/B` и `A2` не трогать.

```yaml
- type: entity
  id: STTestHereticKnowledgeGraspB
  components:
  - type: StoriesHereticKnowledge
    path: STTestHereticPathB
    stage: 2
    pathLocked: true
    requires: [ STTestHereticKnowledgeRootB ]

- type: entity
  id: STTestHereticKnowledgeBridgeAB
  components:
  - type: StoriesHereticKnowledge
    stage: 3
    bridge: [ STTestHereticPathA, STTestHereticPathB ]
    requires: [ STTestHereticKnowledgeA2 ]

- type: entity
  id: STTestHereticKnowledgeB3
  components:
  - type: StoriesHereticKnowledge
    path: STTestHereticPathB
    stage: 3
    requires: [ STTestHereticKnowledgeGraspB, STTestHereticKnowledgeBridgeAB ]
```

Тест:

```csharp
[Test]
public async Task CrossPathThroughBridge()
{
    var testMap = await Pair.CreateTestMap();
    var sys = Server.System<StoriesHereticSystem>();

    await Server.WaitAssertion(() =>
    {
        var uid = SSpawnAtPosition("MobHuman", testMap.GridCoords);
        var ent = new Entity<StoriesHereticComponent>(uid, SEntMan.AddComponent<StoriesHereticComponent>(uid));
        sys.AddPoints(ent, 10);

        Assert.That(sys.TryLearn(ent, "STTestHereticKnowledgeRootA"), Is.True);
        Assert.That(sys.TryLearn(ent, "STTestHereticKnowledgeA2"), Is.True);

        // Закрытое знание чужого пути нельзя даже при выполненных требованиях.
        Assert.That(sys.CanLearn(ent, "STTestHereticKnowledgeGraspB", out _), Is.False);

        // Без моста узел чужого пути закрыт требованиями.
        Assert.That(sys.CanLearn(ent, "STTestHereticKnowledgeB3", out _), Is.False);

        Assert.That(sys.TryLearn(ent, "STTestHereticKnowledgeBridgeAB"), Is.True);
        Assert.That(sys.TryLearn(ent, "STTestHereticKnowledgeB3"), Is.True, "Мост не открыл знание соседнего пути");
        Assert.That(ent.Comp.Path, Is.EqualTo((ProtoId<TagPrototype>) "STTestHereticPathA"), "Переход сменил путь");
    });
}
```

В `using` добавить `Content.Shared.Tag;`, если его нет.

- [ ] **Step 2: Запустить и убедиться, что тест падает.**

Run: `dotnet build -c Release && dotnet test Content.IntegrationTests/Content.IntegrationTests.csproj -c Release --no-build --filter "FullyQualifiedName~HereticKnowledgeTest"`
Expected: FAIL. Сначала YAML-ошибка о полях `pathLocked`/`bridge`, после добавления полей — ассерт на `B3`.

- [ ] **Step 3: Поля компонента.** Добавить в `StoriesHereticKnowledgeComponent` после `Ascension`:

```csharp
    /// <summary>
    /// Знание только для последователей своего пути (Хватка, Метка, улучшение клинка).
    /// Корень (<see cref="ChoosesPath"/>) и <see cref="Ascension"/> закрыты для чужих путей всегда.
    /// Остальные знания пути доступны с чужого пути через мосты — как в дереве Paradise.
    /// </summary>
    [DataField]
    public bool PathLocked;

    /// <summary>
    /// Побочное знание между двумя соседними путями кольца. Только для раскладки окна: доступ задаёт <see cref="Requires"/>.
    /// </summary>
    [DataField]
    public List<ProtoId<TagPrototype>> Bridge = new();
```

- [ ] **Step 4: Правило в `CanLearn`.** Заменить строки 101–102 в `StoriesHereticSystem.Knowledge.cs`:

```csharp
        else if (knowledge.Path != null
                 && knowledge.Path != ent.Comp.Path
                 && (knowledge.PathLocked || knowledge.Ascension || (!knowledge.ChoosesPath && ent.Comp.Path == null)))
            reason = Loc.GetString("stories-heretic-learn-wrong-path");
```

Почему так:
- `ChoosesPath` сюда не входит: выбор второго пути отсекает проверка выше («путь уже выбран»).
- Без выбранного пути знания чужих путей закрыты: мост всё равно требует знание своего пути, но так проще читать отказ.

- [ ] **Step 5: Пометить Пепел.** В `knowledge_ash.yml` добавить `pathLocked: true` в `STHereticKnowledgeAshGrasp`, `STHereticKnowledgeAshMark` и `STHereticKnowledgeAshFieryBlade`, после строки `path: STHereticPathAsh`.

- [ ] **Step 6: Запустить все тесты еретика.**

Run: `dotnet build -c Release && dotnet test Content.IntegrationTests/Content.IntegrationTests.csproj -c Release --no-build --filter "FullyQualifiedName~Tests._Stories.Heretic"`
Expected: PASS, включая старый `LearningRules`: там `A2` от пути B отсекается требованием `RootA`.

- [ ] **Step 7: Commit**

```bash
git add Content.Shared/_Stories/Heretic Resources/Prototypes/_Stories/Heretic/knowledge_ash.yml Content.IntegrationTests/Tests/_Stories/Heretic/HereticKnowledgeTest.cs
git commit -m "feat(heretic): переходы между путями через мосты, как в дереве Paradise

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Кольцо путей и колонки-мосты в окне

**Files:**
- Create: `Content.Client/_Stories/Heretic/UI/StoriesHereticRing.cs`
- Modify: `Content.Client/_Stories/Heretic/UI/StoriesHereticKnowledgeNode.cs` (колонка получает `BridgeTo`)
- Modify: `Content.Client/_Stories/Heretic/UI/StoriesHereticKnowledgeBoundUserInterface.cs:40-64`
- Modify: `Content.Client/_Stories/Heretic/UI/StoriesHereticKnowledgeTreeControl.cs` (подложка моста — градиент двух палитр)
- Test: `Content.IntegrationTests/Tests/_Stories/Heretic/HereticKnowledgeUiTest.cs`

**Interfaces:**
- Consumes: `StoriesHereticKnowledgeComponent.Bridge` (Task 1).
- Produces:
  - `StoriesHereticKnowledgeColumn(string? Path, string Name, string? BridgeTo = null)`;
  - `StoriesHereticRing.BuildColumns(IReadOnlyCollection<string> paths, IReadOnlyCollection<(string A, string B)> bridges) → List<(string? Path, string? BridgeTo)>`. Нулевой элемент — общая колонка `(null, null)`;
  - `StoriesHereticRing.ColumnOf(List<(string? Path, string? BridgeTo)> columns, string? path, IReadOnlyList<string> bridge) → int`.

- [ ] **Step 1: Тест раскладки.** Добавить в `HereticKnowledgeUiTest.cs`:

```csharp
[Test]
public async Task RingColumnsFollowParadiseOrder()
{
    await Pair.Client.WaitAssertion(() =>
    {
        var columns = StoriesHereticRing.BuildColumns(
            ["STHereticPathVoid", "STHereticPathAsh", "STHereticPathBlade", "STHereticPathFlesh"],
            [("STHereticPathFlesh", "STHereticPathVoid"), ("STHereticPathBlade", "STHereticPathVoid"), ("STHereticPathFlesh", "STHereticPathLock")]);

        Assert.That(columns, Is.EqualTo(new List<(string?, string?)>
        {
            (null, null),
            ("STHereticPathAsh", null),
            ("STHereticPathFlesh", "STHereticPathLock"), // полумост: соседа нет, мост прижат к нашему пути с его стороны
            ("STHereticPathFlesh", null),
            ("STHereticPathFlesh", "STHereticPathVoid"),
            ("STHereticPathVoid", null),
            ("STHereticPathVoid", "STHereticPathBlade"),
            ("STHereticPathBlade", null),
        }));

        Assert.That(StoriesHereticRing.ColumnOf(columns, null, ["STHereticPathVoid", "STHereticPathFlesh"]), Is.EqualTo(4));
        Assert.That(StoriesHereticRing.ColumnOf(columns, "STHereticPathBlade", []), Is.EqualTo(7));
        Assert.That(StoriesHereticRing.ColumnOf(columns, null, []), Is.EqualTo(0));
    });
}
```

- [ ] **Step 2: Запустить и убедиться, что тест падает.**

Run: `dotnet build -c Release`
Expected: ошибка компиляции «StoriesHereticRing не найден».

- [ ] **Step 3: `StoriesHereticRing.cs`**

```csharp
namespace Content.Client._Stories.Heretic.UI;

/// <summary>
/// Порядок путей в окне — кольцо дерева Paradise. Побочные знания-мосты стоят между соседями.
/// Это раскладка окна, а не правило игры: доступ к знаниям задают их требования.
/// </summary>
public static class StoriesHereticRing
{
    public static readonly string[] Order =
    [
        "STHereticPathAsh", "STHereticPathMoon", "STHereticPathLock", "STHereticPathFlesh",
        "STHereticPathVoid", "STHereticPathBlade", "STHereticPathRust", "STHereticPathCosmos",
    ];

    /// <summary>
    /// Колонки слева направо: общая, затем пути по кольцу (начиная с Пепла) и мосты между ними.
    /// Мост к пути, которого нет в окне, ставится рядом со своим путём с той стороны, где по кольцу лежит сосед.
    /// Мост записывается как (путь слева, путь справа по кольцу).
    /// </summary>
    public static List<(string? Path, string? BridgeTo)> BuildColumns(
        IReadOnlyCollection<string> paths,
        IReadOnlyCollection<(string A, string B)> bridges)
    {
        var shown = Order.Where(paths.Contains).ToList();
        // Пути, которых нет в кольце (тестовые), — в конец по алфавиту.
        shown.AddRange(paths.Where(p => !Order.Contains(p)).OrderBy(p => p));

        var normalized = new HashSet<(string Left, string Right)>();
        foreach (var (a, b) in bridges)
        {
            normalized.Add(IsClockwiseNeighbour(a, b) ? (a, b) : (b, a));
        }

        var columns = new List<(string? Path, string? BridgeTo)> { (null, null) };
        foreach (var path in shown)
        {
            // Полумост слева: сосед слева по кольцу не показан.
            foreach (var (left, right) in normalized)
            {
                if (right == path && !shown.Contains(left))
                    columns.Add((path, left));
            }

            columns.Add((path, null));

            foreach (var (left, right) in normalized)
            {
                if (left == path)
                    columns.Add((path, right));
            }
        }

        return columns;
    }

    public static int ColumnOf(List<(string? Path, string? BridgeTo)> columns, string? path, IReadOnlyList<string> bridge)
    {
        if (bridge.Count == 2)
        {
            for (var i = 0; i < columns.Count; i++)
            {
                var (p, to) = columns[i];
                if (to != null && ((p == bridge[0] && to == bridge[1]) || (p == bridge[1] && to == bridge[0])))
                    return i;
            }
        }

        for (var i = 0; i < columns.Count; i++)
        {
            if (path != null && columns[i].Path == path && columns[i].BridgeTo == null)
                return i;
        }

        return 0;
    }

    /// <summary>
    /// <paramref name="b"/> — следующий после <paramref name="a"/> путь по кольцу.
    /// </summary>
    private static bool IsClockwiseNeighbour(string a, string b)
    {
        var ia = Array.IndexOf(Order, a);
        var ib = Array.IndexOf(Order, b);
        return ia >= 0 && ib >= 0 && (ia + 1) % Order.Length == ib;
    }
}
```

В тесте полумост Плоть–Замок нормализуется в `(Lock, Flesh)`, потому что Замок стоит по кольцу перед Плотью. Замка в окне нет, поэтому мост ставится слева от Плоти как `(Flesh, Lock)`. Мост Пустота–Клинок нормализуется в `(Void, Blade)` и ставится справа от Пустоты.

- [ ] **Step 4: Колонка с мостом.** В `StoriesHereticKnowledgeNode.cs`:

```csharp
/// <summary>
/// Колонка дерева. <see cref="Path"/> — ID тега пути, <c>null</c> — общие знания.
/// <see cref="BridgeTo"/> — второй путь моста; у колонки пути — <c>null</c>.
/// </summary>
public sealed record StoriesHereticKnowledgeColumn(string? Path, string Name, string? BridgeTo = null);
```

- [ ] **Step 5: BUI.** В `StoriesHereticKnowledgeBoundUserInterface.Update` заменить сбор `paths`/`columns` и расчёт `column` (строки 40–64):

```csharp
        // Колонки идут по кольцу Paradise, мосты — между соседними путями.
        var paths = system.AllKnowledge
            .Where(k => k.Comp.Path != null)
            .Select(k => k.Comp.Path!.Value.Id)
            .Distinct()
            .ToList();

        var bridges = system.AllKnowledge
            .Where(k => k.Comp.Bridge.Count == 2)
            .Select(k => (k.Comp.Bridge[0].Id, k.Comp.Bridge[1].Id))
            .Distinct()
            .ToList();

        var layout = StoriesHereticRing.BuildColumns(paths, bridges);
        var columns = layout
            .Select(c => new StoriesHereticKnowledgeColumn(c.Path,
                c.Path == null
                    ? Loc.GetString("stories-heretic-ui-column-general")
                    : c.BridgeTo == null ? Loc.GetString($"stories-heretic-path-{c.Path}") : string.Empty,
                c.BridgeTo))
            .ToList();
```

Внутри цикла по знаниям:

```csharp
            var column = StoriesHereticRing.ColumnOf(layout, knowledge.Path?.Id, knowledge.Bridge.Select(b => b.Id).ToList());
```

Поле `Path` узла для моста остаётся `knowledge.Path?.Id`, то есть `null`. Значит, печать и связи у моста общей палитры, а подложку колонки красит TreeControl.

- [ ] **Step 6: Подложка моста.** В `StoriesHereticKnowledgeTreeControl.Draw` в цикле колонок: если `_columns[i].BridgeTo` не `null`, нарисовать две половины колонки тёмными тонами двух палитр и не рисовать черту под заголовком.

```csharp
            var column = _columns[i];
            if (column.BridgeTo != null)
            {
                var mid = (left + right) / 2;
                handle.DrawRect(new UIBox2(left, 0, mid, PixelSize.Y), StoriesHereticPalette.Get(column.Path).Dark.WithAlpha(0.25f));
                handle.DrawRect(new UIBox2(mid, 0, right, PixelSize.Y), StoriesHereticPalette.Get(column.BridgeTo).Dark.WithAlpha(0.25f));
                continue;
            }
```

Вставить сразу после вычисления `left`/`right`, а `palette` вычислять после этого блока.

- [ ] **Step 7: Тесты и Release-клиент.**

Run: `dotnet build -c Release && dotnet test Content.IntegrationTests/Content.IntegrationTests.csproj -c Release --no-build --filter "FullyQualifiedName~HereticKnowledgeUiTest"`
Expected: PASS. Старый `KnowledgeWindowBuilds` передаёт колонки без `BridgeTo` — параметр по умолчанию.

Затем запустить Release-клиент: `dotnet run --project Content.Client -c Release --no-build`. Ожидание: главное меню открывается, в логе нет `Sandbox violation`.

- [ ] **Step 8: Commit**

```bash
git add Content.Client/_Stories/Heretic/UI Content.IntegrationTests/Tests/_Stories/Heretic/HereticKnowledgeUiTest.cs
git commit -m "feat(heretic): колонки окна знаний по кольцу Paradise, мосты между путями

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Ингредиент-лужа

**Files:**
- Modify: `Content.Shared/_Stories/Heretic/Components/StoriesHereticRitualComponent.cs` (класс `StoriesHereticIngredient`)
- Modify: `Content.Shared/_Stories/Heretic/StoriesHereticRitualSystem.cs:308-324` (`Matches`)
- Create: в `Resources/Prototypes/_Stories/Heretic/entities.yml` ничего; список крови — в YAML ритуалов путей (PR б)
- Test: `Content.IntegrationTests/Tests/_Stories/Heretic/HereticRitualTest.cs`

**Interfaces:**
- Produces:
  - `StoriesHereticIngredient.Reagents` (`List<ProtoId<ReagentPrototype>>?`);
  - `StoriesHereticIngredient.ReagentAmount` (`FixedPoint2`, по умолчанию 10).
  - Лужа считается одним ингредиентом. `consume` удаляет её целиком.

- [ ] **Step 1: Тест.** В `[TestPrototypes]` файла `HereticRitualTest.cs` добавить:

```yaml
- type: entity
  id: STTestHereticRitualBlood
  components:
  - type: StoriesHereticRitual
    ingredients:
    - name: stories-heretic-ingredient-blood
      reagents: [ Blood ]
    results:
    - STTestHereticResult
```

Тест:

```csharp
[Test]
public async Task BloodPuddleIngredient()
{
    var testMap = await Pair.CreateTestMap();
    var rituals = Server.System<StoriesHereticRitualSystem>();
    var puddles = Server.System<SharedPuddleSystem>();

    await Server.WaitAssertion(() =>
    {
        var coords = testMap.GridCoords;
        var uid = SSpawnAtPosition("MobHuman", coords);
        var comp = SEntMan.AddComponent<StoriesHereticComponent>(uid);
        comp.Rituals.Clear();
        comp.Rituals.Add("STTestHereticRitualBlood");
        var heretic = new Entity<StoriesHereticComponent>(uid, comp);
        var rune = SSpawnAtPosition("STHereticRune", coords);

        // 5 ед. мало.
        Assert.That(puddles.TrySpillAt(coords, new Solution("Blood", 5), out _, sound: false), Is.True);
        Assert.That(rituals.TryPerform(heretic, rune), Is.False);

        // Лужа на тайле сливается в одну: 5 + 10 = 15 ед.
        Assert.That(puddles.TrySpillAt(coords, new Solution("Blood", 10), out var puddle, sound: false), Is.True);
        Assert.That(rituals.TryPerform(heretic, rune), Is.True);
        Assert.That(SEntMan.Deleted(puddle) || SEntMan.IsQueuedForDeletion(puddle), Is.True, "Лужа не поглощена");
    });
}
```

`using`: `Content.Shared.Chemistry.Components;`, `Content.Shared.Fluids;`. Сверить имя и сигнатуру `TrySpillAt` в `Content.Shared/Fluids/SharedPuddleSystem*.cs`: если серверная, взять `Content.Server.Fluids.EntitySystems.PuddleSystem`.

- [ ] **Step 2: Запустить и убедиться, что тест падает.**

Run: `dotnet build -c Release && dotnet test Content.IntegrationTests/Content.IntegrationTests.csproj -c Release --no-build --filter "FullyQualifiedName~BloodPuddleIngredient"`
Expected: FAIL (поле `reagents` неизвестно).

- [ ] **Step 3: Поля ингредиента** (после `OrganCategory`):

```csharp
    /// <summary>
    /// Лужа на руне, в которой не меньше <see cref="ReagentAmount"/> любого из реагентов (кровь, пепел, рвота).
    /// </summary>
    [DataField]
    public List<ProtoId<ReagentPrototype>>? Reagents;

    [DataField]
    public FixedPoint2 ReagentAmount = 10;
```

`using Content.Shared.Chemistry.Reagent; using Content.Shared.FixedPoint;`.

- [ ] **Step 4: Сопоставление.** В `StoriesHereticRitualSystem`:
- добавить зависимость `[Dependency] private SharedSolutionContainerSystem _solutions = default!;`;
- добавить в `Matches` перед `return false;` блок ниже.

```csharp
        if (ingredient.Reagents != null
            && TryComp<PuddleComponent>(uid, out var puddle)
            && _solutions.TryGetSolution(uid, puddle.SolutionName, out _, out var solution))
        {
            var amount = FixedPoint2.Zero;
            foreach (var reagent in ingredient.Reagents)
            {
                amount += solution.GetTotalPrototypeQuantity(reagent);
            }

            if (amount >= ingredient.ReagentAmount)
                return true;
        }
```

`using Content.Shared.Chemistry.EntitySystems; using Content.Shared.Fluids.Components; using Content.Shared.FixedPoint;`. Если поиск `LookupFlags.Uncontained` в `TryPerform` не находит лужу (тест упадёт на втором `TryPerform`), заменить флаг на `LookupFlags.Uncontained | LookupFlags.Static`.

- [ ] **Step 5: Локаль.** В `Resources/Locale/{ru-RU,en-US}/_stories/heretic.ftl` рядом с другими `stories-heretic-ingredient-*` добавить:
- ru: `stories-heretic-ingredient-blood = лужа крови`;
- en: `stories-heretic-ingredient-blood = pool of blood`.

- [ ] **Step 6: Запустить и убедиться, что тест проходит.** Та же команда. Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add Content.Shared/_Stories/Heretic Resources/Locale Content.IntegrationTests/Tests/_Stories/Heretic/HereticRitualTest.cs
git commit -m "feat(heretic): ингредиент ритуала — лужа с реагентом

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Цели из знаний и цель «Призвать монстров»

**Files:**
- Modify: `Content.Shared/_Stories/Heretic/Components/StoriesHereticKnowledgeComponent.cs` (+ `Objectives`, `RequiredSummons`)
- Modify: `Content.Shared/_Stories/Heretic/Components/StoriesHereticComponent.cs` (+ `SummonCount`, `RequiredSummons`)
- Modify: `Content.Shared/_Stories/Heretic/StoriesHereticSystem.cs` (`ObjectivesComplete`, `RegisterSummon`)
- Modify: `Content.Shared/_Stories/Heretic/StoriesHereticSystem.Knowledge.cs` (`Grant`)
- Modify: `Content.Server/_Stories/Heretic/StoriesHereticObjectiveSystem.cs`, `StoriesHereticObjectiveConditionComponent.cs`, `StoriesHereticRoleComponent.cs`
- Modify: `Resources/Prototypes/_Stories/Heretic/objectives.yml`, `Resources/Locale/{ru-RU,en-US}/_stories/heretic.ftl`
- Test: `Content.IntegrationTests/Tests/_Stories/Heretic/HereticProgressTest.cs`

**Interfaces:**
- Produces:
  - `StoriesHereticSystem.RegisterSummon(Entity<StoriesHereticComponent> ent)`;
  - `StoriesHereticComponent.SummonCount` / `RequiredSummons` (`int`);
  - `StoriesHereticKnowledgeComponent.Objectives` (`List<EntProtoId>`);
  - `StoriesHereticKnowledgeComponent.RequiredSummons` (`int`);
  - прототип `STHereticObjectiveSummon`;
  - `StoriesHereticObjectiveKind.Summon`.

- [ ] **Step 1: Тест.** В `HereticProgressTest.cs` добавить тестовые прототипы и тест. Прототипы:

```yaml
- type: entity
  id: STTestHereticKnowledgeSummoner
  components:
  - type: StoriesHereticKnowledge
    requiredSummons: 2
    objectives: [ STHereticObjectiveSummon ]
```

Тест:

```csharp
[Test]
public async Task KnowledgeAddsSummonObjective()
{
    var testMap = await Pair.CreateTestMap();
    var heretics = Server.System<StoriesHereticSystem>();
    var minds = Server.System<SharedMindSystem>();

    await Server.WaitAssertion(() =>
    {
        var uid = SSpawnAtPosition("MobHuman", testMap.GridCoords);
        var mindId = minds.CreateMind(null);
        minds.TransferTo(mindId, uid);
        Server.System<RoleSystem>().MindAddRole(mindId, "STMindRoleHeretic");
        var ent = new Entity<StoriesHereticComponent>(uid, SEntMan.AddComponent<StoriesHereticComponent>(uid));

        heretics.AddPoints(ent, 5);
        Assert.That(heretics.TryLearn(ent, "STTestHereticKnowledgeSummoner"), Is.True);
        Assert.That(ent.Comp.RequiredSummons, Is.EqualTo(2));

        var mind = SEntMan.GetComponent<MindComponent>(mindId);
        Assert.That(mind.Objectives.Any(o => SEntMan.GetComponent<MetaDataComponent>(o).EntityPrototype?.ID == "STHereticObjectiveSummon"));

        ent.Comp.SacrificeCount = ent.Comp.RequiredSacrifices;
        ent.Comp.HeadSacrificeCount = ent.Comp.RequiredHeadSacrifices;
        ent.Comp.LearnedCount = ent.Comp.RequiredKnowledge;
        Assert.That(heretics.ObjectivesComplete(ent.Comp), Is.False, "Цель призывов не учтена");

        heretics.RegisterSummon(ent);
        heretics.RegisterSummon(ent);
        Assert.That(heretics.ObjectivesComplete(ent.Comp), Is.True);
    });
}
```

`using`: `System.Linq; Content.Server.Roles; Content.Shared.Mind;`. Если в файле уже есть похожая подготовка разума, взять её.

- [ ] **Step 2: Запустить и убедиться, что тест падает.**

Run: `dotnet build -c Release && dotnet test Content.IntegrationTests/Content.IntegrationTests.csproj -c Release --no-build --filter "FullyQualifiedName~KnowledgeAddsSummonObjective"`
Expected: FAIL (неизвестные поля и методы).

- [ ] **Step 3: Поля.**

В `StoriesHereticKnowledgeComponent` (после `LocksAscension`):

```csharp
    /// <summary>
    /// Цели, которые получает разум еретика при изучении (Принцип Голода — «Призвать монстров»).
    /// </summary>
    [DataField]
    public List<EntProtoId> Objectives = new();

    /// <summary>
    /// Сколько призывов нужно для Вознесения после изучения. Больший из известных порогов побеждает.
    /// </summary>
    [DataField]
    public int RequiredSummons;
```

В `StoriesHereticComponent` (после `HeadSacrificeCount`):

```csharp
    /// <summary>
    /// Призванные монстры, чью роль занял игрок. Гули не считаются (Paradise).
    /// </summary>
    [DataField, AutoNetworkedField]
    public int SummonCount;

    [DataField, AutoNetworkedField]
    public int RequiredSummons;
```

- [ ] **Step 4: Логика в `StoriesHereticSystem`.**

`ObjectivesComplete`:

```csharp
        return comp.LearnedCount >= comp.RequiredKnowledge
               && comp.SacrificeCount >= comp.RequiredSacrifices
               && comp.HeadSacrificeCount >= comp.RequiredHeadSacrifices
               && comp.SummonCount >= comp.RequiredSummons;
```

После `RegisterSacrifice`:

```csharp
    public void RegisterSummon(Entity<StoriesHereticComponent> ent)
    {
        ent.Comp.SummonCount++;
        Dirty(ent);
    }
```

В `Grant` (`StoriesHereticSystem.Knowledge.cs`) после блока `GrantPoints`:

```csharp
        if (knowledge.RequiredSummons > ent.Comp.RequiredSummons)
            ent.Comp.RequiredSummons = knowledge.RequiredSummons;
```

- [ ] **Step 5: Сервер — выдача целей и прогресс.** В `StoriesHereticObjectiveSystem`:
- добавить зависимости `[Dependency] private SharedMindSystem _mind = default!;` и `[Dependency] private StoriesHereticSystem _heretic = default!;`;
- в `Initialize` подписаться: `SubscribeLocalEvent<StoriesHereticComponent, StoriesHereticKnowledgeLearnedEvent>(OnKnowledgeLearned);`;
- добавить обработчик:

```csharp
    private void OnKnowledgeLearned(Entity<StoriesHereticComponent> ent, ref StoriesHereticKnowledgeLearnedEvent args)
    {
        if (!_heretic.TryGetKnowledge(args.Knowledge, out var knowledge)
            || knowledge.Objectives.Count == 0
            || !_mind.TryGetMind(ent, out var mindId, out var mind))
            return;

        foreach (var objective in knowledge.Objectives)
        {
            _mind.TryAddObjective(mindId, mind, objective);
        }
    }
```

В `Save` добавить `progress.SummonCount = heretic.SummonCount; progress.RequiredSummons = heretic.RequiredSummons;`.

В `GetCounts`: `StoriesHereticObjectiveKind.Summon => (progress.SummonCount, progress.RequiredSummons),`.

В `StoriesHereticRoleComponent` добавить поля `[DataField] public int SummonCount;` и `[DataField] public int RequiredSummons;`.

В `StoriesHereticObjectiveKind` добавить `Summon,`.

Цель добавляется после `Grant`, а `RequiredSummons` к этому моменту уже выставлен, поэтому заголовок получает правильное число.

- [ ] **Step 6: YAML и локаль.** В `objectives.yml`:

```yaml
- type: entity
  parent: STBaseHereticObjective
  id: STHereticObjectiveSummon
  components:
  - type: StoriesHereticObjectiveCondition
    kind: Summon
    title: stories-heretic-objective-summon
```

В `heretic.ftl`:
- ru: `stories-heretic-objective-summon = Призовите монстров Мансуса: { $count }`;
- en: `stories-heretic-objective-summon = Summon Mansus monsters: { $count }`.

- [ ] **Step 7: Запустить и убедиться, что тест проходит.** Сначала новый тест, затем все тесты еретика. Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add Content.Shared/_Stories/Heretic Content.Server/_Stories/Heretic Resources/Prototypes/_Stories/Heretic/objectives.yml Resources/Locale Content.IntegrationTests/Tests/_Stories/Heretic/HereticProgressTest.cs
git commit -m "feat(heretic): цели из знаний и цель «Призвать монстров»

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Призывы

**Files:**
- Modify: `Content.Shared/_Stories/Heretic/Components/StoriesHereticRitualComponent.cs` (эффект `Summon`)
- Create: `Content.Server/_Stories/Heretic/Summons/StoriesHereticSummonRitualComponent.cs`
- Create: `Content.Server/_Stories/Heretic/Summons/StoriesHereticSummonComponent.cs`
- Create: `Content.Server/_Stories/Heretic/Summons/StoriesHereticSummonSystem.cs`
- Modify: `Resources/Prototypes/_Stories/Heretic/antag.yml` (роль разума слуги)
- Create: `Resources/Prototypes/_Stories/Heretic/summons.yml` (база мобов-призывов)
- Create: `Resources/Locale/{ru-RU,en-US}/_stories/heretic_summons.ftl`
- Test: `Content.IntegrationTests/Tests/_Stories/Heretic/HereticSummonTest.cs`

**Interfaces:**
- Consumes: `StoriesHereticSystem.RegisterSummon` (Task 4), `StoriesHereticRitualEffectEvent`.
- Produces:
  - эффект `StoriesHereticRitualEffect.Summon`;
  - компонент ритуала `StoriesHereticSummonRitual { Mob: EntProtoId }`;
  - на мобе `StoriesHereticSummonComponent { Master, Ingredients, ExpireTime, Bound }`;
  - абстрактный прототип `STHereticSummonBase`;
  - роль разума `STMindRoleHereticMonster`;
  - `StoriesHereticSummonSystem.IsServantOf(EntityUid servant, EntityUid master) → bool`. Им пользуются «Сшить плоть» (PR б) и Тюрьма (PR в).

- [ ] **Step 1: Тест** `HereticSummonTest.cs`:

```csharp
#pragma warning disable RA0002
using Content.IntegrationTests.Fixtures;
using Content.Server._Stories.Heretic.Summons;
using Content.Shared._Stories.Heretic;
using Content.Shared._Stories.Heretic.Components;
using Content.Shared.Mind;
using Content.Shared.Mobs.Systems;
using Content.Shared.Damage;
using Content.Shared.Damage.Systems;
using Robust.Shared.GameObjects;
using Robust.Shared.Timing;

namespace Content.IntegrationTests.Tests._Stories.Heretic;

[TestFixture]
public sealed class HereticSummonTest : GameTest
{
    [TestPrototypes]
    private const string Prototypes = @"
- type: entity
  parent: [ MobHuman, STHereticSummonBase ]
  id: STTestHereticSummonMob
  components:
  - type: GhostRole
    name: test
    description: test
    rules: test
    mindRoles:
    - STMindRoleHereticMonster

- type: entity
  id: STTestHereticRitualSummon
  components:
  - type: StoriesHereticRitual
    effect: Summon
    ingredients:
    - name: stories-heretic-ingredient-knife
      whitelist:
        tags: [ Knife ]
  - type: StoriesHereticSummonRitual
    mob: STTestHereticSummonMob
";

    private int CountKnives()
    {
        var count = 0;
        var query = SEntMan.AllEntityQueryEnumerator<MetaDataComponent>();
        while (query.MoveNext(out var uid, out var meta))
        {
            if (meta.EntityPrototype?.ID == "KitchenKnife" && !SEntMan.IsQueuedForDeletion(uid))
                count++;
        }

        return count;
    }

    [Test]
    public async Task UnclaimedSummonReturnsIngredients()
    {
        var testMap = await Pair.CreateTestMap();
        var rituals = Server.System<StoriesHereticRitualSystem>();
        var timing = Server.ResolveDependency<IGameTiming>();
        var mob = EntityUid.Invalid;

        await Server.WaitAssertion(() =>
        {
            var coords = testMap.GridCoords;
            var uid = SSpawnAtPosition("MobHuman", coords);
            var comp = SEntMan.AddComponent<StoriesHereticComponent>(uid);
            comp.Rituals.Clear();
            comp.Rituals.Add("STTestHereticRitualSummon");
            var rune = SSpawnAtPosition("STHereticRune", coords);
            SSpawnAtPosition("KitchenKnife", coords);

            Assert.That(rituals.TryPerform((uid, comp), rune), Is.True);
            Assert.That(CountKnives(), Is.Zero, "Нож не поглощён");

            var query = SEntMan.EntityQueryEnumerator<StoriesHereticSummonComponent>();
            Assert.That(query.MoveNext(out mob, out var summon), Is.True, "Моб не призван");
            Assert.That(summon!.Master, Is.EqualTo(uid));
            summon.ExpireTime = timing.CurTime;
        });

        await Pair.RunTicksSync(2);

        await Server.WaitAssertion(() =>
        {
            Assert.That(SEntMan.Deleted(mob) || SEntMan.IsQueuedForDeletion(mob), Is.True, "Невостребованный призыв остался");
            Assert.That(CountKnives(), Is.EqualTo(1), "Ингредиенты не вернулись");
        });
    }

    [Test]
    public async Task ClaimedSummonCountsAndDropsOnDeath()
    {
        var testMap = await Pair.CreateTestMap();
        var rituals = Server.System<StoriesHereticRitualSystem>();
        var minds = Server.System<SharedMindSystem>();
        var damageable = Server.System<DamageableSystem>();
        var heretic = default(Entity<StoriesHereticComponent>);
        var mob = EntityUid.Invalid;

        await Server.WaitAssertion(() =>
        {
            var coords = testMap.GridCoords;
            var uid = SSpawnAtPosition("MobHuman", coords);
            heretic = (uid, SEntMan.AddComponent<StoriesHereticComponent>(uid));
            heretic.Comp.Rituals.Clear();
            heretic.Comp.Rituals.Add("STTestHereticRitualSummon");
            var rune = SSpawnAtPosition("STHereticRune", coords);
            SSpawnAtPosition("KitchenKnife", coords);
            Assert.That(rituals.TryPerform(heretic, rune), Is.True);

            var query = SEntMan.EntityQueryEnumerator<StoriesHereticSummonComponent>();
            query.MoveNext(out mob, out _);

            var mindId = minds.CreateMind(null);
            minds.TransferTo(mindId, mob);
            Assert.That(heretic.Comp.SummonCount, Is.EqualTo(1), "Призыв не засчитан");
            Assert.That(SEntMan.GetComponent<StoriesHereticSummonComponent>(mob).Bound, Is.True);

            damageable.TryChangeDamage(mob, new DamageSpecifier { DamageDict = new() { { "Blunt", 1000 } } }, true);
        });

        await Pair.RunTicksSync(2);

        await Server.WaitAssertion(() =>
        {
            Assert.That(SEntMan.Deleted(mob) || SEntMan.IsQueuedForDeletion(mob), Is.True, "Мёртвый призыв не схлопнулся");
            Assert.That(CountKnives(), Is.EqualTo(1), "Призыв не выронил ингредиенты");
        });
    }
}
```

- [ ] **Step 2: Запустить и убедиться, что тест падает** (неизвестные типы). Run: `dotnet build -c Release`. Expected: ошибки компиляции.

- [ ] **Step 3: Эффект.** В `StoriesHereticRitualEffect` (после `Ascension`) добавить `Summon,`.

- [ ] **Step 4: Компоненты.**

`StoriesHereticSummonRitualComponent.cs`:

```csharp
using Robust.Shared.Prototypes;

namespace Content.Server._Stories.Heretic.Summons;

/// <summary>
/// На прототипе ритуала с эффектом Summon: кого призвать.
/// </summary>
[RegisterComponent, Access(typeof(StoriesHereticSummonSystem))]
public sealed partial class StoriesHereticSummonRitualComponent : Component
{
    [DataField(required: true)]
    public EntProtoId Mob;
}
```

`StoriesHereticSummonComponent.cs`:

```csharp
using Robust.Shared.Prototypes;
using Robust.Shared.Serialization.TypeSerializers.Implementations.Custom;

namespace Content.Server._Stories.Heretic.Summons;

/// <summary>
/// Призванный монстр Мансуса. Пока роль не занята, ждёт игрока; при смерти роняет ингредиенты рецепта (Paradise).
/// </summary>
[RegisterComponent, AutoGenerateComponentPause, Access(typeof(StoriesHereticSummonSystem))]
public sealed partial class StoriesHereticSummonComponent : Component
{
    [DataField]
    public EntityUid? Master;

    /// <summary>
    /// Прототипы поглощённых ингредиентов — вернутся на пол, если роль не займут или моб умрёт.
    /// </summary>
    [DataField]
    public List<EntProtoId> Ingredients = new();

    [DataField]
    public TimeSpan ClaimTime = TimeSpan.FromMinutes(5);

    [DataField(customTypeSerializer: typeof(TimeOffsetSerializer)), AutoPausedField]
    public TimeSpan ExpireTime;

    /// <summary>
    /// Роль занята игроком.
    /// </summary>
    [DataField]
    public bool Bound;
}
```

- [ ] **Step 5: Система** `StoriesHereticSummonSystem.cs`:

```csharp
using Content.Server.Chat.Managers;
using Content.Server.Popups;
using Content.Shared._Stories.Heretic;
using Content.Shared._Stories.Heretic.Components;
using Content.Shared._Stories.Heretic.Events;
using Content.Shared.Mind.Components;
using Content.Shared.Mobs;
using Content.Shared.Popups;
using Robust.Shared.Player;
using Robust.Shared.Prototypes;
using Robust.Shared.Timing;

namespace Content.Server._Stories.Heretic.Summons;

public sealed partial class StoriesHereticSummonSystem : EntitySystem
{
    [Dependency] private IChatManager _chat = default!;
    [Dependency] private IComponentFactory _factory = default!;
    [Dependency] private IGameTiming _timing = default!;
    [Dependency] private StoriesHereticSystem _heretic = default!;
    [Dependency] private PopupSystem _popup = default!;

    public override void Initialize()
    {
        base.Initialize();

        SubscribeLocalEvent<StoriesHereticRitualEffectEvent>(OnRitualEffect);
        SubscribeLocalEvent<StoriesHereticSummonComponent, MindAddedMessage>(OnMindAdded);
        SubscribeLocalEvent<StoriesHereticSummonComponent, MobStateChangedEvent>(OnMobStateChanged);
    }

    public override void Update(float frameTime)
    {
        base.Update(frameTime);

        var now = _timing.CurTime;
        var query = EntityQueryEnumerator<StoriesHereticSummonComponent>();
        while (query.MoveNext(out var uid, out var summon))
        {
            if (!summon.Bound && now >= summon.ExpireTime)
                Collapse((uid, summon), "stories-heretic-summon-unclaimed");
        }
    }

    public bool IsServantOf(EntityUid servant, EntityUid master)
    {
        return TryComp<StoriesHereticSummonComponent>(servant, out var summon) && summon.Master == master;
    }

    private void OnRitualEffect(ref StoriesHereticRitualEffectEvent args)
    {
        if (args.Effect != StoriesHereticRitualEffect.Summon
            || !ProtoMan.TryIndex(args.Ritual, out var proto)
            || !proto.TryComp<StoriesHereticSummonRitualComponent>(out var ritual, _factory))
            return;

        var mob = Spawn(ritual.Mob, Transform(args.Rune).Coordinates);
        var summon = EnsureComp<StoriesHereticSummonComponent>(mob);
        summon.Master = args.Heretic;
        summon.ExpireTime = _timing.CurTime + summon.ClaimTime;
        foreach (var uid in args.Consumed)
        {
            if (MetaData(uid).EntityPrototype is { } ingredient)
                summon.Ingredients.Add(ingredient.ID);
        }

        _popup.PopupEntity(Loc.GetString("stories-heretic-summon-waiting"), mob, args.Heretic);
        args.Success = true;
    }

    private void OnMindAdded(Entity<StoriesHereticSummonComponent> ent, ref MindAddedMessage args)
    {
        if (ent.Comp.Bound)
            return;

        ent.Comp.Bound = true;
        if (TryComp<StoriesHereticComponent>(ent.Comp.Master, out var heretic))
        {
            _heretic.RegisterSummon((ent.Comp.Master.Value, heretic));
            _popup.PopupEntity(Loc.GetString("stories-heretic-summon-claimed", ("mob", ent.Owner)), ent.Comp.Master.Value, ent.Comp.Master.Value, PopupType.Medium);
        }

        if (TryComp<ActorComponent>(ent, out var actor) && ent.Comp.Master is { } master)
            _chat.DispatchServerMessage(actor.PlayerSession, Loc.GetString("stories-heretic-summon-briefing", ("master", master)));
    }

    private void OnMobStateChanged(Entity<StoriesHereticSummonComponent> ent, ref MobStateChangedEvent args)
    {
        if (args.NewMobState == MobState.Dead)
            Collapse(ent, "stories-heretic-summon-collapse");
    }

    /// <summary>
    /// Моб растворяется и оставляет ингредиенты рецепта: ритуал не пропадает впустую.
    /// </summary>
    private void Collapse(Entity<StoriesHereticSummonComponent> ent, LocId message)
    {
        var coords = Transform(ent).Coordinates;
        foreach (var ingredient in ent.Comp.Ingredients)
        {
            Spawn(ingredient, coords);
        }

        ent.Comp.Ingredients.Clear();
        _popup.PopupCoordinates(Loc.GetString(message, ("mob", ent.Owner)), coords, PopupType.MediumCaution);
        QueueDel(ent);
    }
}
```

Если поглощённый ингредиент — тело (у будущих рецептов таких нет), его прототип `MobHuman` вернётся пустым мобом. Поэтому рецепты призывов не используют трупы. Это правило указать в комментарии над `Ingredients`.

- [ ] **Step 6: YAML.**

В `antag.yml`:

```yaml
- type: entity
  parent: BaseMindRoleAntag
  id: STMindRoleHereticMonster
  name: Heretic Monster Role
  components:
  - type: MindRole
    antagPrototype: STHeretic
    exclusiveAntag: false
    roleType: TeamAntagonist
```

`summons.yml`:

```yaml
- type: entity
  abstract: true
  id: STHereticSummonBase
  categories: [ HideSpawnMenu ]
  components:
  - type: StoriesHereticSummon
  - type: GhostTakeoverAvailable
  - type: GhostRole
    rules: stories-heretic-summon-rules
    mindRoles:
    - STMindRoleHereticMonster
    raffle:
      settings: short
```

Сверить с `Resources/Prototypes/_Stories/Entities/Mobs/Demons/shadow_demon.yml`, нужен ли `GhostTakeoverAvailable` рядом с `GhostRole`. Взять тот же набор. Конкретные мобы (с `MobState`, спрайтом и боем) наследуют эту базу вместе со своим родителем-мобом через `parent: [ <моб>, STHereticSummonBase ]`.

- [ ] **Step 7: Локаль** `heretic_summons.ftl` (ru; en — тот же смысл по-английски):

```
stories-heretic-summon-rules = Вы — монстр Мансуса, призванный еретиком. Служите хозяину и помогайте ему вознестись. Вы антагонист.
stories-heretic-summon-waiting = Ритуал зовёт сквозь Мансус… Кто-то должен откликнуться.
stories-heretic-summon-claimed = Ваш зов услышан: { $mob } пришёл служить.
stories-heretic-summon-briefing = Вы — тварь, явившаяся через Врата Мансуса. Ваш хозяин — { $master }. Служите ему любой ценой.
stories-heretic-summon-unclaimed = { CAPITALIZE($mob) } так и не обрёл душу и растворяется.
stories-heretic-summon-collapse = { CAPITALIZE($mob) } схлопывается сам в себя!
```

- [ ] **Step 8: Запустить** `HereticSummonTest`, затем все тесты еретика. Expected: PASS.

- [ ] **Step 9: Commit**

```bash
git add Content.Shared/_Stories/Heretic Content.Server/_Stories/Heretic/Summons Resources/Prototypes/_Stories/Heretic Resources/Locale Content.IntegrationTests/Tests/_Stories/Heretic/HereticSummonTest.cs
git commit -m "feat(heretic): призывы Мансуса с призрачной ролью и возвратом ингредиентов

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Гули

**Files:**
- Create: `Content.Server/_Stories/Heretic/Ghouls/StoriesHereticGhoulComponent.cs`
- Create: `Content.Server/_Stories/Heretic/Ghouls/StoriesHereticRaisedComponent.cs`
- Create: `Content.Server/_Stories/Heretic/Ghouls/StoriesHereticGhoulSystem.cs`
- Modify: `Resources/Prototypes/_Stories/Heretic/antag.yml` (роль `STMindRoleHereticGhoul`)
- Create: `Resources/Prototypes/_Stories/Heretic/ghouls.yml` (прототип обращения `STHereticGhoul`, база для путей)
- Create: `Resources/Locale/{ru-RU,en-US}/_stories/heretic_ghouls.ftl`
- Test: `Content.IntegrationTests/Tests/_Stories/Heretic/HereticGhoulTest.cs`

**Interfaces:**
- Consumes: `Content.Server._Stories.Conversion.ConversionSystem` (`CanConvert`, `TryConvert`, `TryRevert`, `GetEntitiesConvertedBy`).
- Produces:
  - `StoriesHereticGhoulSystem.TryRaise(EntityUid master, EntityUid corpse, ProtoId<ConversionPrototype> conversion, int limit, [NotNullWhen(false)] out string? reason) → bool`;
  - `StoriesHereticGhoulSystem.CountGhouls(EntityUid master, ProtoId<ConversionPrototype> conversion) → int`;
  - компонент `StoriesHereticGhoul { CritThreshold, DeadThreshold }`. Его выдаёт прототип обращения; при выдаче компонент меняет пороги, при снятии у живого возвращает старые;
  - прототип обращения `STHereticGhoul` (25/50).
  - Пути в PR (б) добавят `STHereticGhoulVoiceless` и `STHereticGhoulShattered` по этому образцу.

- [ ] **Step 1: Тест** `HereticGhoulTest.cs`:

```csharp
#pragma warning disable RA0002
using Content.IntegrationTests.Fixtures;
using Content.Server._Stories.Heretic.Ghouls;
using Content.Shared.Damage;
using Content.Shared.Damage.Systems;
using Content.Shared.Mind;
using Content.Shared.Mobs;
using Content.Shared.Mobs.Systems;
using Robust.Shared.GameObjects;

namespace Content.IntegrationTests.Tests._Stories.Heretic;

[TestFixture]
public sealed class HereticGhoulTest : GameTest
{
    [Test]
    public async Task RaiseGhoulRules()
    {
        var testMap = await Pair.CreateTestMap();
        var ghouls = Server.System<StoriesHereticGhoulSystem>();
        var damageable = Server.System<DamageableSystem>();
        var mobState = Server.System<MobStateSystem>();
        var thresholds = Server.System<MobThresholdSystem>();
        var minds = Server.System<SharedMindSystem>();
        var master = EntityUid.Invalid;
        var corpse = EntityUid.Invalid;

        EntityUid Corpse()
        {
            var uid = SSpawnAtPosition("MobHuman", testMap.GridCoords);
            minds.TransferTo(minds.CreateMind(null), uid);
            damageable.TryChangeDamage(uid, new DamageSpecifier { DamageDict = new() { { "Blunt", 250 } } }, true);
            Assert.That(mobState.IsDead(uid));
            return uid;
        }

        await Server.WaitAssertion(() =>
        {
            master = SSpawnAtPosition("MobHuman", testMap.GridCoords);
            var alive = SSpawnAtPosition("MobHuman", testMap.GridCoords);
            Assert.That(ghouls.TryRaise(master, alive, "STHereticGhoul", 1, out _), Is.False, "Поднят живой");

            corpse = Corpse();
            Assert.That(ghouls.TryRaise(master, corpse, "STHereticGhoul", 1, out _), Is.True);
            Assert.That(mobState.IsAlive(corpse), Is.True);
            Assert.That(thresholds.TryGetThresholdForState(corpse, MobState.Dead, out var dead) && dead == 50, "Пороги гуля не выставлены");
            Assert.That(ghouls.CountGhouls(master, "STHereticGhoul"), Is.EqualTo(1));

            Assert.That(ghouls.TryRaise(master, Corpse(), "STHereticGhoul", 1, out _), Is.False, "Лимит не сработал");

            // Смерть гуля освобождает слот, тело остаётся трупом.
            damageable.TryChangeDamage(corpse, new DamageSpecifier { DamageDict = new() { { "Blunt", 100 } } }, true);
        });

        await Pair.RunTicksSync(2);

        await Server.WaitAssertion(() =>
        {
            Assert.That(mobState.IsDead(corpse), Is.True, "Снятие обращения оживило труп");
            Assert.That(ghouls.CountGhouls(master, "STHereticGhoul"), Is.Zero);
            Assert.That(ghouls.TryRaise(master, corpse, "STHereticGhoul", 1, out _), Is.False, "Тот же труп поднят дважды");
        });
    }
}
```

Сверить имя `MobThresholdSystem.TryGetThresholdForState` в `Content.Shared/Mobs/Systems/MobThresholdSystem.cs`. Если другое, взять аналог, возвращающий порог состояния.

- [ ] **Step 2: Запустить и убедиться, что тест падает** (ошибка компиляции).

- [ ] **Step 3: Компоненты.**

`StoriesHereticRaisedComponent.cs`:

```csharp
namespace Content.Server._Stories.Heretic.Ghouls;

/// <summary>
/// Труп уже служил гулем — поднять его второй раз нельзя (Paradise: «once per corpse»).
/// </summary>
[RegisterComponent, Access(typeof(StoriesHereticGhoulSystem))]
public sealed partial class StoriesHereticRaisedComponent : Component;
```

`StoriesHereticGhoulComponent.cs`:

```csharp
using Content.Shared.FixedPoint;

namespace Content.Server._Stories.Heretic.Ghouls;

/// <summary>
/// Выдаётся прототипом обращения гуля. Хрупкость гуля по tg: доля обычного здоровья (25/50/125 из 100).
/// </summary>
[RegisterComponent, Access(typeof(StoriesHereticGhoulSystem))]
public sealed partial class StoriesHereticGhoulComponent : Component
{
    [DataField(required: true)]
    public FixedPoint2 CritThreshold;

    [DataField(required: true)]
    public FixedPoint2 DeadThreshold;

    [DataField]
    public FixedPoint2? OriginalCrit;

    [DataField]
    public FixedPoint2? OriginalDead;
}
```

- [ ] **Step 4: Система** `StoriesHereticGhoulSystem.cs`:

```csharp
using System.Diagnostics.CodeAnalysis;
using Content.Server._Stories.Conversion;
using Content.Server.Chat.Managers;
using Content.Shared._Stories.Conversion;
using Content.Shared.Administration.Systems;
using Content.Shared.Examine;
using Content.Shared.Humanoid;
using Content.Shared.Mobs;
using Content.Shared.Mobs.Systems;
using Robust.Shared.Player;
using Robust.Shared.Prototypes;

namespace Content.Server._Stories.Heretic.Ghouls;

public sealed partial class StoriesHereticGhoulSystem : EntitySystem
{
    [Dependency] private IChatManager _chat = default!;
    [Dependency] private ConversionSystem _conversion = default!;
    [Dependency] private MobStateSystem _mobState = default!;
    [Dependency] private MobThresholdSystem _thresholds = default!;
    [Dependency] private RejuvenateSystem _rejuvenate = default!;

    public override void Initialize()
    {
        base.Initialize();

        SubscribeLocalEvent<StoriesHereticGhoulComponent, ComponentStartup>(OnStartup);
        SubscribeLocalEvent<StoriesHereticGhoulComponent, ComponentShutdown>(OnShutdown);
        SubscribeLocalEvent<StoriesHereticGhoulComponent, MobStateChangedEvent>(OnMobStateChanged);
        SubscribeLocalEvent<StoriesHereticGhoulComponent, ExaminedEvent>(OnExamined);
    }

    public int CountGhouls(EntityUid master, ProtoId<ConversionPrototype> conversion)
    {
        return _conversion.GetEntitiesConvertedBy(master, conversion).Count;
    }

    public bool TryRaise(EntityUid master,
        EntityUid corpse,
        ProtoId<ConversionPrototype> conversion,
        int limit,
        [NotNullWhen(false)] out string? reason)
    {
        reason = null;
        if (!HasComp<HumanoidProfileComponent>(corpse) || !_mobState.IsDead(corpse))
            reason = Loc.GetString("stories-heretic-ghoul-invalid");
        else if (HasComp<StoriesHereticRaisedComponent>(corpse))
            reason = Loc.GetString("stories-heretic-ghoul-already");
        else if (CountGhouls(master, conversion) >= limit)
            reason = Loc.GetString("stories-heretic-ghoul-limit");
        else if (!_conversion.CanConvert(corpse, conversion, master))
            reason = Loc.GetString("stories-heretic-ghoul-protected");

        if (reason != null)
            return false;

        _rejuvenate.PerformRejuvenate(corpse);
        EnsureComp<StoriesHereticRaisedComponent>(corpse);
        _conversion.TryConvert(corpse, conversion, master);

        if (TryComp<ActorComponent>(corpse, out var actor))
            _chat.DispatchServerMessage(actor.PlayerSession, Loc.GetString("stories-heretic-ghoul-briefing", ("master", master)));

        return true;
    }

    private void OnStartup(Entity<StoriesHereticGhoulComponent> ent, ref ComponentStartup args)
    {
        if (_thresholds.TryGetThresholdForState(ent, MobState.Critical, out var crit))
            ent.Comp.OriginalCrit = crit;
        if (_thresholds.TryGetThresholdForState(ent, MobState.Dead, out var dead))
            ent.Comp.OriginalDead = dead;

        _thresholds.SetMobStateThreshold(ent, ent.Comp.CritThreshold, MobState.Critical);
        _thresholds.SetMobStateThreshold(ent, ent.Comp.DeadThreshold, MobState.Dead);
    }

    /// <summary>
    /// Пороги возвращаются только живому (снятие майндшилдом или админом). Мёртвый гуль остаётся трупом:
    /// с обычными порогами накопленный урон снова оказался бы «живым».
    /// </summary>
    private void OnShutdown(Entity<StoriesHereticGhoulComponent> ent, ref ComponentShutdown args)
    {
        if (TerminatingOrDeleted(ent) || _mobState.IsDead(ent))
            return;

        if (ent.Comp.OriginalDead is { } dead)
            _thresholds.SetMobStateThreshold(ent, dead, MobState.Dead);
        if (ent.Comp.OriginalCrit is { } crit)
            _thresholds.SetMobStateThreshold(ent, crit, MobState.Critical);
    }

    private void OnMobStateChanged(Entity<StoriesHereticGhoulComponent> ent, ref MobStateChangedEvent args)
    {
        if (args.NewMobState != MobState.Dead || !TryComp<ConversionableComponent>(ent, out var conversions))
            return;

        // Гуль умер — служба окончена, слот лимита освобождается.
        foreach (var id in new List<string>(conversions.ActiveConversions.Keys))
        {
            if (ProtoMan.TryIndex<ConversionPrototype>(id, out var proto) && proto.Components.ContainsKey(Factory.GetComponentName<StoriesHereticGhoulComponent>()))
                _conversion.TryRevert(ent, id);
        }
    }

    private void OnExamined(Entity<StoriesHereticGhoulComponent> ent, ref ExaminedEvent args)
    {
        args.PushMarkup(Loc.GetString("stories-heretic-ghoul-examine", ("ent", ent.Owner)));
    }
}
```

Сверить: `Factory` — свойство `EntitySystem`; если его нет, внедрить `[Dependency] private IComponentFactory _factory`. Также сверить, как `ComponentRegistry` хранит имена (ключ без суффикса `Component`). Удаление `StoriesHereticGhoulComponent` при `TryRevert` выполнит сама `ConversionSystem` через `RemoveComponents(proto.Components)`.

- [ ] **Step 5: YAML.**

В `antag.yml`:

```yaml
- type: entity
  parent: BaseMindRoleAntag
  id: STMindRoleHereticGhoul
  name: Heretic Ghoul Role
  components:
  - type: MindRole
    antagPrototype: STHeretic
    exclusiveAntag: false
    roleType: TeamAntagonist
```

`ghouls.yml`:

```yaml
# Dead — для проверки CanConvert до оживления, Alive — для TryConvert после PerformRejuvenate.
# «Только труп» проверяет StoriesHereticGhoulSystem.TryRaise.
- type: conversion
  id: STHereticGhoul
  allowedMobStates: [ Dead, Alive ]
  blacklist:
    components:
    - Zombie
    - StoriesHeretic
  components:
  - type: StoriesHereticGhoul
    critThreshold: 25
    deadThreshold: 50
  mindRoles:
  - STMindRoleHereticGhoul
```

`TryConvert` повторяет проверки `CanConvert` уже после оживления, поэтому в списке оба состояния.

- [ ] **Step 6: Локаль** `heretic_ghouls.ftl` (ru и en):

```
stories-heretic-ghoul-invalid = Это тело не подходит.
stories-heretic-ghoul-already = Этот труп уже служил Мансусу.
stories-heretic-ghoul-limit = Больше слуг вы удержать не сможете.
stories-heretic-ghoul-protected = Что-то защищает этот разум.
stories-heretic-ghoul-briefing = Вы — шаркающий труп, возвращённый через Врата Мансуса. Ваш хозяин — { $master }. Помогайте ему любой ценой.
stories-heretic-ghoul-examine = [color=gray]Кожа { $ent } серая и иссохшая, глаза пустые.[/color]
```

- [ ] **Step 7: Запустить** `HereticGhoulTest` и все тесты еретика. Expected: PASS.

- [ ] **Step 8: Commit**

```bash
git add Content.Server/_Stories/Heretic/Ghouls Resources/Prototypes/_Stories/Heretic Resources/Locale Content.IntegrationTests/Tests/_Stories/Heretic/HereticGhoulTest.cs
git commit -m "feat(heretic): гули на основе обращения Conversion

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Плащ Тени

**Files:**
- Modify: `Content.Shared/_Stories/Heretic/Events/StoriesHereticActionEvents.cs` (событие действия)
- Create: `Content.Shared/_Stories/Heretic/Cloak/StoriesHereticShadowCloakComponent.cs`
- Create: `Content.Shared/_Stories/Heretic/Cloak/StoriesHereticShadowCloakSystem.cs`
- Modify: `Resources/Prototypes/_Stories/Heretic/knowledge.yml`, `actions.yml`, `entities.yml` (эффект дыма)
- Create: `Resources/Locale/{ru-RU,en-US}/_stories/heretic_cloak.ftl`
- Test: `Content.IntegrationTests/Tests/_Stories/Heretic/HereticCloakTest.cs`

**Interfaces:**
- Consumes: `StoriesHereticRequiresFocusComponent` (на действии, как у Пепла).
- Produces:
  - `StoriesHereticShadowCloakEvent : InstantActionEvent { Duration = 3 мин, InterruptCooldown = 3 мин, MinCooldown = 6 с, SpeedMultiplier = 1.25, DamageThreshold = 25, InterruptSlowdown = 0.5, InterruptSlowTime = 2 мин }`;
  - компонент `StoriesHereticShadowCloak` на еретике, пока плащ активен.

Поведение (спецификация §3):
- **Включение:**
  - добавить `IdentityBlocker` (FULL), запомнив, был ли он уже на мобе;
  - снять тег `FootstepSound`, запомнив, был ли он;
  - заспавнить дым `STHereticShadowCloakSmoke`, прикрепив к мобу;
  - вызвать `IdentitySystem.QueueIdentityUpdate`;
  - вызвать `MovementSpeedModifierSystem.RefreshMovementSpeedModifiers`.
- **Голос:** подписка на `TransformSpeakerNameEvent` подменяет имя на `stories-heretic-cloak-voice` («Неизвестный»).
- **Скорость:** подписка на `RefreshMovementSpeedModifiersEvent` даёт `×1.25`.
- **Урон** (`DamageChangedEvent`, только при увеличении): накопить урон. Если накоплено ≥ 25, с шансом `накоплено %` сорвать плащ.
- **Сон и крит** (`MobStateChangedEvent` не Alive, `SleepStateChangedEvent`) — сорвать.
- **Срыв:** снять плащ, нокдаун 0,5 с, статус замедления `StatusEffectSlowdown` на 2 мин (через `MovementModStatusSystem`, см. `Content.Shared/Movement/Systems/MovementModStatusSystem.cs`), перезарядка действия 3 мин.
- **Повторное нажатие** — ручное снятие, перезарядка `max(осталось, 6 с)`.
- **Конец срока** (`Update` на сервере): снять без штрафа, перезарядка 6 с.
- **Снятие** восстанавливает всё запомненное.

- [ ] **Step 1: Тест** `HereticCloakTest.cs`. Проверяет:
  1. После действия имя, видимое другим (`Identity.Name(heretic, SEntMan, observer)`), не равно настоящему.
  2. Скорость выше.
  3. Удар на 100 Blunt снимает плащ (шанс 100%) и возвращает имя.

```csharp
#pragma warning disable RA0002
using Content.IntegrationTests.Fixtures;
using Content.Shared._Stories.Heretic.Cloak;
using Content.Shared._Stories.Heretic.Components;
using Content.Shared._Stories.Heretic.Events;
using Content.Shared.Damage;
using Content.Shared.Damage.Systems;
using Content.Shared.IdentityManagement;
using Content.Shared.Movement.Components;
using Robust.Shared.GameObjects;

namespace Content.IntegrationTests.Tests._Stories.Heretic;

[TestFixture]
public sealed class HereticCloakTest : GameTest
{
    [Test]
    public async Task CloakHidesAndBreaksOnDamage()
    {
        var testMap = await Pair.CreateTestMap();
        var damageable = Server.System<DamageableSystem>();
        var heretic = EntityUid.Invalid;
        var observer = EntityUid.Invalid;
        var realName = string.Empty;
        var baseSpeed = 0f;

        await Server.WaitAssertion(() =>
        {
            heretic = SSpawnAtPosition("MobHuman", testMap.GridCoords);
            observer = SSpawnAtPosition("MobHuman", testMap.GridCoords);
            SEntMan.AddComponent<StoriesHereticComponent>(heretic);
            realName = SEntMan.GetComponent<MetaDataComponent>(heretic).EntityName;
            baseSpeed = SEntMan.GetComponent<MovementSpeedModifierComponent>(heretic).CurrentSprintSpeed;

            var ev = new StoriesHereticShadowCloakEvent { Performer = heretic };
            SEntMan.EventBus.RaiseLocalEvent(heretic, ev);
            Assert.That(ev.Handled, Is.True);
        });

        await Pair.RunTicksSync(2);

        await Server.WaitAssertion(() =>
        {
            Assert.That(SEntMan.HasComponent<StoriesHereticShadowCloakComponent>(heretic));
            Assert.That(Identity.Name(heretic, SEntMan, observer), Is.Not.EqualTo(realName));
            Assert.That(SEntMan.GetComponent<MovementSpeedModifierComponent>(heretic).CurrentSprintSpeed, Is.GreaterThan(baseSpeed));

            damageable.TryChangeDamage(heretic, new DamageSpecifier { DamageDict = new() { { "Blunt", 30 } } }, true);
            damageable.TryChangeDamage(heretic, new DamageSpecifier { DamageDict = new() { { "Blunt", 70 } } }, true);
        });

        await Pair.RunTicksSync(2);

        await Server.WaitAssertion(() =>
        {
            Assert.That(SEntMan.HasComponent<StoriesHereticShadowCloakComponent>(heretic), Is.False, "Плащ не сорван уроном");
            Assert.That(Identity.Name(heretic, SEntMan, observer), Is.EqualTo(realName));
        });
    }
}
```

После первого удара (30) накоплено 30, то есть шанс 30%. Второй удар доводит накопленное до 100, это шанс 100%, поэтому тест детерминирован. Если плащ сорвался на первом ударе, второй ничего не ломает.

- [ ] **Step 2: Запустить и убедиться, что тест падает** (ошибка компиляции).

- [ ] **Step 3: Событие** в `StoriesHereticActionEvents.cs`:

```csharp
public sealed partial class StoriesHereticShadowCloakEvent : InstantActionEvent
{
    [DataField]
    public TimeSpan Duration = TimeSpan.FromMinutes(3);

    [DataField]
    public TimeSpan MinCooldown = TimeSpan.FromSeconds(6);

    [DataField]
    public TimeSpan InterruptCooldown = TimeSpan.FromMinutes(3);

    [DataField]
    public float SpeedMultiplier = 1.25f;

    [DataField]
    public float DamageThreshold = 25f;

    [DataField]
    public TimeSpan InterruptKnockdown = TimeSpan.FromSeconds(0.5);

    [DataField]
    public TimeSpan InterruptSlowTime = TimeSpan.FromMinutes(2);

    [DataField]
    public EntProtoId Smoke = "STHereticShadowCloakSmoke";
}
```

- [ ] **Step 4: Компонент** `StoriesHereticShadowCloakComponent.cs`:

```csharp
using Robust.Shared.GameStates;
using Robust.Shared.Serialization.TypeSerializers.Implementations.Custom;

namespace Content.Shared._Stories.Heretic.Cloak;

/// <summary>
/// Еретик укрыт Плащом Тени: личность и голос скрыты, шаги беззвучны, скорость выше (tg shadow_cloak).
/// </summary>
[RegisterComponent, NetworkedComponent, AutoGenerateComponentState, AutoGenerateComponentPause]
[Access(typeof(StoriesHereticShadowCloakSystem))]
public sealed partial class StoriesHereticShadowCloakComponent : Component
{
    [DataField, AutoNetworkedField]
    public float SpeedMultiplier = 1.25f;

    [DataField(customTypeSerializer: typeof(TimeOffsetSerializer)), AutoPausedField]
    public TimeSpan EndTime;

    [DataField]
    public float DamageTaken;

    [DataField]
    public float DamageThreshold = 25f;

    [DataField]
    public EntityUid? Action;

    [DataField]
    public EntityUid? Smoke;

    [DataField]
    public bool HadIdentityBlocker;

    [DataField]
    public bool HadFootstepTag;

    [DataField]
    public TimeSpan MinCooldown;

    [DataField]
    public TimeSpan InterruptCooldown;

    [DataField]
    public TimeSpan InterruptKnockdown;

    [DataField]
    public TimeSpan InterruptSlowTime;
}
```

- [ ] **Step 5: Система** `StoriesHereticShadowCloakSystem.cs`. Логику брать из блока «Поведение» выше. Опорные вызовы:
  - `_identity.QueueIdentityUpdate(uid)`;
  - `_speed.RefreshMovementSpeedModifiers(uid)`;
  - `_tag.RemoveTag` / `_tag.AddTag(uid, "FootstepSound")`;
  - `_actions.SetCooldown(actionId, cooldown)`;
  - `_stun.TryKnockdown(uid, time, true)`;
  - `_movementMod.TryUpdateMovementSpeedModDuration(uid, MovementModStatusSystem.Slowdown, time, 0.5f)`. Сверить имя метода и константы в `MovementModStatusSystem.cs:93`.
  - Шанс: `_random.Prob(Math.Min(1f, comp.DamageTaken / 100f))`.
  - Спавн дыма: `PredictedSpawnAttachedTo(smoke, new EntityCoordinates(uid, default))`, удаление — `PredictedQueueDel`.
  - Решения о шансе и срыве принимаются только на сервере (`if (_net.IsClient) return;`), потому что `IRobustRandom` не предсказуем.
  - Звук включения — случайный `/Audio/Effects/…` из tg `curse1..6`, если звуки перенесены в Task 12. До этого — `/Audio/Magic/staff_animation.ogg`.

- [ ] **Step 6: YAML.**
  - Знание в `knowledge.yml`:

```yaml
- type: entity
  parent: STHereticKnowledgeBase
  id: STHereticKnowledgeShadowCloak
  name: Cloak of Shadow
  description: Hide your identity in purple smoke. Faster movement, quiet steps. Requires a focus.
  components:
  - type: StoriesHereticKnowledge
    starting: true
    cost: 0
    actions:
    - STActionHereticShadowCloak
```

  - Действие в `actions.yml`:

```yaml
- type: entity
  id: STActionHereticShadowCloak
  name: Cloak of Shadow
  description: Disappear into the shadows for three minutes. Taking damage may break the cloak.
  categories: [ HideSpawnMenu ]
  components:
  - type: Action
    itemIconStyle: NoItem
    useDelay: 6
  - type: Sprite
    sprite: _Stories/Heretic/actions.rsi
    layers:
    - state: ninja_cloak
  - type: StoriesHereticRequiresFocus
  - type: InstantAction
    event: !type:StoriesHereticShadowCloakEvent
```

  - Дым в `entities.yml`: сущность `STHereticShadowCloakSmoke` с `Sprite` (`_Stories/Heretic/effects.rsi`, state `curse`, `drawdepth: Overdoors`, `noRot: true`) и `PointLight` (`color: "#7a3fb0"`, `radius: 1.5`, `energy: 0.6`). Пока Task 12 не перенёс `curse`, взять существующий фиолетовый эффект из `Resources/Textures/Effects/` и заменить его в Task 12.

- [ ] **Step 7: Локаль** `heretic_cloak.ftl` (ru; en аналогично) и строки `ent-` в `entities.ftl`:

```
stories-heretic-cloak-voice = Неизвестный
stories-heretic-cloak-on = Вы растворяетесь в тенях.
stories-heretic-cloak-on-others = { CAPITALIZE($ent) } исчезает в клубах фиолетового дыма!
stories-heretic-cloak-off = Тени отпускают вас.
stories-heretic-cloak-broken = Плащ Тени сорван!
```

  В `entities.ftl`:

```
ent-STHereticKnowledgeShadowCloak = Плащ Тени
    .desc = Скройте лицо в фиолетовом дыму: шаги беззвучны, движения быстрее. Требует фокус.
ent-STActionHereticShadowCloak = Плащ Тени
    .desc = Растворитесь в тенях на три минуты. Урон может сорвать плащ.
ent-STHereticShadowCloakSmoke = тёмный дым
```

- [ ] **Step 8: Запустить** `HereticCloakTest` и все тесты еретика. Expected: PASS. Тест `HereticFocusTest`, если он перебирает действия с фокусом, должен остаться зелёным.

- [ ] **Step 9: Commit**

```bash
git add Content.Shared/_Stories/Heretic Resources/Prototypes/_Stories/Heretic Resources/Locale Content.IntegrationTests/Tests/_Stories/Heretic/HereticCloakTest.cs
git commit -m "feat(heretic): стартовое знание Плащ Тени

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Испорченные органы — замена при жертвоприношении

**Files:**
- Create: `Content.Server/_Stories/Heretic/Organs/StoriesHereticCorruptedOrganComponent.cs`
- Create: `Content.Server/_Stories/Heretic/Organs/StoriesHereticCorruptedOrganSystem.cs`
- Modify: `Content.Server/_Stories/Heretic/StoriesHereticSacrificeSystem.cs:247-258`
- Create: `Resources/Prototypes/_Stories/Heretic/organs.yml`
- Create: `Resources/Locale/{ru-RU,en-US}/_stories/heretic_organs.ftl`
- Test: `Content.IntegrationTests/Tests/_Stories/Heretic/HereticOrganTest.cs`

**Interfaces:**
- Produces:
  - `StoriesHereticCorruptedOrganSystem.CurseOrgans(EntityUid victim, EntityCoordinates dropAt) → int` (число заменённых);
  - `StoriesHereticCorruptedOrganSystem.IsSuppressed(EntityUid body) → bool` (святая вода в крови);
  - `StoriesHereticCorruptedOrganComponent { Kind }`;
  - перечисление `StoriesHereticCorruptedOrganKind { Eyes, Tongue, Liver, Stomach, Heart, Lungs, Appendix }`;
  - маркер на теле `StoriesHereticCorruptedBodyComponent`, через него эффекты подписываются на события тела;
  - прототипы `STHereticCorruptedEyes`, `…Tongue`, `…Liver`, `…Stomach`, `…Heart`, `…Lungs`, `…Appendix`. Это наследники `OrganHuman*` с окраской `#8a3fd1`.

- [ ] **Step 1: Тест** `HereticOrganTest.cs`:

```csharp
#pragma warning disable RA0002
using System.Linq;
using Content.IntegrationTests.Fixtures;
using Content.Server._Stories.Heretic.Organs;
using Content.Shared.Body;
using Robust.Shared.Containers;
using Robust.Shared.GameObjects;

namespace Content.IntegrationTests.Tests._Stories.Heretic;

[TestFixture]
public sealed class HereticOrganTest : GameTest
{
    [Test]
    public async Task SacrificeCursesTwoToFourOrgans()
    {
        var testMap = await Pair.CreateTestMap();
        var organs = Server.System<StoriesHereticCorruptedOrganSystem>();
        var containers = Server.System<SharedContainerSystem>();

        await Server.WaitAssertion(() =>
        {
            var victim = SSpawnAtPosition("MobHuman", testMap.GridCoords);
            var replaced = organs.CurseOrgans(victim, testMap.GridCoords);
            Assert.That(replaced, Is.InRange(2, 4));

            var inBody = containers.GetContainer(victim, BodyComponent.ContainerID).ContainedEntities
                .Count(SEntMan.HasComponent<StoriesHereticCorruptedOrganComponent>);
            Assert.That(inBody, Is.EqualTo(replaced), "Испорченные органы не в теле");

            // Старые органы выпали на пол.
            var dropped = 0;
            var query = SEntMan.AllEntityQueryEnumerator<OrganComponent, TransformComponent>();
            while (query.MoveNext(out var uid, out var organ, out var xform))
            {
                if (organ.Body == null && !containers.IsEntityInContainer(uid) && !SEntMan.HasComponent<StoriesHereticCorruptedOrganComponent>(uid))
                    dropped++;
            }

            Assert.That(dropped, Is.EqualTo(replaced), "Старые органы не выпали");
            Assert.That(SEntMan.HasComponent<StoriesHereticCorruptedBodyComponent>(victim));
        });
    }
}
```

Сверить, что у `OrganComponent` есть поле `Body` (`Content.Shared/Body/OrganComponent.cs`). Если имя другое, заменить.

- [ ] **Step 2: Запустить и убедиться, что тест падает** (ошибка компиляции).

- [ ] **Step 3: Компоненты.** В `StoriesHereticCorruptedOrganComponent.cs` два компонента и перечисление:

```csharp
namespace Content.Server._Stories.Heretic.Organs;

/// <summary>
/// Орган, вставленный Мансусом в принесённую жертву (tg corrupted_organ).
/// </summary>
[RegisterComponent, Access(typeof(StoriesHereticCorruptedOrganSystem))]
public sealed partial class StoriesHereticCorruptedOrganComponent : Component
{
    [DataField(required: true)]
    public StoriesHereticCorruptedOrganKind Kind;

    [DataField]
    public TimeSpan NextEffect;

    /// <summary>
    /// Шанс взорваться при извлечении из живого тела.
    /// </summary>
    [DataField]
    public float BurstChance = 0.25f;
}

/// <summary>
/// На теле, в котором есть испорченные органы, — для подписки на события тела (речь, питьё).
/// </summary>
[RegisterComponent, Access(typeof(StoriesHereticCorruptedOrganSystem))]
public sealed partial class StoriesHereticCorruptedBodyComponent : Component;

public enum StoriesHereticCorruptedOrganKind : byte
{
    Eyes,
    Tongue,
    Liver,
    Stomach,
    Heart,
    Lungs,
    Appendix,
}
```

- [ ] **Step 4: Замена.** Перед кодом прочитать:
  - `Content.Shared/Body/BodySystem.cs` (обработка вставки в `body_organs`);
  - `Content.Shared/Body/InitialBodySystem.cs` (как при спавне органы вставляются и связываются с родителем через `OrganRelationSystem.Relate`).

Повторить тот же порядок. Код системы:

```csharp
using Content.Shared._Stories.Heretic.Components;
using Content.Shared.Body;
using Content.Shared.Chemistry.EntitySystems;
using Robust.Shared.Containers;
using Robust.Shared.Map;
using Robust.Shared.Prototypes;
using Robust.Shared.Random;

namespace Content.Server._Stories.Heretic.Organs;

public sealed partial class StoriesHereticCorruptedOrganSystem : EntitySystem
{
    [Dependency] private IRobustRandom _random = default!;
    [Dependency] private SharedContainerSystem _containers = default!;
    [Dependency] private OrganRelationSystem _relations = default!;

    /// <summary>
    /// Категория штатного органа → испорченная замена. Категории — из Resources/Prototypes/Body/organ_categories (сверить ID).
    /// </summary>
    private static readonly (ProtoId<OrganCategoryPrototype> Category, EntProtoId Corrupted)[] Replacements =
    [
        ("Eyes", "STHereticCorruptedEyes"),
        ("Tongue", "STHereticCorruptedTongue"),
        ("Liver", "STHereticCorruptedLiver"),
        ("Stomach", "STHereticCorruptedStomach"),
        ("Heart", "STHereticCorruptedHeart"),
        ("Lungs", "STHereticCorruptedLungs"),
        ("Appendix", "STHereticCorruptedAppendix"),
    ];

    private readonly List<(EntityUid Organ, EntProtoId Corrupted)> _candidates = new();

    public int CurseOrgans(EntityUid victim, EntityCoordinates dropAt)
    {
        if (!_containers.TryGetContainer(victim, BodyComponent.ContainerID, out var container))
            return 0;

        _candidates.Clear();
        foreach (var organ in container.ContainedEntities)
        {
            if (!TryComp<OrganComponent>(organ, out var comp) || HasComp<StoriesHereticCorruptedOrganComponent>(organ))
                continue;

            foreach (var (category, corrupted) in Replacements)
            {
                if (comp.Category == category)
                    _candidates.Add((organ, corrupted));
            }
        }

        _random.Shuffle(_candidates);
        var count = Math.Min(_random.Next(2, 5), _candidates.Count);
        for (var i = 0; i < count; i++)
        {
            Replace(victim, container, _candidates[i].Organ, _candidates[i].Corrupted, dropAt);
        }

        if (count > 0)
            EnsureComp<StoriesHereticCorruptedBodyComponent>(victim);

        return count;
    }

    private void Replace(EntityUid body, BaseContainer container, EntityUid old, EntProtoId corrupted, EntityCoordinates dropAt)
    {
        var parent = CompOrNull<ChildOrganComponent>(old)?.Parent;
        _containers.Remove(old, container, destination: dropAt);

        var organ = Spawn(corrupted, dropAt);
        _containers.Insert(organ, container);
        if (parent != null && HasComp<ChildOrganComponent>(organ))
            _relations.Relate(parent.Value, organ);
    }
}
```

Если `_containers.Remove` в этой версии не принимает `destination`, сначала удалить из контейнера, затем `_transform.SetCoordinates(old, dropAt)`. `Next(2, 5)` даёт 2–4.

- [ ] **Step 5: Вызов в жертвоприношении.** В `StoriesHereticSacrificeSystem`:
- добавить зависимость `[Dependency] private StoriesHereticCorruptedOrganSystem _organs = default!;`;
- в `Sacrifice` после `_rejuvenate.PerformRejuvenate(victim);` добавить код ниже.

```csharp
        // Как curse_organs в tg: только у жертвы с разумом; выпавшие органы годятся для ритуалов.
        if (_mind.TryGetMind(victim, out _, out _) && _organs.CurseOrgans(victim, runeCoords) > 0)
            _popup.PopupCoordinates(Loc.GetString("stories-heretic-organs-forced-out", ("victim", victim)), runeCoords, PopupType.MediumCaution);
```

- [ ] **Step 6: Прототипы** `organs.yml`. Пример для глаз, остальные шесть так же с родителями `OrganHumanTongue`, `OrganHumanLiver`, `OrganHumanStomach`, `OrganHumanHeart`, `OrganHumanLungs`, `OrganHumanAppendix`:

```yaml
- type: entity
  parent: OrganHumanEyes
  id: STHereticCorruptedEyes
  name: corrupt orbs
  description: Wide dilated pupils and no iris. Something behind them is looking back.
  components:
  - type: Sprite
    color: "#8a3fd1"
  - type: StoriesHereticCorruptedOrgan
    kind: Eyes
```

Если у `OrganHuman*` спрайт задан слоями, краска `color` на `Sprite` всё равно тонирует все слои. Проверить в игре в Task 13.

Тексты `name`/`description` (en) и `ent-`-переводы (ru):

| ID | en name | ru name | ru desc |
|---|---|---|---|
| …Eyes | corrupt orbs | испорченные глаза | Расширенные зрачки, радужки нет. Что-то смотрит изнутри. |
| …Tongue | corrupt tongue | испорченный язык | Покрыт мерцающими точками, будто звёздами. |
| …Liver | corrupt liver | испорченная печень | Пульсирует и пузырится. |
| …Stomach | corrupt stomach | испорченный желудок | Жаждет не еды. |
| …Heart | corrupt heart | испорченное сердце | Бьётся не в такт ничему живому. |
| …Lungs | corrupt lungs | испорченные лёгкие | Выдыхают то, чего не вдыхали. |
| …Appendix | corrupt appendix | испорченный аппендикс | Внутри что-то шевелится. |

- [ ] **Step 7: Локаль** `heretic_organs.ftl`: `stories-heretic-organs-forced-out = Органы { $victim } выдавливаются наружу!` (en: `Several organs force themselves out of { $victim }!`).

- [ ] **Step 8: Запустить** `HereticOrganTest` и `HereticSacrificeTest`. Expected: PASS.

- [ ] **Step 9: Commit**

```bash
git add Content.Server/_Stories/Heretic Resources/Prototypes/_Stories/Heretic/organs.yml Resources/Locale Content.IntegrationTests/Tests/_Stories/Heretic/HereticOrganTest.cs
git commit -m "feat(heretic): испорченные органы у принесённых в жертву

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: Испорченные органы — общее поведение и эффекты глаз, языка, печени

**Files:**
- Modify: `Content.Server/_Stories/Heretic/Organs/StoriesHereticCorruptedOrganSystem.cs` (новый partial `…System.Effects.cs`)
- Modify: `Resources/Locale/{ru-RU,en-US}/_stories/heretic_organs.ftl`
- Test: `Content.IntegrationTests/Tests/_Stories/Heretic/HereticOrganTest.cs`

**Interfaces:**
- Consumes: `CurseOrgans`, компоненты из Task 8.
- Produces: `IsSuppressed(EntityUid body)`.

Поведение (спецификация §5.1):
- **Святая вода.** `IsSuppressed(body)` проверяет `Holywater` в растворе крови (`BloodstreamComponent.ChemicalSolutionName` или аналог; сверить в `Content.Shared/Body/Systems/BloodstreamSystem.cs`) при количестве > 0. Все эффекты ниже начинаются с `if (IsSuppressed(body)) return;`.
- **Взрыв при извлечении.** Подписка на `OrganGotRemovedEvent` у `StoriesHereticCorruptedOrganComponent`. Если тело было живым, святой воды нет и выпал шанс 25%:
  - 30 Heat всем мобам в радиусе 1 (`EntityLookupSystem.GetEntitiesInRange<DamageableComponent>`);
  - вспышка `STHereticCurseBlast` (Task 12);
  - удаление органа.
  
  После извлечения пересчитать маркер тела: если испорченных органов не осталось, снять `StoriesHereticCorruptedBodyComponent`.
- **Глаза.** Тик раз в секунду (`Update`, поле `NextEffect`). Если на теле глаза: найти в радиусе 7 и в прямой видимости (`ExamineSystemShared.InRangeUnOccluded`) еретика с фокусом (`StoriesHereticSystem.HasFocus`). Если нашёлся: `StatusEffectsSystem.TryUpdateStatusEffectDuration(body, "StatusEffectBlindness", 5 с)`, сообщение `stories-heretic-organs-eyes-burn`, следующий срабатывание не раньше чем через 30 с.
- **Язык.** Подписка на `StoriesHereticCorruptedBodyComponent, AccentGetEvent`. Если в теле есть испорченный язык: каждое слово с шансом 15% заменить случайным из датасета `STHereticShadowspeak` (новый `dataset` в `datasets.yml`: `"Ф'ТАГН"`, `"Й'ГНАЙИ"`, `"НГ'АХ"`, `"Р'ЛЕЙХ"`, `"ВГАХ'НАГЛ"`, `"М'ЛВАЛЬ"`, `"Ш'ГГОТ"`, `"ЗИ'ЛАРК"`). Осмотр рта не делаем: в SS14 нет осмотра рта.
- **Печень.** Подписка на `StoriesHereticCorruptedBodyComponent, IngestingEvent`. Если в теле есть испорченная печень, с шансом 20% добавить в `args.Split` 5 ед. случайного реагента из списка: `Rum`, `Vodka`, `SpaceMirage`, `Happiness`, `MindbreakerToxin`, `Laughter`. Сверить ID реагентов в `Resources/Prototypes/Reagents`; отсутствующие выкинуть. Сообщение `stories-heretic-organs-liver-bubble`.

- [ ] **Step 1: Тесты.** Добавить в `HereticOrganTest`:
  1. Тело с испорченными глазами и еретик с фокусом рядом (`MobHuman` + `StoriesHeretic` + `STHereticAmberFocus` в слоте шеи через `InventorySystem.TryEquip`). Через `RunSeconds(2)` у тела есть статус `StatusEffectBlindness`.
  2. То же, но в кровь жертвы добавлено 5 ед. `Holywater` (`SharedSolutionContainerSystem.TryAddReagent`). Слепоты нет.
  3. Извлечение испорченного органа из живого тела при `BurstChance = 1` удаляет орган.

Чтобы тест 1 был детерминирован, вставлять глаза через `CurseOrgans` нельзя (случайный выбор). Добавить в систему публичный метод:

```csharp
    /// <summary>
    /// Заменяет орган указанной категории испорченным. Для тестов и админов; <see cref="CurseOrgans"/> выбирает случайно.
    /// </summary>
    public bool TryCurseOrgan(EntityUid victim, ProtoId<OrganCategoryPrototype> category, EntityCoordinates dropAt)
```

Он ищет кандидата нужной категории и вызывает `Replace`. Внутри `CurseOrgans` тоже использовать `Replace`.

- [ ] **Step 2: Запустить и убедиться, что тесты падают.**
- [ ] **Step 3: Реализовать** поведение выше в `StoriesHereticCorruptedOrganSystem.Effects.cs` (partial).
- [ ] **Step 4: Локаль.** В `heretic_organs.ftl`:

```
stories-heretic-organs-eyes-burn = Ваши глаза жжёт при взгляде на { $heretic }!
stories-heretic-organs-liver-bubble = В животе что-то пузырится…
stories-heretic-organs-burst = { CAPITALIZE($organ) } лопается вспышкой проклятого пламени!
```

- [ ] **Step 5: Запустить** `HereticOrganTest`. Expected: PASS.
- [ ] **Step 6: Commit** `feat(heretic): эффекты испорченных глаз, языка и печени, святая вода`.

---

### Task 10: Испорченные органы — желудок, сердце, лёгкие, аппендикс

**Files:**
- Modify: `Content.Server/_Stories/Heretic/Organs/StoriesHereticCorruptedOrganSystem.Effects.cs`
- Modify: `Resources/Locale/{ru-RU,en-US}/_stories/heretic_organs.ftl`
- Test: `Content.IntegrationTests/Tests/_Stories/Heretic/HereticOrganTest.cs`

Поведение (тик раз в секунду, тот же `Update`, по `Kind`):
- **Желудок.** Каждую секунду `SatiationSystem.ModifyValue(body, SatiationSystem.Hunger, -2)`, сверх обычного голода: «втрое быстрее». Сверить знак и масштаб с базовой скоростью в `Resources/Prototypes/Body/Satiation` и подобрать множитель так, чтобы итог был ×3. Если поле `NextEffect` прошло, сообщение `stories-heretic-organs-stomach-blood`, следующее через 30 с.
  - Выпитая кровь (подписка на `IngestingEvent`, в `Split` есть реагент крови, тот же список, что у лужи крови) ставит `SatedUntil = now + 3 мин`. Пока срок не прошёл, голод не ускоряется. Поле `SatedUntil` добавить в `StoriesHereticCorruptedOrganComponent`.
- **Сердце.** Раз в `rand(6, 45)` с: `SharedStaminaSystem.TakeStaminaDamage(body, 20)` и сообщение `stories-heretic-organs-heart-hand`.
- **Лёгкие.** Раз в секунду с шансом 15%, если тело дышит (есть `RespiratorComponent`, не в крите): выпустить на тайл 30 моль `Miasma` или `Plasma` (50/50) через `AtmosphereSystem.GetContainingMixture(body, …)?.AdjustMoles(gas, 30)` и сообщение `stories-heretic-organs-lungs-cough`. Сверить API в `Content.Server/Atmos/EntitySystems/AtmosphereSystem*.cs`.
- **Аппендикс.** Раз в секунду с шансом 2%: `VomitSystem.Vomit(body)` и нокдаун 0,5 с.

- [ ] **Step 1: Тест.** Испорченное сердце (`TryCurseOrgan(..., "Heart", ...)`), затем `NextEffect` выставить в «сейчас» и прогнать 2 секунды. Урон стамине у тела > 0 (`StaminaComponent.StaminaDamage`).
- [ ] **Step 2: Запустить и убедиться, что тест падает.**
- [ ] **Step 3: Реализовать.**
- [ ] **Step 4: Локаль.**

```
stories-heretic-organs-stomach-blood = Кровь… Вам нужна кровь…
stories-heretic-organs-heart-hand = Призрачная рука сжимает ваше сердце!
stories-heretic-organs-lungs-cough = Вы выкашливаете облако странного газа!
```

- [ ] **Step 5: Запустить** `HereticOrganTest`. Expected: PASS.
- [ ] **Step 6: Commit** `feat(heretic): эффекты испорченных желудка, сердца, лёгких и аппендикса`.

---

### Task 11: Расчленение трупа клинком еретика

**Files:**
- Create: `Content.Server/_Stories/Heretic/StoriesHereticDismemberSystem.cs`
- Modify: `Content.Shared/_Stories/Heretic/Events/StoriesHereticEvents.cs` (DoAfter-событие)
- Modify: `Resources/Locale/{ru-RU,en-US}/_stories/heretic.ftl`
- Test: `Content.IntegrationTests/Tests/_Stories/Heretic/HereticDismemberTest.cs`

**Interfaces:**
- Consumes: `DetachableOrganSystem.Detach(Entity<DetachableOrganComponent?>) → EntityUid?` (`Content.Shared/Body/DetachableOrganSystem.cs:21`), `StoriesHereticBladeComponent`.
- Produces:
  - `StoriesHereticDismemberSystem.TryDismember(EntityUid user, EntityUid corpse, EntityUid organ) → bool`, запускает DoAfter 5 с;
  - `StoriesHereticDismemberSystem.Dismember(EntityUid corpse, EntityUid organ) → EntityUid?`, мгновенно, для тестов;
  - `StoriesHereticDismemberDoAfterEvent`.

Поведение:
- **Глагол** в `GetVerbsEvent<AlternativeVerb>` на мобе с `BodyComponent`. Условия: моб мёртв (`MobStateSystem.IsDead`), у пользователя есть `StoriesHereticComponent`, в активной руке предмет с `StoriesHereticBladeComponent`, моб не в Теневом мире (нет `StoriesHereticShadowRealmComponent`).
  - Для каждого органа в `body_organs` с `DetachableOrganComponent` и категорией из списка `Head`, `ArmLeft`, `ArmRight`, `LegLeft`, `LegRight` (сверить ID категорий в `Resources/Prototypes/Body/`) — отдельный глагол «Отделить: {название органа}». Категория глаголов `stories-heretic-dismember-category`.
- **DoAfter** 5 с, `BreakOnMove`, `NeedHand`.
- **По завершении** вызвать `Detach`, звук `/Audio/Effects/gib1.ogg`, сообщение.

- [ ] **Step 1: Тест.**

```csharp
[Test]
public async Task DismemberOnlyCorpses()
{
    var testMap = await Pair.CreateTestMap();
    var dismember = Server.System<StoriesHereticDismemberSystem>();
    var damageable = Server.System<DamageableSystem>();
    var containers = Server.System<SharedContainerSystem>();

    await Server.WaitAssertion(() =>
    {
        var victim = SSpawnAtPosition("MobHuman", testMap.GridCoords);
        EntityUid Head() => containers.GetContainer(victim, BodyComponent.ContainerID).ContainedEntities
            .First(o => SEntMan.GetComponent<OrganComponent>(o).Category == "Head");

        Assert.That(dismember.Dismember(victim, Head()), Is.Null, "Голову отрезали живому");

        damageable.TryChangeDamage(victim, new DamageSpecifier { DamageDict = new() { { "Blunt", 250 } } }, true);
        var head = Head();
        Assert.That(dismember.Dismember(victim, head), Is.Not.Null);
        Assert.That(containers.GetContainer(victim, BodyComponent.ContainerID).ContainedEntities
            .Any(o => SEntMan.GetComponent<OrganComponent>(o).Category == "Head"), Is.False, "Голова осталась");
    });
}
```

- [ ] **Step 2: Запустить и убедиться, что тест падает.**
- [ ] **Step 3: Реализовать.** В `Dismember` проверить, что моб мёртв и орган в его `body_organs`, затем вызвать `_detachable.Detach(organ)`.
- [ ] **Step 4: Локаль.**

```
stories-heretic-dismember-category = Расчленить
stories-heretic-dismember-verb = Отделить: { $part }
stories-heretic-dismember-done = { CAPITALIZE($user) } отсекает { $part } от тела { $corpse }.
```

- [ ] **Step 5: Запустить** `HereticDismemberTest`. Expected: PASS.
- [ ] **Step 6: Commit** `feat(heretic): расчленение трупа клинком еретика`.

---

### Task 12: Спрайты и звуки из tg

**Files:**
- Modify: `Resources/Textures/_Stories/Heretic/actions.rsi` (+ `ninja_cloak`)
- Create/Modify: `Resources/Textures/_Stories/Heretic/effects.rsi` (+ `curse`, `curse_blast` 64×64)
- Modify: `Resources/Prototypes/_Stories/Heretic/entities.yml` (дым, вспышка `STHereticCurseBlast` с `TimedDespawn` 0,3 с)
- Create: `Resources/Audio/_Stories/Heretic/curse1.ogg` … `curse6.ogg` + `attributions.yml`

- [ ] **Step 1: Скачать** во временную папку вне репо (`$TMPDIR/heretic-tg/`):
  - `icons/mob/actions/actions_minor_antag.dmi` (state `ninja_cloak`; сверить, где state лежит, по дереву tg);
  - `icons/effects/effects.dmi` (`curse`);
  - `icons/effects/64x64.dmi` (`curse_blast`);
  - `sound/effects/curse/curse1.ogg` … `curse6.ogg`.

```bash
C=ca79910d79497cf1ab56ebcf8db52a7289a3d412
D="$TMPDIR/heretic-tg"; mkdir -p "$D"
for f in icons/mob/actions/actions_minor_antag.dmi icons/effects/effects.dmi icons/effects/64x64.dmi; do
  curl -sL "https://raw.githubusercontent.com/tgstation/tgstation/$C/$f" -o "$D/$(basename $f)"
done
```

Перед скачиванием сверить пути по списку файлов: `gh api "repos/tgstation/tgstation/git/trees/$C?recursive=1" --jq '.tree[].path' | grep -E 'curse|ninja_cloak|64x64'`.

- [ ] **Step 2: Конвертировать** нужные состояния в существующие RSI. Сначала прочитать `--help`, затем запустить конвертер:

```bash
python -I Tools/_Stories/dmi_to_rsi.py --help
python -I Tools/_Stories/dmi_to_rsi.py "$D/actions_minor_antag.dmi" --states ninja_cloak --into Resources/Textures/_Stories/Heretic/actions.rsi
```

Аргументы конвертера сверить по `--help` и по тому, как он вызывался в подпроекте 1 (`git log -p --follow Resources/Textures/_Stories/Heretic/actions.rsi/meta.json | head`). Состояние 64×64 нельзя положить в RSI 32×32: нужен отдельный `curse_blast.rsi` с `"size": {"x": 64, "y": 64}`.

- [ ] **Step 3: Атрибуция.** В каждом новом и изменённом `meta.json` должны быть `"license": "CC-BY-SA-3.0"` и `"copyright"` с упоминанием tg и коммита. В `Resources/Audio/_Stories/Heretic/attributions.yml` добавить запись для каждого `curse*.ogg`:

```yaml
- files: ["curse1.ogg", "curse2.ogg", "curse3.ogg", "curse4.ogg", "curse5.ogg", "curse6.ogg"]
  license: "CC-BY-SA-3.0"
  copyright: "Taken from tgstation at https://github.com/tgstation/tgstation/commit/ca79910d79497cf1ab56ebcf8db52a7289a3d412"
  source: "https://github.com/tgstation/tgstation/tree/ca79910d79497cf1ab56ebcf8db52a7289a3d412/sound/effects/curse"
```

- [ ] **Step 4: Подключить.**
  - Дым `STHereticShadowCloakSmoke` перевести на `effects.rsi:curse`.
  - Добавить вспышку `STHereticCurseBlast` (`curse_blast.rsi`, `TimedDespawn: lifetime: 0.3`) и использовать её во взрыве органа (Task 9).
  - Звук включения Плаща Тени заменить на `SoundCollection` `STHereticCurse` из шести файлов (`Resources/Prototypes/_Stories/SoundCollections/heretic.yml`, создать при отсутствии).

- [ ] **Step 5: Проверка.**

Run: `dotnet build -c Release && dotnet test Content.IntegrationTests/Content.IntegrationTests.csproj -c Release --no-build --filter "FullyQualifiedName~HereticKnowledgeUiTest|FullyQualifiedName~EntityTest"`
Expected: PASS. RSI-тест проверяет иконки, `EntityTest` спавнит новые сущности.

- [ ] **Step 6: Commit** `feat(heretic): спрайты Плаща Тени и вспышки проклятия из tg`.

---

### Task 13: Финальная проверка PR (а)

- [ ] **Step 1: Сборка и линтер.**

Run: `dotnet build -c Release`, затем `dotnet run --project Content.YAMLLinter -c Release --no-build`
Expected: 0 ошибок, 0 предупреждений линтера по `_Stories/Heretic`.

- [ ] **Step 2: Тесты.**

Run: `dotnet test Content.IntegrationTests/Content.IntegrationTests.csproj -c Release --no-build --filter "FullyQualifiedName~Tests._Stories.Heretic|FullyQualifiedName~EntityTest|FullyQualifiedName~AllGamePresets|FullyQualifiedName~Secret"`
Expected: PASS все.

- [ ] **Step 3: Release-клиент и плейтест.** Запустить сервер и клиент командами из `.superpowers/heretic/NOTES.md` («Локальный запуск как на боевом»), выдать себе еретика админ-вербом. Проверить:
  - окно знаний: колонки идут Пепел → (мосты появятся с путями в PR б–г), клиент без `Sandbox violation`;
  - Плащ Тени: дым, имя скрыто в чате и при осмотре, срыв уроном;
  - жертвоприношение второго клиента: органы выпали, у жертвы фиолетовые органы; проверить тонировку в руке;
  - отделение головы трупа клинком через ПКМ → «Расчленить».

- [ ] **Step 4: Заметки.** В `.superpowers/heretic/NOTES.md` отметить «PR (а) готов». Записать найденные при плейтесте нюансы в раздел «Подводные камни».

- [ ] **Step 5: Push ветки** — только после подтверждения пользователя. PR (а) открывается после слияния #452 (см. NOTES.md).
