# Battery Arbitrage Optimization with DAM and RTM

## 1. Introduction
This document describes the optimization model for battery arbitrage in both the **Day-Ahead Market (DAM)** and **Real-Time Market (RTM)** while accounting for battery degradation costs. The optimization is implemented as a **rolling-horizon, multi-stage MILP** (Mixed-Integer Linear Programming) model.

## 2. Mathematical Formulation

### 2.1 Objective Function
The goal is to **maximize total arbitrage profit** over the optimization horizon, considering both DAM and RTM profits while accounting for battery degradation costs.

#### DAM Optimization (Runs at 10:00 Day-1)
$$
\text{Profit}_{DAM} = \sum_{t=0}^{95} \Delta t \left( P^{DAM}_t \cdot d_t - P^{DAM}_t \cdot c_t \right) - \sum_{t=0}^{95} C_{deg}(c_t, d_t)
$$

#### RTM Optimization (Starts at 22:00 Day-1 and runs every 30 min)
$$
\text{Profit}_{RTM} = \sum_{t=0}^{47} \Delta t \left( P^{RTM}_t \cdot d_t - P^{RTM}_t \cdot c_t \right) - \sum_{t=0}^{47} C_{deg}(c_t, d_t)
$$

where:
- **$P^{DAM}_t$**, **$P^{RTM}_t$** are market prices at time $t$
- **$c_t$**, **$d_t$** are charging and discharging decisions
- **$C_{deg}(c_t, d_t)$** is the degradation cost, computed using the **Rainflow algorithm**

### 2.2 Constraints

#### 1. Battery Energy Balance
$$
x_{t+1} = x_t + \eta_c c_t - \frac{d_t}{\eta_d}
$$
- **$x_t$**: State-of-Charge (SoC)
- **$\eta_c, \eta_d$**: Charging and discharging efficiency

#### 2. Power Limits
$$
0 \leq c_t \leq P_{ch}^{max} \cdot y_t
$$
$$
0 \leq d_t \leq P_{dis}^{max} \cdot z_t
$$

#### 3. Mutually Exclusive Charging/Discharging
$$
y_t + z_t \leq 1
$$

#### 4. DAM Schedules are Fixed Once Cleared
After 13:30 on **Day-1**, DAM schedules are known and cannot be changed:
$$
c_t^{DAM} = c_t^{committed}, \quad d_t^{DAM} = d_t^{committed}
$$

#### 5. RTM Participation Constraints
Battery flexibility for RTM depends on available capacity after honoring DAM commitments.
$$
c_t^{RTM} + c_t^{DAM} \leq P_{ch}^{max}, \quad d_t^{RTM} + d_t^{DAM} \leq P_{dis}^{max}
$$

## 3. Battery Degradation Modeling (Rainflow Algorithm)
To model degradation, we apply **Rainflow Counting** to extract battery cycles from SoC profiles. The degradation cost function is:
$$
C_{deg} = \sum_{j} \left( \alpha \cdot E_j \cdot f_j \right)
$$
where:
- **$E_j$** is the energy throughput of cycle $j$
- **$f_j$** is the cycle frequency
- **$\alpha$** is the degradation cost per cycle

## 4. Execution Timeline
The optimization runs in sequential steps:

| Time | Event |
|------|-------|
| 10:00 D-1 | DAM optimization using **forecasted DAM & RTM prices** |
| 13:30 D-1 | DAM schedules are **finalized** & RTM forecasts updated |
| 22:00 D-1 | RTM optimization for **next 24 hours** starts |
| Every 30 min | RTM optimization updates based on **new forecasts & cleared bids** |

## 5. Plots & Visualization
The final outputs include:
- **DAM & RTM Price Trajectories**
- **Battery SoC Profile**
- **DAM & RTM Bidding Schedules**
- **Cycle Depth Distribution (Rainflow Analysis)**

---
This framework ensures optimal battery participation in **both markets** while managing degradation costs effectively.
