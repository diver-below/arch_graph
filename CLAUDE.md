# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview
# Technical Task: System Architecture Graph Viewer

## 1. Overview

Build a comfortable, zoomable, and pannable user interface for visualizing a system architecture.  
The UI renders a directed/undirected graph consisting of nodes (modules, databases, transports, proxies) and connections between them.  
Nodes and their metadata are provided by a backend service, but for initial development and demonstration a static demo file will be used.

The final deliverable is a self-contained frontend application that runs first on localhost and later on a virtual machine as a **systemd user unit**.

---

## 2. Functional Requirements

### 2.1 Graph Rendering
- Display all nodes and connections from the data source.
- Nodes are positioned inside a large grid; each node occupies one cell.
- Connections are drawn as flexible lines (e.g., curved or orthogonal) between the connected nodes.
- The whole graph is placed inside a scrollable/zoomable viewport.

### 2.2 Zoom and Pan
- Support mouse wheel / pinch‑to‑zoom for zooming in/out.
- Support click‑and‑drag (or touch‑drag) to pan the viewport.
- Zoom and pan must be smooth and intuitive (e.g., scaling toward the cursor).

### 2.3 Node and Connection Selection
- **Click / tap** on a node or a connection line to select it.
- When a **node** is selected:
  - The graph automatically **pans and zooms** to bring that node and **all its direct connections** into view.
  - All other nodes and connections become **greyed out** (low opacity / desaturated) while the selected node and its directly connected nodes/edges remain fully colored/highlighted.
- When a **connection** is selected:
  - Both source and target nodes are highlighted; the rest greyed out.
  - Graph pans/zooms to fit those two nodes and the selected edge.

### 2.4 Info Pop‑up
- A **pop‑up panel** appears on the **right side** of the screen.
- Content of the pop‑up:
  - **Name / ID** of the selected element (node or connection).
  - **Description** (plain text or markdown) of the element.
  - For a node: a **copyable list** of all its incoming/outgoing connections (source, target, type/label).
  - For a connection: source node name, target node name, connection type/protocol.
- The pop‑up must support **copy to clipboard** for the connection list (e.g., a “Copy” button next to each item or for the whole list).

### 2.5 Zones and Groups
- The architecture is divided into **network zones** (e.g., "our system zone", "external zone"). Each zone contains only the nodes that belong to it.
- Inside a system zone there are **namespace zones** that group pods/modules belonging to the same namespace.
- Certain node types (e.g., all "get" modules) must be visually **grouped together** inside their zone/namespace.
- The user must be able to visually distinguish zones (e.g., by colored backgrounds or borders).

### 2.6 Grid Layout
- The whole map behaves like a **big grid**.
- Each node is placed in a distinct cell.
- Zones and groups are aligned to the grid (rectangular areas).
- Coordinates can be provided in the data or auto‑calculated; the grid ensures a structured, non‑overlapping layout.

---

## 3. UI Layout Details

| Element          | Description                                                                 |
|------------------|-----------------------------------------------------------------------------|
| Main viewport    | Full browser window, contains the zoomable/pannable graph canvas.           |
| Pop‑up panel     | Fixed on the right side, width ~300‑400px, toggles visible when a selection exists. Shows detailed info. |
| Graph canvas     | Renders all nodes and edges, handles click/tap events, zoom, pan.           |
| Zone backgrounds | Semi‑transparent rectangles behind nodes, labelled with zone/namespace name.|
| Node appearances | Different shapes/colours by node type (see Domain Model).                   |
| Edge lines       | Flexible curves or step‑lines, with optional arrowheads, coloured by type.  |

---

## 4. Domain Model (Architecture Entities)

The following entities appear in the graph. They must be represented by the data model and correctly rendered.

### 4.1 Hierarchical Structure (top‑down)
- **Network Zones** (e.g., "our zone", "external zone")
  - **Systems** (e.g., system‑A, system‑B)
    - **Namespaces** (e.g., namespace‑1, namespace‑2)
      - **Pods / Modules** (the actual executable units)
    - **Databases** (associated with a system)
    - **Transports** (e.g., Kafka, Zookeeper – system‑level)
    - **Proxies** (e.g., Gateway, Load Balancer – at system boundaries)

