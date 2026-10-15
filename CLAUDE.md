# Instructions for agents

1. Read `docs/CONTEXT.md` first: it holds the plan, the decisions made so far and the open
   questions. Keep it true when things change, and add dated notes rather than rewriting history.
2. Do not start the app until Steve says so. Until then this repo holds the plan and the pinned
   submodules only.
3. The engines live in the submodules (`upstream/pdfcraft`, `upstream/cadcraft`). Change them in
   their own repos (Azurehawke/pdfcraft branch `revucraft`, Azurehawke/cadcraft), then update the
   pin here; never edit submodule contents from this repo.
4. Follow the ground rules in `docs/CONTEXT.md` §9 and each upstream repo's own `CLAUDE.md` and
   `AGENTS.md` (never panic, clean-room, Rust only, attribution for assets).
5. Commits are authored as Steve; end commit messages with the attribution lines the environment
   asks for.
