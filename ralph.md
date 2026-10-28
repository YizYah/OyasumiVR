# OyasumiVR

OyasumiVR is a Windows desktop app for VRChat users, built with Tauri. The Angular frontend
(`src-ui`) talks to a Rust Tauri backend (`src-core`), which talks over gRPC to a C# SteamVR
overlay sidecar (`src-overlay-sidecar`, with its own Angular UI in `src-overlay-ui`) and to Rust
sidecars for elevated actions and Steam Frame support.

## Setup

Install dependencies:

    npm install

## Verification

Full native checks (Rust and C# components, including format/lint/build/test for each) need
Windows and its toolchains, and are not verifiable here. The commands below are the portable,
cross-platform subset: ESLint/Prettier/translation/readme checks, the two Angular (`ng build`)
frontend builds, and the Vitest suite for the web code.

<!-- RALPH_CHECK_COMMANDS
npm run check:format:web
npm run check:lint:web
npm run check:translations
npm run check:generated:readmes
RALPH_CHECK_COMMANDS_END -->

<!-- RALPH_BUILD_COMMANDS
npm run check:build:ui
npm run check:build:overlay-ui
RALPH_BUILD_COMMANDS_END -->

<!-- RALPH_REGRESSION_COMMANDS
npm run check:test:web
RALPH_REGRESSION_COMMANDS_END -->