### 4.2 Module (Pod) Types
Each module must be rendered with a distinct colour/shape:

| Type        | Description                                                     |
|-------------|-----------------------------------------------------------------|
| get         | Receives external requests, calls syncer, rsc‑core, ext systems |
| syncer      | Receives from get, calls external master‑systems, DB, Kafka     |
| put         | Connects to external Kafka, calls syncer and bps                |
| bps         | Works with DB, Kafka, external DB‑orchestrator, Zookeeper       |
| rsc‑core    | Three subtypes: rsc‑write, rsc‑read, rsc‑monitor (details below)|
| ui          | Composed of ui‑proxy, ui‑front, ui‑back                         |
| db          | PostgreSQL (slow) or Ignite (fast)                              |
| transport   | Our Kafka, Zookeeper                                            |
| proxy       | Gateway or Load Balancer (at zone/system borders)               |

**RSC‑Core Subtypes:**
- `rsc‑write` – reads from our Kafka, writes to DB.
- `rsc‑read`  – reads from DB.
- `rsc‑monitor` – reads/writes Zookeeper, requests rsc‑write, receives from get modules.

**UI Subtypes:**
- `ui‑proxy` – connects to external auth, redirects to ui‑front.
- `ui‑front` – static forms, gets requests from ui‑proxy, requests ui‑back.
- `ui‑back` – receives from ui‑front, writes to our Kafka, reads/writes DBs.

### 4.3 Connections
Connections carry a `type` or `protocol` label, e.g.:
- HTTP request
- Kafka topic (read/write)
- DB query (SQL)
- Zookeeper access
- gRPC call

---

## 5. Data Model Specification

The frontend loads its data from a **separate file**, not embedded in this document.  
Two files will be provided:

1. **Data schema / model description** (e.g., `data-model.md` or `schema.json`) – defines the structure of nodes, connections, zones, groups, coordinates.
2. **Demo data file** (e.g., `demo-data.json`) – contains actual demo nodes and connections.

The agent must **parse** these files at runtime (or build‑time) and render the graph accordingly.  
The data must include at least:

- **Zones**: id, name, bounding box (grid coordinates), parent (optional).
- **Systems**: id, name, zoneId.
- **Namespaces**: id, name, systemId, zoneId.
- **Nodes**: id, name, type (one of the enumerated types), subtype (where applicable), parent (namespace, system, or zone), grid position (x, y cell), description, additional metadata.
- **Connections**: id, sourceNodeId, targetNodeId, label/type, direction (optional), description.
- **Groups**: optional grouping of nodes (e.g., all "get" modules) for visual clustering.

The file format will be JSON. The agent is free to transform it internally.

> **Note:** The actual data model content is **not** included in this .md – it will be provided in the aforementioned separate files.

---

## 6. Technical Stack & Constraints

- **Runtime**: Browser (modern Chrome, Firefox, Edge).
- **No backend required** for the demo – the app is a static site that reads the demo data file (e.g., via `fetch` or as an imported module).
- **No heavy framework is mandatory**, but using a lightweight library for graph rendering is encouraged (e.g., D3.js, Cytoscape.js, vis‑network, or custom SVG/Canvas).
- The code must be **self‑contained** in a directory with an `index.html`, assets, and data files.
- Deployment target: first **localhost** (just open `index.html` or run via a simple HTTP server), later a Linux VM as a **systemd user unit** serving the static files with a lightweight HTTP server (e.g., `python3 -m http.server` or `nginx`).

---

## 7. Non‑functional Requirements

- **Performance**: The graph should handle up to ~200 nodes and ~500 edges without noticeable lag during zoom/pan.
- **Responsiveness**: The pop‑up should adapt to screen width; on narrow screens (< 768px) it may overlay the graph instead of being fixed on the right.
- **Accessibility**: Basic keyboard navigation (arrow keys to pan, +/- for zoom, Tab to select elements) is a plus.
- **Clipboard**: The connection list in the pop‑up must have a copy button, and the standard `Ctrl+C` on selected text must work.

---

## 8. Implementation Plan

