# Markup (DMK & Scheme) & Commission Calculation Logic

> **Last Updated:** 2026-06-04  
> **Source Files:**
> - `src/app/shared/services/calculation-orchestration.service.ts`
> - `src/app/shared/lib/common.service.ts`
> - `src/app/modules/admin/tours/edit/edit.component.ts`

---

## 1. TỔNG QUAN KIẾN TRÚC TÍNH TOÁN

```mermaid
flowchart TB
    subgraph TRIGGERS["🔔 TRIGGERS"]
        A1["Load Tour (GetOneToursByID)"]
        A2["Add/Edit/Delete Service"]
        A3["Change Markup/Margin"]
        A4["Apply Markup Scheme"]
        A5["Change Currency"]
        A6["Change Pax/Passenger"]
        A7["Change Commission Settings"]
    end

    subgraph ORCHESTRATION["🎯 ORCHESTRATION LAYER"]
        B1["refresh_grid()"]
        B2["ActionRunTotalChange()"]
        B3["reCalculatorLandingHotel()"]
        B4["CalculationOrchestrationService.calculateFullTourTotals()"]
    end

    subgraph PRICING["💰 PRICING LAYER (common.service.ts)"]
        C1["runPrice()"]
        C2["runPriceCalculator()"]
        C3["applyPricingLogicBasedOnServiceType()"]
        C4["GridFooterTotalTour()"]
    end

    subgraph COMMISSION["💹 COMMISSION LAYER"]
        D1["aggregateCommissionsFromServices()"]
        D2["calculateCommissionsFromGrandTotal()"]
        D3["calculateAndUpdateAgentCommissions()"]
        D4["runApplyCommissions()"]
    end

    A1 & A2 & A3 & A4 & A5 & A6 & A7 --> B1 & B2
    B1 --> B3
    B3 --> B4
    B2 --> B4
    B4 --> C1
    C1 --> C2 --> C3
    C3 --> C4
    B4 --> D3
    D3 --> D1 & D2
    C4 --> D4

    style TRIGGERS fill:#fff3cd
    style ORCHESTRATION fill:#d1ecf1
    style PRICING fill:#d4edda
    style COMMISSION fill:#f8d7da
```

---

## 2. LUỒNG TÍNH TOÁN MARKUP: DMK vs SCHEME

```mermaid
flowchart TD
    START(["🏁 Bắt đầu tính markup cho 1 service"]) --> CHECK_DMK{"info.DMK<br/>(Direct Markup)?"}

    CHECK_DMK -->|"✅ DMK = true<br/>(Direct Markup)"| DMK_FLOW
    CHECK_DMK -->|"❌ DMK = false<br/>(Markup Scheme)"| SCHEME_FLOW

    subgraph DMK_FLOW["📐 DIRECT MARKUP (DMK)"]
        D1["Mỗi service có markup riêng<br/>service.markup = giá trị người dùng set"]
        D2["Service KHÔNG có markupSchemes<br/>→ dùng service.markup trực tiếp"]
        D3["Extra Markup được cộng thêm:<br/>service.markup + ExtraDirectMarkupAllLanding<br/>hoặc + ExtraDirectMarkupAllHotel"]
        D4["Tính giá: pricebuy → markup → pricesell"]
        D5["Gọi Calculator tương ứng theo PriceType"]

        D1 --> D2 --> D3 --> D4 --> D5
    end

    subgraph SCHEME_FLOW["📊 MARKUP SCHEME"]
        S1["Tất cả service dùng chung 1 scheme<br/>MarkupAllLanding / MarkupAllHotel"]
        S2["Service có markupSchemes object<br/>chứa tiers markup theo pax range"]
        S3["Markup = MarkupAllLanding<br/>+ ExtraDirectMarkupAllLanding<br/>(hoặc tương tự cho Hotel)"]
        S4["Nếu service.keepDMK = true<br/>→ giữ markup riêng, KHÔNG ghi đè"]
        S5["Gọi Calculator tương ứng theo PriceType<br/>(có xét đến markupSchemes)"]

        S1 --> S2 --> S3 --> S4 --> S5
    end

    subgraph CALCULATORS["🧮 CALCULATORS (theo PriceType)"]
        C1["GroupCalculator<br/>→ Dành cho GROUP / shared service"]
        C2["PERSONCalculator<br/>→ Dành cho PERSON / per-person"]
        C3["PriceBandCalculator<br/>→ Dành cho PRICEBAND<br/>(theo số pax: min-max tiers)"]
        C4["AccommodationCalculator<br/>→ Dành cho ACCOMMODATIONFEE<br/>(tính room, night, occupancy)"]
        C5["ACCUMULATEDCalculator<br/>→ Dành cho ACCUMULATEDCOST<br/>(cộng dồn chi phí)"]
    end

    D5 --> CALCULATORS
    S5 --> CALCULATORS

    C1 & C2 & C3 & C4 & C5 --> PAXBREAK["Tạo paxBreak[]<br/>Mỗi tier: pricebuy, pricesell,<br/>priceBeforeCommissions,<br/>pricebuyTLs, pricesellTLs"]

    PAXBREAK --> APPLY_COMM["Chạy runApplyCommissions()<br/>nếu có agentCommissions"]

    APPLY_COMM --> SET_FIELDS["Set các trường output:<br/>ld_unitPrice, ld_totalPrice,<br/>ld_totalPriceBeforeCommissions,<br/>unit_Total, sgl_Total, acc_Total,..."]

    SET_FIELDS --> GRID_FOOTER["GridFooterTotalTour()<br/>Tổng hợp tất cả service<br/>→ gridFooterTotalTour[]"]

    style DMK_FLOW fill:#cce5ff
    style SCHEME_FLOW fill:#d4edda
    style CALCULATORS fill:#fff3cd
```

