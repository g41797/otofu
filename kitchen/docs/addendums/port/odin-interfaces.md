# Emulating "Interfaces" in Odin

Odin intentionally lacks native interfaces/traits to maintain explicit control over polymorphism and data layout. Instead, developers use procedure groups, parametric polymorphism, vtables, and reflection to achieve similar patterns. This document covers all major variants with their trade-offs, usage guidelines, and library organization patterns.

## Goals & Philosophy

**Purpose**: Express "works with any type supporting operations X, Y, Z" while preserving:
- Predictable static dispatch where possible
- Explicit data layouts 
- Zero-cost abstractions when feasible
- Extensibility without central registration

**Odin Constraints**:
- No implicit method dispatch
- No trait bounds (but `where` clauses approximate this)
- Polymorphism via procedure overloading, generics, or explicit dispatch tables

## Variant 1: Procedure Groups (Static Overload Resolution)

**When**: Fixed set of known types, simple call sites, zero runtime cost

**Core Idea**: Name your "interface" as a procedure group. Overloads define implementations.

```odin
package drawable

Point :: struct { x, y: f32 }
Sprite :: struct { pos: Point, tex_id: u32 }
Text :: struct { pos: Point, content: string }

// INTERFACE = procedure group name
draw :: proc {draw_point, draw_sprite, draw_text}

// IMPLEMENTATIONS
draw_point :: proc(p: ^Point) { 
    fmt.printf("Point(%.1f, %.1f)\n", p.x, p.y) 
}
draw_sprite :: proc(s: ^Sprite) { 
    fmt.printf("Sprite@%.1f,%.1f tex=%d\n", s.pos.x, s.pos.y, s.tex_id) 
}
draw_text :: proc(t: ^Text) { 
    fmt.printf("Text@%.1f,%.1f: %s\n", t.pos.x, t.pos.y, t.content) 
}

// Usage - compiler picks correct overload
main :: proc() {
    p: Point = {1, 2}
    s: Sprite = {{3, 4}, 42}
    t: Text = {{5, 6}, "Hello"}
    
    draw(&p)  // → draw_point
    draw(&s)  // → draw_sprite  
    draw(&t)  // → draw_text
}
```

**Library Layout**:
```
drawable/
├── drawable.odin      # exports `draw` group + all overloads
├── impl_point.odin    # draw_point
├── impl_sprite.odin   # draw_sprite  
└── impl_text.odin     # draw_text
```

**Pros**: Zero cost, simple, debuggable  
**Cons**: Not extensible, monomorphic call sites

## Variant 2: Parametric Polymorphism + `where` Constraints (Static Duck Typing)

**When**: Generic algorithms over "any drawable type", compile-time safety

**Core Idea**: Use `where` clauses to require procedures exist for type `T`.

```odin
package renderer

// Generic render loop - works for ANY T with draw(^T)
render_batch :: proc($T: typeid, items: []T)
where draw: proc(^T) {
    for item, i in items {
        fmt.printf("Rendering %d: ", i)
        draw(&item)
    }
}

// Concrete implementations (from Variant 1)
draw_point :: proc(p: ^Point) { ... }
draw_sprite :: proc(s: ^Sprite) { ... }
draw_text :: proc(t: ^Text) { ... }

// Usage - monomorphized per type
main :: proc() {
    points: Point = {{1,2}, {3,4}, {5,6}} [github](https://github.com/odin-lang/Odin/issues/372)
    sprites: Sprite = { ... } [forum.odin-lang](https://forum.odin-lang.org/t/exploring-parametric-polymorphism/403/2)
    
    render_batch(points)   // instantiates with T=Point
    render_batch(sprites)  // instantiates with T=Sprite
}
```

**Library Layout**:
```
renderer/
├── renderer.odin       # generic render_batch(T)
├── drawable.odin       # procedure group + overloads (Variant 1)
└── impl/               # concrete draw_* procs
```

**Pros**: Zero cost, generic, extensible, compile-time checked  
**Cons**: Complex error messages, requires naming discipline

## Variant 3: Explicit Vtables (Manual Dynamic Dispatch)

**When**: Heterogeneous collections, plugins, runtime type selection

**Core Idea**: Struct containing procedure pointers + data pointer.

