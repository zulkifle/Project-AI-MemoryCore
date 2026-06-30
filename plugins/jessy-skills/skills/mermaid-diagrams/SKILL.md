---
name: mermaid-diagrams
description: "MUST use when user says 'create flowchart', 'create diagram', 'draw flow',
             'make infrastructure diagram', 'document the flow', 'system diagram',
             'create sequence diagram', 'draw architecture', or asks to visualise
             a system/API/infrastructure as a diagram. Also triggers on:
             'flowchart', 'mermaid', 'signing flow diagram', 'infra diagram',
             'deployment diagram', 'save as diagram', 'save to docs as diagram'."
---

# Mermaid Diagrams — System Flowchart & Infrastructure Designer
*Reads project source code and generates Mermaid diagrams: sequence flows, flowcharts, and infrastructure maps.*

## Activation

When this skill activates, output:

`Drawing diagrams from source...`

Then execute the protocol below.

## Context Guard

| Context | Status |
|---------|--------|
| **User wants to visualise API flow, system flow, or infrastructure** | ACTIVE — full protocol |
| **User wants to document a signing/auth/data flow** | ACTIVE — full protocol |
| **User asks for a general diagram unrelated to code** | ACTIVE — produce best-fit Mermaid diagram |
| **User asks for UI mockup or wireframe** | DORMANT — different concern |

---

## Diagram Types Supported

| Type | Mermaid Keyword | Use For |
|------|----------------|---------|
| Sequence Diagram | `sequenceDiagram` | API request/response flows, auth flows, service interactions |
| Flowchart | `graph TB` / `graph LR` | System components, decision trees, data flow |
| Infrastructure | `graph TB` with subgraphs | Docker Compose, K8s deployments, cloud topology |
| Class Diagram | `classDiagram` | Domain models, class relationships |
| State Diagram | `stateDiagram-v2` | Status transitions, lifecycle flows |

---

## Protocol

### Step 1: Understand What to Diagram

Ask if not already clear:
- What system or feature? (signing flow, auth flow, infrastructure, etc.)
- Which diagrams? (sequence, infra, component overview, or all)
- Where to save? (default: `docs/[project]-diagrams.md`)

### Step 2: Read Source Code First

**Always read before drawing.** Never generate a diagram from assumptions.

For **API / flow diagrams** — read:
- [ ] Servlet / controller files (entry points)
- [ ] Auth / credential handler
- [ ] Service / handler classes called by controllers
- [ ] DB utility / repository classes
- [ ] Config reader (for env/keystore/properties)

For **infrastructure diagrams** — read:
- [ ] `docker-compose.yaml` / `docker-compose.yml`
- [ ] `Dockerfile`
- [ ] `deployment.yaml` (K8s)
- [ ] `service.yaml` (K8s)
- [ ] `.properties` or config files for ports/paths

### Step 3: Plan the Diagrams

Based on what was read, identify:
- Participants / components (name them as they appear in code)
- Key steps and their order
- Decision branches (auth pass/fail, success/error)
- Storage interactions (DB, filesystem, keystore)
- Infrastructure: containers, volumes, ports, dependencies

### Step 4: Write the Diagrams

Output as Mermaid code blocks inside a markdown file.

**Sequence Diagram rules:**
- Participant names = class names / service names from code
- Messages = actual method calls or HTTP verbs from code
- `alt` blocks for error/auth branches
- `Note over` for important processing steps (e.g., hashing, signing algo)
- Show return responses at the end

**Infrastructure Diagram rules:**
- Use `subgraph` to group: Host / Docker / K8s Cluster / Namespace / Pod
- Show port mappings as edge labels (`8091 → 8080`)
- Show volume mounts as edges with label (`volume → /opt/mtsa/properties`)
- Show image registry pulls where applicable
- `depends_on` as an explicit edge in Docker diagrams

**Component Overview rules:**
- Show all layers: Client → Auth → Endpoints → Core Logic → Storage
- Group by concern using `subgraph`
- Use `LR` (left-right) layout for readability

### Step 5: Save to File

- Default path: `docs/[project-name]-diagrams.md`
- File structure:
  ```
  # [Project] — System Diagrams
  > One-line description of what's covered.
  ---
  ## 1. [Flow Name] — `POST /endpoint`
  [mermaid block]
  ---
  ## 2. ...
  ```
- All diagrams in one file unless user requests separate files

### Step 6: Confirm

Report:
- File path written to
- List of diagrams created (numbered)
- Which renderer to use (GitHub, VS Code + Mermaid Preview extension)

---

## Mermaid Syntax Rules

```
# Sequence Diagram
sequenceDiagram
    participant A as Label A
    participant B as Label B
    A->>B: message
    B-->>A: response
    alt condition
        A-->>C: branch
    end
    Note over A: annotation

# Flowchart
graph TB
    subgraph GroupName["Label"]
        NODE["Node label<br/>second line"]
    end
    A -->|"edge label"| B

# Infrastructure with ports
graph TB
    CLIENT["Client"]
    subgraph DOCKER["Docker Compose"]
        subgraph APP["Container: app"]
            TOMCAT["Tomcat :8080"]
        end
    end
    CLIENT -->|"HTTP :8091"| TOMCAT
```

**Do not use:**
- Backslashes inside Mermaid labels (use forward slash or omit)
- Raw `\n` in node labels (use `<br/>` for line breaks)
- Unquoted labels with special characters (always wrap in `["..."]`)

---

## Mandatory Rules

1. **Always read source code first** — never invent participant names, method calls, or port numbers
2. **Match names to code** — participant names = class names exactly as they appear
3. **Show error branches** — every auth and signing flow must show failure paths
4. **One file, all diagrams** — unless user requests otherwise
5. **Mermaid only** — no PlantUML, no ASCII art; Mermaid renders natively on GitHub
6. **Verify special chars** — test labels for backslashes, colons, pipes that can break parsing

---

## Output Renders In

- GitHub (native, no extension needed)
- VS Code — install: **Markdown Preview Mermaid Support** extension
- Obsidian (native)
- GitLab (native)

---

## Level History

- **Lv.1** — Base: Read source → generate sequenceDiagram (API flows) + graph TB (infrastructure) + component overview, saved to docs/. Confirmed working on MyTrustSignerXML MITI project (sign/verify/getcertinfo flows + Docker Compose + K8s sandbox). (Origin: 2026-06-30)
