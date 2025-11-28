# FLOW ĐƠN GIẢN: ARBITRAGE TOKEN OP

## 🎯 Ý TƯỞNG CHÍNH

```
Mua OP giá RẺ → Bán OP giá CAO → Kiếm lời chênh lệch!
```

**Nhưng không cần vốn!** Dùng flash loan để vay rồi trả trong 1 transaction.

---

## 🔺 3 POOLS TẠO CƠ HỘI

```mermaid
graph TD
    subgraph Pools on Uniswap V3
        P1["🏦 Pool 1<br/>OP/WETH<br/>Giá: 1 OP = 0.0008 WETH<br/>Fee: 0.3%"]
        P2["🏦 Pool 2<br/>OP/USDT<br/>Giá: 1 OP = 2.50 USDT<br/>Fee: 0.3%"]
        P3["🏦 Pool 3<br/>WETH/USDT<br/>Giá: 1 WETH = 3100 USDT<br/>Fee: 0.05%"]
    end

    P1 -.->|Convert| WETH[WETH = 3100 USDT]
    P2 -.->|Direct| USDT[OP = 2.50 USDT]

    WETH -.-> Compare{Compare}
    USDT -.-> Compare

    Compare -->|"OP via WETH:<br/>0.0008 × 3100 = 2.48 USDT"| Route1
    Compare -->|"OP direct:<br/>2.50 USDT"| Route2

    Route1[2.48 USDT]
    Route2[2.50 USDT]

    Route2 -->|"Chênh lệch<br/>0.02 USDT/OP<br/>= 0.8%"| Opportunity[✅ CƠ HỘI!]

    style P1 fill:#FFE5B4
    style P2 fill:#FFE5B4
    style P3 fill:#FFE5B4
    style Opportunity fill:#90EE90
```

---

## 🚀 FLOW 6 BƯỚC ĐƠN GIẢN

### **🎬 BƯỚC 1: User khởi động**

```javascript
// User gọi contract với tham số:
flashSwapTriangular({
    poolWETH_OP: "0x...",    // Pool OP/WETH
    feeOP_USDT: 3000,        // 0.3%
    feeWETH_USDT: 500,       // 0.05%
    wethAmount: 10 WETH      // Vay 10 WETH
})
```

```mermaid
graph LR
    U[👤 User] -->|"Call contract<br/>Vay 10 WETH"| C[📝 Contract]

    style U fill:#90EE90
    style C fill:#87CEEB
```

**Console:**
```
🟢 User initiates arbitrage with 10 WETH
```

---

### **💸 BƯỚC 2: Vay WETH, nhận OP**

Contract gọi pool OP/WETH để vay:

```mermaid
sequenceDiagram
    participant C as 📝 Contract
    participant P1 as 🏦 Pool OP/WETH

    C->>P1: swap(10 WETH, vay OP tokens)
    Note over P1: Tính toán:<br/>10 WETH ÷ 0.0008 = 12,500 OP<br/>Phí 0.3% → phải trả 10.03 WETH
    P1->>C: Gửi 12,500 OP
    Note over C: ✅ Nhận 12,500 OP<br/>⚠️ Nợ 10.03 WETH
```

**Tính toán:**
```
Vay: 10 WETH
Giá: 1 OP = 0.0008 WETH → 1 WETH = 1250 OP

Contract nhận: 10 × 1250 = 12,500 OP
Contract nợ:   10 × (1 + 0.003) = 10.03 WETH
```

**Console:**
```
💰 STEP 2: Borrowed from OP/WETH pool
   ✓ Received: 12,500 OP tokens
   ⚠ Must repay: 10.03 WETH
```

---

### **🔄 BƯỚC 3: Pool gọi callback**

```mermaid
graph LR
    P[🏦 Pool OP/WETH] -->|"uniswapV3SwapCallback()<br/>(amount0, amount1, data)"| C[📝 Contract]

    style P fill:#FFD700
    style C fill:#87CEEB
```

**Dữ liệu nhận được:**
```javascript
amount0 = -12,500 OP   // Âm = contract nhận
amount1 = +10.03 WETH  // Dương = contract phải trả

// Contract decode:
opAmount = 12,500 OP
wethOwed = 10.03 WETH
```

**Console:**
```
📞 STEP 3: Callback received
   Data: -12,500 OP (received), +10.03 WETH (owed)
```

---

### **🔁 BƯỚC 4: Swap OP → USDT**

Contract swap 12,500 OP → USDT qua pool OP/USDT:

```mermaid
graph LR
    C[📝 Contract<br/>12,500 OP] -->|"Swap via Router"| R[🔀 Router]
    R -->|"Call Pool OP/USDT"| P2[🏦 Pool OP/USDT<br/>fee 0.3%]
    P2 -->|"Return USDT"| R
    R -->|"31,156.25 USDT"| C2[📝 Contract<br/>31,156 USDT]

    style C fill:#87CEEB
    style C2 fill:#87CEEB
    style P2 fill:#FFD700
    style R fill:#DDA0DD
```

