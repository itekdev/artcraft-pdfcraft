 ▐▛███▛█   Claude Code v2.1.285
▝▜██████▀  Sonnet 5.5 · Claude Team
 ▝▝   ▝▝   ~/egui/pdfcraft/crates

  Get to finished work sooner with Opus 5.5. Switch anytime with /model.

❯ /plugin                                                                       

❯ /skills 
  ⎿  No changes

❯ What does the overall call graph look like when opening a PDF to view         

  Called codegraph 2 times (ctrl+o to expand)

● This is the desktop path in crates/ (not the newgui/ stub). I read
  open_unguarded and the per-frame canvas code only through their signatures and
  call edges, so the middle steps are partly inferred.

  Overview

   user: CLI arg / File▸Open / drop / Recent / OS event / web ?file=
          │
          ▼
   ┌─ L7  ui-egui ─────────────────────────────────────────────────────────┐
   │ PdfCraftApp::open_path(path)            [lib.rs:879]  fs::read        │
   │   └─ open_bytes(name, path, bytes)      [lib.rs:726]                  │
   │        ├─ open_converted()  (image/text → new PDF; early return)      │
   │        └─ try_open()                    [lib.rs:734]                  │
   └───────────────┬───────────────────────────────────────────────────────┘
                   │ session.open(name, path, Arc<bytes>, password)
                   ▼
   ┌─ engine ──────────────────────────────────────────────────────────────┐
   │ Session::open                           [engine/lib.rs:1966]          │
   │   └─ guard(catch panics) → open_unguarded                             │
   │        ├─ pdfcraft_cos::Document::open_with_password   (parse, repair)│
   │        │     └─ crypt (password auth) → NeedsPassword/WrongPassword   │
   │        ├─ pdfcraft_render::inspect → DocInfo                          │
   │        │     (pages, outline, annots, fields, layers, attachments)    │
   │        ├─ (XFA layout if the PDF is a dynamic XFA form)               │
   │        └─ push_document                 [engine/lib.rs:2118]          │
   │             ├─ RenderPool::new(bytes, threads, config)                │
   │             │    └─ spawn_worker × N → each owns a PageRenderer       │
   │             │         └─ Pdf::new_with_password (hayro) [raster.rs]   │
   │             ├─ Editor{cos, undo, redo}                                │
   │             └─ forms::fields, edit::marks_present, annot::links::list,│
   │                sign::signatures_of  (all derived from the cos doc)    │
   └───────────────┬───────────────────────────────────────────────────────┘
                   │ DocId
                   ▼
   back in try_open:
     ├─ NeedsPassword/WrongPassword → set password_prompt, return
     ├─ pick right panel (Comments / Bookmarks / none)
     ├─ DocView::new(id, &info, view_defaults)   [canvas.rs:321]
     ├─ views.push; active = Some(i)
     ├─ apply_initial_view(i, doc.initial_view())  [lib.rs:689]
     │     (panel, layout, zoom/fit, start page)
     ├─ select_mode(default_mode) / mode_override
     ├─ update `recent` list
     └─ take_js_output → handle_js  (form open-scripts)

  Rendering (every frame after open)

  The open call renders nothing. Pages are drawn lazily on later frames:

  eframe update() → PdfCraftApp
    └─ canvas (DocView) per frame:
         ├─ work out visible pages/tiles from scroll, zoom, layout, viewport
         ├─ RenderPool::set_queue(Vec<RenderRequest>)   most urgent first
         ├─ RenderPool::try_recv() → RenderedPage        [DocView::receive]
         │     └─ rgba → egui texture, cached in view.pages / tiles / thumbs
         └─ paint textures + overlays (selection, find, comments, form fields)

   worker thread (raster.rs):
     PageRenderer::render(req)
       └─ render_page(pdf, cache, settings, req)   hayro interpreter
            kinds: Pixels | Text (PageText, glyph boxes)
            watchdog: STUCK_AFTER = 20s → replacement worker
            no threads (wasm): rendered inline inside try_recv

  Points worth knowing

  - Two parsers. cos parses the document for editing, and lopdf (inside inspect)
    and hayro parse it for display. This is the bootstrap debt noted in
    CLAUDE.md. The same bytes are parsed up to three times, and a file that cos
    can't read still opens read-only (read_only_reason).
  - Panic containment sits at three layers: guard around open_unguarded,
    catch_unwind around the cos parse and hayro parse, and the pool watchdog.
  - Other entry points go through the same Session::open → push_document. They
    are doc_open in crates/automation/src/lib.rs:612 (CLI/MCP), open_new
    (combine/extract) and open_revision. page_render there calls
    renderer(id).render(...) and skips the egui path.
  - newgui/ is an untracked mock. Its Session::open ignores the bytes and
    returns a blank Document, so it has no parsing or rendering.

  To get exact source for the middle of the chain, I can pull up open_unguarded
  or the canvas request-queue code (DocView frame loop) next.

