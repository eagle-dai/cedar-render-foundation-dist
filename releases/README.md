# Releases

Each Cedar Render Foundation release has its own directory.

```
releases/
├── README.md
├── v0.01/
│   └── README.md
└── v0.1.0/
    └── README.md
```

Each release directory should record at least:

- Cedar release version
- package filename
- base CEF version
- Chromium version
- platform / architecture
- custom capabilities
- artifact SHA-256
- source / build information
- compatibility notes

## Versions

- [v0.1.0](v0.1.0/README.md) — clean Cedar-built CEF baseline, no custom export
- [v0.01](v0.01/README.md) — adds the `cef_request_raw_snapshot` export
