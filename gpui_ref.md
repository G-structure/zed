# GPUI Comprehensive Reference

> A complete reference for the GPUI framework: architecture, core APIs, component libraries, ecosystem tools, and real-world application patterns. Compiled from source analysis of Zed's `crates/gpui/`, 3 component libraries, 7 applications, and 6 utility libraries.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Core Types & Traits](#2-core-types--traits)
3. [Rendering Pipeline](#3-rendering-pipeline)
4. [Layout System (Taffy/Flexbox)](#4-layout-system)
5. [Styling API (Tailwind-like)](#5-styling-api)
6. [Event System & Actions](#6-event-system--actions)
7. [Entity/State Management](#7-entitystate-management)
8. [Concurrency Model](#8-concurrency-model)
9. [Platform Abstraction](#9-platform-abstraction)
10. [Key Macros & Derives](#10-key-macros--derives)
11. [gpui-component Library (60+ Components)](#11-gpui-component-library)
12. [Ecosystem Libraries](#12-ecosystem-libraries)
13. [Application Patterns](#13-application-patterns)
14. [Ecosystem Projects](#14-ecosystem-projects)
15. [Quick Start Templates](#15-quick-start-templates)

---

## 1. Architecture Overview

GPUI is a hybrid immediate/retained-mode, GPU-accelerated UI framework for Rust. It powers the Zed code editor and is designed for high-performance desktop applications.

**Rendering backends:** Metal (macOS), WGPU (Linux/Windows), WASM (Web), Test (headless)

### Module Structure (`crates/gpui/src/`)

| Module | Purpose |
|--------|---------|
| `app.rs`, `app/` | Application context, entity management, global state |
| `element.rs`, `elements/` | Element system — div, text, img, list, canvas, etc. |
| `window.rs` | Window state, event dispatching, rendering context |
| `view.rs` | View rendering and caching |
| `scene.rs` | Paint operation batching and GPU primitives |
| `style.rs`, `styled.rs` | Tailwind-inspired styling API |
| `geometry.rs` | 2D coordinates, bounds, sizes, transforms |
| `platform.rs`, `platform/` | Cross-platform abstractions |
| `action.rs` | Action system for keyboard-driven UI |
| `executor.rs` | Async task management (foreground/background) |
| `subscription.rs` | Event subscriptions and observations |
| `input.rs` | Text input handling |
| `keymap.rs`, `key_dispatch.rs` | Keyboard event routing |
| `text_system/` | Font rendering and text layout |
| `taffy.rs` | Flexbox/grid layout engine integration |

---

## 2. Core Types & Traits

### Application Types

```rust
// Top-level application builder
pub struct Application(Rc<AppCell>);
impl Application {
    pub fn with_assets(self, asset_source: impl AssetSource) -> Self
    pub fn with_http_client(self, http_client: Arc<dyn HttpClient>) -> Self
    pub fn run<F>(self, on_finish_launching: F) where F: FnOnce(&mut App)
}

// Root context — all application state
pub struct App { /* ... */ }
```

### Context Types

| Type | When available | Derefs to |
|------|---------------|-----------|
| `App` | Root context, global state | — |
| `Context<T>` | Updating an `Entity<T>` | `App` |
| `AsyncApp` | Inside `cx.spawn()`, across `.await` | — |
| `AsyncWindowContext` | Inside window-scoped `cx.spawn_in()` | — |
| `Window` | Passed as separate `window` arg | — |

```rust
// Context<T> gives you both App access and entity-specific mutation
pub struct Context<'a, T> {
    app: &'a mut App,
    entity_state: WeakEntity<T>,
}
impl<'a, T> Deref for Context<'a, T> { type Target = App; }
```

### Entity Types

```rust
// Strong handle to state of type T
pub struct Entity<T> { /* EntityId + Arc<RwLock<EntityRefCounts>> */ }
impl<T> Entity<T> {
    pub fn entity_id(&self) -> EntityId
    pub fn downgrade(&self) -> WeakEntity<T>
    pub fn read(&self, cx: &App) -> &T
    pub fn read_with<R>(&self, cx: &App, read: impl FnOnce(&T, &App) -> R) -> R
    pub fn update<R>(&self, cx: &mut App, update: impl FnOnce(&mut T, &mut Context<T>) -> R) -> R
    pub fn update_in<R>(&self, cx: &mut AsyncWindowContext, update: impl FnOnce(&mut T, &mut Window, &mut Context<T>) -> R) -> Result<R>
}

// Weak handle — prevents circular references, returns Result
pub struct WeakEntity<T> { /* ... */ }
impl<T> WeakEntity<T> {
    pub fn upgrade(&self) -> Option<Entity<T>>
    pub fn read_with<R>(...) -> Result<R>
    pub fn update<R>(...) -> Result<R>
}
```

### Rendering Traits

```rust
// Stateful views — Entity<T> where T: Render is a "view"
pub trait Render: 'static + Sized {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement;
}

// Stateless components — consumed on render
pub trait RenderOnce: 'static {
    fn render(self, cx: &mut App) -> impl IntoElement;
}

// Low-level element trait
pub trait Element: 'static + IntoElement {
    type RequestLayoutState: 'static;
    type PrepaintState: 'static;
    fn id(&self) -> Option<ElementId>;
    fn request_layout(...) -> (LayoutId, Self::RequestLayoutState);
    fn prepaint(...) -> Self::PrepaintState;
    fn paint(...);
}

// Conversion to elements
pub trait IntoElement: Sized {
    type Element: Element;
    fn into_element(self) -> Self::Element;
    fn into_any_element(self) -> AnyElement;
}
```

### Window

```rust
pub struct Window { /* layout, rendering, event state */ }
// Passed as `window: &mut Window` — always comes before `cx` in signatures
// Used for: focus, actions, drawing, input state, layout
```

---

## 3. Rendering Pipeline

**Flow: Render -> Layout -> Prepaint -> Paint -> Scene -> GPU**

### Stage 1: Render (View -> Element Tree)
`Render::render()` is called, returning an element tree via fluent API.

### Stage 2: Layout (Element -> LayoutId via Taffy)
```rust
Element::request_layout(window, cx) -> (LayoutId, RequestLayoutState)
// Taffy computes flexbox layout, returns LayoutId reference
```

### Stage 3: Prepaint (Compute Rendering State)
```rust
Element::prepaint(bounds, request_layout_state, window, cx) -> PrepaintState
// Registers hitboxes, content masks, cached render state
```

### Stage 4: Paint (Render GPU Primitives)
```rust
Element::paint(bounds, request_layout_state, prepaint_state, window, cx)
// Inserts primitives into Scene
```

### GPU Primitive Types (scene.rs)
```rust
pub enum Primitive {
    Shadow(Shadow),
    Quad(Quad),           // Rectangle with border/shadow/corner radius
    Path(Path<ScaledPixels>),
    Underline(Underline),
    MonochromeSprite(MonochromeSprite),
    SubpixelSprite(SubpixelSprite),
    PolychromeSprite(PolychromeSprite),
    Surface(PaintSurface),
}
```

### View Caching
Views can opt into caching via `.cached()`. Cache invalidated by `cx.notify()`, `window.refresh()`, or entity changes.

---

## 4. Layout System

GPUI uses the **taffy** crate for flexbox and grid layout.

### Style Struct (key fields)
```rust
pub struct Style {
    pub display: Display,           // Block, Flex, Grid, None
    pub position: Position,         // Absolute, Relative, Fixed
    pub size: Size<Length>,
    pub min_size: Size<Length>,
    pub max_size: Size<Length>,
    pub margin: Edges<Length>,
    pub padding: Edges<DefiniteLength>,
    pub gap: Size<DefiniteLength>,
    pub flex_direction: FlexDirection,
    pub flex_wrap: FlexWrap,
    pub flex_grow: f32,
    pub flex_shrink: f32,
    pub align_items: Option<AlignItems>,
    pub justify_content: Option<JustifyContent>,
    pub background: Option<Fill>,
    pub border_color: Option<Hsla>,
    pub corner_radii: Corners<AbsoluteLength>,
    pub box_shadow: Vec<BoxShadow>,
    pub text: TextStyleRefinement,
    pub opacity: Option<f32>,
    pub grid_cols: Option<u16>,
    pub grid_rows: Option<u16>,
}
```

### Length Types
```rust
pub enum Length { Definite(DefiniteLength), Auto }
pub enum DefiniteLength { Pixels(Pixels), Rems(f32), Relative(f32) }
```

---

## 5. Styling API

Tailwind-like method chaining via the `Styled` trait.

### Layout
```rust
div()
    .flex()                     // display: flex
    .flex_col()                 // flex-direction: column
    .flex_row()                 // flex-direction: row
    .flex_1()                   // flex: 1 1 0%
    .flex_auto()                // flex: 1 1 auto
    .flex_none()                // flex: none
    .gap(px(8.))                // gap: 8px
    .justify_center()           // justify-content: center
    .items_center()             // align-items: center
    .items_start()
    .items_end()
```

### Sizing
```rust
    .w(px(200.))                // width: 200px
    .h(px(100.))                // height: 100px
    .w_full()                   // width: 100%
    .h_full()                   // height: 100%
    .min_w(px(50.))
    .max_w(px(500.))
    .size_full()                // width: 100%; height: 100%
```

### Spacing
```rust
    .m(px(8.))                  // margin: 8px
    .mx(px(16.))                // margin-left/right: 16px
    .my(px(8.))                 // margin-top/bottom: 8px
    .mt(px(4.))                 // margin-top: 4px
    .p(px(12.))                 // padding: 12px
    .px(px(16.))                // padding-left/right: 16px
    .py(px(8.))                 // padding-top/bottom: 8px
```

### Colors & Borders
```rust
    .bg(cx.theme().colors.background)
    .text_color(gpui::white())
    .border_1()                 // border-width: 1px
    .border_color(gpui::red())
    .rounded(px(4.))            // border-radius: 4px
    .rounded_md()               // border-radius: medium
    .shadow()                   // box-shadow
```

### Text
```rust
    .text_size(px(14.))
    .font_weight(FontWeight::BOLD)
    .text_center()
    .text_ellipsis()
    .truncate()
    .line_clamp(2)
```

### Visibility & Cursor
```rust
    .hidden()
    .visible()
    .cursor_pointer()
    .cursor_text()
    .overflow_hidden()
    .overflow_scroll()
```

### Conditional Styling
```rust
    .when(is_selected, |this| this.bg(blue_500()))
    .when_some(optional_label, |this, label| this.child(label))
```

### Interactive States
```rust
    .hover(|style| style.bg(gray_100()))
    .group_hover("group-name", |style| style.visible())
```

---

## 6. Event System & Actions

### Mouse Events
```rust
element
    .on_mouse_down(MouseButton::Left, |event, window, cx| { ... })
    .on_mouse_up(MouseButton::Left, |event, window, cx| { ... })
    .on_mouse_move(|event, window, cx| { ... })
    .on_click(|event, window, cx| { ... })
    .on_scroll_wheel(|event, window, cx| { ... })
```

### Keyboard Events
```rust
element
    .on_key_down(|event, window, cx| { ... })
    .on_key_up(|event, window, cx| { ... })
```

### Actions

Actions are dispatched via keyboard bindings or code.

```rust
// Define simple actions (no data)
actions!(my_namespace, [Save, Quit, Undo, Redo]);

// Define actions with data
#[derive(Clone, PartialEq, serde::Deserialize, schemars::JsonSchema, Action)]
#[action(namespace = editor)]
pub struct SelectNext { pub replace_newest: bool }

// Register keybindings
cx.bind_keys([
    KeyBinding::new("cmd-s", Save, None),
    KeyBinding::new("cmd-q", Quit, None),
]);

// Handle actions on elements
element.on_action(|action: &Save, window, cx| { ... })

// Dispatch actions in code
window.dispatch_action(Save.boxed_clone(), cx);
focus_handle.dispatch_action(&Save, window, cx);
```

### Listener Pattern (entity context)
```rust
// cx.listener wraps closure to provide entity access
element.on_click(cx.listener(|this: &mut MyView, event, window, cx: &mut Context<MyView>| {
    this.handle_click(event, cx);
}))
```

### Dispatch Phases
```rust
pub enum DispatchPhase {
    Capture,  // Root -> focused element
    Bubble,   // Focused element -> root (default)
}
```

---

## 7. Entity/State Management

### Creating Entities
```rust
let entity: Entity<MyView> = cx.new(|cx: &mut Context<MyView>| MyView { counter: 0 });

// For circular references
let reservation = cx.reserve_entity::<MyView>();
let entity = cx.insert_entity(reservation, |cx| MyView { ... });
```

### Reading & Updating
```rust
// Read
let value = entity.read(cx).counter;
entity.read_with(cx, |view, cx| view.counter);

// Update
entity.update(cx, |view, cx| { view.counter += 1; cx.notify(); });

// Update with window access
entity.update_in(window_cx, |view, window, cx| { ... });
```

### Notifications (trigger re-render)
```rust
cx.notify()  // Marks view as needing re-render
```

### Event Emission
```rust
// Declare what events an entity can emit
impl EventEmitter<MyEvent> for MyView {}

// Emit events
cx.emit(MyEvent { data: 42 });

// Subscribe to events from another entity
let sub = cx.subscribe(&other_entity, |this, other, event: &MyEvent, cx| {
    this.handle_event(event, cx);
});
// Store sub in Vec<Subscription> to keep it alive
```

### Observations
```rust
// Watch for entity changes (triggered by notify())
cx.observe(&other_entity, |this, other, cx| { ... });

// Watch self
cx.observe_self(|this, cx| { ... });

// Watch entity release (drop)
cx.on_release(|this, cx| { ... });
cx.observe_release(&other, |this, other, cx| { ... });

// Watch globals
cx.observe_global::<MyGlobal>(|this, cx| { ... });
```

### Global State
```rust
pub trait Global: 'static {}

cx.set_global(MyGlobal { ... });
cx.update_global(|global: &mut MyGlobal, cx| { ... });
let global = cx.global::<MyGlobal>();
```

---

## 8. Concurrency Model

Single-threaded rendering with background task support.

### Foreground Tasks (main thread)
```rust
// From App context
cx.spawn(async move |cx: &mut AsyncApp| {
    let data = fetch_data().await;
    cx.update(|cx| { /* update state */ })?;
    Ok(())
}).detach();

// From entity context — provides WeakEntity<T>
cx.spawn(async move |this: WeakEntity<MyView>, cx: &mut AsyncApp| {
    let result = compute().await;
    this.update(cx, |this, cx| {
        this.result = result;
        cx.notify();
    })?;
    Ok(())
}).detach();
```

### Background Tasks (thread pool)
```rust
let task = cx.background_spawn(async move {
    // Must be Send + 'static
    expensive_computation()
});
let result = task.await;
```

### Task Lifecycle
```rust
pub struct Task<T>(/* ... */);
impl<T> Task<T> {
    pub fn ready(val: T) -> Self    // Immediate value
    pub fn detach(self)              // Run forever (don't cancel on drop)
}
// Tasks are Future — can be .await'd
// Dropping a Task cancels it
// Store in a field to tie lifetime to struct
```

### Priority Levels
```rust
pub enum Priority {
    Lowest, Low, Default, High, Highest,
    RealtimeAudio,  // Dedicated thread
}
cx.background_spawn_with_priority(Priority::High, async { ... });
```

---

## 9. Platform Abstraction

### Platform Trait (key methods)
```rust
pub trait Platform: 'static {
    fn run(&self, on_finish_launching: Box<dyn FnOnce()>);
    fn quit(&self);
    fn activate(&self, ignoring_other_apps: bool);
    fn displays(&self) -> Vec<Rc<dyn PlatformDisplay>>;
    fn open_window(...) -> Result<Box<dyn PlatformWindow>>;
    fn open_url(&self, url: &str);
    fn prompt_for_paths(...) -> oneshot::Receiver<Result<Option<Vec<PathBuf>>>>;
    fn read_from_clipboard(&self) -> Option<ClipboardItem>;
    fn write_to_clipboard(&self, item: ClipboardItem);
    fn set_menus(&self, menus: Vec<Menu>, keymap: &Keymap);
}
```

### Implementations
| Platform | Backend |
|----------|---------|
| macOS | Native Cocoa / Metal |
| Linux | Wayland or X11 / WGPU |
| Windows | Win32 / WGPU |
| WASM | Web Canvas |
| Test | Mock (headless) |

---

## 10. Key Macros & Derives

### `#[derive(IntoElement)]` — Stateless components
```rust
#[derive(IntoElement)]
pub struct MyComponent { pub title: SharedString }
impl RenderOnce for MyComponent {
    fn render(self, cx: &mut App) -> impl IntoElement {
        div().child(self.title)
    }
}
```

### `actions!()` — Simple actions
```rust
actions!(editor, [MoveUp, MoveDown, SelectAll]);
// Expands to unit structs with Action impl
```

### `#[derive(Action)]` — Actions with data
```rust
#[derive(Clone, PartialEq, Deserialize, JsonSchema, Action)]
#[action(namespace = editor)]
pub struct GoToLine { pub line: u32 }
```

### `register_action!()` — Registration
```rust
register_action!(MyAction);
```

---

## 11. gpui-component Library

**Repository:** [longbridge/gpui-component](https://github.com/longbridge/gpui-component)
**60+ production-grade components** inspired by macOS/Windows + shadcn/ui design.

### Setup
```rust
fn main() {
    Application::new().with_assets(Assets).run(|cx: &mut App| {
        gpui_component::init(cx);  // MUST be called first
        cx.open_window(WindowOptions::default(), |window, cx| {
            let view = cx.new(|_| MyView);
            cx.new(|cx| Root::new(view, window, cx))  // MUST wrap in Root
        });
    });
}
```

### Component Catalog

#### Input Components
| Component | Type | Key Features |
|-----------|------|-------------|
| **Input** | Entity-based | Rope text, LSP, syntax highlighting, completions |
| **NumberInput** | Entity-based | Numeric validation, step controls |
| **OtpInput** | Entity-based | Masked one-time password entry |
| **Select** | Entity-based | Dropdown with search, delegates |
| **Checkbox** | RenderOnce | checked, label, on_click |
| **Radio** | RenderOnce | Radio group |
| **Switch** | RenderOnce | Toggle with label |
| **Slider** | RenderOnce | Single or range, vertical support |
| **ColorPicker** | Entity-based | Hex, RGB, HSL inputs |
| **DatePicker** | Entity-based | Calendar, date range selection |
| **Form** | RenderOnce | Horizontal/vertical layout, columns |
| **Field** | RenderOnce | Label + control wrapper |

#### Display Components
| Component | Key Features |
|-----------|-------------|
| **Text / TextView** | Markdown & HTML rendering, syntax highlighting |
| **Badge** | Variants: Primary, Secondary, Success, Warning, Danger, Info |
| **Tag** | Categorization labels |
| **Avatar** | Image with fallback initials, status indicator |
| **Icon** | SVG icons from Lucide set (100+) |
| **Kbd** | Keyboard key display |
| **Skeleton** | Loading placeholders |
| **Spinner** | Loading indicator |
| **Progress** | Linear progress bar |
| **ProgressCircle** | Circular progress |
| **Alert** | Info, Success, Warning, Error message boxes |
| **Rating** | Star rating (interactive or display) |
| **DescriptionList** | Key-value pair display |

#### Navigation Components
| Component | Key Features |
|-----------|-------------|
| **TabBar** | Variants: Tab, Outline, Pill, Segmented, Underline |
| **Breadcrumb** | Trail with separator, icons |
| **Pagination** | Page navigation with callbacks |
| **Stepper** | Step progression indicator |
| **Link** | Hyperlink with external support |
| **Sidebar** | Collapsible, header/footer, menu groups |

#### Container Components
| Component | Key Features |
|-----------|-------------|
| **Dialog** | Modal with title, content, footer, OK/Cancel |
| **AlertDialog** | Specialized alert modal |
| **Sheet** | Slide-in panel (left/right/top/bottom), resizable |
| **Popover** | Floating panel with anchor positioning |
| **HoverCard** | Hover-triggered content panel |
| **Tooltip** | Hover tooltip |
| **Accordion** | Collapsible items, single or multiple |
| **Collapsible** | Single collapsible section |
| **GroupBox** | Container with title |

#### Menu Components
| Component | Key Features |
|-----------|-------------|
| **DropdownMenu** | Button-triggered dropdown |
| **ContextMenu** | Right-click menu |
| **PopupMenu** | Generic popup |
| **AppMenuBar** | Application top menu bar |

#### Data Components
| Component | Key Features |
|-----------|-------------|
| **List** | Virtual scrolling, search, keyboard nav |
| **DataTable** | Virtual rows/columns, sorting, pagination, column resize |
| **Table** | Simple stateless table (Header, Body, Row, Cell) |
| **Tree** | Hierarchical view, expand/collapse, keyboard nav |

#### Chart Components
| Component | Key Features |
|-----------|-------------|
| **LineChart** | Multiple series, scales, grid, tooltip |
| **AreaChart** | Stacked area support |
| **BarChart** | Vertical/horizontal, grouped/stacked |
| **PieChart** | Pie/donut with labels |
| **CandlestickChart** | OHLC financial data |

#### Layout Components
| Component | Key Features |
|-----------|-------------|
| **DockArea** | Complex panel management with drag-drop |
| **Dock** | Left/right/bottom panels |
| **DockItem** | Split, Tabs, or Panel variants |
| **TabPanel** | Tabbed container within dock |
| **StackPanel** | Resizable stacked panels |
| **Tiles** | Freeform floating panel arrangement |
| **ResizablePanel** | Panel with resize handle |
| **Scrollable** | Scrollable container with scrollbar |
| **VirtualList** | High-performance virtual scrolling |

#### Utility Components
| Component | Key Features |
|-----------|-------------|
| **Root** | Window-level manager (dialogs, sheets, notifications, focus) |
| **TitleBar** | Platform-specific title bar |
| **FocusTrap** | Constrain Tab/Shift+Tab within bounds |
| **Notification** | Toast notification system |
| **Settings** | Settings page/group/item framework |
| **Divider** | Visual separator |

### Theme System
```rust
// Access current theme
let theme = cx.theme();
let colors = &theme.colors;

// Key color properties
theme.colors.background
theme.colors.foreground
theme.colors.primary / .primary_foreground
theme.colors.danger / .danger_foreground
theme.colors.success / .success_foreground
theme.colors.warning / .warning_foreground
theme.colors.muted / .muted_foreground

// Theme modes
ThemeMode::Light | ThemeMode::Dark | ThemeMode::Auto

// Change theme
Theme::change(mode, config, cx);
```

### Size System
All sizable components support: `XSmall`, `Small`, `Medium`, `Large`, `Size(DefiniteLength)`

### Component Design Patterns
```rust
// Builder pattern (all components)
Button::new("id")
    .label("Click me")
    .primary()
    .on_click(|_, _, _| {})
    .disabled(false)

// Entity-based (stateful components)
let input_state = cx.new(|cx| InputState::new(window, cx));
Input::new(input_state).placeholder("Type here...").cleanable(true)

// Callback pattern
button.on_click(|event, window, cx| { /* ... */ })
switch.on_click(|checked, window, cx| { /* ... */ })
slider.on_change(|value, window, cx| { /* ... */ })
```

---

## 12. Ecosystem Libraries

### gpui-hooks — React-Style Hooks
**Repo:** [leset0ng/gpui-hooks](https://github.com/leset0ng/gpui-hooks)

```rust
#[hook_element]
struct CounterApp {}

impl HookedRender for CounterApp {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        let (count, set_count) = self.use_state(|| 0i32);
        let doubled = self.use_memo(|| count() * 2, [count()]);
        self.use_effect(|| {
            println!("Count changed: {}", count());
            Some(|| println!("Cleanup"))
        }, [count()]);
        div().child(format!("Count: {}", count()))
    }
}
```

Hooks: `use_state`, `use_effect`, `use_memo`, `use_ref`, `use_callback`

### gpui-router — Declarative Routing
**Repo:** [justjavac/gpui-router](https://github.com/justjavac/gpui-router)

```rust
fn main() {
    Application::new().run(|cx: &mut App| {
        router_init(cx);
        // ...
    });
}

Routes::new().child(
    Route::new().path("/").element(|_, _| layout()).children(vec![
        Route::new().index().element(|_, _| home()),
        Route::new().path("about").element(|_, _| about()),
        Route::new().path("users/:id").element(|params, _| user(params)),
        Route::new().path("{*not_match}").element(|_, _| not_found()),
    ]),
)
```

Features: nested routes, dynamic segments, wildcards, lazy evaluation.

### gpui-nav — Screen Navigation
**Repo:** [benodiwal/gpui-nav](https://github.com/benodiwal/gpui-nav)

```rust
navigator.push(screen, cx);          // Add to stack
navigator.pop(cx);                    // Remove from stack
navigator.replace(screen, cx);        // Replace current
navigator.clear_and_push(screen, cx); // Reset and push
```

Stack-based navigation (like mobile apps). Screens implement the `Screen` trait.

### gpui-form — Form Generation
**Repo:** [stayhydated/gpui-form](https://github.com/stayhydated/gpui-form)

Derive macros for automatic form generation from structs with validation.

### gpui-storybook — Component Gallery
**Repo:** [stayhydated/gpui-storybook](https://github.com/stayhydated/gpui-storybook)

Framework for building component showcases with macro-driven story creation, fixtures, and theme switching.

### Additional Libraries
| Library | Purpose | Repo |
|---------|---------|------|
| **gpui-symbols** | Native SF Symbols for GPUI | AprilNEA/gpui-symbols |
| **gpui-video-player** | Video playback | cijiugechu/gpui-video-player |
| **gpui-d3rs** | D3.js-style plotting | pierreaubert/sotf |
| **plotters-gpui** | plotters backend for GPUI | JakkuSakura/plotters-gpui |
| **adabraka-ui** | Additional UI components | Augani/adabraka-ui |
| **gpui-ui-kit** | 40+ composable components | (crates.io) |

---

## 13. Application Patterns

### Minimal App (Setu pattern, ~16 lines)
```rust
fn main() {
    env_logger::init();
    Application::new().with_assets(Assets).run(|cx: &mut App| {
        gpui_component::init(cx);
        init_theme(cx);
        MyApp::register_actions(cx);
        MyApp::register_keybindings(cx);
        MyApp::create_main_window(cx);
        cx.activate(true);
    });
}
```

### Entity Composition (Postman pattern)
```rust
pub struct MyApp {
    sidebar: Entity<Sidebar>,
    editor: Entity<Editor>,
    status_bar: Entity<StatusBar>,
    _subscriptions: Vec<Subscription>,
}

impl MyApp {
    pub fn new(window: &mut Window, cx: &mut Context<Self>) -> Self {
        let sidebar = cx.new(|cx| Sidebar::new(cx));
        let editor = cx.new(|cx| Editor::new(cx));
        let mut subs = Vec::new();
        subs.push(cx.subscribe(&sidebar, |this, _, event, cx| {
            this.handle_sidebar_event(event, cx);
        }));
        Self { sidebar, editor, status_bar: cx.new(|_| StatusBar), _subscriptions: subs }
    }
}

impl Render for MyApp {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div().flex().size_full()
            .child(self.sidebar.clone())
            .child(self.editor.clone())
    }
}
```

### Global State Store (Zedis pattern)
```rust
pub struct AppStore {
    theme_mode: ThemeMode,
    font_size: f32,
    locale: String,
}
impl Global for AppStore {}

// Initialize
cx.set_global(AppStore { ... });

// Access
let store = cx.global::<AppStore>();

// Update
cx.update_global::<AppStore, _>(|store, cx| {
    store.theme_mode = ThemeMode::Dark;
});
```

### Persistent Window Bounds (Zedis pattern)
```rust
// Save with debounce
window.on_window_bounds_changed(cx.listener(|this, window, cx| {
    let bounds = window.bounds();
    cx.spawn(async move |_, cx| {
        cx.background_executor().timer(Duration::from_millis(500)).await;
        save_bounds(bounds).ok();
        Ok(())
    }).detach();
}));
```

### State Stack Navigation (Loungy pattern)
Query-driven architecture where a central `TextInput` drives all interactions. Nested views push/pop state without complexity.

### Action-Driven Architecture (Setu pattern)
```rust
// 130+ keybindings
cx.bind_keys([
    KeyBinding::new("alt-g", HttpGet, None),
    KeyBinding::new("alt-p", HttpPost, None),
    KeyBinding::new("ctrl-tab", NextTab, None),
]);
```

---

## 14. Ecosystem Projects

### Applications

| Project | Description | Key Patterns |
|---------|-------------|-------------|
| [**Zed**](https://github.com/zed-industries/zed) | High-perf code editor | Full GPUI showcase |
| [**Loungy**](https://github.com/MatthiasGrandl/loungy) | App launcher (Spotlight/Raycast) | State stack, query-driven |
| [**nohrs**](https://github.com/noh-rs/nohrs) | macOS file explorer | Library-first, feature-gated GUI |
| [**helix-gpui**](https://github.com/polachok/helix-gpui) | Helix editor frontend | MVVM, complex event flow |
| [**postman-gpui**](https://github.com/847850277/postman-gpui) | HTTP client | Entity composition |
| [**setu**](https://github.com/bajrangCoder/setu) | API testing | Action-driven, minimal |
| [**zedis**](https://github.com/vicanso/zedis) | Redis GUI | Global state, persistence, i18n |
| [**Hummingbird**](https://github.com/143mailliw/hummingbird) | Music player | Media playback, FLAC/MP3/OGG/WAV |
| [**pgui**](https://github.com/duanebester/pgui) | Postgres GUI | Database management |
| [**vleer**](https://github.com/vleerapp/vleer) | Music streaming | OpenMusic API |
| [**Fulgur**](https://github.com/fulgur-app/Fulgur) | Code editor + sync | Encrypted file sync |
| [**coop**](https://github.com/lumehq/coop) | Nostr messaging | P2P messaging |
| [**zqlz**](https://github.com/samurmaykrr/zqlz) | Database IDE | SQLite, Postgres, MySQL, Redis |

### Tooling

| Tool | Description |
|------|-------------|
| [**create-gpui-app**](https://github.com/zed-industries/create-gpui-app) | Scaffold new GPUI app |
| [**Pulsar-Native**](https://github.com/Far-Beyond-Pulsar/Pulsar-Native) | Game engine on GPUI |
| [**React Native GPUI**](https://github.com/rngpui/react-native-gpui) | RN Fabric on Rust/GPUI |

### Learning Resources

| Resource | Link |
|----------|------|
| Official website | [gpui.rs](https://www.gpui.rs/) |
| Official awesome list | [zed-industries/awesome-gpui](https://github.com/zed-industries/awesome-gpui) |
| Community awesome list | [edo-zhou/awesome-gpui](https://github.com/edo-zhou/awesome-gpui) |
| GPUI Book | [MatinAniss/gpui-book](https://github.com/MatinAniss/gpui-book) |
| gpui-component docs | [longbridge.github.io/gpui-component](https://longbridge.github.io/gpui-component/) |
| Ownership blog post | [zed.dev/blog/gpui-ownership](https://zed.dev/blog/gpui-ownership) |
| Rewrite blog post | [zed.dev/blog/why-the-big-rewrite](https://zed.dev/blog/why-the-big-rewrite) |
| YouTube streams | [Duane Bester - Rust Streams](https://www.youtube.com/playlist?list=PLzIkykhdNahwxfVbxgZR69TQSsJc-6Rqq) |

---

## 15. Quick Start Templates

### Bare GPUI (no component library)
```rust
use gpui::prelude::*;
use gpui::{Application, WindowOptions, div, px};

struct MyView { counter: i32 }

impl Render for MyView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div()
            .flex().flex_col().items_center().justify_center()
            .size_full().bg(gpui::white())
            .child(format!("Counter: {}", self.counter))
            .child(
                div().px(px(16.)).py(px(8.))
                    .bg(gpui::blue())
                    .rounded(px(4.))
                    .cursor_pointer()
                    .child("Increment")
                    .on_click(cx.listener(|this, _, _, cx| {
                        this.counter += 1;
                        cx.notify();
                    }))
            )
    }
}

fn main() {
    Application::new().run(|cx: &mut App| {
        cx.open_window(WindowOptions::default(), |_window, cx| {
            cx.new(|_| MyView { counter: 0 })
        }).unwrap();
        cx.activate(true);
    });
}
```

### With gpui-component
```rust
use gpui::prelude::*;
use gpui::{Application, WindowOptions};
use gpui_component::{self, button::Button, Root};

struct MyView;

impl Render for MyView {
    fn render(&mut self, _window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        div().flex().flex_col().gap(px(8.)).p(px(16.))
            .child(Button::new("btn").label("Primary").primary())
            .child(Button::new("btn2").label("Danger").danger())
    }
}

fn main() {
    Application::new().with_assets(Assets).run(|cx: &mut App| {
        gpui_component::init(cx);
        cx.open_window(WindowOptions::default(), |window, cx| {
            let view = cx.new(|_| MyView);
            cx.new(|cx| Root::new(view, window, cx))
        }).unwrap();
        cx.activate(true);
    });
}
```

### Cargo.toml Dependencies
```toml
[dependencies]
# From git (latest)
gpui = { git = "https://github.com/zed-industries/zed", branch = "main" }

# With component library
gpui-component = { git = "https://github.com/longbridge/gpui-component.git" }

# Optional ecosystem
gpui-router = "0.2"
gpui-hooks = { git = "https://github.com/leset0ng/gpui-hooks.git" }
```

---

## 16. Advanced Graphics, Canvas & Custom Drawing

### The Canvas Element — Low-Level Drawing Primitive

GPUI's `canvas()` element gives direct access to the paint API without defining a custom Element:

```rust
use gpui::canvas;

canvas(
    // Phase 1: prepaint — compute state, register hitboxes
    |bounds: Bounds<Pixels>, window: &mut Window, cx: &mut App| {
        // Return any state needed for paint phase
        ()
    },
    // Phase 2: paint — draw to the GPU
    |bounds: Bounds<Pixels>, _state, window: &mut Window, cx: &mut App| {
        // Draw rectangles (quads)
        window.paint_quad(fill(bounds, gpui::red()));

        // Draw vector paths
        let mut builder = PathBuilder::fill();
        builder.move_to(point(px(0.), px(0.)));
        builder.line_to(point(px(100.), px(50.)));
        builder.curve_to(point(px(150.), px(0.)), point(px(200.), px(50.)));
        let path = builder.build().unwrap();
        window.paint_path(path, gpui::blue());
    },
)
.size_full()
```

### Window Paint API — All Drawing Primitives

```rust
// Rectangles with borders, shadows, corner radii
window.paint_quad(quad);

// Vector paths (lines, curves, polygons)
window.paint_path(path, color);

// Compositing layers (nested draw ordering)
window.paint_layer(bounds, |window| { /* nested drawing */ });

// Text glyphs, emoji, SVG, images
window.paint_glyph(...);
window.paint_emoji(...);
window.paint_svg(...);
window.paint_image(...);

// macOS: CoreVideo pixel buffer surfaces
window.paint_surface(bounds, cv_pixel_buffer);

// Shadows
window.paint_shadows(...);
```

### PathBuilder — Lyon-Based Vector Graphics

```rust
use gpui::PathBuilder;

// Stroked path
let mut builder = PathBuilder::stroke(px(2.0));
builder.move_to(start);
builder.line_to(end);
builder.curve_to(control1, control2);  // Cubic bezier
builder.arc_to(radius, sweep_angle, x_rotation);
builder.add_polygon(&[p1, p2, p3, p4]);  // Closed polygon

// Transforms
builder.translate(point(px(10.), px(20.)));
builder.scale(2.0, 2.0);
builder.rotate(radians);

// Dashed lines
builder.dash_array = Some(vec![px(5.0), px(3.0)]);

let path = builder.build().unwrap();
window.paint_path(path, color);
```

### Animation System

```rust
use gpui::Animation;

// Animate any element
element.with_animation(
    "my-animation",
    Animation::new(Duration::from_secs(2))
        .repeat()                    // Loop forever
        .with_easing(ease_in_out),   // Easing function
    |element, delta| {
        // delta: 0.0 -> 1.0 over duration
        element.with_transformation(Transformation::rotate(percentage(delta)))
    },
)

// Built-in easing functions:
// linear, quadratic, ease_in_out, ease_out_quint, bounce, pulsating_between
```

### 60fps Render Loop Pattern (for real-time visualization)

```rust
struct AnimatedView { /* state */ }

impl Render for AnimatedView {
    fn render(&mut self, window: &mut Window, cx: &mut Context<Self>) -> impl IntoElement {
        // Schedule next frame immediately
        cx.defer_in(window, |this, _, cx| { cx.notify(); });

        // Update simulation state
        self.tick();

        // Draw
        canvas(|_, _, _| {}, move |bounds, _, window, _| {
            // Paint thousands of primitives per frame
            for point in &self.points {
                window.paint_quad(fill(
                    gpui::bounds(point.pos, size(px(2.), px(2.))),
                    point.color,
                ));
            }
        }).size_full()
    }
}
```

### Scene Primitives (GPU-level)

```rust
pub enum Primitive {
    Shadow(Shadow),              // Box shadows
    Quad(Quad),                  // Rectangles with border/shadow/corner radius
    Path(Path<ScaledPixels>),    // Vector paths
    Underline(Underline),        // Text decorations
    MonochromeSprite(MonochromeSprite),  // Single-color sprites (icons)
    SubpixelSprite(SubpixelSprite),      // Subpixel-rendered text
    PolychromeSprite(PolychromeSprite),  // Full-color sprites (images)
    Surface(PaintSurface),       // macOS CoreVideo surfaces
}

// 2D transformation matrix for sprites/surfaces
pub struct TransformationMatrix {
    pub rotation_scale: [[f32; 2]; 2],  // 2x2 rotation+scale
    pub translation: [f32; 2],           // Translation vector
}
```

### Surface Element (Hardware Video/External Rendering)

```rust
// macOS only — render CoreVideo pixel buffers
use gpui::surface;
surface(cv_pixel_buffer)
    .object_fit(ObjectFit::Contain)
    .size_full()
```

---

## 17. Visualization & Plotting Libraries

### plotters-gpui — Full Plotters Backend for GPUI

**Repo:** [JakkuSakura/plotters-gpui](https://github.com/JakkuSakura/plotters-gpui)

Implements `plotters::DrawingBackend` using GPUI primitives. Anything plotters can draw, this puts into a GPUI window.

**Available examples:**
- `3d-plot.rs` — 3D surface plots with rotation/projection
- `3d-plot2.rs` — Additional 3D visualization
- `mandelbrot.rs` — Pixel-level fractal rendering
- `animation-cosine.rs` — Real-time animated sine waves at 60fps
- `animation-cpu.rs` — CPU usage animation
- `stock.rs` — Stock chart
- `area-chart.rs` — Area chart
- `normal-dist2.rs` — Statistical distribution
- `window.rs` — Basic windowed plot

**How it works:**

```rust
// 1. Define a chart
struct MyChart;
impl PlottersChart for MyChart {
    fn plot(&mut self, root: &DrawingArea<GpuiBackend, Shift>) -> Result<(), DrawingErrorKind> {
        let mut chart = ChartBuilder::on(root)
            .build_cartesian_3d(-3.0..3.0, -3.0..3.0, -3.0..3.0)?;
        chart.draw_series(SurfaceSeries::xoz(
            x_range, z_range, |x, z| (x*x + z*z).cos()
        ).style(BLUE.mix(0.2).filled()))?;
        Ok(())
    }
}

// 2. Render uses canvas() internally
impl Render for PlottersDrawAreaViewer {
    fn render(&mut self, _: &mut Window, _: &mut Context<Self>) -> impl IntoElement {
        let this = self.clone();
        canvas(
            |_, _, _| {},
            move |bounds, _, window, cx| {
                this.plot(bounds, window, cx).ok();
            },
        ).size_full()
    }
}
```

**Backend translates plotters calls to GPUI:**
- `draw_pixel()` -> `window.paint_quad()` (1px filled quad)
- `draw_line()` -> custom `Line` element with `render_pixels()`
- `draw_rect()` / `fill_polygon()` -> `PathBuilder::fill()` + `window.paint_path()`
- `draw_text()` -> `window.text_system().shape_line()` + `shaped_line.paint()`

### gpui-plot — Higher-Level Plotting with Zoom/Pan

**Repo:** [JakkuSakura/gpui-plot](https://github.com/JakkuSakura/gpui-plot)

Built on top of plotters-gpui with added interactivity:

```rust
// Figures with axes, markers, and animation
let model = FigureModel::new("My Figure".to_string());
let axes_bounds = AxesBounds::new(AxisRange::new(0.0, 100.0), AxisRange::new(0.0, 100.0));
let grid = GridModel::from_numbers(10, 10);
let axes_model = AxesModel::new(axes_bounds, grid);

// Add markers with different shapes
let mut markers = Markers::new();
markers.add_marker(Marker::new(point2(50.0, 50.0), px(10.0))
    .shape(MarkerShape::Circle)
    .color(Hsla::red()));

// MarkerShapes: Circle, Square, TriangleUp, TriangleDown
```

Features: FPS counter, zoom/pan interaction, native GPUI geometry + plotters in same figure.

### gpui-component Charts (D3.js-inspired)

Built into the gpui-component library (`crates/ui/src/plot/`):

- **LineChart** — multi-series with StrokeStyle (Natural, Linear, StepAfter)
- **BarChart** — vertical/horizontal, grouped/stacked
- **AreaChart** — stacked area
- **PieChart** — with `start_angle`, `end_angle`, `pad_angle`
- **CandlestickChart** — OHLC financial data
- **Arc shapes** — inner/outer radius for donut charts

Scale system: `ScaleLinear`, `ScaleBand`, `ScalePoint`, `ScaleOrdinal`

### gpui-toolkit (gpui-d3rs + gpui-px)

**Repo:** [pierreaubert/gpui-toolkit](https://github.com/pierreaubert/gpui-toolkit)

- **gpui-d3rs**: Low-level D3.js-style primitives (scales, colors, contours, Delaunay triangulation, quadtrees)
- **gpui-px**: Plotly Express-style high-level charting API
- **gpui-themes**: Theme editor and showcase
- **gpui-ui-kit**: 40+ components with tests (including volume knobs, potentiometers, vertical sliders)

---

## 18. Game Engine & 3D: Pulsar-Native

**Repo:** [Far-Beyond-Pulsar/Pulsar-Native](https://github.com/Far-Beyond-Pulsar/Pulsar-Native)

A game engine built on GPUI for the editor UI + Bevy for 3D rendering. The most advanced GPUI project in the ecosystem.

### Architecture: 3-Layer GPU Compositor

```
Layer 0 (bottom):  Black background
Layer 1 (middle):  Bevy 3D scene (D3D12, opaque)
Layer 2 (top):     GPUI UI overlay (alpha-blended, transparent)
```

- **Zero-copy GPU texture sharing** between Bevy (D3D12) and GPUI (D3D11) via `OpenSharedResource`
- GPUI renders lazily (only when `needs_render` is true)
- Bevy renders continuously for real-time 3D viewports
- Frame profiling with `profiling::profile_scope!`
- Device error recovery and texture size mismatch handling

### Crate Structure

```
crates/
  engine/              # Core engine (init graph, window, rendering)
  engine_backend/      # GPU backend abstraction
  engine_state/        # Shared engine state
  engine_fs/           # Filesystem layer
  ui/                  # GPUI-based editor UI (fork of gpui-component)
  blueprint_compiler/  # Blueprint system
  plugin_manager/      # Plugin loading
  profiling/           # Performance profiling
  net/                 # Networking
  multiuser_server/    # Multiplayer
  helio-feature-skies/ # Sky rendering feature
  type_db/             # Type database
```

### DAG-Based Initialization

Engine startup uses a dependency graph with Kahn's algorithm for topological sort:

```rust
let mut graph = InitGraph::new();
graph.add_task(InitTask::new(
    LOGGING, "Logging", vec![],
    Box::new(|ctx| { /* init logging */ })
))?;
graph.add_task(InitTask::new(
    DISCORD, "Discord Rich Presence", vec![SET_GLOBAL],
    Box::new(|ctx| { /* init discord */ })
))?;
graph.execute(&mut init_ctx)?;  // Runs in topological order
graph.to_dot();                  // Export as DOT for visualization
```

### Current Status

- Windows D3D11 compositor works (transitioning to WGPU for cross-platform)
- Bevy integration is early stage
- GPUI editor UI is functional (forked gpui-component)
- Plugin system, blueprint compiler, profiling all present

---

## 19. Gaps & Opportunities (What Doesn't Exist Yet)

| Category | Status | Path Forward |
|----------|--------|-------------|
| **Force-directed graph layout** | No GPUI integration | Wire [`fdg`](https://github.com/grantshandy/fdg) simulation to `canvas()` |
| **Large point cloud rendering** | No optimized renderer | Use `paint_quad()` for small sets; need instanced rendering for millions |
| **Node/edge graph editor** | Nothing exists | Build with `canvas()` + drag events + `petgraph` |
| **Physics simulation** | No integration | Wire [Rapier](https://rapier.rs/) to render loop via `canvas()` |
| **GPU-accelerated graph layout** | Not in GPUI | [`vibe-graph-layout-gpu`](https://docs.rs/vibe-graph-layout-gpu) uses WGPU separately |
| **Native 3D rendering** | Not possible in GPUI | GPUI is fundamentally 2D; use Pulsar-Native's Bevy compositor approach |
| **Particle systems** | Nothing exists | Straightforward to build with `canvas()` + `paint_quad()` at 60fps |
| **Custom shaders** | Not exposed | GPUI has fixed shaders (Metal/WGPU); use `Surface` element for external GPU output |

### Bridging Strategy

For graphs, physics, and point clouds, the realistic approach is:

1. **Simulation layer**: Use existing Rust crates (`fdg`, `petgraph`, `rapier`, `nalgebra`)
2. **Render layer**: Use `canvas()` with `window.paint_quad()` and `window.paint_path()`
3. **Interaction layer**: Mouse/keyboard events on the canvas element
4. **60fps loop**: `cx.defer_in(window, |this, _, cx| cx.notify())` for continuous re-render
5. **For true 3D**: Follow Pulsar-Native's shared-texture compositor pattern with Bevy/WGPU

### Relevant Rust Crates for Integration

| Crate | Purpose | Integration |
|-------|---------|-------------|
| [`fdg`](https://github.com/grantshandy/fdg) | Force-directed graph simulation (ForceAtlas2, Kamada-Kawai) | Tick simulation, render nodes via `paint_quad()` |
| [`petgraph`](https://crates.io/crates/petgraph) | Graph data structures (DAG, BFS, DFS, toposort) | Data model for graph visualization |
| [`rapier2d`](https://rapier.rs/) / `rapier3d` | Physics engine (rigid bodies, collisions, joints) | Step physics, render via `canvas()` |
| [`nalgebra`](https://nalgebra.org/) | Linear algebra (vectors, matrices, transforms) | Math for custom rendering |
| [`lyon`](https://github.com/nical/lyon) | 2D path tessellation (already used by GPUI internally) | Complex shape rendering |
| [`plotters`](https://github.com/plotters-rs/plotters) | 2D/3D charting | Via `plotters-gpui` backend |
| [`vibe-graph-layout-gpu`](https://docs.rs/vibe-graph-layout-gpu) | GPU-accelerated Barnes-Hut graph layout | External WGPU compute, render result in GPUI |

---

## Extern Source Repos

All referenced source code has been cloned into `extern/` (gitignored):

```
extern/
  gpui-component/          # Longbridge 60+ component library
  gpui-app/                # gpui-component gallery/example app
  scopeclient-components/  # Additional UI components
  awesome-gpui/            # Official Zed project list
  awesome-gpui-community/  # Community project list
  loungy/                  # App launcher
  nohrs/                   # File explorer
  helix-gpui/              # Helix editor frontend
  postman-gpui/            # HTTP client
  setu/                    # API testing tool
  zedis/                   # Redis GUI
  gpui-hooks/              # React-style hooks
  gpui-router/             # Declarative routing
  gpui-nav/                # Screen navigation
  gpui-form/               # Form generation
  gpui-storybook/          # Component gallery framework
```