---

## 3. CHI TIẾT TỪNG CALCULATOR THEO PRICE TYPE

```mermaid
flowchart LR
    subgraph GROUP["GroupCalculator"]
        G1["Input: data.markup, tiers[]"]
        G2["Với mỗi tier (pax range):<br/>pricesell = pricebuy × (1 + markup/100)"]
        G3["priceBeforeCommissions = pricesell<br/>(trước khi áp commission)"]
        G4["pricesell = runApplyCommissions(pricesell, agentCommissions)"]
    end

    subgraph PRICEBAND["PriceBandCalculator"]
        P1["Input: listOrtherData (tiers theo pax)<br/>data.markup, data.DMK"]
        P2["Tìm tier phù hợp với pax hiện tại"]
        P3["Nếu DMK=false & có markupSchemes:<br/>tính markup từ scheme's tiers"]
        P4["Nếu DMK=true:<br/>dùng data.markup trực tiếp"]
        P5["pricesell = price × (1 + markup/100)<br/>× số pax / min pax của tier"]
        P6["pricesellTLs = priceTLs × (1 + markupTLs/100)"]
        P7["priceBeforeCommissions, pricesell → commissions"]
    end

    subgraph ACCOM["AccommodationCalculator"]
        H1["Input: hotelinfo, noofrooms, noofnights,<br/>markup, occupancy, lsRoomAssigned"]
        H2["Tính unit_buy, sgl_buy từ hotelinfo.price"]
        H3["unit = markup(unit_buy, markup)<br/>sgl = markup(sgl_buy, markup)"]
        H4["Áp dụng hotel rules (supplement/deduction)"]
        H5["Từng tier: tính paxBreak<br/>phân bổ room cost theo occupancy"]
        H6["Extra Bed, Breakfast, Late checkout,..."]
        H7["unit_BeforeCommissions → runApplyCommissions → unit"]
    end

    subgraph ACCUM["ACCUMULATEDCalculator"]
        U1["Input: Cộng dồn cost từ các component"]
        U2["Tính progressive pricing theo tier"]
        U3["Tương tự logic như PriceBand<br/>nhưng cộng dồn"]
    end

    style GROUP fill:#e8f5e9
    style PRICEBAND fill:#fff3e0
    style ACCOM fill:#e3f2fd
    style ACCUM fill:#fce4ec
```

---

## 4. LUỒNG TÍNH TOÁN COMMISSION

