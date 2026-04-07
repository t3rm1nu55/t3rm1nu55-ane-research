# Experiments

Small probes against private ANE/AMX APIs, used to validate reverse-engineering findings before they graduate to `t3rm1nu55-monitorplus`.

**Status:** empty. Nothing has matured from tracking into experimentation yet.

## Guidelines when this directory becomes active

- Each experiment is a small self-contained Rust binary (or Swift probe if that's the only path)
- Include a `README.md` in the experiment's subdirectory explaining: what it probes, what you expect to see, what you actually saw, and whether it worked
- Link back to the `journal.md` entry that triggered the experiment
- Do NOT vendor reverse-engineered headers — declare `extern "C"` signatures manually (same IP-safe policy as the main repo)
- Experiments that work and produce something actionable should open a corresponding issue on `t3rm1nu55-monitorplus`
