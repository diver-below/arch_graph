# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview
# Technical Task: Interactive System Architecture Visualization UI

## 1. Overview
Build a comfortable, zoomable, and pannable user interface for visualizing a complex system architecture.  
The UI renders a directed graph showing modules (pods), databases, transports, proxies, and their interconnections.  
Data is loaded from a JSON file (backend-supplied; a demo file will be used for development).  
The application must run as a **systemd user unit** on a Linux VM.

## 2. Goals
- Provide a clear, navigable overview of system topology.
- Allow users to zoom and pan freely around the graph.
- Enable selection of an edge to inspect its details and connected nodes.
- Highlight the selected subgraph and dim unrelated elements.
- Offer a clean, modern UI with a side panel for selected item information.

## 3. Functional Requirements

### 3.1 Data Loading
- Load graph structure and metadata from a static JSON file (demo file provided).
- The file format is specified in Section 7 (Data Model).
- The UI must handle the demo file out-of-the-box without additional backend.

### 3.2 Graph Rendering
- **Hierarchical layout** based on zones → systems → namespaces → pods/modules.
- Nodes represent:
  - Network Zones (top-level containers)
  - Systems (inside zones)
  - Namespaces (inside systems)
  - Pods / Modules of types: `get`, `syncer`, `put`, `bps`, `rsc-core`, `ui`, `other`
  - Databases: `postgres` (slow) or `ignite` (fast)
  - Transport units: `kafka`, `zookeeper`
  - Proxies: `gateway`, `load_balancer`
- Edges represent directional connections between any of the above (e.g., module → DB, module → transport, proxy → system boundary, etc.).
- Node and edge styling must reflect the type (e.g., different colors/shapes for DBs, transports, proxies, module types).
- Support flexible, non-overlapping curved edges (Bézier curves or similar).

### 3.3 Navigation
- **Zoom**: smooth zooming via scroll/pinch or on-screen buttons (fit-to-screen, zoom in/out).
- **Pan**: drag to move the canvas.
- The graph must remain fully interactive at all zoom levels.
- Optionally, a mini-map for overview.

### 3.4 Edge Selection & Interaction
- **Selecting an edge** (click/tap):
  1. The graph automatically pans/zooms to focus on that edge and its **directly connected nodes** (source and target).
  2. All other nodes and edges become **greyed out** (reduced opacity / muted colors).
  3. The selected edge is highlighted (e.g., thicker, glowing).
  4. A **side panel** appears on the right side of the screen (desktop) or as a bottom sheet / overlay (mobile).
- **Side panel contents**:
  - Description of the edge (from metadata, e.g., protocol, purpose).
  - A **copyable list** of connections: source → target with detailed node names/IDs.
  - Metadata: type of connection (sync/async, request/response, read/write, etc.), technology used.
  - A “Copy to clipboard” button for the connection list.
- **Deselection**: Clicking on empty space or pressing `Esc` clears the selection, restores full color/opacity, and hides the panel.
- While a selection is active, further clicks on other edges replace the selection.

### 3.5 Visual Feedback
- Hover effects on nodes and edges (tooltip with basic info).
- Smooth transitions when focusing on a selection (pan+zoom animation).
- The side panel should slide in/out smoothly.

## 4. Non-Functional Requirements
- **Performance**: Smooth 60fps interaction with up to 200 nodes and 500 edges.
- **Responsiveness**: Works on standard desktop resolutions (1280×720 and above). Mobile support is a plus but not mandatory.
- **Deployment**: Must run as a **systemd user unit** (`systemctl --user`) on a Linux VM. The application should be served by a lightweight HTTP server (e.g., Node.js static server, Python http.server, or nginx) bundled with the UI.
- **Accessibility**: Basic keyboard navigation (Tab to select, Esc to deselect), panel content readable by screen readers (semantic HTML).

## 5. UI/UX Design
- **Layout**: 
  - Full-viewport canvas with graph.
  - Right-side panel (width ~350px) toggled on selection, overlaying but not permanently covering the graph.
  - Optional toolbar at the top/bottom: zoom controls, reset view, legend toggle.
- **Color Palette**:
  - Normal state: distinct colors per node/edge category.
  - Selected state: vibrant highlight.
  - Greyed-out state: low opacity (0.2), grayscale.
- **Typography**: Sans-serif, clean, readable font.
- **Icons**: Optional but encouraged to differentiate node types.

## 6. Architecture & Technology Stack (Suggestion)
The agent may choose the technology, but the following is recommended for simplicity and performance:
- **Frontend**: HTML5 Canvas or SVG with a modern graph library (e.g., Cytoscape.js, D3.js force/graph, vis.js, or GoJS).
- **Framework**: Vanilla JavaScript or a lightweight framework (e.g., Svelte, React, Vue). The final build should be static files (HTML+JS+CSS).
- **Server**: A simple HTTP server to serve static files (e.g., `http-server` npm package, Python `http.server`, or Nginx).
- **Systemd user unit**:
  - Service file stored in `~/.config/systemd/user/`.
  - The unit starts the HTTP server on a specific port (e.g., 8080).
  - Working directory set to the app’s static folder.
  - `Restart=on-failure`.

