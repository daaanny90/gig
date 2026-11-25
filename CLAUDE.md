# gig

Notes for Claude Code sessions that develop this repository.

- `bin/gig` is the whole tool: one Python file, standard library only. Its module docstring describes open, close, init and gig.json: read it first. `bin/roger` links to it for the Herdr mods that call that name.
- Comments explain why, not what. Code, comments, docs and commit messages are in English. Commits: `type(gig): subject`, subject only.
- This repository is public. Never commit personal or client data: no real ticket keys or prefixes, client or company names, personal paths, hosts or IPs, not even as examples, in docstrings or in commit messages. Examples use neutral names (`SHOP-1234`, `XSHOP-1`, "the shop").
- `herdr/roger/main.js` parses some of gig's output (the lines marked "parsed by herdr/roger/main.js"): change both sides together.
- There are no tests yet. Check a change in a throwaway client folder (a few `git init` repositories and a gig.json) and, for the plugin step of `gig init`, a throwaway marketplace you remove afterwards: never in a real client's folder.