### Phase 1 – Localhost Development
1. Set up project structure: `index.html`, CSS, JS, and data files.
2. Implement data loading from `demo-data.json`.
3. Build the grid‑based layout engine – place nodes in cells respecting zones/groups.
4. Render nodes (SVG or Canvas) with different colours/shapes by type.
5. Draw connections as flexible lines (e.g., cubic bezier curves or step‑lines).
6. Implement zoom (d3.zoom or custom) and pan.
7. Add click selection for nodes and edges, toggle grey‑out behaviour.
8. Implement graph auto‑centre and zoom‑to‑fit for selected node + neighbours.
9. Build the right‑side pop‑up with description and copyable connection list.
10. Test with the provided demo data.

### Phase 2 – VM Deployment as systemd User Unit
1. Place all static files (including data) in a dedicated directory, e.g., `/home/<user>/sysarch-viewer/`.
2. Create a systemd user unit that starts a simple HTTP server serving that directory on `localhost:8080`.
   ```ini
   # ~/.config/systemd/user/sysarch-viewer.service
   [Unit]
   Description=System Architecture Viewer

   [Service]
   ExecStart=/usr/bin/python3 -m http.server 8080 --directory /home/<user>/sysarch-viewer
   Restart=always

   [Install]
   WantedBy=default.target
   ```
3. Enable and start the user unit: systemctl --user enable --now sysarch-viewer.
4. Ensure the service is reachable (e.g., by reverse proxy or direct localhost access).

## 9. Deliverables
1. Source code repository with all HTML, CSS, JS, and configuration files.
2. demo-data.json and data-model.md/schema.json (already provided, but must be part of the deliverable).
3. README.md with instructions for:
- Running locally.
- Deploying as a systemd user unit.
4. Systemd unit file (or instructions to create it).

## 10. Acceptance Criteria
- All nodes and connections from the demo file are displayed.
- Zoom and pan work smoothly.
- Zones, namespaces, and groups are visually distinct.
- Clicking a node highlights it, greys out the rest, centres the view on the node and its direct neighbours.
- Right‑side pop‑up shows description and a copyable list of connections.
- Clicking a connection highlights its endpoints and itself; pop‑up shows details.
- The application runs locally by simply opening index.html or via a local HTTP server.
- The systemd user unit successfully serves the application on the VM.

