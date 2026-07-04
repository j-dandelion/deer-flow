# Troubleshooting Guide

This document lists common issues encountered during DeerFlow smoke testing and how to resolve them.

## Code Update Issues

### Issue: `git pull` Fails with a Merge Conflict Warning

**Symptoms**:
```
error: Your local changes to the following files would be overwritten by merge
```

**Solutions**:
1. Option A: Commit local changes first
   ```bash
   git add .
   git commit -m "Save local changes"
   git pull origin main
   ```

2. Option B: Stash local changes
   ```bash
   git stash
   git pull origin main
   git stash pop  # Restore changes later if needed
   ```

3. Option C: Discard local changes (use with caution)
   ```bash
   git reset --hard HEAD
   git pull origin main
   ```

---

## Local Mode Environment Issues

### Issue: Node.js Version Is Too Old

**Symptoms**:
```
Node.js version is too old. Requires 22+, got x.x.x
```

**Solutions**:
1. Install or upgrade Node.js with nvm:
   ```bash
   nvm install 22
   nvm use 22
   ```

2. Or download and install it from the official website: https://nodejs.org/

3. Verify the version:
   ```bash
   node --version
   ```

---

### Issue: pnpm Is Not Installed

**Symptoms**:
```
command not found: pnpm
```

**Solutions**:
1. Install pnpm with npm:
   ```bash
   npm install -g pnpm
   ```

2. Or use the official installation script:
   ```bash
   curl -fsSL https://get.pnpm.io/install.sh | sh -
   ```

3. Verify the installation:
   ```bash
   pnpm --version
   ```

---

### Issue: uv Is Not Installed

**Symptoms**:
```
command not found: uv
```

**Solutions**:
1. Use the official installation script:
   ```bash
   curl -LsSf https://astral.sh/uv/install.sh | sh
   ```

2. macOS users can also install it with Homebrew:
   ```bash
   brew install uv
   ```

3. Verify the installation:
   ```bash
   uv --version
   ```

---

### Issue: nginx Is Not Installed

**Symptoms**:
```
command not found: nginx
```

**Solutions**:
1. macOS (Homebrew):
   ```bash
   brew install nginx
   ```

2. Ubuntu/Debian:
   ```bash
   sudo apt update
   sudo apt install nginx
   ```

3. CentOS/RHEL:
   ```bash
   sudo yum install nginx
   ```

4. Verify the installation:
   ```bash
   nginx -v
   ```

---

### Issue: Port Is Already in Use

**Symptoms**:
```
Error: listen EADDRINUSE: address already in use :::2026
```

**Solutions**:
1. Find the process using the port:
   ```bash
   lsof -i :2026  # macOS/Linux
   netstat -ano | findstr :2026  # Windows
   ```

2. Stop that process:
   ```bash
   kill -9 <PID>  # macOS/Linux
   taskkill /PID <PID> /F  # Windows
   ```

3. Or stop DeerFlow services first:
   ```bash
   make stop
   ```

---

## Local Mode Dependency Installation Issues

### Issue: `make install` Fails Due to Network Timeout

**Symptoms**:
Network timeouts or connection failures occur during dependency installation.

**Solutions**:
1. Configure pnpm to use a mirror registry:
   ```bash
   pnpm config set registry https://registry.npmmirror.com
   ```

2. Configure uv to use a mirror registry:
   ```bash
   uv pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
   ```

3. Retry the installation:
   ```bash
   make install
   ```

---

### Issue: Python Dependency Installation Fails

**Symptoms**:
Errors occur during `uv sync`.

**Solutions**:
1. Clean the uv cache:
   ```bash
   cd backend
   uv cache clean
   ```

2. Resync dependencies:
   ```bash
   cd backend
   uv sync
   ```

3. View detailed error logs:
   ```bash
   cd backend
   uv sync --verbose
   ```

---

### Issue: Frontend Dependency Installation Fails

**Symptoms**:
Errors occur during `pnpm install`.

**Solutions**:
1. Clean the pnpm cache:
   ```bash
   cd frontend
   pnpm store prune
   ```

