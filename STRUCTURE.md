# Repository structure

Source code is private. Available on request.

```
.
├── .dockerignore
├── .gitignore
├── Dockerfile
├── LICENSE
├── README.md
├── docs
│   └── ARCHITECTURE.md
├── lab
│   ├── campus.clab.yml
│   ├── topology.cr.yml
│   └── watcher.yaml
├── netbox
│   ├── postgres.yaml
│   ├── valkey.yaml
│   └── values.yaml
├── pyproject.toml
├── scripts
│   ├── lab.sh
│   └── netbox.sh
├── seed
│   ├── cabling.yml
│   ├── dcim.yml
│   ├── extras.yml
│   └── ipam.yml
├── src
│   └── intentlab
│       ├── __init__.py
│       ├── inject_drift.py
│       ├── prove_commit.py
│       ├── prove_rollback.py
│       ├── push.py
│       ├── render_topology.py
│       ├── seed.py
│       ├── verify.py
│       └── watch.py
├── templates
│   └── srlinux
│       ├── access.j2
│       ├── aggregation.j2
│       ├── base.j2
│       └── core.j2
└── tests
    ├── test_inject_drift.py
    ├── test_prove_rollback.py
    ├── test_push.py
    ├── test_render_topology.py
    ├── test_seed.py
    ├── test_templates.py
    ├── test_verify.py
    └── test_watch.py
```