```mermaid
flowchart TD
    START_COM(["🏁 Bắt đầu tính Commission"]) --> CHECK_APPLY{"info.applyCommissions<br/>= true?"}

    CHECK_APPLY -->|"❌ No"| NO_COM["Reset tất cả commission về 0<br/>grandTotal = PriceBeforeCommissions<br/>agentSellPrice = PriceBeforeCommissions"]

    CHECK_APPLY -->|"✅ Yes"| CHECK_KEEP{"info.keepCommissions<br/>= true?"}

    CHECK_KEEP -->|"✅ Yes<br/>(Giữ nguyên commission cũ)"| KEEP_FLOW["Giữ nguyên rsAgentCommissions<br/>grandTotal = PriceBeforeCommissions<br/>+ sum(preserved commissions)<br/>agentSellPrice = grandTotal"]

    CHECK_KEEP -->|"❌ No<br/>(Tính lại commission)"| CHECK_MODE{"info.optionCommissions<br/>= 'SellPrice'?"}

    CHECK_MODE -->|"✅ SellPrice Mode"| SELLPRICE_FLOW
    CHECK_MODE -->|"❌ Markup/Margin Mode"| MARKUP_FLOW

    subgraph SELLPRICE_FLOW["💹 SELLPRICE MODE: Aggregate từ services"]
        SP1["aggregateCommissionsFromServices()"]
        SP2["1️⃣ Tổng hợp từ LANDING services<br/>(grid_PaxManagementLanding)"]
        SP3["2️⃣ Tổng hợp từ ACCOMMODATION services<br/>(dataHotelPassenger)"]
        SP4["3️⃣ Tổng hợp từ SURCHARGES<br/>(SurchargesHotel)"]
        SP5["4️⃣ Với mỗi commission (theo order):<br/>runningTotal = PriceBeforeCommissions"]
        SP6["5️⃣ Nếu item có miniTour + custom commissions<br/>→ aggregate từ child items"]
        SP7["6️⃣ Xử lý extraAmount:<br/>price = calculatedPrice + extraAmount"]
        SP8["7️⃣ VALIDATE: Nếu price sai → fallback<br/>dùng savedApiPrices hoặc flat calculation"]
        SP9["8️⃣ grandTotal = PriceBeforeCommissions<br/>+ sum(all commission prices)"]

        SP1 --> SP2 --> SP3 --> SP4 --> SP5 --> SP6 --> SP7 --> SP8 --> SP9
    end

    subgraph MARKUP_FLOW["📊 MARKUP/MARGIN MODE: Tính từ grandTotal"]
        M1["calculateCommissionsFromGrandTotal()"]
        M2["runningTotal = PriceBeforeCommissions"]
        M3["Với mỗi commission (theo order):"]
        M4["Nếu có markup > 0:<br/>amount = runningTotal × (markup/100)"]
        M5["Nếu có margin > 0:<br/>markup = margin/(100-margin)×100<br/>amount = runningTotal × (markup/100)"]
        M6["Nếu isExtraApplied:<br/>amount += extraAmount"]
        M7["runningTotal += amount<br/>(compound effect cho commission sau)"]
        M8["grandTotal = runningTotal cuối cùng<br/>agentSellPrice = grandTotal"]

        M1 --> M2 --> M3 --> M4 & M5
        M4 & M5 --> M6 --> M7 --> M3
        M7 -->|"Hết commission"| M8
    end

    NO_COM --> FINAL
    KEEP_FLOW --> FINAL
    SP9 --> FINAL
    M8 --> FINAL

    FINAL(["🏁 KẾT QUẢ CUỐI CÙNG<br/>info.grandTotal<br/>info.agentSellPrice<br/>info.rsAgentCommissions"])

    style SELLPRICE_FLOW fill:#d4edda
    style MARKUP_FLOW fill:#fff3cd
    style NO_COM fill:#f8d7da
    style KEEP_FLOW fill:#cce5ff
```

---

## 5. CHI TIẾT SELLPRICE MODE: AGGREGATE COMMISSIONS TỪ SERVICES