```odin
package shapes

Vec2 :: struct { x, y: f32 }

// INTERFACE DEFINITION
Shape_VTable :: struct #raw_union {
    destroy: proc(data: rawptr),
    area:    proc(data: rawptr) -> f32,
    draw:    proc(data: rawptr),
}

Shape :: struct {
    vtable: ^Shape_VTable,
    data:   rawptr,
}

// CONCRETE TYPES + VTABLES
Circle :: struct { center: Vec2, radius: f32 }
circle_vtable := Shape_VTable {
    destroy = proc(data: rawptr) { free((^Circle)(data)) },
    area    = proc(data: rawptr) -> f32 { 
        c := (^Circle)(data); return PI * c.radius * c.radius 
    },
    draw    = proc(data: rawptr) { 
        c := (^Circle)(data); fmt.println("Circle r=", c.radius) 
    },
}

Rect :: struct { min, max: Vec2 }
rect_vtable := Shape_VTable{ ... }

// CONSTRUCTORS
make_circle :: proc(c: ^Circle) -> Shape { 
    return Shape{&circle_vtable, rawptr(c)} 
}
make_rect :: proc(r: ^Rect) -> Shape { 
    return Shape{&rect_vtable, rawptr(r)} 
}

// GENERIC ALGORITHMS
total_area :: proc(shapes: []Shape) -> f32 {
    result: f32
    for shape in shapes {
        result += shape.vtable.area(shape.data)
    }
    return result
}
```

**Library Layout**:
```
shapes/
├── interface.odin      # Shape, Shape_VTable, total_area()
├── circle.odin         # Circle + circle_vtable + make_circle()
├── rect.odin           # Rect + rect_vtable + make_rect()
└── utils.odin          # common algorithms
```

**Pros**: Runtime polymorphism, stable ABI, FFI friendly  
**Cons**: Boilerplate, manual memory management, indirection cost

## Variant 4: Type Erasure via `any` + Reflection

**When**: Debug tools, inspectors, serialization, dynamic dispatch

**Core Idea**: Store `any` (value+typeid) and dispatch via typeid cases.

```odin
package inspector

import "core:reflect"

Value :: struct { any: any }

inspect :: proc(v: Value) {
    id := v.any.id
    
    switch reflect.union_variant_index(id) {
    case type_info_of(Point).id: 
        p := (^Point)(v.any.data); fmt.println("Point:", p.x, p.y)
    case type_info_of(Sprite).id:
        s := (^Sprite)(v.any.data); fmt.println("Sprite tex:", s.tex_id)
    case: fmt.println("Unknown:", reflect.typeid_to_string(id))
    }
}

// Usage
main :: proc() {
    values := []Value{
        {any({1,2})},
        {any(Sprite{{3,4}, 42})},
    }
    for v in values { inspect(v) }
}
```

**Pros**: Ultimate flexibility, single container type  
**Cons**: Runtime cost, type safety lost

## Decision Matrix

| Requirement ↓ / Variant → | Proc Groups | Parapoly+where | Vtables | any+reflect |
|---------------------------|-------------|----------------|---------|-------------|
| **Zero-cost** | ✅ | ✅ | ❌ | ❌ |
| **Generic algos** | ❌ | ✅ | ✅ | ✅ |
| **Heterogeneous** | ❌ | Limited | ✅ | ✅ |
| **Compile safety** | ✅ | ✅ | Partial | ❌ |
| **Extensible** | Limited | ✅ | ✅ | ✅ |
| **Boilerplate** | Low | Medium | High | Low |

## Library Recommendations

**For shared libraries**:
1. **Primary**: Parapoly + `where` (Variant 2) - best balance
2. **Simple wrapper**: Procedure groups (Variant 1) over parapoly 
3. **Runtime needs**: Vtables (Variant 3) with clear ownership rules
4. **`any`/reflect**: Debug/serialization only

**Naming convention**:
```
your_lib/
├── $OP.odin           # proc group $OP + overloads (Variant 1)
├── generic_$OP.odin   # parapoly generic_$OP(T) where $OP(^T) (Variant 2)
└── $OP_vtable.odin    # VTable + constructors (Variant 3)
```

This pattern scales from simple apps to complex frameworks while staying true to Odin's data-oriented philosophy.
```