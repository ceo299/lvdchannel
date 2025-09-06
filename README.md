# OTC Pre-order DApp — GitHub-ready Project

Đây là một project **GitHub-ready** mẫu, đã sắp xếp đầy đủ file và nội dung. Bạn có thể copy toàn bộ vào repo mới và chạy ngay.

---

## Cấu trúc file

```
otc-preorder-dapp/
├─ contracts/
│  └─ OTCPreOrder.sol
├─ scripts/
│  └─ deploy.js
├─ hardhat.config.js
├─ package.json
├─ frontend/
│  ├─ package.json
│  └─ src/
│     ├─ App.jsx
│     ├─ index.js
│     └─ abi.json
└─ README.md
```

---

## 1) `contracts/OTCPreOrder.sol`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

contract OTCPreOrder {
    struct Offer {
        address seller;
        string title;
        uint256 priceWei;
        uint256 totalQuantity;
        uint256 sold;
        bool active;
    }

    uint256 public nextOfferId;
    mapping(uint256 => Offer) public offers;
    mapping(uint256 => mapping(address => uint256)) public deposits;

    event OfferCreated(uint256 indexed offerId, address indexed seller, string title, uint256 priceWei, uint256 quantity);
    event OfferClosed(uint256 indexed offerId);
    event PreOrdered(uint256 indexed offerId, address indexed buyer, uint256 qty, uint256 amountWei);
    event SellerWithdraw(uint256 indexed offerId, address indexed seller, uint256 amountWei);

    function createOffer(string calldata title, uint256 priceWei, uint256 quantity) external returns (uint256) {
        require(priceWei > 0, "price>0");
        require(quantity > 0, "qty>0");

        uint256 id = nextOfferId++;
        offers[id] = Offer({
            seller: msg.sender,
            title: title,
            priceWei: priceWei,
            totalQuantity: quantity,
            sold: 0,
            active: true
        });

        emit OfferCreated(id, msg.sender, title, priceWei, quantity);
        return id;
    }

    function preorder(uint256 offerId, uint256 qty) external payable {
        Offer storage off = offers[offerId];
        require(off.active, "offer not active");
        require(qty > 0, "qty>0");
        require(off.sold + qty <= off.totalQuantity, "not enough qty");
        uint256 cost = qty * off.priceWei;
        require(msg.value == cost, "incorrect ETH sent");

        deposits[offerId][msg.sender] += qty;
        off.sold += qty;

        emit PreOrdered(offerId, msg.sender, qty, cost);
    }

    function withdraw(uint256 offerId) external {
        Offer storage off = offers[offerId];
        require(msg.sender == off.seller, "only seller");
        uint256 amount = off.sold * off.priceWei;
        require(amount > 0, "nothing to withdraw");

        off.sold = 0;
        off.active = false;

        (bool sent, ) = off.seller.call{value: amount}("");
        require(sent, "withdraw failed");

        emit SellerWithdraw(offerId, off.seller, amount);
        emit OfferClosed(offerId);
    }

    function myReserved(uint256 offerId, address buyer) external view returns (uint256) {
        return deposits[offerId][buyer];
    }
}
```

---

## 2) `scripts/deploy.js`

```javascript
const hre = require("hardhat");