```mermaid
flowchart TD
    AGG_START(["aggregateCommissionsFromServices()"]) --> SAVE_EXTRA["💾 Lưu preservedValues<br/>(isExtraApplied, extraAmount, originalPrice)<br/>từ agentCommissions + rsAgentCommissions"]

    SAVE_EXTRA --> INIT_RS["Khởi tạo / Reset rsAgentCommissions<br/>về 0 (giữ preserved extra settings)"]

    INIT_RS --> LANDING["1️⃣ AGGREGATE LANDING SERVICES"]

    subgraph LANDING_DETAIL["LANDING DETAIL"]
        L1["getGridLandingServices()<br/>→ Lấy từ grid_PaxManagementLanding.data"]
        L2["Với mỗi service (đã lọc active):"]
        L3{"item.miniTour có<br/>custom commissions?"}
        L3 -->|"✅ Yes"| L4["aggregateCommissionsFromMiniTourChildren()<br/>→ Tính từ child items' agentCommissions<br/>theo markup riêng của từng child"]
        L3 -->|"❌ No"| L5["aggregateFromItemCommissions()"]
        L5 --> L5A["Kiểm tra rsAgentCommissions có sẵn:<br/>nếu tất cả selected & price>0 → dùng luôn"]
        L5A --> L5B["Nếu không: tính từ item.agentCommissions<br/>hoặc listOrtherData (PRICEBAND)<br/>hoặc fallback về booking-level"]
        L5B --> L5C["Tính: commissionPrice = basePrice × (markup/100)<br/>với per-item compounding"]
    end

    LANDING --> LANDING_DETAIL

    LANDING_DETAIL --> ACCOM["2️⃣ AGGREGATE ACCOMMODATION SERVICES"]

    subgraph ACCOM_DETAIL["ACCOMMODATION DETAIL"]
        A1["getAccommodationServices()<br/>→ dataHotelPassenger (active, non-TL)"]
        A2["Với mỗi service:"]
        A3{"item.miniTour có<br/>custom commissions?"}
        A3 -->|"✅ Yes"| A4["aggregateCommissionsFromMiniTourChildren()"]
        A3 -->|"❌ No"| A5["aggregateFromItemCommissions()<br/>(tương tự landing)"]
    end

    ACCOM --> ACCOM_DETAIL

    ACCOM_DETAIL --> SURCHARGE["3️⃣ AGGREGATE SURCHARGES"]

    subgraph SUR_DETAIL["SURCHARGE DETAIL"]
        S1["Duyệt SurchargesHotel[]"]
        S2["Với mỗi surcharge có rsAgentCommissions:<br/>cộng trực tiếp price vào info.rsAgentCommissions"]
    end

    SURCHARGE --> SUR_DETAIL

    SUR_DETAIL --> RECALC["4️⃣ RECALCULATE VỚI COMPOUNDING"]

    subgraph RECALC_DETAIL["COMPOUNDING DETAIL"]
        R1["Sort commissions theo order"]
        R2["runningTotal = PriceBeforeCommissions"]
        R3["Với mỗi commission:"]
        R4{"isExtraApplied<br/>hoặc extraAmount > 0?"}
        R4 -->|"✅ Yes"| R5["price = calculatedPrice + extraAmount<br/>→ giữ nguyên user's intended total"]
        R4 -->|"❌ No"| R6["price = aggregated price từ services"]
        R7["runningTotal += price<br/>(cho commission tiếp theo compound)"]
        R8{"Có commission trước bị extra?"}
        R8 -->|"✅ Yes"| R9["Recalculate commission này<br/>từ runningTotal × (markup/100)"]
        R8 -->|"❌ No"| R10["Giữ aggregated price"]
    end

    RECALC --> RECALC_DETAIL

    RECALC_DETAIL --> VALIDATE["5️⃣ VALIDATE & FALLBACK"]

    subgraph VAL_DETAIL["VALIDATION DETAIL"]
        V1{"Có commission nào<br/>price = 0 (nhưng markup>0)<br/>hoặc price = PriceBeforeCommissions?"}
        V1 -->|"✅ Có vấn đề"| V2["FALLBACK:<br/>1. Thử savedApiPrices từ backend<br/>2. Nếu không có → flat calculation"]
        V1 -->|"❌ OK"| V3["Giữ kết quả aggregated"]
    end

    VALIDATE --> VAL_DETAIL

    VAL_DETAIL --> FINAL_AGG["6️⃣ FINAL"]

    subgraph FINAL_DET["FINAL CALCULATION"]
        F1["totalCommissions = sum(rsAgentCommissions.price)"]
        F2["grandTotal = PriceBeforeCommissions + totalCommissions"]
        F3["agentSellPrice = grandTotal"]
    end

    FINAL_AGG --> FINAL_DET

    style LANDING_DETAIL fill:#e8f5e9
    style ACCOM_DETAIL fill:#e3f2fd
    style SUR_DETAIL fill:#fff3e0
    style RECALC_DETAIL fill:#fce4ec
    style VAL_DETAIL fill:#f8d7da
    style FINAL_DET fill:#d4edda
```

