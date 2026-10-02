# Writing a plugin

A `zenohd` plugin is a Rust `cdylib` that implements `zenoh_plugin_trait::Plugin` with Zenoh's types. Start
from `plugins/zenoh-plugin-example` in the main repository.

## Skeleton

```rust
use zenoh::internal::{plugins::{RunningPluginTrait, ZenohPlugin, RunningPlugin}, runtime::DynamicRuntime};
use zenoh_plugin_trait::{plugin_version, plugin_long_version, Plugin, PluginControl};

pub struct MyPlugin;

// Exposes the plugin's vtable so zenohd can find it (only in the cdylib build)
#[cfg(feature = "dynamic_plugin")]
zenoh_plugin_trait::declare_plugin!(MyPlugin);

impl ZenohPlugin for MyPlugin {}

impl Plugin for MyPlugin {
    type StartArgs = DynamicRuntime;
    type Instance = RunningPlugin;

    const DEFAULT_NAME: &'static str = "my";
    const PLUGIN_VERSION: &'static str = plugin_version!();
    const PLUGIN_LONG_VERSION: &'static str = plugin_long_version!();

    fn start(name: &str, runtime: &Self::StartArgs) -> zenoh::Result<Self::Instance> {
        // Read this plugin's own config section: plugins/<name>
        let config = runtime.get_config().get_plugin_config(name)?;
        // Open a session on the router's runtime
        // let session = zenoh::session::init(runtime.clone()).await?;   (inside your async task)
        // spawn your work, keep a handle/flag to stop it…
        Ok(Box::new(MyRunning { /* … */ }))
    }
}

struct MyRunning { /* … */ }
impl PluginControl for MyRunning {}
impl RunningPluginTrait for MyRunning {
    // Optional: accept runtime config changes (the default rejects them)
    // fn config_checker(&self, path: &str, current: &JsonKeyValueMap, new: &JsonKeyValueMap)
    //     -> zenoh::Result<Option<JsonKeyValueMap>> { … }

    // Optional: answer admin-space queries under @/<zid>/router/status/plugins/<name>/**
    // fn adminspace_getter(&self, key_expr: &KeyExpr, plugin_status_key: &str)
    //     -> zenoh::Result<Vec<Response>> { … }
}
```

`Cargo.toml`:

```toml
[lib]
name = "zenoh_plugin_my"          # zenohd looks for lib<name>.so
crate-type = ["cdylib"]

[features]
default = ["dynamic_plugin"]
dynamic_plugin = []

[dependencies]
zenoh = { version = "=1.10.1", features = ["default", "internal", "plugins", "unstable"] }
zenoh-util = "=1.10.1"           # for JsonKeyValueMap in config_checker
zenoh-plugin-trait = "=1.10.1"
```

## Rules of thumb

- **Async runtime.** A dynamic plugin can't reuse `zenohd`'s tokio runtime handle. The example creates its
  own (`TOKIO_RUNTIME`, 2 workers, 50 blocking threads) and uses `Handle::try_current()` to pick the right
  one. The REST plugin does the same, which is why it has `work_thread_num` and `max_block_thread_num`.
- **Stopping.** When the plugin is removed from the config, its `RunningPlugin` is dropped or stopped. Use a
  flag or cancellation token to end your tasks.
- **Config changes.** Implement `config_checker` to accept, rewrite (`Ok(Some(new))`) or reject (`Err`)
  changes under `plugins/<name>/…`. Without it every change is rejected with
  `Runtime configuration change not supported`.
- **`__required__`.** Read it from your config. If it's true, panic on unrecoverable errors. Otherwise log them.
- **Compatibility.** Build with exactly the same rustc, Zenoh version and Zenoh features as the target
  `zenohd` ([details](index.md#binary-compatibility)).
- **Static linking.** Plugins can also be linked statically into a custom router binary with the
  `plugins` feature, which avoids ABI concerns.

## Sources

- `plugins/zenoh-plugin-example/src/lib.rs`, `Cargo.toml`
- `plugins/zenoh-plugin-trait/src/` (`plugin.rs`, `vtable.rs`, `compatibility.rs`)
- `zenoh/src/api/plugins.rs` (`RunningPluginTrait`, `config_checker`, `adminspace_getter`)
