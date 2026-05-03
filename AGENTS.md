# opencode-figma Plugin

Control Figma Desktop directly from OpenCode. Full read/write access via Yolo Mode (CDP). No API key required.

## Quick Reference

| User says | Tool to use |
|-----------|-------------|
| "connect to figma" | `figma_connect` |
| "add shadcn colors" | `figma_tokens` with `preset="shadcn"` |
| "add tailwind colors" | `figma_tokens` with `preset="tailwind"` |
| "show colors on canvas" | `figma_tokens` with `action="visualize"` |
| **"create a button / card / input / badge"** (reusable UI) | **`figma_component` with `action="create-set"` (NOT figma_render or figma_create)** |
| **"use the Button inside a Card"** | **`<Instance component="Button"/>` inside `figma_render` JSX** |
| "create a one-off layout / hero / page" | `figma_render` with JSX |
| "create a rectangle/frame" | `figma_create` with `type="frame"` |
| "convert to component" | `figma_create` with `action="to-component"` |
| "add a text style" | `figma_textstyle` with `action="create"` |
| "add a drop shadow" | `figma_effect` with `action="create"` |
| "make this fill the parent" | `figma_layout` with `horizontal="FILL"` |
| "list variables" | `figma_variable` with `action="list"` |
| "find nodes named X" | `figma_find` with `pattern="X"` |
| "what's on canvas" | `figma_analyze` with `type="canvas"` |
| "export as PNG/SVG" | `figma_export` with `format="png"` |

**Full command reference:** See README.md

---

## Design Tokens

### Add shadcn Colors
```javascript
figma_tokens({ preset: "shadcn" })
// Creates 244 primitives + 32 semantic variables (Light/Dark modes)
```

### Add Tailwind Colors
```javascript
figma_tokens({ preset: "tailwind" })
// Creates 242 primitive colors only
```

### Create Design System
```javascript
figma_tokens({ preset: "ds" })
// IDS Base colors
```

### Delete All Variables
```javascript
figma_variable({ action: "delete-all" })
figma_variable({ action: "delete-all", collection: "primitives" })
```

---

## Fast Variable Binding (var: syntax)

Use `var:name` syntax to bind variables directly at creation:

### Create with var:
```javascript
figma_create({
  type: "rect",
  name: "Card",
  fill: "var:card",
  stroke: "var:border"
})

figma_create({
  type: "text",
  content: "Hello",
  fill: "var:foreground"
})

figma_create({
  type: "icon",
  name: "star",
  icon: "lucide:star",
  fill: "var:primary"
})
```

### JSX Render with var:
```javascript
figma_render({
  jsx: `<Frame bg="var:card" stroke="var:border" rounded={12} p={24}>
    <Text color="var:foreground" size={18}>Title</Text>
  </Frame>`
})
```

### FLOAT variables (spacing, radius, opacity, stroke weight)

- **`var:` on colors** resolves **COLOR** tokens (`fill`, `stroke`, `bg`, `color` in JSX).
- **`var:` on numbers** resolves **FLOAT** tokens: use **`figma_set`** (`gap`, `paddingLeft`, `radius`, `strokeWidth`, `opacity`, `fontSize`, `width`, `height`, …) or JSX props **`gap`**, **`p`** / **`px`** / **`py`**, **`rounded`**, **`strokeWidth`**, **`opacity`**, **`size`** (text). First `/` separates collection from variable name (same as colors).

```javascript
figma_set({ property: "gap", value: "var:semantic/spacing/md", nodeId: "123:456" })
figma_set({ property: "paddingLeft", value: "var:spacing/4" })
```

---

## Connection Modes

### Yolo Mode (Default - Recommended)
Patches Figma once, then connects directly via CDP. If Figma wasn’t running, it launches and waits for a **canvas tab** — open any file from Recents if you only see home (~90s window).
```javascript
figma_connect({ mode: "yolo" })
```

### Safe Mode (Plugin-based)
Uses Figma plugin, no patching. Start plugin each session.
```javascript
figma_connect({ mode: "safe" })
```

---

## Creating Components (THE RIGHT WAY)