---

## 6. MA TRẬN QUYẾT ĐỊNH: MARKUP × COMMISSION

```mermaid
flowchart TB
    subgraph MATRIX["📋 MA TRẬN CÁC TRƯỜNG HỢP"]
        direction TB

        subgraph R1["HÀNG 1: DMK=true, optionCommissions='SellPrice'"]
            R1A["Markup: Mỗi service markup riêng (DMK)"]
            R1B["Commission: Aggregate từ rsAgentCommissions của từng service"]
            R1C["Kết quả: grandTotal = sum(service prices) + sum(aggregated commissions)"]
        end

        subgraph R2["HÀNG 2: DMK=true, optionCommissions='Markup/Margin'"]
            R2A["Markup: Mỗi service markup riêng (DMK)"]
            R2B["Commission: Tính từ grandTotal × (markup/100) cho từng commission"]
            R2C["Kết quả: grandTotal = PriceBeforeCommissions + compounded commissions"]
        end

        subgraph R3["HÀNG 3: DMK=false (Scheme), optionCommissions='SellPrice'"]
            R3A["Markup: MarkupAllLanding/MarkupAllHotel từ scheme"]
            R3B["Commission: Aggregate từ rsAgentCommissions của từng service"]
            R3C["Kết quả: grandTotal = sum(scheme-based prices) + sum(aggregated commissions)"]
        end

        subgraph R4["HÀNG 4: DMK=false (Scheme), optionCommissions='Markup/Margin'"]
            R4A["Markup: MarkupAllLanding/MarkupAllHotel từ scheme"]
            R4B["Commission: Tính từ grandTotal × (markup/100)"]
            R4C["Kết quả: grandTotal = PriceBeforeCommissions + compounded commissions"]
        end

        subgraph R5["HÀNG 5: keepCommissions=true (bất kể mode)"]
            R5A["Markup: Theo DMK hoặc Scheme"]
            R5B["Commission: GIỮ NGUYÊN rsAgentCommissions cũ"]
            R5C["Kết quả: grandTotal = PriceBeforeCommissions + preserved commissions"]
        end

        subgraph R6["HÀNG 6: applyCommissions=false"]
            R6A["Markup: Theo DMK hoặc Scheme"]
            R6B["Commission: TẤT CẢ VỀ 0"]
            R6C["Kết quả: grandTotal = PriceBeforeCommissions"]
        end
    end

    style R1 fill:#e8f5e9
    style R2 fill:#c8e6c9
    style R3 fill:#b2dfdb
    style R4 fill:#80cbc4
    style R5 fill:#fff9c4
    style R6 fill:#ffccbc
```

---

## 7. THỨ TỰ THỰC THI TOÀN BỘ (Sequence Diagram)

