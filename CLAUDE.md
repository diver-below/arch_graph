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
