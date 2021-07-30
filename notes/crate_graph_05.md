# Summary of ERA05: crate graph

## FileSet
file sets were previously covered in the [vfs lecture](vfs_02.md).

## CrateGraph
The `CrateGraph` is a graph which depicts the dependencies between the crates. Each crate is a node and each edge can be considered as a dependency relationship.
``` rust
pub struct CrateGraph {
    arena: FxHashMap<CrateId, CrateData>,
}
```
``` rust
#[derive(Debug, Clone)]
pub struct CrateData {
    pub root_file_id: FileId,
    pub edition: Edition,
    /// A name used in the package's project declaration: for Cargo projects,
    /// its `[package].name` can be different for other project types or even
    /// absent (a dummy crate for the code snippet, for example).
    ///
    /// For purposes of analysis, crates are anonymous (only names in
    /// `Dependency` matters), this name should only be used for UI.
    pub display_name: Option<CrateDisplayName>,
    pub cfg_options: CfgOptions,
    pub potential_cfg_options: CfgOptions,
    pub env: Env,
    pub dependencies: Vec<Dependency>,
    pub proc_macro: Vec<ProcMacro>,
}
```
Each Crate is allocated a 32-bit unique ID which is then mapped to the `CrateData`.

## CrateData
The properties in this type are not cargo-specific and make sense for all Rust projects with any build system.

### root_file_id
The root file of the crate (lib.rs or main.rs in case of cargo). `FileId` is the ID allocated to a file by the vfs.

### edition
The edition of source code.
``` rust
pub enum Edition {
    Edition2015,
    Edition2018,
    Edition2021,
}
```

### env
The environment at crate's compile time.
``` rust
pub struct Env {
    entries: FxHashMap<String, String>,
}
```