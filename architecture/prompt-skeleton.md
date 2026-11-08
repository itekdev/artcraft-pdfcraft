Please use the analyzed results in `architecture/gui-hierarchy.md` and the source code in this Rust repository to build a new pure gui template app which using the egui framework in `crate/ui-egui` and `apps\pdfcraft\main.rs` as the app but remove all other document processings crates or dependencies to make it a portable egui app template skeleton.

Do not modify application code.
Generate the code in `newgui/*`.

**Investigation**
1. Use the analyzed results in `architecture/gui-hierarchy.md`
2. Use the application's entry point and implementation of `eframe::App`.
3. Remove all other dependecies as much as possible to ma


**Deliverables**

Create a working app in `newgui/` and dependecies in `newgui/crates/` containing:
- An egui app having the main screens and their nested UI components as the current app.
- Make the app running like a document editor with a empty document.
- Make other functions of document processing, loading, etc to crates as interfaces only without real functions.


**Accuracy requirements**
- Follow the same hierarchy from the original source code.
- Copy dependencies in `crates/*` into the `newgui/crates/*`
- Generate code or modify code only in `newgui/*` and `newgui/crates/*`
- Generate new `Cargo.toml` and necessary project configurations to  `newgui`.


