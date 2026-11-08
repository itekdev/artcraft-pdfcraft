Analyze this Rust repository and reconstruct the GUI component hierarchy of the egui/eframe application.

Do not modify application code.

**Investigation**
1. Find the application's entry point and implementation of `eframe::App`.
2. Trace the current `ui()` method and all relevant helper functions that construct the GUI.
3. Identify important containers: `CentralPanel`, `SidePanel`, `TopBottomPanel`, `Window`, `Area`, `ScrollArea`, `Grid`, `CollapsingHeader`, menus, tabs, and nested `ui.horizontal()` / `ui.vertical()` layouts.
4. Follow helper-function calls across files, including reusable components and custom widgets.
5. Identify conditional rendering and dynamic UI, including dialogs, popups, optional panels, and mode-dependent screens.

**Deliverables**

Create `architecture/gui-hierarchy.md` containing:
- A tree of the main screens and their nested UI components.
- A short description of each component's responsibility.
- Source file paths and line references.
- State dependencies for each major component.
- Uncertainties or parts that could not be traced.

Include Mermaid diagrams for:
- The top-level application layout.
- At least one representative screen in greater detail.

**Accuracy requirements**
- Derive the hierarchy from actual source code.
- Distinguish UI containers from application components and individual widgets.
- Treat egui as immediate-mode UI; do not invent a persistent widget tree.
- Represent function calls and conditional branches accurately.
- Do not collapse unrelated components merely because they share a module.
- If a helper function is called from multiple places, represent the reuse explicitly.

Summarize the files examined and the evidence supporting the resulting diagrams. Do not change application behavior.