### 🎯 Monorepo Goal: Nx + React (GUI) + NestJS (Backend) + Dockerized Ethereum

```
my-eth-monorepo/
│
├── apps/
│   ├── gui/                  # React frontend (user interface)
│   └── api/                  # NestJS backend
│
├── libs/
│   └── ethereum/             # Shared Ethereum logic (ethers.js, ABI, utils)
│
├── docker/
│   ├── jwt.hex               # JWT secret for Ethereum node communication
│   ├── docker-compose.yml    # Runs geth + lighthouse clients
│   └── geth.sh               # Script to generate accounts or automate node startup
│
├── nx.json
├── workspace.json
├── tsconfig.base.json
└── package.json
```

---

### ⚙️ Step 1: Create the Monorepo
```bash
npx create-nx-workspace@latest my-eth-monorepo --preset=apps --package-manager=npm
cd my-eth-monorepo
```

---

### 🚀 Step 2: Add Frontend & Backend
```bash
nx generate @nrwl/react:application gui --style=css --routing=true
nx generate @nrwl/nest:application api
```

---

### 🔗 Step 3: Add Ethereum Shared Library
```bash
nx generate @nrwl/js:lib ethereum
```
Then inside `libs/ethereum/`:
- Install and expose ethers.js:
```bash
npm install ethers
```
- Create shared functions:
```ts
// libs/ethereum/src/lib/eth.service.ts
import { ethers } from 'ethers';

export function getProvider(): ethers.JsonRpcProvider {
  return new ethers.JsonRpcProvider("http://localhost:8545");
}
```

---

### 🧱 Step 4: Dockerized Ethereum Node (Goerli-compatible)
```yaml
# docker/docker-compose.yml
version: '3.8'
services:
  execution:
    image: ethereum/client-go:stable
    command: >
      --goerli
      --http
      --http.api eth,net,web3
      --authrpc.addr 0.0.0.0
      --authrpc.port 8551
      --authrpc.jwtsecret /geth/jwt.hex
    volumes:
      - ./geth-data:/root/.ethereum
      - ./jwt.hex:/geth/jwt.hex
    ports:
      - "8545:8545"
      - "8551:8551"

  consensus:
    image: sigp/lighthouse:latest
    command: >
      lighthouse bn
      --network=goerli
      --execution-endpoint=http://execution:8551
      --execution-jwt=/lighthouse/jwt.hex
    volumes:
      - ./lighthouse-data:/root/.lighthouse
      - ./jwt.hex:/lighthouse/jwt.hex
    depends_on:
      - execution
    ports:
      - "9000:9000"
```
Generate JWT:
```bash
openssl rand -hex 32 > docker/jwt.hex
```

---

### 🧠 Step 5: Connect Backend to Ethereum
Inside `apps/api/src/app/app.service.ts`:
```ts
import { Injectable } from '@nestjs/common';
import { getProvider } from '@my-eth-monorepo/ethereum';

@Injectable()
export class AppService {
  async getBlockNumber(): Promise<number> {
    const provider = getProvider();
    return await provider.getBlockNumber();
  }
}
```

---

### 🧪 Step 6: GUI Calls Backend
Add fetch call inside React `apps/gui/src/app/app.tsx`:
```ts
import { useEffect, useState } from 'react';

export function App() {
  const [block, setBlock] = useState<number>(0);

  useEffect(() => {
    fetch('/api')
      .then(res => res.json())
      .then(data => setBlock(data.block));
  }, []);

  return <div>Latest Block: {block}</div>;
}
```
Proxy `apps/gui/proxy.conf.json`:
```json
{
  "/api": {
    "target": "http://localhost:3333",
    "secure": false
  }
}
```
Update `angular.json` or `project.json` to use proxy config.

---

### 🏃‍♂️ Step 7: Run Everything
```bash
# Start Ethereum node (once)
docker compose -f docker/docker-compose.yml up

# Start backend
nx serve api

# Start frontend
nx serve gui
```

---