```mermaid
sequenceDiagram
    participant UI as 🖥️ UI (edit.component.ts)
    participant REFRESH as refresh_grid()
    participant RECALC as reCalculatorLandingHotel()
    participant ORCH as CalculationOrchestrationService
    participant PRICE as runPrice() / runPriceCalculator()
    participant CALC as Calculator (Group/Person/PriceBand/Accom/ACCUM)
    participant COMM as Commission Layer
    participant DB as 💾 Database

    UI->>REFRESH: Trigger (add/edit/delete service, change markup, etc.)
    REFRESH->>PRICE: runPrice(info)
    PRICE->>PRICE: runPriceCalculator(info)
    PRICE->>CALC: applyPricingLogicBasedOnServiceType()
    CALC->>CALC: Tính paxBreak[] (pricebuy, pricesell, priceBeforeCommissions)
    CALC->>CALC: runApplyCommissions() trên từng service
    CALC->>CALC: GridFooterTotalTour() → tổng hợp tất cả service
    PRICE-->>REFRESH: Hoàn thành pricing

    REFRESH->>RECALC: reCalculatorLandingHotel()
    RECALC->>ORCH: calculateFullTourTotals(context)
    ORCH->>ORCH: calculatePaxManagementLanding()
    ORCH->>ORCH: calculateHotelPassenger()
    ORCH->>ORCH: calculateSurcharges()
    ORCH->>ORCH: calculateSurchargeCommissions()
    ORCH->>ORCH: calculateAndUpdateAgentCommissions()

    alt optionCommissions = 'SellPrice'
        ORCH->>COMM: aggregateCommissionsFromServices()
        COMM->>COMM: 1. Aggregate từ Landing services
        COMM->>COMM: 2. Aggregate từ Accommodation services
        COMM->>COMM: 3. Aggregate từ Surcharges
        COMM->>COMM: 4. Compounding + Extra handling
        COMM->>COMM: 5. Validate & Fallback nếu cần
        COMM->>COMM: 6. Tính grandTotal
    else optionCommissions = 'Markup/Margin'
        ORCH->>COMM: calculateCommissionsFromGrandTotal()
        COMM->>COMM: Tính từng commission từ grandTotal × markup/margin
        COMM->>COMM: Compound effect: runningTotal += commission amount
    end

    ORCH-->>RECALC: Trả về CalculationResult
    RECALC-->>REFRESH: grandTotal, agentSellPrice

    REFRESH->>UI: Cập nhật UI
    UI->>DB: _save('update') (debounced)
```

---

## 8. CÔNG THỨC TÍNH TOÁN CHÍNH

### 8.1. Công thức Markup → Sell Price

| Loại Service | Công thức |
|---|---|
| **Group / Person** | `pricesell = pricebuy × (1 + markup/100)` |
| **PriceBand** | `pricesell = (price × (1 + markup/100) × paxCount) / minPax` |
| **Accommodation** | `unit = markup(unit_buy, markup)` → `unit_BeforeCommissions` → `runApplyCommissions()` → `unit` |
| **ACCUMULATED** | Tương tự PriceBand nhưng cộng dồn |

### 8.2. Công thức Commission (SellPrice Mode)

```
commissionPrice = basePrice × (markup/100)
```

Với **compounding**:
```
commissionPrice[i] = (PriceBeforeCommissions + Σ commissionPrice[0..i-1]) × (markup[i]/100)
```

### 8.3. Công thức Commission (Markup/Margin Mode)

```
runningTotal = PriceBeforeCommissions
commissionAmount[i] = runningTotal × (markup[i]/100)
runningTotal += commissionAmount[i]  // compound cho commission sau
grandTotal = runningTotal cuối cùng
```

### 8.4. Chuyển đổi Margin ↔ Markup

```
margin = (markup × 100) / (100 + markup)
markup = (margin × 100) / (100 - margin)
```

### 8.5. Công thức Extra Amount

```
price = calculatedPrice + extraAmount
grandTotal = PriceBeforeCommissions + Σ(price)
```

---

## 9. CÁC BIẾN QUAN TRỌNG