**CRITICAL**: When the user asks to "create a Button", "design Cards", "build a
component library", or anything that implies a *reusable* UI element — use
`figma_component`, **NOT** `figma_render` or `figma_create`. Never produce
"Button 1", "Button 2", "Button 3" as separate frames. Make ONE Button
component-set with State variants, then use **instances**.

### Pattern 1 — A component with multiple states (Button, Input, Badge…)

```javascript
figma_component({
  action: "create-set",
  name: "Button",
  variants: [
    {
      properties: { State: "Default" },
      jsx: '<Frame name="Button" flex="row" gap={8} px={16} py={10} bg="var:primary" rounded={8} justify="center" items="center"><Text name="Label" size={14} weight="medium" color="var:primary-foreground">Click me</Text></Frame>'
    },
    {
      properties: { State: "Hover" },
      jsx: '<Frame name="Button" flex="row" gap={8} px={16} py={10} bg="var:primary" rounded={8} justify="center" items="center" opacity={0.9}><Text name="Label" size={14} weight="medium" color="var:primary-foreground">Click me</Text></Frame>'
    },
    {
      properties: { State: "Disabled" },
      jsx: '<Frame name="Button" flex="row" gap={8} px={16} py={10} bg="var:muted" rounded={8} justify="center" items="center" opacity={0.5}><Text name="Label" size={14} weight="medium" color="var:muted-foreground">Click me</Text></Frame>'
    }
  ]
})
```

The variant property key/value pairs (`State=Default`, `State=Hover`) become
real Figma variant properties on the component set, so the user can switch
states from the right-rail.

### Pattern 2 — A simple, single-variant component (Card, Header, Footer…)

```javascript
figma_component({
  action: "create",
  name: "Card",
  jsx: `<Frame flex="col" gap={12} p={20} bg="var:card" rounded={12} stroke="var:border" w={320}>
    <Text name="Title" size={18} weight="semibold" color="var:card-foreground">Card title</Text>
    <Text name="Description" size={14} color="var:muted-foreground">Description here.</Text>
    <Instance name="Action" component="Button" variant="State=Default" Label="View details"/>
  </Frame>`
})
```

> Inside a component, name the text nodes (`name="Title"`, `name="Description"`)
> so they can be overridden per-instance later.

### Pattern 3 — Use the components inside layouts via `<Instance>`

```javascript
figma_render({
  jsx: `<VStack gap={32} p={48} bg="var:background">
    <Text size={32} weight="bold" color="var:foreground">Buttons</Text>
    <HStack gap={16}>
      <Instance component="Button" variant="State=Default" Label="Primary"/>
      <Instance component="Button" variant="State=Hover" Label="Hover"/>
      <Instance component="Button" variant="State=Disabled" Label="Disabled"/>
    </HStack>

    <Text size={32} weight="bold" color="var:foreground">Cards</Text>
    <HStack gap={24}>
      <Instance component="Card" Title="Welcome" Description="Get started in seconds." Action="Get started"/>
      <Instance component="Card" Title="Pricing" Description="Pay only for what you use." Action="See pricing"/>
    </HStack>
  </VStack>`
})
```

**How `<Instance>` overrides work:**
- `component="Card"` — looks up the component by name (or id, or `Set/Variant`).
- `variant="State=Hover, Size=Lg"` — sets variant properties on the instance.
- Any other prop (`Title="Welcome"`, `Action="Get started"`) is matched against
  a child node by name. If the child is a TEXT node, its characters are set.
  If the child is itself an INSTANCE (nested component), the override is set
  on its inner `Label` / first text node — so `Action="Get started"` on a
  Card propagates into the Card's Button instance automatically.

---

## Complex Components

For complex multi-element components, use `figma_eval` with native Figma API:

```javascript
figma_eval({
  code: `
    const colors = {
      bg: { r: 0.09, g: 0.09, b: 0.11 },
      card: { r: 0.11, g: 0.11, b: 0.13 },
      primary: { r: 0.23, g: 0.51, b: 0.97 }
    };
    // Create components...
  `
})
```

---

## JSX Syntax (`figma_render` and `figma_component`)