2. Remove node_modules and the lock file:
   ```bash
   cd frontend
   rm -rf node_modules pnpm-lock.yaml
   ```

3. Reinstall:
   ```bash
   cd frontend
   pnpm install
   ```

---

## Local Mode Service Startup Issues

### Issue: Services Exit Immediately After Startup

**Symptoms**:
Processes exit quickly after running `make dev-daemon`.

**Solutions**:
1. Check log files:
   ```bash
   tail -f logs/gateway.log
   tail -f logs/frontend.log
   tail -f logs/nginx.log
   ```

2. Check whether config.yaml is configured correctly
3. Check environment variables in the .env file
4. Confirm that required ports are not occupied
5. Stop all services and restart:
   ```bash
   make stop
   make dev-daemon
   ```

---

### Issue: Nginx Fails to Start Because Temp Directories Do Not Exist

**Status: RESOLVED in-repo (nginx.local.conf already pins all five temp
paths; serve.sh passes `-e logs/nginx-error.log` for the bootstrap log).**

Kept here as historical context in case the config regresses.

**Original symptoms** (Fedora/RHEL package builds of nginx):
```
nginx: [alert] could not open error log file: open() "/var/log/nginx/error.log" failed (13: Permission denied)
nginx: [emerg] mkdir() "/var/lib/nginx/tmp/client_body" failed (13: Permission denied)
```
(macOS homebrew reports the same shape but with different paths:
`/opt/homebrew/var/run/nginx/client_body_temp` and friends.)

**Two root causes, both fixed:**

1. **Bootstrap error log.** nginx tries to open its compiled-in default
   error log (`/var/log/nginx/error.log` on Fedora, owned root:root
   drwx--x--x) BEFORE parsing the config — so the config-level
   `error_log` directive never gets a chance to redirect. Fix: pass
   `-e logs/nginx-error.log` on the nginx command line (serve.sh
   already does this). `-p $REPO_ROOT` does NOT help because the
   default error-log path is absolute.

2. **Temp directories.** nginx falls back to its compiled-in
   `/var/lib/nginx/tmp/*` paths (Fedora: owned root:nginx drwxrwx---,
   unwritable by jared). Fix: add the five `*_temp_path` directives
   at the top of the `http {}` block in `docker/nginx/nginx.local.conf`
   (already present). `serve.sh` creates the `temp/` subdirectories
   at startup.

If you see either error after a clean `make dev`, the config has
regressed — re-check both directives above are still in place.

---

### Issue: Nginx Fails to Start (General)

**Symptoms**:
The nginx process fails to start or reports an error.

**Solutions**:
1. Check the nginx configuration:
   ```bash
   nginx -t -c docker/nginx/nginx.local.conf -p .
   ```

2. Check nginx logs:
   ```bash
   tail -f logs/nginx.log
   ```

3. Ensure no other nginx process is running:
   ```bash
   ps aux | grep nginx
   ```

4. If needed, stop existing nginx processes:
   ```bash
   pkill -9 nginx
   ```

---

### Issue: Frontend Compilation Fails

**Symptoms**:
Compilation errors appear in `frontend.log`.

**Solutions**:
1. Check frontend logs:
   ```bash
   tail -f logs/frontend.log
   ```

2. Check whether Node.js version is 22+
3. Reinstall frontend dependencies:
   ```bash
   cd frontend
   rm -rf node_modules .next
   pnpm install
   ```

4. Restart services:
   ```bash
   make stop
   make dev-daemon
   ```

---

### Issue: Gateway Fails to Start

**Symptoms**:
Errors appear in `gateway.log`.

**Solutions**:
1. Check gateway logs:
   ```bash
   tail -f logs/gateway.log
   ```

2. Check whether config.yaml exists and has valid formatting
3. Check whether Python dependencies are complete:
   ```bash
   cd backend
   uv sync
   ```

4. Confirm that the Gateway process is running normally.

---

## Docker-Related Issues

### Issue: Docker Commands Cannot Run

**Symptoms**:
```
Cannot connect to the Docker daemon
```

