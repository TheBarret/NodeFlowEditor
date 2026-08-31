# Flowed - A Node Flow Engine Concept for PyGame

**Target Environment:** Python 3.10+
**Renderer:** Pygame (latest)
**Architecture:** Headless Core + Visual Overlay

---

## 1. Core Data Model (Pygame-Free Domain)

The foundation is a pure Python directed graph. It contains zero references to Pygame,  
screen coordinates, or rendering routines, ensuring full unit testability and headless execution.  

### Sockets & Ports

* **Identity:** Each port possesses a globally unique identifier (e.g., `UUID` or `node_id:port_name`).
* **Direction:** Defined explicitly as `PortDirection.INPUT` or `PortDirection.OUTPUT`.
* **Data Types:** Associated with a type signature (e.g., `Number`, `String`, `ImageBuffer`, `Any`).
* **Connectivity:** Sockets track maximum allowed connections (typically 1 for inputs, multi-connection for outputs).

### Node Structure

**Identity:**  
Contains a unique `node_id` and a functional type identifier.  

**Port Mapping:**  
Maintains dictionaries of input and output ports indexed by `port_id`.  

**State & Parameter Separation:**  
**`params` (Persistent):** User-configured parameters (e.g., slider values, text input, operation modes).  
                             Serialized to file and tracked in the undo history.  
**`runtime_state` (Volatile):** Execution-only transient storage (e.g., cached evaluation outputs, internal buffers).  
                                  Excluded from graph saves and cleared on graph load.  

**Recomputation Optimization:**  
Employs an `is_dirty` boolean flag. Nodes recompute only when `is_dirty` is `True` or when upstream input parameters change.  

### Connections (Edges)

Explicit directed pairs referencing unique port identifiers:   
  ` (source_output_port_id) -> (target_input_port_id) `  

---

## 2. Graph Operations & Mutation Layer

All structural changes pass through a single, centralized `Graph` API surface.  
The visual interface and external scripts never directly manipulate node lists or socket tuples.  

* **API Methods:**
* `add_node(node_type, position)` / `remove_node(node_id)`
* `connect(source_port_id, target_port_id)` / `disconnect(edge_id)`
* `set_param(node_id, param_key, value)`


* **Centralized Validation (`validate_connection`):**
Single source of truth evaluating whether a prospective edge is legal before creation.  

Checks for:
* Port direction validity (Output $\rightarrow$ Input only).
* Data type compatibility or allowed implicit conversion rules.
* Cycle formation constraints.


* **Command & History Stack:**
* All mutations are encapsulated as Command objects (`AddNodeCommand`, `ConnectCommand`).
* Supports native **Undo / Redo** operations and graph state serialization/deserialization without UI dependencies.

---

## 3. Execution Controller & Cycle Management

The evaluation engine determines execution order based on graph structure and node execution contracts.  

### Execution Contracts

**Standard Nodes (`evaluate()`):**  
Participates in pull-based execution. Recursively requests data from upstream input ports when marked dirty.   

**Delay / Buffer Nodes (`evaluate_cached()`):**   
Acts as a boundary node (representing unit delay $z^{-1}$).  
Returns cached state from the previous evaluation tick **without** making recursive upstream calls.  

### Engine Evaluation Workflow

1. **Startup Cycle Analysis:** Run Tarjan’s or Johnson’s algorithm to detect cycles across the graph topology.  
2. **DAG Evaluation (No Cycles):**  
   Perform a Topological Sort. Nodes execute strictly in topological order using a pull-based demand engine.    
3. **Cyclic Graph Evaluation (Cycles Present):**
* Inspect all detected cycles. Every cycle **must** contain at least one node implementing the `evaluate_cached()` delay contract.  
* If a cycle lacks a delay node, the engine halts execution and raises a clear `InvalidCycleError`.  
* Valid cyclic graphs are evaluated via tick-based step execution,  
  using delay nodes as memory buffers to prevent infinite recursion stacks.

---

## 4. Canvas Management & Rendering Engine

Handles the mapping from world-space graph structures to Pygame pixel rendering.