## 7. Data Model (Demo File Format)
The demo JSON file (e.g., `demo-architecture.json`) follows this structure:

```json
{
  "zones": [
    {
      "id": "zone-dmz",
      "name": "DMZ",
      "systems": [
        {
          "id": "sys-front",
          "name": "Front System",
          "namespaces": [
            {
              "id": "ns-api",
              "name": "api-gateway",
              "pods": [
                {
                  "id": "pod-gateway",
                  "type": "proxy",
                  "subtype": "gateway",
                  "name": "API Gateway"
                }
              ]
            }
          ],
          "databases": [
            {
              "id": "db-postgres-1",
              "type": "postgres",
              "name": "Main DB"
            }
          ],
          "transports": [
            {
              "id": "transport-kafka-1",
              "type": "kafka",
              "name": "Kafka Cluster"
            }
          ],
          "proxies": [
            {
              "id": "proxy-lb-1",
              "type": "load_balancer",
              "name": "LB Public"
            }
          ]
        }
      ]
    }
  ],
  "edges": [
    {
      "id": "edge-001",
      "source": "pod-get-1",
      "target": "db-postgres-1",
      "metadata": {
        "description": "GET module reads from PostgreSQL",
        "protocol": "tcp",
        "direction": "unidirectional",
        "connectionType": "read",
        "technology": "jdbc"
      },
      "connections": [
        { "sourceName": "GET-module", "targetName": "Main DB (Postgres)" }
      ]
    }
  ]
}

All nodes (pods, databases, transports, proxies) are uniquely referenced by their id.

The edges array defines directed connections. connections provides human-readable labels for the pop-up copy list.


8. Detailed Module Behavior (Informational for Visualization)
The following describes the expected connections and roles of each module type. The demo file should include edges that reflect these relationships to create a realistic example.

get – receives requests from external systems, connects to syncer, rsc-core (read/monitor), and external master systems.

syncer – receives online requests from get, connects to external master systems, DBs, and writes/reads from internal Kafka topics.

put – connects to external Kafka master-system topics, talks to syncer and bps.

bps – reads/writes DBs, writes to Kafka, interacts with external DB-orchestrator, writes/reads Zookeeper (schema).

rsc-core subtypes:

rsc-write – reads from internal Kafka, writes to DB.

rsc-read – reads from DB.

rsc-monitor – requests to rsc-write, reads/writes Zookeeper, receives requests from get.

ui subtypes:

ui-proxy – connects to external auth system, forwards requests to ui-front.

ui-front – serves static forms, receives from ui-proxy, requests to ui-back.

ui-back – receives from ui-front, writes to internal Kafka, reads/writes DBs.

The UI does not enforce any logic, but this information should be reflected in the demo data to showcase a realistic architecture.

9. Demo Data Requirements
The agent must create a rich demo JSON file (demo-architecture.json) that includes:

At least 2 zones, 3 systems, multiple namespaces.

Examples of all node types: get, syncer, put, bps, rsc-core (all three), ui (all three), postgres, ignite, kafka, zookeeper, gateway, load_balancer.

At least 15 edges with descriptive metadata and connection lists.

Connections based on the behavior described in Section 8.

The demo file must be placed in the static assets folder.

10. Implementation Tasks for Coding Agent
Initialize project with chosen tooling, produce a static web app.

Create the demo JSON file as per Section 9.

Implement data parser to load the JSON and build a graph model.

Layout and render the graph:

Use a hierarchical or force-directed layout with clustering by zones/systems/namespaces.

Apply distinct visual styles for node categories.

Add zoom and pan interactions.

Implement edge selection:

Click handler on edges.

Focus animation (pan & zoom to fit the edge and its incident nodes).

Grey-out all other elements.

Build the side panel:

Display edge description, connection list, metadata.

Implement copy-to-clipboard.

Add deselection logic (click background / Esc).

Ensure responsiveness (desktop primarily).

Package for deployment:

Set up a minimal HTTP server (e.g., serve npm package, or a small Node.js script).

Create a systemd user unit file (system-architecture-ui.service) that starts the server on boot.

Provide a simple install script or instructions.

11. Acceptance Criteria
The UI loads the demo file and renders a clear, interactive graph.

User can zoom (scroll/pinch) and pan (drag) smoothly.

Clicking an edge highlights it, focuses on the source/target nodes, and opens a right-side panel with correct information.

Clicking background or pressing Esc restores normal view and closes the panel.

The “Copy connections” button copies the list as plain text.

The application runs as a systemd user unit and is reachable via a web browser (e.g., http://localhost:8080).

12. Out of Scope
Backend integration / dynamic data loading (the app works with a static file).

Authentication / user management.

Editing or saving graph structures.

Real-time updates (WebSocket, etc.).

Mobile-first design (basic responsiveness is acceptable).

13. Deliverables
Complete source code with build instructions.

Static build output (if any build step is used) or directly served source.

demo-architecture.json file.

system-architecture-ui.service user unit file.

README.md with deployment steps.
