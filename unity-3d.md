# Unity Project Conventions for AI Agents

This document defines how to build a Unity project. Any AI agent (Claude Code, Copilot CLI, or similar) working in this repository must read and follow these conventions before generating code, creating files, or modifying the project.

This guide is project-agnostic. It applies to any Unity project using the architectural patterns described below.

---

## 1. Architectural Principles

Every decision in this project follows four principles:

1. **Manager-driven** — Global systems are owned by singleton Manager MonoBehaviours that coordinate high-level concerns (match lifecycle, spawning, selection, camera, UI). Managers are the only singletons in the project.
2. **Component-based** — Entity behavior is composed from small, focused MonoBehaviour components. Prefer composition over inheritance. Deep class hierarchies are prohibited.
3. **Data-configured** — All tunable values (stats, costs, timings, thresholds) live in ScriptableObject assets. No magic numbers in code. Every designer-facing value is a data asset field.
4. **Feature-folder oriented** — Each system, entity type, and world object owns a self-contained folder with its scripts, prefabs, data, and materials. Related files live together.

---

## 2. Folder Structure

### Layout

```
Assets/Game/
├── Entities/
│   ├── Scripts/           ← Shared entity components and base definitions
│   ├── {EntityType}/      ← Per-type folder (repeat for each entity)
│   │   ├── Scripts/
│   │   ├── Prefabs/
│   │   ├── Data/
│   │   └── Materials/
├── World/
│   ├── Scripts/           ← Shared world object base components
│   ├── {WorldFeature}/    ← Per-feature folder
│   │   ├── Scripts/
│   │   ├── Prefabs/
│   │   ├── Data/
│   │   └── Materials/
├── Systems/
│   ├── {SystemName}/      ← Per-system folder
│   │   ├── Scripts/
│   │   ├── Prefabs/
│   │   ├── Data/
│   │   ├── Editor/        ← Optional: custom inspectors
│   │   └── Tests/         ← Optional: EditMode/PlayMode tests
├── UI/
│   ├── Scripts/           ← UI controller scripts
│   ├── UXML/              ← UI Toolkit layout documents
│   ├── USS/               ← UI Toolkit stylesheets
│   ├── Prefabs/           ← Any hybrid UI prefabs
│   └── Data/              ← UI-related data assets
├── Factions/ (or Teams/, Categories/)
│   ├── Scripts/
│   └── {FactionName}/Data/
├── Input/
│   ├── Scripts/
│   └── {InputMapName}.inputactions
├── Scenes/
│   └── {SceneName}.unity
└── Settings/              ← Project-wide configuration assets
```

### Rules

- **Do not invent top-level folders** outside the structure above. New features go into the appropriate existing domain.
- New systems go under `Systems/{SystemName}/`.
- New entity types go under `Entities/{EntityType}/`.
- **No `Common/`, `Shared/`, `Utils/`, or `Helpers/` folders.** Shared code lives in the parent domain's `Scripts/` folder.
- **Data folders hold only `.asset` files** (ScriptableObject instances). Never put MonoBehaviours or plain C# classes there.
- Each system folder contains at minimum `Scripts/` and optionally `Prefabs/`, `Data/`, `Editor/`, `Tests/`.

---

## 3. Naming Conventions

### Files and classes

