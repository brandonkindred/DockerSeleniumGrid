# Docker Selenium Grid

[![Docker](https://img.shields.io/badge/docker-compose-blue?logo=docker)](https://docs.docker.com/compose/)
[![Selenium](https://img.shields.io/badge/selenium-grid-43B02A?logo=selenium)](https://www.selenium.dev/documentation/grid/)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://github.com/brandonkindred/DockerSeleniumGrid/pulls)

> **A Docker Selenium Grid for scalable, parallel cross-browser testing with Chrome, Firefox, and PhantomJS nodes — deployable in seconds with Docker Compose or Docker Swarm.**

Spin up a complete **Selenium Grid in Docker** for **automated browser testing**, **end-to-end (E2E) testing**, **continuous integration (CI/CD) pipelines**, and **parallel test execution** across multiple browsers — all from a single `docker-compose.yml` file.

> **Heads up:** This repo pins `selenium/*:3.7.1`, i.e. **Selenium Grid 3** (the legacy hub/node architecture and `/wd/hub` endpoint). The PhantomJS node uses the official `selenium/node-phantomjs` image — note that the upstream **PhantomJS project has been unmaintained since 2018**; prefer headless Chrome or Firefox for new work.

---

## Table of Contents

- [Why Docker Selenium Grid?](#why-docker-selenium-grid)
- [Features](#features)
- [Architecture](#architecture)
- [Quick Start](#quick-start)
- [Prerequisites](#prerequisites)
- [Usage](#usage)
- [Configuration](#configuration)
- [Scaling Nodes](#scaling-nodes)
- [Connecting Your Tests](#connecting-your-tests)
- [Docker Swarm Deployment](#docker-swarm-deployment)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)
- [Keywords](#keywords)

---

## Why Docker Selenium Grid?

Running **Selenium WebDriver tests** locally is slow and brittle. **Selenium Grid** lets you distribute tests across multiple browsers and machines in parallel — and **Docker** makes the entire grid **reproducible, isolated, and disposable**.

This repository gives you a **ready-to-run Selenium Grid in Docker** so you can:

- Run **parallel Selenium tests** in **Chrome, Firefox, and PhantomJS (headless)** simultaneously.
- Eliminate "works on my machine" issues with **containerized browser environments**.
- Drop the grid into any **CI/CD pipeline** (Jenkins, GitHub Actions, GitLab CI, CircleCI, TeamCity).
- Scale up or down with a single command using **Docker Compose** or **Docker Swarm**.

## Features

- **Single-file stack** — one `docker-compose.yml` defines the hub and all browser nodes.
- **Multi-browser support** — Chrome, Firefox, and PhantomJS (headless) nodes out of the box.
- **Docker Swarm–first** — overlay networking, deploy constraints, and restart policies already configured.
- **Tuned for stability** — session timeouts, restart policies, and `/dev/shm` mount to prevent Chrome crashes.
- **Lightweight** — based on official `selenium/*` images.
- **Horizontally scalable** — replicate browser nodes across a Swarm cluster on demand.
- **CI/CD friendly** — integrates with Jenkins, GitHub Actions, GitLab CI, and any tool that can talk to a Selenium hub.

## Architecture

```
                      ┌──────────────────────┐
   Your test code ───►│  Selenium Hub :4444  │◄─── WebDriver API
                      └──────────┬───────────┘
                                 │ (overlay network)
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
       ┌───────────┐      ┌────────────┐     ┌──────────────┐
       │  Chrome   │      │  Firefox   │     │  PhantomJS   │
       │   Node    │      │    Node    │     │    Node      │
       └───────────┘      └────────────┘     └──────────────┘
```

- **Hub** — exposes port `4444`, routes WebDriver requests to available nodes.
- **Nodes** — each runs a real browser in a container and registers with the hub.
- **Private overlay network** — keeps inter-container traffic isolated from the host.

## Quick Start

Clone, initialise Swarm (required for the overlay network — see caveat below), then deploy:

```bash
git clone https://github.com/brandonkindred/DockerSeleniumGrid.git
cd DockerSeleniumGrid
docker swarm init                                  # required: overlay network
docker stack deploy -c docker-compose.yml selenium
```

Open the **Selenium Grid console** in your browser:

```
http://localhost:4444/grid/console
```

You should see Chrome, Firefox, and PhantomJS nodes registered.

> **Why `docker stack deploy` and not `docker-compose up`?**
> The `private` network in `docker-compose.yml` uses `driver: overlay`, which **requires Docker Swarm mode**. Running `docker-compose up` on a non-Swarm host fails with `This node is not a swarm manager`. If you'd rather use plain Compose on one machine, change the network to `driver: bridge`.
>
> **Single-host port caveat:** The `chrome` and `firefox` services both publish host port `5557` (and PhantomJS uses `5556`). On a **single Docker host** only one of them can claim each port — the second replica will fail. Either remove the `ports:` mapping from the conflicting services (the hub talks to nodes over the internal `private` network, so an exposed host port isn't required), reassign one to a free port (e.g. `5558:5556`), or deploy across multiple Swarm nodes where each replica binds its port on a different machine.

## Prerequisites

- **Docker Engine** 17.06+ (or Docker Desktop on macOS/Windows)
- **Docker Swarm mode enabled** (`docker swarm init`) — required because the `private` network uses the `overlay` driver
- **Docker Compose** 1.16+ — only if you switch the network to `bridge` and run with `docker-compose` instead of `docker stack deploy`

## Usage

Deploy the stack (Swarm):

```bash
docker stack deploy -c docker-compose.yml selenium
```

Inspect services and logs:

```bash
docker stack services selenium
docker service logs -f selenium_hub
```

Tear everything down:

```bash
docker stack rm selenium
```

If you've switched the network to `bridge` for single-host Compose use:

```bash
docker-compose up -d
docker-compose logs -f hub
docker-compose down
```

## Configuration

Hub behaviour is tuned via environment variables in `docker-compose.yml`:

| Variable                  | Default  | Unit    | Description                                              |
| ------------------------- | -------- | ------- | -------------------------------------------------------- |
| `GRID_BROWSER_TIMEOUT`    | `120000` | seconds | How long a node waits for a browser to respond before killing the session. |
| `GRID_TIMEOUT`            | `120000` | seconds | How long the hub waits before considering a session abandoned. |
| `GRID_MAX_SESSION`        | `100`    | count   | Maximum concurrent sessions the hub will accept.         |
| `GRID_CLEAN_UP_CYCLE`     | `5000`   | ms      | How often the hub cleans up stale sessions.              |

> **Note:** `GRID_BROWSER_TIMEOUT` and `GRID_TIMEOUT` are passed to Selenium Grid 3's `-browserTimeout` / `-timeout` flags, which expect **seconds**. The defaults shown above (`120000` seconds ≈ 33 hours) are effectively "no timeout"; lower them if you need stuck sessions reaped sooner.

Each node mounts `/dev/shm` to give Chrome/Firefox enough shared memory for stable rendering — a common fix for the dreaded "tab crashed" error.

## Scaling Nodes

In **Docker Swarm**, scale each browser service across the cluster:

```bash
docker service scale selenium_chrome=4
docker service scale selenium_firefox=2
```

(Replace `selenium_` with whatever stack name you used in `docker stack deploy`.) Because each replica can land on a different Swarm node, the host port `5557` (or `5556` for PhantomJS) does not conflict.

With plain **Docker Compose on a single host**, `--scale chrome=4` will fail because all replicas try to bind the same host port. To scale locally, first remove the `ports:` block from the browser services in `docker-compose.yml` — the hub reaches nodes over the internal `private` network and doesn't need them published — then:

```bash
docker-compose up -d --scale chrome=4 --scale firefox=2
```

## Connecting Your Tests

Point your **WebDriver** client at the hub URL `http://localhost:4444/wd/hub`.

### Python (selenium)

```python
from selenium import webdriver
from selenium.webdriver.common.desired_capabilities import DesiredCapabilities

driver = webdriver.Remote(
    command_executor="http://localhost:4444/wd/hub",
    desired_capabilities=DesiredCapabilities.CHROME,
)
driver.get("https://example.com")
print(driver.title)
driver.quit()
```

### Java (JUnit / TestNG)

```java
DesiredCapabilities caps = DesiredCapabilities.firefox();
WebDriver driver = new RemoteWebDriver(
    new URL("http://localhost:4444/wd/hub"), caps);
driver.get("https://example.com");
driver.quit();
```

### JavaScript (WebdriverIO)

```js
const { remote } = require('webdriverio');

const browser = await remote({
  hostname: 'localhost',
  port: 4444,
  path: '/wd/hub',
  capabilities: { browserName: 'chrome' }
});
await browser.url('https://example.com');
await browser.deleteSession();
```

## Docker Swarm Deployment

The compose file is already Swarm-aware (`deploy.mode`, `placement.constraints`, `restart_policy`). Deploy as a stack:

```bash
docker stack deploy -c docker-compose.yml selenium
docker stack services selenium
```

The hub is pinned to a manager node; browser nodes can run anywhere in the cluster, joined together by the `private` overlay network.

## Troubleshooting

**Nodes not registering with the hub?**
Check that `HUB_PORT_4444_TCP_ADDR=hub` resolves on the shared network. In Swarm, ensure the overlay network is attached to every service.

**Chrome crashes mid-test?**
Make sure `/dev/shm` is mounted (default in this repo). On hosts with low shared memory, increase it with `--shm-size=2g`.

**`session not created` errors?**
Bump `GRID_MAX_SESSION` and scale up nodes — you're probably saturating the grid.

**Stale sessions?**
Lower `GRID_TIMEOUT` so the hub reclaims dead sessions faster.

## Contributing

Issues and pull requests are welcome! If you find a bug or have a feature request, please [open an issue](https://github.com/brandonkindred/DockerSeleniumGrid/issues).

## License

No `LICENSE` file is currently checked in. Until one is added, treat the code as "all rights reserved" by the repository owner. If you plan to depend on it, please [open an issue](https://github.com/brandonkindred/DockerSeleniumGrid/issues) asking for an explicit license (MIT or Apache-2.0 are common choices for projects like this).

## Keywords

`selenium grid docker` · `docker selenium grid` · `selenium grid docker compose` · `selenium docker swarm` · `parallel selenium testing` · `headless browser testing docker` · `selenium chrome docker` · `selenium firefox docker` · `phantomjs docker` · `cross browser testing docker` · `webdriver docker` · `ci cd selenium grid` · `containerized browser testing` · `automated browser testing`