**Solutions**:
1. Confirm that Docker Desktop is running
2. macOS: check whether the Docker icon appears in the top menu bar
3. Linux: run `sudo systemctl start docker`
4. Run `docker info` again to verify

---

### Issue: `make docker-init` Fails to Pull the Image

**Symptoms**:
```
Error pulling image: connection refused
```

**Solutions**:
1. Check network connectivity
2. Configure a Docker image mirror if needed
3. Check whether a proxy is required
4. Switch to local installation mode if necessary (recommended)

---

## Configuration File Issues

### Issue: config.yaml Is Missing or Invalid

**Symptoms**:
```
Error: could not read config.yaml
```

**Solutions**:
1. Regenerate the configuration file:
   ```bash
   make config
   ```

2. Check YAML syntax:
   - Make sure indentation is correct (use 2 spaces)
   - Make sure there are no tab characters
   - Check that there is a space after each colon

3. Use a YAML validation tool to check the format

---

### Issue: Model API Key Is Not Configured

**Symptoms**:
After services start, API requests fail with authentication errors.

**Solutions**:
1. Edit the .env file and add the API key:
   ```bash
   OPENAI_API_KEY=your-actual-api-key-here
   ```

2. Restart services (local mode):
   ```bash
   make stop
   make dev-daemon
   ```

3. Restart services (Docker mode):
   ```bash
   make docker-stop
   make docker-start
   ```

4. Confirm that the model configuration in config.yaml references the environment variable correctly

---

## Service Health Check Issues

### Issue: Frontend Page Is Not Accessible

**Symptoms**:
The browser shows a connection failure when visiting http://localhost:2026.

**Solutions** (local mode):
1. Confirm that the nginx process is running:
   ```bash
   ps aux | grep nginx
   ```

2. Check nginx logs:
   ```bash
   tail -f logs/nginx.log
   ```

3. Check firewall settings

**Solutions** (Docker mode):
1. Confirm that the nginx container is running:
   ```bash
   docker ps | grep nginx
   ```

2. Check nginx logs:
   ```bash
   cd docker && docker compose -p deer-flow-dev -f docker-compose-dev.yaml logs nginx
   ```

3. Check firewall settings

---

### Issue: API Gateway Health Check Fails

**Symptoms**:
Accessing `/health` returns an error or times out.

**Solutions** (local mode):
1. Check gateway logs:
   ```bash
   tail -f logs/gateway.log
   ```

2. Confirm that config.yaml exists and has valid formatting
3. Check whether Python dependencies are complete
4. Confirm that the Gateway process is running normally.

**Solutions** (Docker mode):
1. Check gateway container logs:
   ```bash
   make docker-logs-gateway
   ```

2. Confirm that config.yaml is mounted correctly
3. Check whether Python dependencies are complete
4. Confirm that the Gateway process is running normally.

---

## Common Diagnostic Commands

### Local Mode Diagnostics

#### View All Service Processes
```bash
ps aux | grep -E "(uvicorn|next|nginx)" | grep -v grep
```

#### View Service Logs
```bash
# View all logs
tail -f logs/*.log

# View specific service logs
tail -f logs/gateway.log
tail -f logs/frontend.log
tail -f logs/nginx.log
```

#### Stop All Services
```bash
make stop
```

#### Fully Reset the Local Environment
```bash
make stop
make clean
make config
make install
make dev-daemon
```

---

### Docker Mode Diagnostics

#### View All Container Status
```bash
docker ps -a
```

#### View Container Resource Usage
```bash
docker stats
```

#### Enter a Container for Debugging
```bash
docker exec -it deer-flow-gateway sh
```

#### Clean Up All DeerFlow-Related Containers and Images
```bash
make docker-stop
cd docker && docker compose -p deer-flow-dev -f docker-compose-dev.yaml down -v
```

#### Fully Reset the Docker Environment
```bash
make docker-stop
make clean
make config
make docker-init
make docker-start
```

---

## Get More Help

If the solutions above do not resolve the issue:
1. Check the GitHub issues for the project: https://github.com/bytedance/deer-flow/issues
2. Review the project documentation: README.md and the `backend/docs/` directory
3. Open a new issue and include detailed error logs
