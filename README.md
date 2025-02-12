# Battery Bidding Optimization – Detailed Explanation

This document describes an algorithm that handles battery bidding in two sequential markets: the Day-Ahead Market (DAM) and the Real-Time Market (RTM). The algorithm includes:

- **DAM Bidding (10:00 D-1):**  
  Optimize bids for all 96 blocks (15-minute resolution) of the delivery day using forecasted DAM prices.

- **DAM Update (13:00 D-1):**  
  Update the DAM schedule based on actual schedules and known DAM prices.

- **RTM Bidding (from 22:00 D-1 until 23:00 D):**  
  Run a rolling optimization every 30 minutes. At each update, new RTM price forecasts are provided and the battery's state-of-charge (SoC) is updated before optimizing the bid for the next 30-minute block.

---

## 1. Day-Ahead Market (DAM) Optimization

### **Time Resolution**

- **Blocks:** 96 blocks (15-minute intervals over 24 hours).
- **Time Step:**  
  $$\Delta t_{DAM} = 0.25 \text{ hours}.$$

### **Objective Function**

The goal is to maximize the arbitrage profit over the entire DAM period. The objective is defined as:

$$
\text{Profit}_{DAM} = \sum_{t=0}^{95} \Delta t_{DAM} \left( P_t \cdot d_t - P_t \cdot c_t \right)
$$

Where:
- $$P_t$$ is the forecasted DAM price (in \$/MWh) at time block $$t$$.
- $$c_t$$ is the charging power (MW) at time block $$t$$.
- $$d_t$$ is the discharging power (MW) at time block $$t$$.

### **Constraints**

1. **Initial State-of-Charge (SoC):**

   $$x_0 = x_{\text{initial}},$$

   where, for example, $$x_{\text{initial}} = 500 \text{ MWh}.$$

2. **Battery Dynamics (SoC Update):**

   For each time block $$t = 0, 1, \ldots, 95$$:

   $$
   x_{t+1} = x_t + \Delta t_{DAM} \left( \eta_c \cdot c_t - \frac{1}{\eta_d} \cdot d_t \right)
   $$

   Here, $$\eta_c$$ and $$\eta_d$$ are the charging and discharging efficiencies, respectively.

3. **Charging Power Limits:**

   With a binary decision variable $$y_t$$ indicating a charging bid:
   - If $$y_t = 1$$:
     
     $$
     10 \leq c_t \leq P_{ch}^{max}
     $$
     
   - If $$y_t = 0$$ then $$c_t = 0.$$

4. **Discharging Power Limits:**

   With a binary decision variable $$z_t$$ for discharging:
   - If $$z_t = 1$$:
     
     $$
     10 \leq d_t \leq P_{dis}^{max}
     $$
     
   - If $$z_t = 0$$ then $$d_t = 0.$$

5. **Mutual Exclusivity:**

   The battery cannot charge and discharge simultaneously:

   $$
   y_t + z_t \leq 1 \quad \text{for all } t.
   $$

---

## 2. DAM Schedule Update (13:00 D-1)

At 13:00 on the day before delivery, the actual DAM schedules and prices become known. The previously optimized DAM schedule is updated (overridden) using these actual values. This is achieved by re-running the DAM optimization with the updated price data and/or actual dispatch schedules.

---

## 3. Real-Time Market (RTM) Optimization

### **Time Resolution**

- **Blocks:** 48 blocks (30-minute intervals over 24 hours).
- **Time Step:**  
  $$\Delta t_{RTM} = 0.5 \text{ hours}.$$

### **Objective Function**

For each 30-minute block in RTM, the optimization maximizes the profit for that block:

$$
\text{Profit}_{RTM,t} = \Delta t_{RTM} \left( P^{RT}_t \cdot d^{RT}_t - P^{RT}_t \cdot c^{RT}_t \right)
$$

Where:
- $$P^{RT}_t$$ is the RTM price forecast for block $$t$$.
- $$c^{RT}_t$$ and $$d^{RT}_t$$ are the charging and discharging decisions for that 30-minute interval.

### **RTM Constraints**

The RTM constraints mirror those in DAM:
1. **Charging Limits:**  
   If $$y^{RT}_t = 1$$, then
   $$
   10 \leq c^{RT}_t \leq P_{ch}^{max}.
   $$
2. **Discharging Limits:**  
   If $$z^{RT}_t = 1$$, then
   $$
   10 \leq d^{RT}_t \leq P_{dis}^{max}.
   $$
3. **Mutual Exclusivity:**  
   $$
   y^{RT}_t + z^{RT}_t \leq 1.
   $$

After each RTM block optimization, the battery’s SoC is updated as follows:

$$
x_{\text{new}} = x_{\text{prev}} + \Delta t_{RTM} \left( \eta_c \cdot c^{RT}_t - \frac{1}{\eta_d} \cdot d^{RT}_t \right).
$$

RTM optimization runs every 30 minutes (rolling update) until 23:00 on the day of delivery. At each update stage, the RTM price predictions are refreshed, and the optimization uses the updated SoC.

---

## 4. Code Structure Explanation

### **DAM Optimization (10:00 D-1)**
- **Inputs:** Forecasted DAM prices for 96 blocks (15-minute resolution).
- **Decision Variables:**  
  - $$c_t$$: Charging power.  
  - $$d_t$$: Discharging power.  
  - $$y_t$$, $$z_t$$: Binary variables for charging/discharging decisions.  
  - $$x_t$$: Battery state-of-charge.
- **Objective:**  
  $$\max \sum_{t=0}^{95} \Delta t_{DAM} (P_t \cdot d_t - P_t \cdot c_t).$$
- **Constraints:**  
  - Initial SoC.  
  - SoC dynamics: $$x_{t+1} = x_t + \Delta t_{DAM} \left( \eta_c c_t - \frac{1}{\eta_d} d_t \right).$$  
  - Charging/Discharging limits and mutual exclusivity.

### **DAM Schedule Update (13:00 D-1)**
- **Process:**  
  At 13:00, the DAM optimization is updated using actual price and dispatch data. This re-optimizes the schedule for the remaining blocks of the day.

### **RTM Optimization (22:00 D-1 Onwards)**
- **Inputs:** RTM price forecasts for 48 blocks (30-minute resolution).
- **Procedure:**  
  - For each 30-minute block, a separate MILP is formulated with RTM prices.
  - The objective is to maximize profit for that block.
  - After solving, the battery’s SoC is updated using the dispatch result:
    
    $$
    x_{\text{new}} = x_{\text{prev}} + \Delta t_{RTM} \left( \eta_c \cdot c^{RT}_t - \frac{1}{\eta_d} \cdot d^{RT}_t \right).
    $$
    
  - The RTM price forecast is refreshed for the next block.
- **Outcome:**  
  This rolling optimization continues every 30 minutes until 23:00 on the delivery day, ensuring the bids are dynamically adjusted in real time.

### **Plotting**
- **DAM Plot:**  
  Displays the battery’s state-of-charge trajectory over the DAM period.
- **RTM Plot:**  
  Shows the RTM price forecasts or results over the RTM period.

---

## Conclusion

This integrated framework enables optimal bidding for both DAM and RTM:
- The **DAM optimization** sets the initial schedule based on forecasted prices.
- **DAM schedule updates** incorporate actual market data when available.
- **RTM bidding** uses rolling 30-minute optimizations to continuously update the battery dispatch and bidding strategy up to the delivery time.

This approach maximizes arbitrage profit while respecting battery operational constraints (e.g., charging/discharging limits, SoC dynamics) in both market environments.

Feel free to adjust the parameters or constraints to match your specific market conditions or battery characteristics.
````markdown