**Tính toán:**
```
Input: 12,500 OP
Giá: 1 OP = 2.50 USDT
Fee: 0.3%

USDT trước phí = 12,500 × 2.50 = 31,250 USDT
USDT sau phí   = 31,250 × 0.997 = 31,156.25 USDT

✅ Nhận: 31,156.25 USDT
```

**Console:**
```
🔵 STEP 4: Swap OP → USDT
   Input: 12,500 OP
   Output: 31,156.25 USDT
   Fee: 0.3% (93.75 USDT)
```

---

### **🔁 BƯỚC 5: Swap USDT → WETH**

Contract swap USDT → WETH qua pool WETH/USDT:

```mermaid
graph LR
    C[📝 Contract<br/>31,156 USDT] -->|"Swap via Router"| R[🔀 Router]
    R -->|"Call Pool WETH/USDT"| P3[🏦 Pool WETH/USDT<br/>fee 0.05%]
    P3 -->|"Return WETH"| R
    R -->|"10.0454 WETH"| C2[📝 Contract<br/>10.0454 WETH]

    style C fill:#87CEEB
    style C2 fill:#87CEEB
    style P3 fill:#FFD700
    style R fill:#DDA0DD
```

**Tính toán:**
```
Input: 31,156.25 USDT
Giá: 1 WETH = 3100 USDT
Fee: 0.05%

WETH trước phí = 31,156.25 ÷ 3100 = 10.0504 WETH
WETH sau phí   = 10.0504 × 0.9995 = 10.0454 WETH

✅ Nhận: 10.0454 WETH
```

**Console:**
```
🟡 STEP 5: Swap USDT → WETH
   Input: 31,156.25 USDT
   Output: 10.0454 WETH
   Fee: 0.05% (0.005 WETH)
```

---

### **💰 BƯỚC 6: Trả nợ & Nhận profit**

```mermaid
graph TD
    C[📝 Contract<br/>có 10.0454 WETH] --> Compare{So sánh}

    Compare -->|"Nhận được: 10.0454 WETH<br/>Phải trả: 10.03 WETH"| Calc[Tính toán]

    Calc -->|"10.0454 > 10.03<br/>✅ CÓ LỜI!"| Profit["💰 Profit<br/>0.0154 WETH<br/>≈ $47.74"]

    Profit --> Pay1[Trả Pool OP/WETH<br/>10.03 WETH]
    Profit --> Pay2[Gửi User<br/>0.0154 WETH]

    style C fill:#87CEEB
    style Profit fill:#90EE90
    style Pay1 fill:#FFD700
    style Pay2 fill:#90EE90
```

**Code trong contract:**
```javascript
// So sánh
wethReceived = 10.0454 WETH
wethOwed = 10.03 WETH

if (wethReceived > wethOwed) {
    profit = wethReceived - wethOwed
          = 10.0454 - 10.03
          = 0.0154 WETH

    // Trả nợ
    weth.transfer(poolWETH_OP, 10.03 WETH)

    // Gửi profit
    weth.transfer(user, 0.0154 WETH)
}
```

**Console:**
```
✅ STEP 6: Calculate profit
   Received: 10.0454 WETH
   Must pay: 10.03 WETH
   ━━━━━━━━━━━━━━━━━━━━
   💰 PROFIT: 0.0154 WETH

   ✓ Paid pool: 10.03 WETH
   ✓ Sent to user: 0.0154 WETH (≈ $47.74 USD)

🎉 ARBITRAGE COMPLETED SUCCESSFULLY!
```

---

## 📊 TỔNG QUAN TOÀN BỘ FLOW

```mermaid
graph TD
    Start["👤 USER<br/>Vay 10 WETH"] --> Step1

    Step1["🏦 POOL OP/WETH<br/>━━━━━━━━━━━<br/>IN: Vay 10 WETH<br/>OUT: 12,500 OP<br/>OWE: 10.03 WETH"] --> Step2

    Step2["🏦 POOL OP/USDT<br/>━━━━━━━━━━━<br/>IN: 12,500 OP<br/>OUT: 31,156 USDT<br/>FEE: 0.3%"] --> Step3

    Step3["🏦 POOL WETH/USDT<br/>━━━━━━━━━━━<br/>IN: 31,156 USDT<br/>OUT: 10.0454 WETH<br/>FEE: 0.05%"] --> Step4

    Step4{{"💰 TÍNH TOÁN<br/>━━━━━━━━━━━<br/>Nhận: 10.0454 WETH<br/>Nợ: 10.03 WETH<br/>Profit: 0.0154 WETH"}} --> Step5

    Step5["✅ HOÀN TẤT<br/>━━━━━━━━━━━<br/>Trả pool: 10.03 WETH<br/>User nhận: 0.0154 WETH<br/>≈ $47.74 USD"] --> End["🎉 SUCCESS!"]

    style Start fill:#90EE90
    style Step1 fill:#FFE5B4
    style Step2 fill:#FFE5B4
    style Step3 fill:#FFE5B4
    style Step4 fill:#87CEEB
    style Step5 fill:#90EE90
    style End fill:#90EE90
```

