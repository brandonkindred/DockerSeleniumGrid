# Docker Selenium Grid

[![Docker](https://img.shields.io/badge/docker-compose-blue?logo=docker)](https://docs.docker.com/compose/)
[![Selenium](https://img.shields.io/badge/selenium-grid-43B02A?logo=selenium)](https://www.selenium.dev/documentation/grid/)
[![License: MIT](https://img.shields.io/badge/license-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](https://github.com/brandonkindred/DockerSeleniumGrid/pulls)

> **A production-ready Docker Selenium Grid for scalable, parallel cross-browser testing with Chrome, Firefox, and PhantomJS nodes — deployable in seconds with Docker Compose or Docker Swarm.**

Spin up a complete **Selenium Grid in Docker** for **automated browser testing**, **end-to-end (E2E) testing**, **continuous integration (CI/CD) pipelines**, and **parallel test execution** across multiple browsers — all from a single `docker-compose.yml` file.

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

- **One-command setup** — `docker-compose up` and your grid is online.
- **Multi-browser support** — Chrome, Firefox, and PhantomJS (headless) nodes out of the box.
- **Docker Swarm-ready** — overlay networking and deploy constraints already configured.
- **Tuned for stability** — session timeouts, restart policies, and `/dev/shm` mount to prevent Chrome crashes.
- **Lightweight** — based on official `selenium/*` images.
- **Easy to scale** — add or remove browser nodes on demand.
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

Clone, start, and run tests in under a minute:

```bash
git clone https://github.com/brandonkindred/DockerSeleniumGrid.git
cd DockerSeleniumGrid
docker-compose up -d
```

Open the **Selenium Grid console** in your browser:

```
http://localhost:4444/grid/console
```

You should see one Chrome, one Firefox, and one PhantomJS node registered.

## Prerequisites

- **Docker Engine** 17.06+ (or Docker Desktop on macOS/Windows)
- **Docker Compose** 1.16+
- For Swarm deployments: a Swarm-initialised cluster (`docker swarm init`)

## Usage

Start the grid in detached mode:

```bash
docker-compose up -d
```

Tail the hub logs:

```bash
docker-compose logs -f hub
```

Stop and remove everything:

```bash
docker-compose down
```

## Configuration

Hub behaviour is tuned via environment variables in `docker-compose.yml`:

| Variable                  | Default  | Description                                              |
| ------------------------- | -------- | -------------------------------------------------------- |
| `GRID_BROWSER_TIMEOUT`    | `120000` | Time (ms) a node waits for browser command before killing it. |
| `GRID_TIMEOUT`            | `120000` | Time (ms) the hub waits before reclaiming an idle session. |
| `GRID_MAX_SESSION`        | `100`    | Max concurrent sessions across the grid.                 |
| `GRID_CLEAN_UP_CYCLE`     | `5000`   | How often (ms) the hub cleans up stale sessions.         |

Each node mounts `/dev/shm` to give Chrome/Firefox enough shared memory for stable rendering — a common fix for the dreaded "tab crashed" error.

## Scaling Nodes

Add more browser instances on demand with Compose:

```bash
docker-compose up -d --scale chrome=4 --scale firefox=2
```

In Docker Swarm:

```bash
docker service scale dockerseleniumgrid_chrome=4
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

Released under the [MIT License](https://opensource.org/licenses/MIT).

## Keywords

`selenium grid docker` · `docker selenium grid` · `selenium grid docker compose` · `selenium docker swarm` · `parallel selenium testing` · `headless browser testing docker` · `selenium chrome docker` · `selenium firefox docker` · `phantomjs docker` · `cross browser testing docker` · `webdriver docker` · `ci cd selenium grid` · `containerized browser testing` · `automated browser testing`