| Type | Pattern | Example |
|------|---------|---------|
| Manager (singleton) | `{System}Manager.cs` | `SpawnManager.cs` |
| Component (behavior) | `{Feature}Component.cs` | `HealthComponent.cs` |
| Entity marker | `{EntityType}Component.cs` | `PlayerUnitComponent.cs` |
| ScriptableObject class | `{Feature}Definition.cs` | `UnitDefinition.cs` |
| ScriptableObject asset | `{DescriptiveName}.asset` | `WarriorUnitDefinition.asset` |
| Interface | `I{Name}.cs` | `IDamageable.cs` |
| Enum | `{Name}.cs` | `UnitRole.cs` |
| Data record (plain C#) | `{Name}.cs` | `FactionRecord.cs` |
| Struct (value type) | `{Name}.cs` | `DamageInfo.cs` |
| Prefab | `{Name}.prefab` | `Warrior.prefab` |
| UI document | `{PanelName}.uxml` | `HUD.uxml` |
| UI stylesheet | `{Scope}.uss` | `HUD.uss`, `Common.uss` |
| Input actions asset | `{Name}.inputactions` | `GameplayInput.inputactions` |
| Scene | `{Purpose}.unity` | `Gameplay_MVP.unity` |
| UI controller | `{Panel}Controller.cs` | `HUDController.cs` |

### Code style

- **PascalCase**: public fields, properties, methods, classes, structs, enums, delegates.
- **camelCase**: private fields, local variables, parameters.
- **No underscore prefix** on serialized private fields (Unity Inspector displays them cleanly without it).
- Use `[SerializeField]` for private fields exposed to the Inspector. Never make fields `public` solely for Inspector visibility.
- Use `[Header("Section")]` to group serialized fields in components with many fields.
- **One class per file.** File name must exactly match the class/struct/interface name.

### Namespaces

Use `{ProjectName}.{Domain}` namespaces consistently:

```csharp
namespace MyProject.Entities { }
namespace MyProject.Systems.Spawning { }
namespace MyProject.World { }
namespace MyProject.UI { }
```

Every file must have a namespace. No code in the global namespace.

---

## 4. Modern C# in Unity

Use modern C# features where Unity supports them, but respect Unity's runtime constraints.

### Use freely

```csharp
// Null-conditional and null-coalescing
OnDeath?.Invoke();
var name = unit?.DisplayName ?? "Unknown";

// Expression-bodied members
public bool IsDead => currentHP <= 0;
public int MaxHP => definition.maxHP;

// Pattern matching
if (target is IDamageable damageable) { damageable.TakeDamage(info); }
if (collider.TryGetComponent<HealthComponent>(out var health)) { }

// String interpolation
Debug.Log($"Unit {unitName} took {damage} damage, {currentHP} HP remaining");

// Tuples for multi-return
public (bool success, int remaining) TryConsume(int amount) { }

// Switch expressions
var color = role switch
{
    UnitRole.Worker  => Color.yellow,
    UnitRole.Soldier => Color.red,
    _                => Color.white
};

// Records and init-only setters (for data transfer objects, not MonoBehaviours)
public record DamageEvent(int Amount, GameObject Source, Vector3 HitPoint);

// Using declarations (no braces needed)
using var reader = new StreamReader(path);

// Collection expressions and LINQ (outside hot paths)
var alive = units.Where(u => !u.IsDead).ToList();
```

### Use with caution

```csharp
// Async/await — Use UniTask or Unity's Awaitable (Unity 6+).
// Do NOT use System.Threading.Tasks in gameplay code. Unity is not thread-safe.

// LINQ in Update / hot paths — Allocates garbage. Cache results or use loops.

// Nullable reference types (#nullable enable) — Useful for data classes,
// but Unity serialization ignores them. Avoid on MonoBehaviour fields.
```

### Never use in Unity gameplay code

```csharp
// System.Threading.Thread / Task.Run for gameplay logic
// Finalizers / destructors (~ClassName)
// Static constructors that depend on Unity systems
// System.Reflection in hot paths (Editor code only)
```

### Unity-specific quirks

- `== null` on Unity objects checks both C# null AND destroyed objects. This is intentional. Use it.
- `GetComponent<T>()` returns null if not found (no exception). Always null-check or use `TryGetComponent`.
- `TryGetComponent<T>(out var comp)` avoids a GC allocation when the component is missing. Prefer it.
- `ScriptableObject` instances are **shared**. Never mutate them at runtime. Use runtime records for mutable state.
- `Destroy()` is deferred to end of frame. Use `DestroyImmediate()` only in Editor scripts.
- Coroutines allocate per-`yield`. For new code prefer `async`/`Awaitable` (Unity 6+) or direct timer patterns.
- `MonoBehaviour` serialized references default to null. Always null-check before use.

---

## 5. MonoBehaviour Patterns

### Managers (singletons)

Managers are the **only singletons** in the project. They coordinate systems, own global state, and provide stable access points.

```csharp
namespace MyProject.Systems.Colony
{
    public class ColonyManager : MonoBehaviour
    {
        public static ColonyManager Instance { get; private set; }

        [Header("Configuration")]
        [SerializeField] private FactionDefinition[] factions;

        private readonly Dictionary<int, FactionRecord> records = new();

        private void Awake()
        {
            if (Instance != null && Instance != this)
            {
                Destroy(gameObject);
                return;
            }
            Instance = this;
        }

        private void OnDestroy()
        {
            if (Instance == this) Instance = null;
        }
    }
}
```

**Manager rules:**

- Each manager is a prefab in its system folder, placed in the scene at edit time.
- Managers reference each other via `{Manager}.Instance`. No serialized cross-references between managers.
- Managers must not contain entity-specific logic. They coordinate; they don't specialize.
- Managers expose **C# events** for system-to-system communication and optionally **UnityEvents** for designer-wired responses (see Section 8).
- Managers are **not** `DontDestroyOnLoad`. They live and die with the gameplay scene.

### Components

Components are MonoBehaviours attached to entity or world object prefabs. Each implements a **single, focused behavior**.

```csharp
namespace MyProject.Entities
{
    public class HealthComponent : MonoBehaviour, IDamageable
    {
        [Header("Runtime")]
        [SerializeField] private int currentHP;

        public int CurrentHP => currentHP;
        public bool IsDead => currentHP <= 0;

        // C# events for system communication
        public event System.Action<DamageInfo> OnDamaged;
        public event System.Action OnDeath;

        // UnityEvents for designer-wired responses (VFX, SFX, animation)
        [Header("Designer Events")]
        [SerializeField] private UnityEvent<int> onDamagedVisual;
        [SerializeField] private UnityEvent onDeathVisual;

        public void Initialize(int maxHP)
        {
            currentHP = maxHP;
        }

        public void TakeDamage(DamageInfo info)
        {
            if (IsDead) return;
            currentHP = Mathf.Max(0, currentHP - info.Amount);

            OnDamaged?.Invoke(info);
            onDamagedVisual?.Invoke(info.Amount);

            if (IsDead)
            {
                OnDeath?.Invoke();
                onDeathVisual?.Invoke();
            }
        }
    }
}
```

**Component rules:**

- Components never reference Managers in `Awake()`. Use `Start()` or an explicit `Initialize()` method.
- Components expose `Initialize()` for configuration by the spawning system. They do not self-configure.
- Components communicate **outward** via events. They communicate **inward** via public methods.
- Components cache sibling references in `Awake()` via `TryGetComponent<T>()` on their own GameObject.
- Components must not call `GetComponent<>()` on **other GameObjects** in hot paths. Cache at init.
- Use `[RequireComponent(typeof(T))]` when a component always needs a sibling.

### Interfaces for cross-cutting concerns

```csharp
public interface IDamageable
{
    void TakeDamage(DamageInfo info);
    bool IsDead { get; }
}

public interface ISelectable
{
    void Select();
    void Deselect();
    bool IsSelected { get; }
}

public interface ICommandable
{
    void IssueCommand(CommandData command);
}
```

Check interfaces via `TryGetComponent<T>()` or pattern matching rather than `GetComponent<Interface>()` in hot paths.

---

## 6. ScriptableObject Data Assets

### Definition pattern

```csharp
namespace MyProject.Entities
{
    [CreateAssetMenu(fileName = "New Unit", menuName = "Game/Unit Definition")]
    public class UnitDefinition : ScriptableObject
    {
        [Header("Identity")]
        public string displayName;
        public Sprite icon;
        public GameObject prefab;

        [Header("Stats")]
        public int maxHP;
        public float moveSpeed;
        public int attackDamage;
        public float attackRange;

        [Header("Economy")]
        public int cost;
        public float productionTime;
    }
}
```

**Rules:**

- Always include `[CreateAssetMenu]` with a descriptive menu path.
- Fields on ScriptableObjects are `public` (they are pure data containers).
- Assets live in the `Data/` subfolder of the owning feature folder.
- **Never modify ScriptableObject values at runtime.** Use runtime records for mutable state.

---

## 7. Prefab Construction

### Hierarchy

```
EntityRoot (GameObject)               ← Components, NavMeshAgent, Collider
├── Visual (child)                    ← MeshRenderer / SpriteRenderer
├── SelectionIndicator (child, off)   ← Toggled by selection component
└── SensorRange (child, optional)     ← Trigger collider for detection
```

### Rules

1. **Root GameObject**: Named after the entity. All functional components go here.
2. **Visual child**: Named `Visual` or `Model`. Holds the renderer. Separates visuals from logic.
3. **Collider on root**: Sized to match the entity's physical presence.
4. **Selection indicator**: A disabled child toggled by the selection component.
5. **Layer assignment**: Define project layers (e.g., `Entity`, `WorldObject`, `Terrain`). Assign in the prefab.
6. **No self-configuration**: The spawning system calls `Initialize()` on each component after instantiation.

### Prefab workflow for agents

1. Create the root GameObject in the scene.
2. Add a visual child with the appropriate renderer.
3. Add all required components to the root.
4. Add a collider to the root.
5. Create a data asset in the entity's `Data/` folder.
6. Drag the root into the entity's `Prefabs/` folder.
7. Delete the scene instance.
8. Reference the prefab in the data asset's `prefab` field.

---

## 8. Events & Communication

### Dual-event philosophy

This project uses **two event systems** for different audiences:

| System | Audience | Purpose | Wired in |
|--------|----------|---------|----------|
| **C# events** (`event Action<T>`) | Code / systems | System-to-system communication, game logic | Code (`OnEnable`/`OnDisable`) |
| **UnityEvents** (`UnityEvent<T>`) | Designers / player UX | VFX, SFX, animation triggers, UI feedback | Inspector |

Both can fire from the same moment. C# events drive the simulation. UnityEvents drive the player experience.

### C# events (system communication)

Used when **code subscribes to code**. Managers, components, and systems use these.

```csharp
// DECLARING
public event System.Action<int> OnResourceChanged;
public event System.Action OnEntityDestroyed;

// RAISING
OnResourceChanged?.Invoke(currentAmount);

// SUBSCRIBING
private void OnEnable()
{
    ResourceManager.Instance.OnResourceChanged += HandleResourceChanged;
}
private void OnDisable()
{
    ResourceManager.Instance.OnResourceChanged -= HandleResourceChanged;
}
```

**Rules:**

- Always subscribe in `OnEnable()`, unsubscribe in `OnDisable()`. Never in `Awake()` or `Start()`.
- C# events drive logic. They never directly trigger visual effects.

### UnityEvents (designer / player UX)

Used when **designers wire responses in the Inspector** — particle effects, audio sources, animation triggers, screen shake, etc.

```csharp
using UnityEngine.Events;

public class HealthComponent : MonoBehaviour
{
    [Header("Designer Events")]
    [SerializeField] private UnityEvent<int> onDamagedVisual;
    [SerializeField] private UnityEvent onDeathVisual;
    [SerializeField] private UnityEvent onHealedVisual;

    public void TakeDamage(DamageInfo info)
    {
        // ... game logic, C# event ...
        onDamagedVisual?.Invoke(info.Amount);
        if (IsDead) onDeathVisual?.Invoke();
    }
}
```

**UnityEvent rules:**

- Always `[SerializeField] private`.
- Name with a `Visual` or `Feedback` suffix to distinguish from system events.
- Used for **fire-and-forget notifications** to VFX, SFX, animation. They never modify game state.
- Designers drag particle systems, audio sources, or animator triggers onto these in the Inspector.
- UnityEvents are **optional**. The game functions correctly without any wired.
- UnityEvents are preferred over coupling VFX/SFX scripts directly to gameplay components.

### When to use which

| Scenario | Use |
|----------|-----|
| Manager notifies another manager | C# event |
| Component notifies its manager | C# event |
| UI controller updates a label or bar | C# event |
| Damage triggers a particle effect | UnityEvent |
| Death triggers a sound effect | UnityEvent |
| Selection shows a highlight | UnityEvent |
| Production complete plays a fanfare | UnityEvent |
| Any response a designer should tweak without code | UnityEvent |

---

## 9. Input Handling (Unity Input System)

This project uses the **Unity Input System** package (`com.unity.inputsystem`). The legacy `UnityEngine.Input` class is not used.

### Setup

1. Create an `.inputactions` asset in `Assets/Game/Input/` (e.g., `GameplayInput.inputactions`).
2. Define Action Maps (e.g., `Gameplay`, `UI`) with Actions and Bindings for keyboard, mouse, and gamepad.
3. Enable **"Generate C# Class"** on the asset to produce a type-safe wrapper.

### Input reader pattern

A single `InputReader` ScriptableObject translates Input System callbacks into semantic C# events.

```csharp
using UnityEngine;
using UnityEngine.InputSystem;

namespace MyProject.Input
{
    [CreateAssetMenu(fileName = "InputReader", menuName = "Game/Input Reader")]
    public class InputReader : ScriptableObject, GameplayInput.IGameplayActions
    {
        private GameplayInput inputActions;

        public event System.Action<Vector2> OnPointerPosition;
        public event System.Action OnPrimaryAction;        // Left click / tap
        public event System.Action OnSecondaryAction;      // Right click
        public event System.Action<Vector2> OnPanInput;
        public event System.Action<float> OnZoomInput;
        public event System.Action OnCancelAction;
        public event System.Action OnToggleDebug;

        private void OnEnable()
        {
            inputActions ??= new GameplayInput();
            inputActions.Gameplay.SetCallbacks(this);
            inputActions.Gameplay.Enable();
        }

        private void OnDisable()
        {
            inputActions?.Gameplay.Disable();
        }

        public void OnPrimaryClick(InputAction.CallbackContext context)
        {
            if (context.performed) OnPrimaryAction?.Invoke();
        }

        public void OnSecondaryClick(InputAction.CallbackContext context)
        {
            if (context.performed) OnSecondaryAction?.Invoke();
        }

        public void OnPan(InputAction.CallbackContext context)
        {
            OnPanInput?.Invoke(context.ReadValue<Vector2>());
        }

        public void OnZoom(InputAction.CallbackContext context)
        {
            OnZoomInput?.Invoke(context.ReadValue<float>());
        }
    }
}
```

The ScriptableObject approach is preferred — it's shared across scenes and can be referenced by any MonoBehaviour via a serialized field. Alternatively, the InputReader can be a MonoBehaviour on a dedicated `InputManager` GameObject.

### Input rules

- **No component calls `Keyboard.current`, `Mouse.current`, or `Gamepad.current` directly.** All input flows through the InputReader.
- Gameplay systems subscribe to `InputReader` events in `OnEnable()` and unsubscribe in `OnDisable()`.
- The InputReader translates raw input into **semantic actions** (e.g., `OnPrimaryAction`, not `OnLeftMouseButtonDown`). This keeps input device-agnostic.
- Action Maps are switched when context changes (e.g., `Gameplay` → `UI` when a menu opens).
- Input rebinding uses the Input System's built-in rebinding API.

---

## 10. UI (UI Toolkit: UXML + USS)

This project uses **UI Toolkit** for all runtime UI. Legacy Canvas/UGUI is not used for new UI work.

### File organization

```
Assets/Game/UI/
├── Scripts/
│   ├── HUDController.cs
│   ├── CommandPanelController.cs
│   ├── DebugOverlayController.cs
│   └── TooltipController.cs
├── UXML/
│   ├── HUD.uxml
│   ├── CommandPanel.uxml
│   ├── DebugOverlay.uxml
│   └── Tooltip.uxml
├── USS/
│   ├── Common.uss           ← Shared variables, resets, typography
│   ├── HUD.uss
│   ├── CommandPanel.uss
│   └── DebugOverlay.uss
└── Data/
    └── UISettings.asset
```

### UXML structure

Each UI panel is a self-contained `.uxml` document.

```xml
<!-- HUD.uxml -->
<ui:UXML xmlns:ui="UnityEngine.UIElements">
    <Style src="project://database/Assets/Game/UI/USS/Common.uss" />
    <Style src="project://database/Assets/Game/UI/USS/HUD.uss" />

    <ui:VisualElement name="hud-root" class="hud-panel">
        <ui:VisualElement name="resource-bar" class="resource-bar">
            <ui:Label name="food-label" text="Food: 0" class="resource-text" />
            <ui:Label name="population-label" text="Pop: 0" class="resource-text" />
        </ui:VisualElement>
    </ui:VisualElement>
</ui:UXML>
```

### USS styling

All visual styling lives in USS files. Never set styles inline in C# except for values that change dynamically at runtime (e.g., health bar width percentage).

```css
/* Common.uss — shared design tokens */
:root {
    --color-primary: #4488FF;
    --color-danger: #FF4444;
    --color-text: #EEEEEE;
    --color-bg: rgba(0, 0, 0, 0.7);
    --font-size-sm: 12px;
    --font-size-md: 16px;
    --font-size-lg: 24px;
    --spacing-sm: 4px;
    --spacing-md: 8px;
    --spacing-lg: 16px;
    --radius-sm: 4px;
    --radius-md: 8px;
}

/* HUD.uss */
.hud-panel {
    position: absolute;
    width: 100%;
    height: 100%;
    flex-direction: column;
}

.resource-bar {
    flex-direction: row;
    padding: var(--spacing-md);
    background-color: var(--color-bg);
    border-bottom-left-radius: var(--radius-md);
    border-bottom-right-radius: var(--radius-md);
    align-self: center;
}

.resource-text {
    color: var(--color-text);
    font-size: var(--font-size-md);
    margin-right: var(--spacing-lg);
    -unity-font-style: bold;
}

/* State classes — toggled from C# */
.hidden { display: none; }
.disabled { opacity: 0.4; }
.active { border-color: var(--color-primary); border-width: 2px; }
```

### UI controller pattern

Each panel has a controller MonoBehaviour that queries elements by name and binds to game events.

```csharp
using UnityEngine;
using UnityEngine.UIElements;

namespace MyProject.UI
{
    [RequireComponent(typeof(UIDocument))]
    public class HUDController : MonoBehaviour
    {
        private Label foodLabel;
        private Label populationLabel;

        private void OnEnable()
        {
            var root = GetComponent<UIDocument>().rootVisualElement;

            // Cache element queries
            foodLabel = root.Q<Label>("food-label");
            populationLabel = root.Q<Label>("population-label");

            // Subscribe to game events
            if (ResourceManager.Instance != null)
                ResourceManager.Instance.OnResourceChanged += UpdateFood;
            if (PopulationManager.Instance != null)
                PopulationManager.Instance.OnPopulationChanged += UpdatePopulation;
        }

        private void OnDisable()
        {
            if (ResourceManager.Instance != null)
                ResourceManager.Instance.OnResourceChanged -= UpdateFood;
            if (PopulationManager.Instance != null)
                PopulationManager.Instance.OnPopulationChanged -= UpdatePopulation;
        }

        private void UpdateFood(int amount) => foodLabel.text = $"Food: {amount}";
        private void UpdatePopulation(int count) => populationLabel.text = $"Pop: {count}";
    }
}
```

### UI button and interaction handling

```csharp
private void OnEnable()
{
    var root = GetComponent<UIDocument>().rootVisualElement;

    var produceBtn = root.Q<Button>("produce-btn");
    produceBtn.clicked += OnProduceClicked;

    // Hover/focus states are handled entirely in USS via pseudo-classes:
    // Button:hover { background-color: ... }
    // Button:active { background-color: ... }
    // Button:disabled { opacity: 0.4; }
}

private void OnDisable()
{
    var root = GetComponent<UIDocument>().rootVisualElement;
    var produceBtn = root.Q<Button>("produce-btn");
    if (produceBtn != null) produceBtn.clicked -= OnProduceClicked;
}

private void OnProduceClicked()
{
    // Call into game system — UI never modifies state directly
    selectedProducer?.QueueProduction(selectedDefinition);
}
```

### UI rules

- **UI code never modifies game state directly.** It calls public methods on gameplay components or managers.
- **UI code never uses `GetComponent<>()` to search the scene for game entities.** It receives data through events or references provided by the selection system.
- **UI updates are event-driven.** Controllers subscribe to C# events and update elements in the handler. No polling in `Update()` (exception: animating transitions).
- **Each controller manages one panel.** No monolithic UI scripts.
- **Query elements by name** using `root.Q<T>("element-name")`. Cache queries in `OnEnable()`.
- **Use USS classes for state changes** (add/remove `.active`, `.disabled`, `.hidden`) rather than setting style properties in C#.
- **USS variables in `:root`** for all shared values. No hardcoded values in individual rules.
- **Hover, focus, and active states** are defined in USS via pseudo-classes (`:hover`, `:active`, `:focus`), not in C#.
- Each UI panel is a separate `UIDocument` component on its own GameObject in the scene.

---

## 11. Spawn & Initialization Flow

When a new entity is created at runtime:

1. The **spawning system** instantiates the prefab from the data asset's `prefab` field.
2. For each component, the spawning system calls `Initialize()` with the relevant data:
   ```csharp
   var health = instance.GetComponent<HealthComponent>();
   health.Initialize(definition.maxHP);

   var movement = instance.GetComponent<MovementComponent>();
   movement.Initialize(definition.moveSpeed, definition.agentRadius);

   var team = instance.GetComponent<TeamComponent>();
   team.Initialize(factionDefinition);
   ```
3. The spawning system **registers the entity** with the appropriate manager.
4. The entity's behavior components begin working on the next update cycle.

**No entity should configure itself.** The spawning system is the single, predictable initialization path.

---

## 12. Periodic / Scheduled Logic

For game logic that should not run every frame (AI decisions, scanning, evaluation), use timer-based scheduling.

### Simple timer pattern

```csharp
public class SensorComponent : MonoBehaviour
{
    [SerializeField] private float scanInterval = 0.25f;
    private float scanTimer;

    private void Update()
    {
        scanTimer -= Time.deltaTime;
        if (scanTimer <= 0f)
        {
            scanTimer = scanInterval;
            PerformScan();
        }
    }

    private void PerformScan() { /* detection logic */ }
}
```

### Interface-based scheduling (advanced)

For projects with many scheduled systems, use a centralized scheduler:

```csharp
public interface IScheduledUpdate
{
    void OnScheduledUpdate(float deltaTime);
    float Interval { get; }
}
```

Components implement this and register with a scheduler manager. This centralizes timing, enables pause, and allows diagnostic overlays.

**General rule: If it affects visuals or input responsiveness, use `Update()`. If it makes a game decision, schedule it at a slower cadence.**

---

## 13. Navigation & Movement

### Setup

- Use `com.unity.ai.navigation`.
- `NavMeshSurface` on the terrain or a dedicated bake object.
- Obstacles: `NavMeshObstacle` with `Carve = true` or baked as static geometry.
- NavMesh is baked in the Editor. No runtime baking unless explicitly needed.

### Movement component

```csharp
public class MovementComponent : MonoBehaviour
{
    private NavMeshAgent agent;

    private void Awake() => agent = GetComponent<NavMeshAgent>();

    public void Initialize(float speed, float radius, float acceleration, int avoidancePriority)
    {
        agent.speed = speed;
        agent.radius = radius;
        agent.acceleration = acceleration;
        agent.avoidancePriority = avoidancePriority;
    }

    public void MoveTo(Vector3 position) => agent.SetDestination(position);
    public void Stop() => agent.ResetPath();

    public bool HasArrived =>
        !agent.pathPending
        && agent.remainingDistance <= agent.stoppingDistance + 0.1f
        && (!agent.hasPath || agent.velocity.sqrMagnitude < 0.01f);
}
```

---

## 14. Scene Setup

```
Scene
├── --- MANAGERS ---
├── GameManager
├── SpawnManager
├── SelectionManager
├── CameraManager
├── (other managers)
├── --- WORLD ---
├── Terrain
├── Obstacles/
├── (world features)
├── --- ENTITIES ---
├── (spawned at runtime or placed for testing)
├── --- CAMERA ---
├── CameraRig
├── --- UI ---
├── HUDDocument             ← UIDocument + HUDController
├── CommandPanelDocument    ← UIDocument + CommandPanelController
└── DebugOverlayDocument    ← UIDocument (disabled by default)
```

- Managers are prefabs placed at edit time, not spawned at runtime.
- Each UI panel is a separate GameObject with a `UIDocument` referencing its `.uxml` and a controller MonoBehaviour.
- Use `--- SECTION ---` named empty GameObjects as hierarchy separators.

---

## 15. Package Dependencies

### Required

| Package | Purpose |
|---------|---------|
| `com.unity.inputsystem` | Input handling |
| `com.unity.ai.navigation` | NavMesh pathfinding |
| `com.unity.ui` | UI Toolkit runtime |
| `com.unity.cinemachine` | Camera rigs |

### Optional

| Package | Purpose |
|---------|---------|
| `com.unity.test-framework` | EditMode / PlayMode tests |
| `com.unity.editorcoroutines` | Editor async workflows |

Do not add unlisted packages without explicit approval.

---

## 16. Common Mistakes to Avoid

| Mistake | Correct Approach |
|---------|-----------------|
| Deep inheritance hierarchies | Composition: attach focused components to prefab |
| Hardcoding stats in scripts | ScriptableObject definitions for all tunable values |
| `FindObjectOfType<>()` at runtime | `Manager.Instance` for managers, cached refs for components |
| `GetComponent<>()` in `Update()` on other objects | Cache at initialization |
| Separate code for manual vs. automated behavior | One system with priority levels |
| Gameplay logic in UI scripts | UI reads and displays only |
| Legacy `Input.GetKey()` | Unity Input System with InputReader |
| Canvas/UGUI for new UI | UI Toolkit (UXML + USS) |
| Scattering files across folders | Feature-folder ownership |
| `public` fields for Inspector | `[SerializeField] private` |
| Mutating ScriptableObjects at runtime | Runtime records for mutable state |
| One giant controller for everything | Each manager owns one responsibility |
| Inline styles in C# UI code | USS classes; only dynamic values in code |
| `GameObject.Find()` or `FindWithTag()` | Known references; `root.Q<T>()` for UI |
| Missing `OnDisable()` unsubscribe | Every subscribe has a matching unsubscribe |
| VFX/SFX scripts tightly coupled to gameplay | UnityEvents for designer-wired feedback |

---

## 17. Agent Workflow

When picking up a task, follow this sequence:

1. **Read the task** — Requirements and acceptance criteria.
2. **Read this conventions doc** — Confirm patterns and placement.
3. **Check dependencies** — Verify prerequisite systems exist.
4. **Create ScriptableObject definitions first** — `.cs` definition classes for new data.
5. **Create component scripts** — MonoBehaviours following the patterns above.
6. **Create UI (if needed)** — `.uxml` layout → `.uss` styles → controller `.cs`.
7. **Create or update prefabs** — Assemble following the prefab rules.
8. **Create data assets** — `.asset` files in the correct `Data/` folders.
9. **Wire the scene** — Add objects, assign references.
10. **Test against acceptance criteria** — Enter Play mode and verify.
11. **Commit** with a descriptive message.

### Commit format

```
feat({scope}): {brief description}
fix({scope}): {brief description}
refactor({scope}): {brief description}
docs({scope}): {brief description}
```

---

## 18. Quick Reference

```
QUESTION                                     → ANSWER
Where do shared entity scripts go?           → Entities/Scripts/
Where do entity-specific scripts go?         → Entities/{Type}/Scripts/
Where does a manager live?                   → Systems/{System}/Scripts/
Where do data assets go?                     → {Feature}/Data/
How to expose a field to Inspector?          → [SerializeField] private
How do entities get their stats?             → Spawner calls Initialize(definition)
How to handle input?                         → Subscribe to InputReader events
How does UI update?                          → Subscribe to C# events, update elements
How do designers wire VFX/SFX?              → UnityEvents on components, wired in Inspector
How to build a UI panel?                     → .uxml layout + .uss styles + Controller.cs
How to query UI elements?                    → root.Q<T>("element-name")
What drives entity automation?               → Scheduled logic or task system with priorities
How is team/faction color applied?           → TeamComponent sets material color at spawn
What event system for code?                  → C# events (event Action<T>)
What event system for designer feedback?     → UnityEvents (serialized, Inspector-wired)
What input system?                           → Unity Input System (com.unity.inputsystem)
What UI system?                              → UI Toolkit (UXML + USS)
Can I use LINQ?                              → Yes, but not in Update() or hot paths
Can I use async/await?                       → Yes, with Awaitable (Unity 6+) or UniTask
Can I mutate a ScriptableObject at runtime?  → No. Use a runtime record instead.
```