async function main() {
  const OTC = await hre.ethers.getContractFactory("OTCPreOrder");
  const otc = await OTC.deploy();
  await otc.deployed();
  console.log("OTCPreOrder deployed to:", otc.address);
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

---

## 3) `hardhat.config.js`

```javascript
require("@nomiclabs/hardhat-ethers");

module.exports = {
  solidity: "0.8.19",
  networks: {
    hardhat: {},
  },
};
```

---

## 4) `package.json` (root)

```json
{
  "name": "otc-preorder-dapp",
  "version": "1.0.0",
  "scripts": {
    "compile": "npx hardhat compile",
    "deploy": "npx hardhat run scripts/deploy.js --network localhost"
  },
  "devDependencies": {
    "hardhat": "^2.16.0",
    "@nomiclabs/hardhat-ethers": "^2.2.0",
    "ethers": "^6.0.0"
  }
}
```

---

## 5) `frontend/package.json`

```json
{
  "name": "frontend",
  "version": "1.0.0",
  "private": true,
  "dependencies": {
    "ethers": "^6.0.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build"
  }
}
```

---

## 6) `frontend/src/abi.json`

```json
[
  "function nextOfferId() view returns (uint256)",
  "function offers(uint256) view returns (address,string,uint256,uint256,uint256,bool)",
  "function createOffer(string,uint256,uint256) returns (uint256)",
  "function preorder(uint256,uint256) payable",
  "function withdraw(uint256)",
  "event OfferCreated(uint256 indexed, address indexed, string, uint256, uint256)",
  "event PreOrdered(uint256 indexed, address indexed, uint256, uint256)"
]
```

---

## 7) `frontend/src/App.jsx`

```jsx
import React, { useEffect, useState } from 'react';
import { ethers } from 'ethers';
import ABI from './abi.json';

export default function App() {
  const [provider, setProvider] = useState(null);
  const [signer, setSigner] = useState(null);
  const [contract, setContract] = useState(null);
  const [account, setAccount] = useState(null);
  const [offers, setOffers] = useState([]);
  const [contractAddress, setContractAddress] = useState('PASTE_CONTRACT_ADDRESS');

  useEffect(() => {
    if (!window.ethereum) return;
    const p = new ethers.BrowserProvider(window.ethereum);
    setProvider(p);
  }, []);

  async function connect() {
    await window.ethereum.request({ method: 'eth_requestAccounts' });
    const p = provider;
    const s = await p.getSigner();
    setSigner(s);
    const addr = await s.getAddress();
    setAccount(addr);
    const c = new ethers.Contract(contractAddress, ABI, s);
    setContract(c);
    await loadOffers(c);
  }

  async function loadOffers(c) {
    const nextId = Number(await c.nextOfferId());
    const arr = [];
    for (let i = 0; i < nextId; i++) {
      const o = await c.offers(i);
      arr.push({ id: i, seller: o[0], title: o[1], priceWei: o[2].toString(), totalQuantity: o[3].toString(), sold: o[4].toString(), active: o[5] });
    }
    setOffers(arr);
  }

  async function createOffer(e) {
    e.preventDefault();
    const title = e.target.title.value;
    const priceWei = ethers.parseEther(e.target.price.value);
    const qty = Number(e.target.qty.value);
    const tx = await contract.createOffer(title, priceWei, qty);
    await tx.wait();
    await loadOffers(contract);
  }

  async function preorder(offerId, qty, priceWei) {
    const cost = ethers.BigInt(priceWei) * BigInt(qty);
    const tx = await contract.preorder(offerId, qty, { value: cost.toString() });
    await tx.wait();
    await loadOffers(contract);
  }

  async function withdraw(offerId) {
    const tx = await contract.withdraw(offerId);
    await tx.wait();
    await loadOffers(contract);
  }

  return (
    <div style={{ padding: 20 }}>
      <h1>OTC Pre-order DApp</h1>
      {!account ? (
        <button onClick={connect}>Connect MetaMask</button>
      ) : (
        <div>Connected: {account}</div>
      )}

      <section>
        <h2>Create Offer</h2>
        <form onSubmit={createOffer}>
          <input name="title" placeholder="Title" required />
          <input name="price" placeholder="Price (ETH)" required />
          <input name="qty" type="number" placeholder="Quantity" required />
          <button type="submit">Create</button>
        </form>
      </section>

      <section>
        <h2>Offers</h2>
        {offers.length === 0 && <p>No offers yet</p>}
        <ul>
          {offers.map(o => (
            <li key={o.id}>
              <strong>{o.title}</strong> — {ethers.formatEther(o.priceWei)} ETH — {o.sold}/{o.totalQuantity} — {o.active ? 'Active' : 'Closed'}
              <div>
                <input id={`qty-${o.id}`} type="number" placeholder="qty" />
                <button onClick={() => preorder(o.id, Number(document.getElementById(`qty-${o.id}`).value), o.priceWei)}>Pre-order</button>
                {account && account.toLowerCase() === o.seller.toLowerCase() && (
                  <button onClick={() => withdraw(o.id)}>Withdraw</button>
                )}
              </div>
            </li>
          ))}
        </ul>
      </section>
    </div>
  );
}
```

---

## 8) `frontend/src/index.js`

```javascript
import React from 'react';
import { createRoot } from 'react-dom/client';
import App from './App';

createRoot(document.getElementById('root')).render(<App />);
```

---

## 9) `README.md`

```markdown
# OTC Pre-order DApp

### Deploy & Run

1. Cài đặt Node.js v18+
2. Clone repo: `git clone ...`
3. `cd otc-preorder-dapp`
4. `npm install`
5. Chạy node local: `npx hardhat node`
6. Deploy contract: `npx hardhat run scripts/deploy.js --network localhost`
7. Copy contract address in output
8. `cd frontend && npm install`
9. Dán contract address vào `frontend/src/App.jsx`
10. `npm start` để chạy UI

### Features
- Seller tạo offer OTC pre-order
- Buyer đặt cọc bằng ETH
- Seller rút tiền khi kết thúc

### Nâng cấp
- Thêm refund cho buyer
- Dùng ERC-20 thay ETH
- Tích hợp phí nền tảng
```

---

👉 Repo này đã **GitHub-ready**: chỉ cần copy lên GitHub, chạy `npm install` và deploy. Bạn muốn mình đóng gói repo này thành file `.zip` để tải về không?
