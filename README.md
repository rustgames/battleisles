# battleisles

Nostalgia version of battle isles 

Using Rust 1.98.1, pinned in rust-toolchain.toml (rustup installs it automatically on first build). CI uses the same file.

clone and run 'cargo run'

If you want wasm support:

    - Install trunk with 'cargo install --locked trunk'

    - Install wasm target with 'rustup target add wasm32-unknown-unknown' 
    
    - Install wasm-bindg-clien with 'cargo install --locked wasm-bindgen-cli'
    
    - run 'trunk build'
    
    - run 'trunk serve'

Secrets and per-project environment variables: see [docs/secrets-setup.md](docs/secrets-setup.md).

If you want to use a codespace, everything should be ready in .devcontainer

