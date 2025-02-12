# **Battery Arbitrage Optimization in DAM & RTM**

## **Introduction**
This project optimizes battery arbitrage in both the **Day-Ahead Market (DAM)** and **Real-Time Market (RTM)**. The model follows a **rolling-horizon multi-stage optimization**, continuously updating predictions and schedules as new information becomes available.

## **Optimization Stages**
1. **10:00 (D-1): DAM & RTM Pre-optimization**
   - Predict **DAM** and **RTM** prices for the entire delivery day.
   - Optimize bids for **DAM**, considering potential profits from **RTM**.
   
2. **13:30 (D-1): DAM Cleared Results Update**
   - DAM prices and **accepted schedules** are now known.
   - Update battery commitment and available flexibility for RTM.
   - Update RTM price predictions and re-optimize.

3. **22:00 (D-1) Onward: RTM Optimization (Rolling Horizon)**
   - **RTM market opens**: Optimize for the next 48 time blocks (00:30 to 24:00).
   - Every **30 minutes**, update:
     - RTM cleared schedules.
     - Battery **state-of-charge (SoC)**.
     - New RTM price predictions.
   - Re-optimize for the next 30-minute window while considering future profits.

## **Mathematical Formulation**

### **Objective Function**
The goal is to maximize total arbitrage profit from both DAM and RTM:

$$
\max \sum_{t=0}^{95} \Delta t_{DAM} \left( P_t^{DAM} \cdot d_t - P_t^{DAM} \cdot c_t \right) + \sum_{t=0}^{47} \Delta t_{RTM} \left( P_t^{RTM} \cdot d_t - P_t^{RTM} \cdot c_t \right)
$$

where:
- \(P_t^{DAM}, P_t^{RTM}\) = Market prices in DAM and RTM.
- \(c_t, d_t\) = Charging and discharging power (MW).
- \(\Delta t_{DAM} = 15\) min, \(\Delta t_{RTM} = 30\) min.

### **Constraints**
1. **Battery State-of-Charge (SoC) Dynamics**

$$
 x_{t+1} = x_t + \Delta t \cdot \left( \eta_c \cdot c_t - \frac{d_t}{\eta_d} \right)
$$

2. **Charging/Discharging Limits**

$$
0 \leq c_t \leq y_t \cdot P_{ch}^{max}, \quad 10 \cdot y_t \leq c_t
$$

$$
0 \leq d_t \leq z_t \cdot P_{dis}^{max}, \quad 10 \cdot z_t \leq d_t
$$

where \( y_t, z_t \) are binary variables indicating whether charging or discharging occurs.

3. **Mutual Exclusivity**

$$
y_t + z_t \leq 1 \quad \forall t
$$

4. **Respect DAM Commitments**
   - DAM schedules must be followed.
   - RTM **cannot override DAM commitments**.
   
5. **RTM Rolling Horizon**
   - Every 30 minutes, update:
     - RTM price predictions.
     - Cleared results from the previous RTM auction.
     - Battery SoC.

## **Code Structure**

### **1. Inputs & Forecasting Functions**
- `forecast_DAM_nextday()`: Predicts DAM & RTM prices for day D.
- `update_DAM_results()`: Updates DAM cleared results at 13:30.
- `update_RTM_predictions()`: Updates rolling RTM forecasts every 30 min.
- `update_RTM_results()`: Fetches RTM cleared schedules.

### **2. DAM Optimization (10:00 D-1)**
- Uses predicted DAM & RTM prices to optimize bids.
- Ensures battery flexibility for potential RTM profits.
- Solves Mixed Integer Linear Program (MILP).

### **3. RTM Rolling Optimization (From 22:00 D-1)**
- At **each 30-min interval**:
  - Fetch **RTM clearing results** and **update SoC**.
  - Fetch **new RTM predictions**.
  - Re-optimize the remaining period.

## **Logging & Monitoring**
- **Detailed logs for each RTM interval**, including:
  - RTM cleared schedules.
  - SoC updates.
  - New RTM price forecasts.
  - Updated bids for the next time block.

## **Conclusion**
This framework ensures **profit maximization across both DAM and RTM**, using real-time updates and a rolling-horizon strategy. The battery is dynamically scheduled while respecting DAM commitments and leveraging RTM flexibility.
