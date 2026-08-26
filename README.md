# Docker Practice Lab

A hands-on Docker learning path from first container to advanced operations, designed for an Ubuntu VirtualBox lab.

## Lab environment

- Ubuntu Linux guest in VirtualBox
- Docker Engine and Docker Compose v2
- NAT adapter for internet access
- Host-only adapter for SSH and published lab services
- Exercises progress from basic container lifecycle to security, registries, BuildKit, remote contexts, Swarm, and capstone projects

## Start here

1. Open the [interactive HTML manual](docs/Docker_Lab_Practice_Manual_Basic_to_Expert.html) or the [Markdown manual](DOCKER-LAB-PRACTICE-MANUAL.md).
2. Confirm the prerequisites and take the recommended VirtualBox snapshot.
3. Begin with Module 1 and run each command block individually.
4. Complete each **Pass Check** before moving to the next lab.
5. Use the cleanup command only after completing the exercise.

## Contents

| Stage | Modules | Focus |
|---|---:|---|
| Foundation | 1–3 | Containers, processes, logs, images, layers, cleanup |
| Builder | 4–7 | Volumes, bind mounts, networking, Dockerfiles, Compose |
| Operator | 8–10 | Troubleshooting, health checks, resource controls, security, daemon operations |
| Expert | 11–13 | Registries, BuildKit, remote contexts, Swarm, capstone projects |

The repository includes browser, Markdown, and Word editions:

- [Docker lab manual — Interactive HTML](docs/Docker_Lab_Practice_Manual_Basic_to_Expert.html)
- [Docker lab manual — Markdown](DOCKER-LAB-PRACTICE-MANUAL.md)
- [Docker lab manual — Word](docs/Docker_Lab_Practice_Manual_Basic_to_Expert.docx)

## Safety notes

- Use the included credentials only inside an isolated disposable training environment.
- Never commit real passwords, tokens, private keys, or internal certificates.
- Do not expose the Docker daemon on unauthenticated TCP port `2375`.
- Review Docker objects before running prune or volume-removal commands.
- Do not bypass TLS verification when working behind an inspecting firewall; trust only an approved CA.

## Progress checklist

- [ ] Modules 1–3: container and image fundamentals
- [ ] Modules 4–5: storage and networking
- [ ] Module 6: Dockerfiles and image builds
- [ ] Module 7: Docker Compose
- [ ] Modules 8–9: operations and security
- [ ] Modules 10–11: daemon administration and registries
- [ ] Module 12: orchestration concepts
- [ ] Module 13: capstones and final practical exam

## Projects

- [Unix Username Generator](projects/unix-username-generator/) — a containerized Nginx web app that creates lowercase Unix-style usernames and copies them to the clipboard.

## License

Educational lab material. Add a formal license before accepting external contributions or reuse.