✻ Churned for 22s · done 2:07 PM

※ recap: You asked for the overall call graph when opening a PDF to view, and I've mapped it from the UI's open_path through the engine's Session::open to the render pool. Next, I can pull the source for open_unguarded or the canvas render-queue loop if you want more detail. (disable recaps in /config)

❯ please explain or graphify the ui tree of this app. And list steps for        
  customizing the ui function layout, e.g., remove some functions or tabs.      

  Called codegraph 2 times, ran 3 shell commands (ctrl+o to expand)

● PdfCraft UI tree (crates/ui-egui)

  Everything is built in PdfCraftApp::ui() (lib.rs:1518), which runs every
  frame. Panel order matters in egui: panels are claimed outside-in, and the
  central panel gets what's left.

  PdfCraftApp::ui()
  ├─ [full_screen && doc open] → CentralPanel{ canvas::document_area } + dialogs
  + toast, return
  │
  ├─ TOP  chrome::tab_strip                  chrome.rs:9    (38px)
  │    ├─ Home button (active=None)
  │    ├─ one tab() per DocView  (name, dirty dot, close ×)
  │    ├─ "+ Open" ghost button
  │    └─ right side: theme button, shortcuts, Discord
  │
  ├─ TOP  chrome::mode_bar                   chrome.rs:123  (48px)
  │    ├─ main_menu()  hamburger: File / Edit / Pages / View / Help   
  chrome.rs:165
  │    ├─ mode tabs: All tools · Read · Edit · Convert · E-Sign       (enum 
  Mode)
  │    └─ right side: search box (opens palette), Document props, Save, Print
  │
  ├─ RIGHT chrome::right_rail                chrome.rs:246  (48px, only if doc 
  open)
  │    └─ icon buttons → app.right = Some(RightPanel::…)
  │         Comments, Bookmarks, Pages, Fields, Layers, Attachments, Signatures,
  │         (Accessibility if a check ran), Search, Compare, page-display menu
  │
  ├─ RIGHT panels::right_panel               panels.rs:384  (hidden when mode ==
  Read)
  │    └─ match RightPanel → Comments / Bookmarks / Pages thumbs / Fields / 
  Layers /
  │       Attachments / Signatures / Accessibility / Search / Compare
  │
  ├─ LEFT  panels::left_panel                panels.rs:17   (272px, hidden in 
  Read)
  │    └─ match app.left
  │         ├─ LeftPanel::AllTools → all_tools(): rows from TOOL_GROUPS (14 
  shown, "View more")
  │         └─ LeftPanel::Tool(id) → tool_detail(): catalog::group(id).sections 
  → items
  │                                  each item = a command id run via 
  app.run_command()
  │
  ├─ CENTRAL
  │    ├─ active == None → home::show        (recent files, drop zone)
  │    └─ active == Some(i) → canvas::document_area
  │         ├─ notices banner (repaired / secured / signatures)
  │         ├─ pages: Single / Continuous / TwoUp, tiles from RenderPool
  │         ├─ per-tool overlays: comments, forms, redact, crop, measure, links,
  sign…
  │         └─ canvas::quick_bar   canvas.rs:2288  floating vertical strip
  │              Select, Hand, Comment/Highlight/Draw groups with flyouts, …
  │
  └─ OVERLAYS: palette::show (Ctrl/⌘K), dialogs::show (enum Dialog), 
  widgets::toast

  How the pieces are wired

  Concept: Mode (5 workspaces)
  Defined in: lib.rs:97
  Notes: select_mode (lib.rs:1042) maps each mode to a left panel: Edit→"edit",
  Convert→"export", Sign→"fill_sign", others→AllTools. It is also persisted as
  default_mode.
  ────────────────────────────────────────
  Concept: LeftPanel / RightPanel / QuickTool
  Defined in: lib.rs:121, 128, 146
  Notes: Enums, so removing a variant makes the compiler list every use.
  ────────────────────────────────────────
  Concept: Tool catalog
  Defined in: crates/engine/src/catalog.rs (TOOL_GROUPS, ToolGroup)
  Notes: The data behind the left panel, the command palette (palette.rs:63),
  the
  icons (icons.rs:64) and the i18n tests (i18n/mod.rs).
  ────────────────────────────────────────
  Concept: Commands
  Defined in: pdfcraft_engine::commands, run_command / execute (lib.rs:1022)
  Notes: Menus, palette and tools all go through string command ids like
  "file.save".
  ────────────────────────────────────────
  Concept: Theme tokens
  Defined in: theme.rs Tokens
  Notes: Colours and radius.

  Steps to customize the layout

  A. Remove or reorder tools in the left panel (easiest, data only)
  1. Edit TOOL_GROUPS in crates/engine/src/catalog.rs. Delete a ToolGroup to
     drop the whole tool, or delete individual item(...) entries in its sections
     to drop single functions.
  2. The left panel, command palette and icon preloading all follow from that
     data.
  3. Check COLLAPSED_TOOLS = 14 (panels.rs:11). It sets how many rows show
     before "View more". Lower it, or remove the expand logic in all_tools().
  4. Run cargo test -p pdfcraft-ui-egui -p pdfcraft-engine. Tests in i18n/mod.rs
     iterate TOOL_GROUPS. Other tests may reference group ids such as "edit" or
     "measure" through LeftPanel::Tool("…").
  5. If you remove a group that select_mode points at ("edit", "export",
     "fill_sign"), left_panel falls back to AllTools (panels.rs:32). Update
     select_mode anyway.

  B. Remove a mode tab (All tools / Read / Edit / Convert / E-Sign)
  1. In chrome.rs:135-137, delete the tuple from the mode array.
  2. In lib.rs:97, remove the Mode variant and its parse() and serde entries
     (lib.rs:107). The compiler then flags the remaining match arms:
     select_mode, the CLI --mode option and the default_mode setting in
     Preferences. Stored default_mode values that no longer parse must fall back
     to a default.
  3. self.mode != Mode::Read gates the side panels at lib.rs:1540 and 1546. Keep
     Read if you want that distraction-free behaviour.

  C. Remove right-rail icons and their panels
  1. Delete the rail_button(...) line in chrome.rs:278-290.
  2. If the panel should be gone entirely, remove the RightPanel variant and its
     arm in right_panel() (panels.rs:384). Also remove the places that set it,
     such as canvas.rs:1144 (Notice::Signatures), apply_initial_view and
     try_open (they auto-open Comments or Bookmarks).
  3. If you only want to hide it, delete the rail button and leave the enum.

  D. Trim the top bars
  - Tab strip: delete the theme, shortcuts or Discord buttons in tab_strip
    (chrome.rs:45-61). To drop Home, change app.active = None handling and
    home::show.
  - Mode bar: remove Print, Save or Info (chrome.rs:146-154), or the search box
    (chrome.rs:157).
  - Hamburger menu: edit main_menu (chrome.rs:165). Its entries are driven by
    command registry groups.

  E. Remove quick-bar tools (floating strip)
  1. In canvas.rs:2288 quick_bar, delete the icons::button(...) blocks. The
     comment groups come from comments::GROUPS.
  2. Remove the matching QuickTool variants only if nothing else produces them
     (the keyboard shortcuts V and H, the palette, measure_ui::command).

  F. Remove a function everywhere, including keyboard and palette
  1. Remove its command id from the catalog (step A) and from
     pdfcraft_engine::commands.
  2. Remove the menu entry, any shortcut handling, and the automation tool in
     crates/automation/src/tools.rs if it should not be scriptable.
     tool_table_is_well_formed in crates/automation/tests/automation.rs checks
     that table.

  G. Verify
  1. Run cargo fmt --check && cargo clippy --workspace -- -D warnings && cargo 
     test --workspace. These are the repo's quality gates.
  2. Look at the result: pdfcraft --control /tmp/c doc.pdf, then pdfcraft-cli ui
     --control /tmp/c screenshot --out x.png.
  3. Update parity/acrobat-features.toml if you hid something marked shipped.
     Run cargo xtask parity.

  Caveats

  - Removing features moves away from the repo's Acrobat-parity goal, so a
    runtime switch may suit a white-label build better. A small UiLayout struct
    on PdfCraftApp, loaded from settings and holding hidden tool ids, hidden
    modes and hidden rail panels, could filter TOOL_GROUPS, the mode array and
    rail_button. That is more work up front, but it avoids forking the catalog
    and enum code.
  - newgui/ is a separate, untracked template that mirrors this tree, with the
    same panel names and a flattened QuickTool. If you are customizing there,
    the same chrome.rs and panels.rs structure applies, but its catalog is
    catalog::TOOL_GROUPS in the template engine.
  - I read the quick-bar and rail code only partway. I haven't read the
    ToolSection and ToolItem definitions or the command registry, so check those
    before deleting items.

  I can draft the runtime UiLayout config approach, or make one specific
  removal, if you want either.

✻ Sautéed for 42s · done 3:17 PM

❯ /export ./architecture/ui-how2modify.md                                       
  ⎿  Conversation exported to:
     /home/ericden/egui/pdfcraft/crates/architecture/ui-how2modify.md