---

## 💹 TRACKING BALANCE

| Step | Action | OP Balance | USDT Balance | WETH Balance | Notes |
|------|--------|------------|--------------|--------------|-------|
| 0 | Start | 0 | 0 | 0 | Contract rỗng |
| 1 | Vay từ Pool1 | **+12,500** | 0 | 0 | Nợ 10.03 WETH |
| 2 | Swap OP→USDT | 0 | **+31,156** | 0 | - |
| 3 | Swap USDT→WETH | 0 | 0 | **+10.0454** | - |
| 4 | Trả nợ Pool1 | 0 | 0 | 10.0454 - 10.03 | Trả 10.03 |
| 5 | Gửi User | 0 | 0 | **0** | Gửi 0.0154 cho User |
| ✅ | **User nhận** | - | - | **+0.0154** | **Profit: $47.74** |

---

## 🎯 CALL FUNCTION VỚI THAM SỐ THỰC TẾ

### **JavaScript/TypeScript:**

```javascript
const { ethers } = require("ethers");

// Setup contract
const contract = new ethers.Contract(
    CONTRACT_ADDRESS,
    ABI,
    signer
);

// Tham số arbitrage
const params = {
    poolWETH_OP: "0x68F5C0A2DE713a54991E01858Fd27a3832401849",  // Pool OP/WETH (ví dụ)
    tokenIntermediate: "0x4200000000000000000000000000000000000042",  // OP token
    feeOP_USDT: 3000,     // 0.3%
    feeWETH_USDT: 500,    // 0.05%
    wethAmount: ethers.utils.parseEther("10")  // 10 WETH
};

// CALL FUNCTION
const tx = await contract.flashSwapTriangular(params);
await tx.wait();

console.log("✅ Arbitrage completed!");
```

### **Solidity:**

```solidity
// Call trực tiếp từ contract khác hoặc EOA
IArbitrage(contractAddress).flashSwapTriangular(
    ArbitrageParams({
        poolWETH_OP: 0x68F5C0A2DE713a54991E01858Fd27a3832401849,
        tokenIntermediate: 0x4200000000000000000000000000000000000042,
        feeOP_USDT: 3000,
        feeWETH_USDT: 500,
        wethAmount: 10 * 10**18  // 10 WETH in wei
    })
);
```

---

## 📈 ROI CALCULATION

```
Investment: 0 WETH (dùng flash loan!)
Gas cost: ~$30 (350k gas × 30 gwei)
Profit: 0.0154 WETH = $47.74

Net profit: $47.74 - $30 = $17.74

ROI: ∞ (vì không cần vốn ban đầu!)
Profit margin: $17.74 / $47.74 = 37% (sau trừ gas)
```

**Với số lượng lớn hơn:**

```
Amount: 100 WETH
Profit: 0.154 WETH = $477.4
Gas cost: ~$30
Net profit: $447.4

→ Tỷ lệ profit/gas tốt hơn nhiều!
```

---

## ⚡ TÓM TẮT 1 PHÚT

### **Điều kiện cần:**
```
✅ OP/WETH pool: Giá 1 OP = 0.0008 WETH
✅ OP/USDT pool: Giá 1 OP = 2.50 USDT
✅ WETH/USDT pool: Giá 1 WETH = 3100 USDT

Tính giá OP qua 2 routes:
Route 1 (via WETH): 0.0008 × 3100 = 2.48 USDT
Route 2 (direct): 2.50 USDT

Chênh lệch: 2.50 - 2.48 = 0.02 USDT (0.8%)
```

### **Flow đơn giản:**
```
WETH → OP → USDT → WETH
10 → 12,500 → 31,156 → 10.0454

Profit: 0.0454 - 0.03 (phí) = 0.0154 WETH ✅
```

### **1 lệnh duy nhất:**
```javascript
await contract.flashSwapTriangular({
    poolWETH_OP,
    feeOP_USDT: 3000,
    feeWETH_USDT: 500,
    wethAmount: "10 WETH"
});

// → Nhận: 0.0154 WETH ($47.74) 🎉
```

---

**VẬY LÀ XONG! Arbitrage OP token đơn giản và có lời!** 🚀💰