| Biến | Vị trí | Ý nghĩa |
|---|---|---|
| `info.DMK` | `edit.component.ts` | `true` = Direct Markup (mỗi service markup riêng), `false` = dùng Markup Scheme |
| `info.MarkupAllLanding` | `edit.component.ts` | Markup % cho tất cả landing services (Scheme mode) |
| `info.MarkupAllHotel` | `edit.component.ts` | Markup % cho tất cả accommodation services (Scheme mode) |
| `info.ExtraDirectMarkupAllLanding` | `edit.component.ts` | Extra markup cộng thêm vào landing |
| `info.ExtraDirectMarkupAllHotel` | `edit.component.ts` | Extra markup cộng thêm vào hotel |
| `info.applyCommissions` | `edit.component.ts` | `true` = có áp dụng commission |
| `info.keepCommissions` | `edit.component.ts` | `true` = giữ nguyên commission cũ, không tính lại |
| `info.optionCommissions` | `edit.component.ts` | `'SellPrice'` = aggregate từ services, khác = tính từ grandTotal |
| `info.agentCommissions[]` | `edit.component.ts` | Template commission (markup, margin, selected) |
| `info.rsAgentCommissions[]` | `edit.component.ts` | Kết quả commission đã tính (price, amount, extraAmount) |
| `info.PriceBeforeCommissions` | `edit.component.ts` | Tổng giá trước khi áp commission |
| `info.grandTotal` | `edit.component.ts` | Tổng giá cuối cùng sau commission |
| `info.agentSellPrice` | `edit.component.ts` | Giá bán cho agent (= grandTotal) |
| `service.markup` | `common.service.ts` | Markup % của từng service (DMK mode) |
| `service.markupSchemes` | `common.service.ts` | Markup scheme object (Scheme mode) |
| `service.keepDMK` | `common.service.ts` | `true` = giữ markup riêng, không bị scheme ghi đè |
| `service.ld_totalPrice` | `common.service.ts` | Tổng giá sau markup + commission |
| `service.ld_totalPriceBeforeCommissions` | `common.service.ts` | Tổng giá sau markup, trước commission |
| `service.rsAgentCommissions[]` | `common.service.ts` | Commission đã tính cho service này |

---

## 10. CÁC ĐIỂM CẦN LƯU Ý (Edge Cases)

1. **keepDMK = true**: Trong Scheme mode, nếu 1 service có `keepDMK = true`, markup của service đó sẽ không bị ghi đè bởi scheme. Dùng cho trường hợp user muốn chỉnh markup riêng cho 1 service cụ thể.

2. **Extra Amount**: Khi user nhập extra amount (commission extra dialog), `isExtraApplied = true` và `extraAmount` được lưu. Khi tính lại commission, extra amount được bảo toàn.

3. **Validation Fallback**: Nếu kết quả aggregate commission bị sai (price = 0 nhưng markup > 0, hoặc price = PriceBeforeCommissions), hệ thống sẽ fallback về saved API prices từ backend hoặc tính flat từ PriceBeforeCommissions.

4. **Per-Item Compounding**: Trong SellPrice mode, commission được tính với per-item compounding. Nghĩa là commission thứ 2 sẽ tính trên (basePrice + commission1), không phải trên basePrice gốc.

5. **Tour Leader**: Các service có tour leader sẽ có thêm `pricesellTLs`, `pricebuyTLs` trong paxBreak, được tính riêng.

6. **QuotePrice (Multiple Quotes)**: Khi tour có nhiều QuotePrice group, mỗi group có markup và commission riêng. Tổng grandTotal là sum của tất cả các group.

7. **Currency Conversion**: Tất cả giá được convert qua currency hiện tại của tour trước khi tính toán.

8. **Cancellation**: Service bị cancel sẽ có giá 0 (hoặc giá trị cancel policy nếu có).

9. **MiniTour / Excursions**: Service có miniTour (link excursions) sẽ có Items_Calculator con. Commission có thể được custom riêng cho từng child item (`hasCustomCommissions = true`).

10. **UngroupAcc**: Khi `UngroupAcc = true`, accommodation được hiển thị riêng lẻ thay vì gộp nhóm.

---

## 11. DEBUG LOGS

Các console.log quan trọng trong code để debug:

```
[DEBUG] === AGGREGATED TOTALS (before recalculation) ===
[DEBUG] PriceBeforeCommissions: ...
[DEBUG] CommissionName | aggregated: ... | markup: ...

[DEBUG] aggregateFromItemCommissions: STEP3 (calculate from source) for ...
[DEBUG] === FINAL TOTALS ===
[DEBUG] grandTotal: ... | agentSellPrice: ...
[DEBUG] === FALLBACK APPLIED - Commission prices were suspicious, recalculated ===
```
