# Install Node.js and npm on WSL2

### 1. Install cURL

```bash
sudo apt-get install curl
```

### 2. Install nvm

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
```

> restart the WSL.

### 3. Verify installation

```bash
command -v nvm
```

It should return `nvm`.

### 4. Install Node.js

```bash
nvm install --lts # Latest stable LTS release (recommended)
nvm install node # Current release
```

### 5. Verify installation

```bash
node --version # node.js
npm --version # npm
```