# Repository structure

Source code is private. Available on request.

```
.
├── .github
│   ├── publish
│   │   └── LICENSE
│   └── workflows
│       └── publish-docs.yml
├── .gitignore
├── CLAUDE.md
├── LICENSE
├── README.md
├── docs
│   ├── ARCHITECTURE.md
│   └── OPERATIONS.md
├── lab
│   ├── campus.clab.yml
│   └── topology.cr.yml
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
│       ├── prove_commit.py
│       ├── push.py
│       ├── render_topology.py
│       ├── seed.py
│       └── verify.py
├── templates
│   └── srlinux
│       ├── access.j2
│       ├── aggregation.j2
│       ├── base.j2
│       └── core.j2
└── tests
    ├── test_push.py
    ├── test_render_topology.py
    ├── test_seed.py
    ├── test_templates.py
    └── test_verify.py
```
