default: build

all: test

test: build
	cargo test

build:
	cargo build --target wasm32-unknown-unknown --release
	soroban contract optimize --wasm ../target/wasm32-unknown-unknown/release/wowmax_stellar_router.wasm
	@rm ../target/wasm32-unknown-unknown/release/wowmax_stellar_router.wasm

fmt:
	cargo fmt --all --check

clean:
	cargo clean