### Camera Transformation Model

Centralized in a unified `Camera` object storing `position` (`Vector2`) and `scale` (`float`).  

$$\text{Screen Point} = (\text{World Point} - \text{Camera.position}) \times \text{Camera.scale}$$  

$$\text{World Point} = \left(\frac{\text{Screen Point}}{\text{Camera.scale}}\right) + \text{Camera.position}$$  

### Performance & Viewport Optimization

**Viewport Culling:**  
Prior to rendering, compute screen-space bounding boxes (AABB) for nodes.  
Render only nodes and edges intersecting the active camera viewport rect.  

**Connection Geometry Caching:**  
Compute Bézier curve paths (using horizontal control point offsets relative to port positions)  
only when connected node positions change or screen zoom shifts. 

**Layer Z-Ordering:**  
Maintain strict render ordering:  
* Background Grid $\rightarrow$  
* Inactive Edges $\rightarrow$  
* Unselected Nodes $\rightarrow$  
* Selected Nodes & Connected Edges $\rightarrow$ Active Dragging Wire.  

---

## 5. Interaction State Machine

Prevents ambiguous UI behaviors by defining clear, non-overlapping input states.  

```
                  ┌──────────────────────────┐
                  │           IDLE           │
                  │ (Hovering, Context Menu) │
                  └────────────┬─────────────┘
                               │
       ┌───────────────────────┼───────────────────────┐
       │ Left-Click Drag       │ Left-Click Port       │ Middle-Click Drag
       ▼                       ▼                       ▼
┌──────────────┐       ┌──────────────┐        ┌──────────────┐
│ DRAGGING_NODE│       │ DRAGGING_WIRE│        │  PAN_ZOOM    │
└──────────────┘       └──────────────┘        └──────────────┘
       │                       │                       │
       └───────────────────────┼───────────────────────┘
                               │ Mouse Release
                               ▼
                  ┌──────────────────────────┐
                  │           IDLE           │
                  └──────────────────────────┘

```

**IDLE:**  
Listens for mouse movements, hover target queries, right-click context menus, and single/box selection triggers.  

**DRAGGING_NODE:**  
Translates selected node world positions by the mouse movement delta scaled by camera zoom.  

**DRAGGING_WIRE:**  
Spawns a temporary Bezier curve from the origin port to the mouse cursor.  
Intersects with `validate_connection()` to provide real-time visual connection feedback  
(e.g., green for valid target, red for invalid target).  

**PAN_ZOOM:**  
Adjusts camera offset based on mouse movement delta, or adjusts camera scale centered  
around the mouse cursor position during scroll wheel events.  

---

## 6. Architectural Layers Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    GRAPH UI ENGINE (Pygame)                     │
│  - Pygame Event Loop & User Input State Machine                 │
│  - Unified Camera Transform (Pan/Zoom Matrix)                   │
│  - Viewport Culling & Bezier Curve Wire Renderer                │
└────────────────────────────────┬────────────────────────────────┘
                                 │ Dispatches User Actions
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                    GRAPH OPERATIONS LAYER                       │
│  - Mutation API (add_node, remove_node, connect, disconnect)    │
│  - Connection Validation Surface (validate_connection)          │
│  - Command Stack (Undo / Redo Buffer & JSON Serialization)      │
└────────────────────────────────┬────────────────────────────────┘
                                 │ Operates On
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                   CORE DATA MODEL (Pygame-Free)                 │
│  - Node [id, params (persistent), runtime_state (volatile)]     │
│  - Port [unique_port_id, type, direction]                       │
│  - Connection [source_port_id -> target_port_id]                │
└────────────────────────────────┬────────────────────────────────┘
                                 │ Reads Structure & Values
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                       EXECUTION CONTROLLER                      │
│  - Topology Cycle Validation (Tarjan's Algorithm)              │
│  - DAG Execution: Topological Sort + Recursive Pull             │
│  - Cyclic Execution: Tick-based Step Engine via Delay Sockets   │
└─────────────────────────────────────────────────────────────────┘

```
