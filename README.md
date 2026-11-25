# System Design

A collection of in-depth system design blogs. Each directory covers one real-world system (URL shortener, Redis, etc.), with a written deep-dive and an architecture diagram.

## Repository Structure

```
system-design/
├── README.md
├── bitly/
│   ├── architecture.md      # In-depth blog on the architecture
│   └── architecture.png     # Architecture diagram (.png or .jpeg)
├── redis/
│   ├── architecture.md
│   └── architecture.png
└── <system-name>/
    ├── architecture.md
    └── architecture.png
```

Every system directory contains exactly two things:

| File | Purpose |
|------|---------|
| `architecture.md` | A blog-style, in-depth explanation of the system's architecture |
| `architecture.png` / `architecture.jpeg` | The architecture diagram, referenced from the blog |

## Systems

| System | Blog | Diagram |
|--------|------|---------|
| Bitly (URL Shortener) | [architecture.md](./bitly/architecture.md) | [diagram](./bitly/architecture.png) |
| Redis | [architecture.md](./redis/architecture.md) | [diagram](./redis/architecture.png) |

> Keep this table updated whenever a new system is added.

## What Each Blog Covers

Each `architecture.md` aims to answer:

1. **Problem and requirements**: what the system does, functional and non-functional requirements
2. **Scale estimates**: traffic, storage, read/write ratio
3. **High-level architecture**: components and how they interact, with the diagram
4. **Deep dives**: data model, storage choice, caching, partitioning, replication, consistency
5. **Trade-offs**: what was chosen, what was rejected, and why
6. **Failure modes**: bottlenecks, single points of failure, and how they are handled
7. **References**: papers, engineering blogs, and docs used

## Adding a New System

1. Create a directory named after the system in lowercase, using hyphens for spaces (e.g. `url-shortener`, `rate-limiter`).
2. Add `architecture.md` with the blog content.
3. Add `architecture.png` (or `.jpeg`) with the diagram.
4. Embed the diagram at the top of the blog:
   ```md
   ![Architecture](./architecture.png)
   ```
5. Add a row to the **Systems** table above.

## Conventions

- One system per directory, one blog per system.
- Diagrams must be readable at full width: use clear labels and a consistent legend.
- Cite sources for any numbers or claims taken from external material.
- Write in your own words; link to original sources instead of copying them.

## Contributing

Suggestions, corrections, and new systems are welcome. Open an issue to propose a system, or submit a pull request following the structure above.

## License

MIT