### Tags

```jsx
<Frame ...>...</Frame>      // explicit frame
<VStack ...>...</VStack>    // frame with flex="col" by default
<HStack ...>...</HStack>    // frame with flex="row" by default
<Stack ...>...</Stack>      // alias for VStack
<Row .../> <Column .../>    // alias for HStack / VStack

<Text ...>Hello</Text>
<H1 .../> <H2 .../> <H3 .../> <H4 .../> <Label .../>  // text with default sizes

<Rect .../> <Ellipse .../>  // shapes

<Instance component="Button" variant="State=Hover" Label="Click me"/>  // reuse a component
```

### Props

```jsx
// Layout
flex="row" | "col"      // (or use HStack / VStack)
gap={16}
p={24}
px={16} py={8}
pl/pr/pt/pb={...}

// Alignment
justify="center"        // main axis: start | center | end | between
items="center"          // cross axis: start | center | end

// Size
w={320} h={200}         // fixed
w="fill" h="fill"       // stretch (inside auto-layout)
w="hug" h="hug"         // shrink to content

// Appearance
bg="#fff"               // background fill
bg="var:card"           // variable
stroke="#000" strokeWidth={2}
rounded={16}
opacity={0.9}

// Text
<Text size={18} weight="bold" color="var:foreground" font="Inter">Hello</Text>
// weight: regular | medium | semibold | bold | light
```

### Reuse via `<Instance>`

```jsx
<Instance component="Button"/>
<Instance component="Button" variant="State=Hover"/>
<Instance component="Card" Title="Welcome" Description="..." Action="Open"/>

// Nested overrides reach inner instances automatically:
//   Card has <Instance name="Action" component="Button" .../>
//   Action="Open" → updates the inner Button's Label text node
```

---

## Key Rules

1. **Reusable UI = `figma_component`.** Buttons, Cards, Inputs, Badges, Avatars, Tags — anything used more than once goes through `figma_component`. Never produce duplicate frames named "Button 1", "Button 2", "Card 1", etc.
2. **Layouts that consume components use `<Instance>`** in `figma_render` — not raw frames that look like the component.
3. **One-off compositions** (a hero, a screen, a marketing section) → `figma_render`.
4. **A single primitive** (just a frame, just a circle) → `figma_create`.
5. **Name your text nodes** inside components (`name="Title"`, `name="Label"`, `name="Action"`) so consumers can override them via `<Instance Title="..."/>`.
6. **Check connection first** — call `figma_connect` if any tool returns "Cannot connect to Figma".
7. **Don't use `figma_eval` to create nodes** — it has no smart positioning, things stack at (0,0).

---

## Onboarding ("Initiate Project")

**Never show terminal commands to users.** Run silently, give friendly feedback.

1. Run connection check silently
2. Ask connection mode (Yolo or Safe)
3. Run `figma_connect` (or with `mode="safe"`)
4. When connected, say: "Connected! What would you like to create?"

If permission error (macOS): System Settings → Privacy → Full Disk Access → Add Terminal

---

## Variable Visualization

"Show colors on canvas" / "display variables":

```javascript
figma_tokens({ action: "visualize" })              // All collections
figma_tokens({ action: "visualize", filter: "primitives" }) // Filter
```

Creates shadcn-style color swatches bound to variables.

---

## Website Recreation

```javascript
// Not yet implemented - coming soon
// figma_recreate({ url: "https://example.com" })
```

---

## Speed Daemon

Connection auto-starts daemon for 10x faster commands.

```javascript
figma_daemon({ action: "status" })
figma_daemon({ action: "restart" })
```

---

## Troubleshooting

### Not Connected
```
figma_connect({ mode: "yolo" })
// or
figma_connect({ mode: "safe" })
```

### macOS Permission Error
Grant Full Disk Access to Terminal:
1. System Settings → Privacy & Security → Full Disk Access
2. Click + and add Terminal
3. Quit Terminal completely (Cmd+Q) and reopen

### Plugin Not Detected (Safe Mode)
In Figma: **Plugins → Development → FigCli**
Terminal should show: `Plugin connected!`
