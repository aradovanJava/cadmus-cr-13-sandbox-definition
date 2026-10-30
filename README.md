# CADMUS-CR-13 · Secure Coding .NET/C# — sandbox definition

Sandbox definition for the CADMUS CyberRangeᶜᶻ platform (cr.cybersecurity-training.cti.gr) used by
the CADMUS bootcamp **BTCMP-02 Secure Coding**, Cyber Range course
**CADMUS-CR-13 Secure Coding .NET/C#**.

It implements section 2 *Training Scenario Architecture* of the
*CADMUS Cyber Range Training Scenario Framework* for CR-13: a Windows 11
developer workstation plus two Docker hosts carrying the SAST and DAST tools.
The application under test is
[`aradovanJava/cadmus-vulnshop-dotnet-cr`](https://github.com/aradovanJava/cadmus-vulnshop-dotnet-cr),
the participant build of the deliberately vulnerable CR-13 reference
application (ASP.NET Core minimal API, .NET 10, OWASP Top 10:2025).

The design is that of the tested Java sandbox,
[`cadmus-cr-14-sandbox-definition`](https://github.com/aradovanJava/cadmus-cr-14-sandbox-definition);
only the SC1 software stack and the application differ.

> **Training system.** The application under test contains a working OS
> command injection (remote code execution) path. It is meant to run only
> inside this sandbox. See `SAFETY.md` in the application repository.

## Topology

```
                 internet-connection (WAN)
                           |
                        router  10.10.10.1
                           |
   dev-lan 10.10.10.0/24 --+-----------------------+----------------------+
                           |                       |                      |
                 sc1  10.10.10.10         sq1  10.10.10.20        owz1  10.10.10.30
                 VM1  Windows 11          VM3  SonarQube          VM2  OWASP ZAP
                 trainee workstation      (Docker, SAST)          (Docker, DAST)
```

| Host | Framework ID | Image | Flavor | Role |
|---|---|---|---|---|
| `sc1` | VM1 · SC1 | `windows-11-x86_64` | `c4_r8_d60` | Developer workstation, RDP access |
| `sq1` | VM3 · SQ1 | `ubuntu-noble-x86_64` | `c4_r8_d40` | SonarQube Community Build in Docker |
| `owz1` | VM2 · OWZ1 | `ubuntu-noble-x86_64` | `c2_r4_d40` | OWASP ZAP in Docker (desktop UI via Webswing) |
| `router` | — | `debian-12-x86_64` | `standard.small` | Gateway |

### Software on SC1 (framework table 2.1, VM1)

.NET 10 SDK and .NET 9 SDK (side by side), SonarScanner for .NET
(`dotnet sonarscanner`, machine-wide), Visual Studio Code with the C# and
C# Dev Kit extensions, Git, Node.js LTS, Python 3, SQLMap, Snyk CLI, Postman,
and Apache Maven with Oracle JDK 25. The application is cloned to
`C:\Lab\cadmus-vulnshop-dotnet-cr` and built once; a README and browser
shortcuts are placed on the desktop, with an environment check at
`C:\Lab\verify-cr-env.ps1` and a PATH helper at `C:\Lab\fix-path.ps1`.

## Access

| | |
|---|---|
| Account on all three machines | `user` / `Password123` — the platform's standard sandbox account (administrator on SC1 with RDP, sudo + docker on SQ1/OWZ1) |
| SonarQube | `http://localhost:9000` on SC1 (forwarded to SQ1); first login `admin` / `admin`, then a forced password change |
| OWASP ZAP desktop | `http://owz1:8080/zap/` from SC1; proxy/API on `owz1:8090` |
| Application from SC1 | `http://127.0.0.1:5080/swagger` after `tools\run-app.ps1` |
| Application as ZAP sees it | `http://sc1:5080`, OpenAPI at `http://sc1:5080/swagger/v1/swagger.json` |

Credentials and every other tunable are in `provisioning/group_vars/all.yml`.

## Design decisions

1. **SQ1 and OWZ1 run Ubuntu 24.04, not Windows 11.** Table 2.1 lists Windows 11
   for the two Docker hosts. Docker on Windows needs WSL 2 / Hyper-V, i.e.
   nested virtualisation, which OpenStack sandboxes generally do not offer.
   The tools, ports and roles are exactly those of the framework; only the
   host OS of the two containers differs.
2. **.NET 10 and .NET 9.** The framework lists .NET 9; the reference
   application targets .NET 10 (LTS). Both SDKs are installed with Microsoft's
   `dotnet-install` script into `C:\Program Files\dotnet`.
3. **Apache Maven is installed** because the framework's VM1 list includes it
   (inherited from the Java variant of the module). It plays no part in the
   .NET levels.
4. **The application stays bound to `127.0.0.1`.** Because the framework puts
   ZAP on a separate machine, SC1 forwards `10.10.10.10:5080` to
   `127.0.0.1:5080` (`netsh interface portproxy`) and a Windows Firewall rule
   admits that port from OWZ1 (`10.10.10.30`) only.
5. **SonarQube is reachable as `http://localhost:9000` on SC1** through a second
   port forward, so `tools\scan-sonar.ps1` in the application works unchanged.
6. **ZAP needs the OpenAPI definition.** The application is an API with no HTML
   pages to crawl, so level A2 has trainees import
   `http://sc1:5080/swagger/v1/swagger.json` into ZAP before the active scan.
7. **Tools are started for the trainee by default** (`cr13_autostart_tools: true`).
   Set it to `false` if levels A1/A3 should include starting them.
8. **SonarQube is pulled from the ECR Public mirror of the Docker Official
   Image first**, falling back to Docker Hub, because a pool of sandboxes
   pulling anonymously from Docker Hub runs into its rate limit.
9. **SC1 restarts as the last provisioning step.** In CR-14 a desktop session
   that existed during provisioning kept an older PATH, so tools that were
   installed looked missing. The restart gives the first trainee session the
   final PATH; `C:\Lab\fix-path.ps1` repairs a stale window if it ever happens.
10. **Level E1 (GitHub Actions)** runs on GitHub, outside the sandbox. Trainees
    fork `cadmus-vulnshop-dotnet-cr` to their own account and extend
    `.github/workflows/01-starter-build.yml` in the fork.

## Mapping to the training levels

| Level | Machine(s) | What the sandbox provides |
|---|---|---|
| 0 · Access | SC1 | RDP account, desktop README, environment check |
| A1/A2 · DAST | SC1 → OWZ1 → SC1 | ZAP desktop at `owz1:8080/zap/`, target `http://sc1:5080` with its OpenAPI definition |
| A3/A4 · SAST | SC1 → SQ1 | SonarQube at `localhost:9000`, SonarScanner for .NET, Snyk |
| C1/C2 · Research and fixes | SC1 | VS Code with C# Dev Kit, source in `C:\Lab\cadmus-vulnshop-dotnet-cr` |
| D1 · Patch and redeploy | SC1 | Git, .NET SDK (`dotnet publish`), local run and rescan |
| E1 · CI/CD | GitHub | Trainee's fork with the starter workflow |

## Using it on the platform

1. **Sandboxes → Definition → Create**
   Git URL `https://github.com/aradovanJava/cadmus-cr-13-sandbox-definition.git`,
   revision = the full SHA of the commit to use (not `main`, see the notes below).
2. **Sandboxes → Pool → Create** from that definition; size = number of trainees.
3. **Allocate.** SC1 dominates the time (.NET SDKs, VS Code, JDK, Chocolatey):
   count on 30–60 minutes for a fresh allocation.
4. Link the pool to the training instance built from the CR-13 training definition.

## Repository layout

```
topology.yml                              hosts, router, network, addressing
provisioning/
├── playbook.yml                          SQ1, OWZ1, then SC1
├── requirements.yml                      roles and collections
├── group_vars/all.yml                    credentials, IPs, images, package list
└── templates/
    ├── sq1-docker-compose.yml.j2         SonarQube
    ├── owz1-docker-compose.yml.j2        OWASP ZAP (Webswing)
    ├── sc1-README.txt.j2                 desktop README on SC1
    ├── verify-cr-env.ps1.j2              environment check on SC1
    └── fix-path.ps1.j2                   PATH helper on SC1
tools/
├── verify-cr-env.ps1                     rendered copy of the environment check
└── fix-path.ps1                          rendered copy of the PATH helper
```

The `tools/` copies can be fetched straight onto SC1 if the on-disk ones are
ever edited by accident:

```powershell
irm https://raw.githubusercontent.com/aradovanJava/cadmus-cr-13-sandbox-definition/main/tools/verify-cr-env.ps1 -OutFile C:\Lab\verify-cr-env.ps1
```

## Notes for instructors

- **SC1 is managed over SSH** (`mgmt_protocol: ssh`). The Windows 11 image on
  cr.cybersecurity-training.cti.gr does not open the WinRM listener; over SSH
  it answers after a first boot of about five minutes. If Windows 11 ever stops
  working on the platform, `windows-server-2025-x86_64` is a drop-in
  replacement (tested for CR-14).
- **Pin the revision.** The platform reads `topology.yml` when the sandbox
  definition is created and does not pick up later commits on a branch.
  Create the definition with a commit SHA as the revision, and create a new
  definition (and pool) after changing the topology or provisioning.
- **The environment check cannot report a false MISSING.** It decides OK or
  MISSING with `Get-Command` against the machine PATH and only *shows* the
  version; a tool installed but invisible to an old window is reported in
  yellow with a pointer to `fix-path.ps1`.
- Snyk requires a free account: trainees run `snyk auth` once.
- The instructor copy of the application (with the `>>> FIX` remediation notes
  and the reference E1 pipeline `02-secure-pipeline.yml`) is deliberately not
  in any public repository.

---

Co-funded by the European Union (Digital Europe Programme, grant agreement
101190006 — CADMUS). Views and opinions expressed are those of the author(s)
only and do not necessarily reflect those of the European Union or HaDEA.