## JSON Schema defining the data structure
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "System Architecture Graph Data",
  "description": "Schema for the graph data containing zones, systems, namespaces, nodes, connections, and optional groups.",
  "type": "object",
  "required": ["zones", "systems", "namespaces", "nodes", "connections"],
  "properties": {
    "zones": {
      "type": "array",
      "description": "Network zones, e.g. our zone, external zone.",
      "items": { "$ref": "#/definitions/zone" }
    },
    "systems": {
      "type": "array",
      "description": "Systems belonging to zones.",
      "items": { "$ref": "#/definitions/system" }
    },
    "namespaces": {
      "type": "array",
      "description": "Namespaces inside systems (optional grouping).",
      "items": { "$ref": "#/definitions/namespace" }
    },
    "nodes": {
      "type": "array",
      "description": "All graph nodes (modules, databases, transports, proxies).",
      "items": { "$ref": "#/definitions/node" }
    },
    "connections": {
      "type": "array",
      "description": "Directed/undirected connections between nodes.",
      "items": { "$ref": "#/definitions/connection" }
    },
    "groups": {
      "type": "array",
      "description": "Optional visual groupings of nodes (e.g. all get modules together).",
      "items": {
        "type": "object",
        "required": ["id", "name", "nodeIds"],
        "properties": {
          "id": { "type": "string" },
          "name": { "type": "string" },
          "nodeIds": {
            "type": "array",
            "items": { "type": "string" }
          }
        }
      }
    }
  },
  "definitions": {
    "zone": {
      "type": "object",
      "required": ["id", "name"],
      "properties": {
        "id": { "type": "string" },
        "name": { "type": "string" },
        "gridBounds": {
          "type": "object",
          "description": "Optional bounding box in grid coordinates (top-left and bottom-right).",
          "properties": {
            "x1": { "type": "integer", "minimum": 0 },
            "y1": { "type": "integer", "minimum": 0 },
            "x2": { "type": "integer", "minimum": 0 },
            "y2": { "type": "integer", "minimum": 0 }
          }
        },
        "color": { "type": "string", "description": "CSS background color for zone visualization." }
      }
    },
    "system": {
      "type": "object",
      "required": ["id", "name", "zoneId"],
      "properties": {
        "id": { "type": "string" },
        "name": { "type": "string" },
        "zoneId": { "type": "string" },
        "gridBounds": {
          "type": "object",
          "properties": {
            "x1": { "type": "integer" },
            "y1": { "type": "integer" },
            "x2": { "type": "integer" },
            "y2": { "type": "integer" }
          }
        }
      }
    },
    "namespace": {
      "type": "object",
      "required": ["id", "name", "systemId", "zoneId"],
      "properties": {
        "id": { "type": "string" },
        "name": { "type": "string" },
        "systemId": { "type": "string" },
        "zoneId": { "type": "string" },
        "gridBounds": {
          "type": "object",
          "properties": {
            "x1": { "type": "integer" },
            "y1": { "type": "integer" },
            "x2": { "type": "integer" },
            "y2": { "type": "integer" }
          }
        }
      }
    },
    "node": {
      "type": "object",
      "required": ["id", "name", "type", "parentId", "gridX", "gridY"],
      "properties": {
        "id": { "type": "string" },
        "name": { "type": "string" },
        "type": {
          "type": "string",
          "enum": ["get", "syncer", "put", "bps", "rsc-core", "ui", "db", "transport", "proxy"]
        },
        "subtype": {
          "type": "string",
          "description": "Subtype for rsc-core (write, read, monitor) or ui (proxy, front, back). Optional.",
          "enum": ["write", "read", "monitor", "proxy", "front", "back", "postgres", "ignite", "kafka", "zookeeper", "gateway", "load_balancer"]
        },
        "parentId": {
          "type": "string",
          "description": "ID of the parent namespace, system, or zone."
        },
        "gridX": { "type": "integer", "description": "Grid column (0-based)." },
        "gridY": { "type": "integer", "description": "Grid row (0-based)." },
        "description": { "type": "string" },
        "metadata": {
          "type": "object",
          "description": "Additional key-value pairs (e.g. version, replicas)."
        }
      }
    },
    "connection": {
      "type": "object",
      "required": ["id", "sourceNodeId", "targetNodeId", "type"],
      "properties": {
        "id": { "type": "string" },
        "sourceNodeId": { "type": "string" },
        "targetNodeId": { "type": "string" },
        "type": {
          "type": "string",
          "description": "Protocol or communication type (e.g. HTTP, Kafka, DB query)."
        },
        "direction": {
          "type": "string",
          "enum": ["directed", "bidirectional"],
          "default": "directed"
        },
        "label": { "type": "string" },
        "description": { "type": "string" }
      }
    }
  }
}
```

## a concrete example that complies with the schema
```json
{
  "zones": [
    {
      "id": "zone-our",
      "name": "Our System Zone",
      "gridBounds": { "x1": 0, "y1": 0, "x2": 8, "y2": 7 },
      "color": "#e8f5e9"
    },
    {
      "id": "zone-ext",
      "name": "External Zone",
      "gridBounds": { "x1": 9, "y1": 0, "x2": 12, "y2": 7 },
      "color": "#fce4ec"
    }
  ],
  "systems": [
    {
      "id": "sys-main",
      "name": "Main Processing System",
      "zoneId": "zone-our",
      "gridBounds": { "x1": 0, "y1": 0, "x2": 3, "y2": 7 }
    },
    {
      "id": "sys-integration",
      "name": "Integration System",
      "zoneId": "zone-our",
      "gridBounds": { "x1": 4, "y1": 0, "x2": 8, "y2": 7 }
    },
    {
      "id": "sys-ext-master",
      "name": "External Master System",
      "zoneId": "zone-ext",
      "gridBounds": { "x1": 9, "y1": 0, "x2": 12, "y2": 7 }
    }
  ],
  "namespaces": [
    {
      "id": "ns-core",
      "name": "core-ns",
      "systemId": "sys-main",
      "zoneId": "zone-our",
      "gridBounds": { "x1": 0, "y1": 1, "x2": 3, "y2": 4 }
    },
    {
      "id": "ns-integration",
      "name": "integration-ns",
      "systemId": "sys-integration",
      "zoneId": "zone-our",
      "gridBounds": { "x1": 4, "y1": 1, "x2": 8, "y2": 4 }
    },
    {
      "id": "ns-ext",
      "name": "external-ns",
      "systemId": "sys-ext-master",
      "zoneId": "zone-ext",
      "gridBounds": { "x1": 9, "y1": 1, "x2": 12, "y2": 4 }
    }
  ],
  "nodes": [
    {
      "id": "get-1",
      "name": "GET Ingestor",
      "type": "get",
      "parentId": "ns-core",
      "gridX": 0,
      "gridY": 1,
      "description": "Receives external requests, calls syncer and rsc-core."
    },
    {
      "id": "get-2",
      "name": "GET Adapter",
      "type": "get",
      "parentId": "ns-integration",
      "gridX": 4,
      "gridY": 1,
      "description": "Handles requests from external master-system."
    },
    {
      "id": "syncer-1",
      "name": "Syncer Hub",
      "type": "syncer",
      "parentId": "ns-core",
      "gridX": 1,
      "gridY": 2,
      "description": "Receives from get, calls external master-systems, DB, Kafka."
    },
    {
      "id": "syncer-2",
      "name": "Syncer Bridge",
      "type": "syncer",
      "parentId": "ns-integration",
      "gridX": 5,
      "gridY": 2,
      "description": "Connects internal get to external systems and transport."
    },
    {
      "id": "put-1",
      "name": "PUT Dispatcher",
      "type": "put",
      "parentId": "ns-core",
      "gridX": 2,
      "gridY": 1,
      "description": "Writes to external Kafka and triggers bps/syncer."
    },
    {
      "id": "bps-1",
      "name": "BPS Engine",
      "type": "bps",
      "parentId": "ns-core",
      "gridX": 2,
      "gridY": 3,
      "description": "Works with DB, Kafka, external DB-orchestrator, Zookeeper."
    },
    {
      "id": "rsc-write-1",
      "name": "RSC Writer",
      "type": "rsc-core",
      "subtype": "write",
      "parentId": "ns-core",
      "gridX": 0,
      "gridY": 3,
      "description": "Reads from our Kafka, writes to DB."
    },
    {
      "id": "rsc-read-1",
      "name": "RSC Reader",
      "type": "rsc-core",
      "subtype": "read",
      "parentId": "ns-core",
      "gridX": 1,
      "gridY": 4,
      "description": "Reads from DB on demand."
    },
    {
      "id": "rsc-monitor-1",
      "name": "RSC Monitor",
      "type": "rsc-core",
      "subtype": "monitor",
      "parentId": "ns-integration",
      "gridX": 6,
      "gridY": 3,
      "description": "Monitors rsc-write, interacts with Zookeeper, answers get queries."
    },
    {
      "id": "ui-proxy-1",
      "name": "UI Proxy",
      "type": "ui",
      "subtype": "proxy",
      "parentId": "ns-integration",
      "gridX": 7,
      "gridY": 1,
      "description": "Connects to external auth, redirects to ui-front."
    },
    {
      "id": "ui-front-1",
      "name": "UI Frontend",
      "type": "ui",
      "subtype": "front",
      "parentId": "ns-integration",
      "gridX": 7,
      "gridY": 2,
      "description": "Static forms, receives from proxy, calls ui-back."
    },
    {
      "id": "ui-back-1",
      "name": "UI Backend",
      "type": "ui",
      "subtype": "back",
      "parentId": "ns-integration",
      "gridX": 7,
      "gridY": 3,
      "description": "Receives from front, writes to Kafka, reads/writes DBs."
    },
    {
      "id": "db-pg-main",
      "name": "PostgreSQL Main",
      "type": "db",
      "subtype": "postgres",
      "parentId": "sys-main",
      "gridX": 1,
      "gridY": 6,
      "description": "Slow but reliable relational store."
    },
    {
      "id": "db-ignite-cache",
      "name": "Ignite Cache",
      "type": "db",
      "subtype": "ignite",
      "parentId": "sys-main",
      "gridX": 2,
      "gridY": 6,
      "description": "Fast in-memory data grid."
    },
    {
      "id": "transport-kafka",
      "name": "Kafka Cluster",
      "type": "transport",
      "subtype": "kafka",
      "parentId": "sys-integration",
      "gridX": 5,
      "gridY": 5,
      "description": "Internal message bus."
    },
    {
      "id": "transport-zk",
      "name": "Zookeeper Ensemble",
      "type": "transport",
      "subtype": "zookeeper",
      "parentId": "sys-integration",
      "gridX": 6,
      "gridY": 5,
      "description": "Coordination and configuration."
    },
    {
      "id": "proxy-gw",
      "name": "API Gateway",
      "type": "proxy",
      "subtype": "gateway",
      "parentId": "zone-our",
      "gridX": 0,
      "gridY": 0,
      "description": "Entry point for external traffic into our zone."
    },
    {
      "id": "proxy-lb",
      "name": "Load Balancer",
      "type": "proxy",
      "subtype": "load_balancer",
      "parentId": "zone-our",
      "gridX": 4,
      "gridY": 0,
      "description": "Distributes internal traffic."
    },
    {
      "id": "ext-master",
      "name": "External Master",
      "type": "get",
      "subtype": null,
      "parentId": "ns-ext",
      "gridX": 10,
      "gridY": 2,
      "description": "External system that sends commands."
    },
    {
      "id": "ext-db-orch",
      "name": "External DB Orchestrator",
      "type": "bps",
      "subtype": null,
      "parentId": "ns-ext",
      "gridX": 10,
      "gridY": 4,
      "description": "Manages external database schemas."
    },
    {
      "id": "ext-auth",
      "name": "External Auth",
      "type": "proxy",
      "subtype": "gateway",
      "parentId": "ns-ext",
      "gridX": 11,
      "gridY": 2,
      "description": "External authentication service."
    }
  ],
  "connections": [
    {
      "id": "c1",
      "sourceNodeId": "proxy-gw",
      "targetNodeId": "get-1",
      "type": "HTTP",
      "direction": "directed",
      "label": "API request"
    },
    {
      "id": "c2",
      "sourceNodeId": "proxy-gw",
      "targetNodeId": "get-2",
      "type": "HTTP",
      "direction": "directed",
      "label": "API request"
    },
    {
      "id": "c3",
      "sourceNodeId": "get-1",
      "targetNodeId": "syncer-1",
      "type": "gRPC",
      "direction": "directed",
      "label": "Process request"
    },
    {
      "id": "c4",
      "sourceNodeId": "get-1",
      "targetNodeId": "rsc-read-1",
      "type": "gRPC",
      "direction": "directed",
      "label": "Read data"
    },
    {
      "id": "c5",
      "sourceNodeId": "get-2",
      "targetNodeId": "ext-master",
      "type": "HTTP",
      "direction": "directed",
      "label": "Fetch external status"
    },
    {
      "id": "c6",
      "sourceNodeId": "get-2",
      "targetNodeId": "syncer-2",
      "type": "gRPC",
      "direction": "directed"
    },
    {
      "id": "c7",
      "sourceNodeId": "syncer-1",
      "targetNodeId": "transport-kafka",
      "type": "Kafka",
      "direction": "bidirectional",
      "label": "read/write topics"
    },
    {
      "id": "c8",
      "sourceNodeId": "syncer-1",
      "targetNodeId": "db-pg-main",
      "type": "SQL",
      "direction": "directed",
      "label": "query"
    },
    {
      "id": "c9",
      "sourceNodeId": "syncer-2",
      "targetNodeId": "ext-master",
      "type": "HTTP",
      "direction": "directed"
    },
    {
      "id": "c10",
      "sourceNodeId": "put-1",
      "targetNodeId": "ext-master",
      "type": "Kafka",
      "direction": "directed",
      "label": "write external topic"
    },
    {
      "id": "c11",
      "sourceNodeId": "put-1",
      "targetNodeId": "syncer-1",
      "type": "gRPC",
      "direction": "directed"
    },
    {
      "id": "c12",
      "sourceNodeId": "put-1",
      "targetNodeId": "bps-1",
      "type": "gRPC",
      "direction": "directed"
    },
    {
      "id": "c13",
      "sourceNodeId": "bps-1",
      "targetNodeId": "db-pg-main",
      "type": "SQL",
      "direction": "bidirectional",
      "label": "read/write"
    },
    {
      "id": "c14",
      "sourceNodeId": "bps-1",
      "targetNodeId": "transport-kafka",
      "type": "Kafka",
      "direction": "directed",
      "label": "write"
    },
    {
      "id": "c15",
      "sourceNodeId": "bps-1",
      "targetNodeId": "transport-zk",
      "type": "Zookeeper",
      "direction": "bidirectional",
      "label": "schema/config"
    },
    {
      "id": "c16",
      "sourceNodeId": "bps-1",
      "targetNodeId": "ext-db-orch",
      "type": "HTTP",
      "direction": "directed"
    },
    {
      "id": "c17",
      "sourceNodeId": "rsc-write-1",
      "targetNodeId": "transport-kafka",
      "type": "Kafka",
      "direction": "directed",
      "label": "read"
    },
    {
      "id": "c18",
      "sourceNodeId": "rsc-write-1",
      "targetNodeId": "db-pg-main",
      "type": "SQL",
      "direction": "directed",
      "label": "write"
    },
    {
      "id": "c19",
      "sourceNodeId": "rsc-write-1",
      "targetNodeId": "db-ignite-cache",
      "type": "SQL",
      "direction": "directed",
      "label": "write"
    },
    {
      "id": "c20",
      "sourceNodeId": "rsc-read-1",
      "targetNodeId": "db-pg-main",
      "type": "SQL",
      "direction": "directed",
      "label": "read"
    },
    {
      "id": "c21",
      "sourceNodeId": "rsc-read-1",
      "targetNodeId": "db-ignite-cache",
      "type": "SQL",
      "direction": "directed",
      "label": "read"
    },
    {
      "id": "c22",
      "sourceNodeId": "rsc-monitor-1",
      "targetNodeId": "rsc-write-1",
      "type": "gRPC",
      "direction": "directed",
      "label": "health check"
    },
    {
      "id": "c23",
      "sourceNodeId": "rsc-monitor-1",
      "targetNodeId": "transport-zk",
      "type": "Zookeeper",
      "direction": "bidirectional"
    },
    {
      "id": "c24",
      "sourceNodeId": "get-1",
      "targetNodeId": "rsc-monitor-1",
      "type": "gRPC",
      "direction": "directed"
    },
    {
      "id": "c25",
      "sourceNodeId": "ui-proxy-1",
      "targetNodeId": "ext-auth",
      "type": "HTTP",
      "direction": "directed",
      "label": "authenticate"
    },
    {
      "id": "c26",
      "sourceNodeId": "ui-proxy-1",
      "targetNodeId": "ui-front-1",
      "type": "HTTP",
      "direction": "directed",
      "label": "redirect"
    },
    {
      "id": "c27",
      "sourceNodeId": "ui-front-1",
      "targetNodeId": "ui-back-1",
      "type": "HTTP",
      "direction": "directed",
      "label": "API call"
    },
    {
      "id": "c28",
      "sourceNodeId": "ui-back-1",
      "targetNodeId": "transport-kafka",
      "type": "Kafka",
      "direction": "directed",
      "label": "write"
    },
    {
      "id": "c29",
      "sourceNodeId": "ui-back-1",
      "targetNodeId": "db-pg-main",
      "type": "SQL",
      "direction": "bidirectional"
    },
    {
      "id": "c30",
      "sourceNodeId": "ui-back-1",
      "targetNodeId": "db-ignite-cache",
      "type": "SQL",
      "direction": "bidirectional"
    },
    {
      "id": "c31",
      "sourceNodeId": "proxy-lb",
      "targetNodeId": "ui-proxy-1",
      "type": "HTTP",
      "direction": "directed"
    },
    {
      "id": "c32",
      "sourceNodeId": "proxy-lb",
      "targetNodeId": "get-2",
      "type": "HTTP",
      "direction": "directed"
    }
  ],
  "groups": [
    {
      "id": "group-gets",
      "name": "GET Modules",
      "nodeIds": ["get-1", "get-2"]
    },
    {
      "id": "group-syncers",
      "name": "Syncer Modules",
      "nodeIds": ["syncer-1", "syncer-2"]
    },
    {
      "id": "group-rsc-cores",
      "name": "RSC Core Modules",
      "nodeIds": ["rsc-write-1", "rsc-read-1", "rsc-monitor-1"]
    }
  ]
}
```
