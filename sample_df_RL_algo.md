# Privacy Preservation Mechanisms in Demand Flexibility Control

## 1. Differential Privacy Framework

### 1.1 Local Differential Privacy Definition
For a mechanism $\mathcal{M}$ and privacy parameter $\epsilon$, the $\epsilon$-local differential privacy guarantee is:

$$\Pr[\mathcal{M}(s_t) \in \mathcal{O}] \leq e^\epsilon \cdot \Pr[\mathcal{M}(s_t') \in \mathcal{O}]$$

for all possible outputs $\mathcal{O}$ and any two adjacent states $s_t, s_t'$ differing in one occupant's data.

### 1.2 Sensitivity Analysis
The local sensitivity of each private state component:

$$\Delta_T = \sup_{s_t, s_t'} \|T_{in,t} - T_{in,t}'\|_2$$ 
$$\Delta_o = \sup_{s_t, s_t'} \|o_t - o_t'\|_2 = 1$$

Global sensitivity for the combined private state:

$$\Delta_S = \sqrt{\Delta_T^2 + \Delta_o^2}$$

## 2. Multi-Level Privacy Protection

### 2.1 Input Perturbation
Apply calibrated Gaussian noise to sensitive measurements:

$$\tilde{T}_{in,t} = T_{in,t} + \mathcal{N}(0, \sigma_T^2)$$
$$\tilde{o}_t = o_t + \mathcal{N}(0, \sigma_o^2)$$

where noise scales are:

$$\sigma_T = \frac{\Delta_T \sqrt{2\ln(1.25/\delta)}}{\epsilon_T}$$
$$\sigma_o = \frac{\Delta_o \sqrt{2\ln(1.25/\delta)}}{\epsilon_o}$$

### 2.2 Feature Transformation
Apply dimensionality reduction with privacy guarantee:

$$z_t = \Pi(s_t^p) + \mathcal{N}(0, \Sigma)$$

where:
- $\Pi$: Random projection matrix
- $\Sigma$: Covariance matrix for privacy preservation

Privacy guarantee for projection:

$$\epsilon_{proj} = \frac{\|\Pi\|_F^2}{2\sigma_{min}^2(\Sigma)}$$

### 2.3 Temporal Privacy
Implement sliding window mechanism:

$$\tilde{s}_t^p = \frac{1}{w}\sum_{i=t-w+1}^t \alpha_i s_i^p + \mathcal{N}(0, \sigma_w^2I)$$

where:
- $w$: Window size
- $\alpha_i$: Decay factors with $\sum \alpha_i = 1$
- $\sigma_w$: Noise scale for temporal aggregation

## 3. Dynamic Privacy Budget Allocation

### 3.1 Budget Management
Total privacy budget constraint:

$$\epsilon_{total} = \epsilon_T + \epsilon_o + \epsilon_{proj} + \epsilon_w \leq \epsilon_{max}$$

Track budget consumption:

$$\epsilon_{remain,t+1} = \epsilon_{remain,t} - \epsilon_t$$

where per-step budget is allocated based on sensitivity:

$$\epsilon_t = \min\{\epsilon_{base}, \epsilon_{remain,t} \cdot \beta_t\}$$

### 3.2 Adaptive Budget Allocation
Dynamic allocation factor:

$$\beta_t = \frac{\exp(-\lambda \|s_t - s_{t-1}\|_2)}{\sum_{i=1}^T \exp(-\lambda \|s_i - s_{i-1}\|_2)}$$

Privacy budget optimization:

$$\epsilon_t^* = \arg\max_{\epsilon_t} \mathbb{E}[r_t | \epsilon_t] \text{ s.t. } \epsilon_t \leq \epsilon_{remain,t}$$

## 4. Privacy-Preserving Training Mechanisms

### 4.1 Gradient Privacy
Apply gradient perturbation during training:

$$\tilde{\nabla}_\theta = \nabla_\theta + \mathcal{N}(0, \sigma_g^2I)$$

where:

$$\sigma_g = \frac{C\sqrt{2\ln(1.25/\delta)}}{\epsilon_g}$$

with clipping threshold $C$.

### 4.2 Loss Function Privacy
Modify critic loss to include privacy penalty:

$$\mathcal{L}_{priv} = \mathcal{L}_{TD} + \lambda_{priv}\|\nabla_\theta \log \pi_\theta(a_t|s_t)\|_2^2$$

### 4.3 Batch Privacy
Implement private mini-batch sampling:

$$\Pr[i \in \mathcal{B}_t] \propto \exp\left(\frac{-\epsilon_b}{2\Delta_b} d(s_i, \bar{s})\right)$$

where:
- $d(s_i, \bar{s})$: Distance to batch mean
- $\Delta_b$: Batch sensitivity
- $\epsilon_b$: Per-batch privacy budget

## 5. Privacy Metrics and Monitoring

### 5.1 Privacy Loss Tracking
Track approximate privacy loss using Rényi Differential Privacy:

$$\text{RDP}_\alpha(\mathcal{M}) = \frac{1}{\alpha-1}\log\mathbb{E}_{x\sim\mathcal{M}(D)}\left[\left(\frac{\mathcal{M}(D)(x)}{\mathcal{M}(D')(x)}\right)^\alpha\right]$$

### 5.2 Privacy Score Calculation
Compute instantaneous privacy score:

$$\text{PrivScore}_t = \exp\left(-\frac{\epsilon_{used,t}}{\epsilon_{max}}\right) \cdot \left(1 - \frac{\|\nabla_\theta \log \pi_\theta(a_t|s_t)\|_2}{C}\right)$$

### 5.3 Privacy Monitoring
Monitor privacy leakage through mutual information:

$$I(S_t^p; Z_t) \leq \frac{\epsilon^2}{8} + \frac{\epsilon}{2}$$

Implement early stopping when:

$$\text{PrivScore}_t < \text{threshold} \text{ or } \epsilon_{remain,t} < \epsilon_{min}$$

## 6. Implementation Guidelines

### 6.1 Privacy Parameter Selection
Choose parameters satisfying:

$$\epsilon_{total} = \sum_{t=1}^T \epsilon_t \leq \epsilon_{target}$$
$$\delta \leq \frac{1}{|\mathcal{D}|}$$

where $|\mathcal{D}|$ is the size of the private dataset.

### 6.2 Composition Rules
Apply advanced composition theorem:

For $k$ mechanisms, each $(\epsilon, \delta)$-DP, the composition is $(O(\sqrt{k\log(1/\delta')}\epsilon), k\delta + \delta')$-DP.

### 6.3 Privacy-Utility Trade-off
Optimize utility under privacy constraints:

$$\max_{\theta} \mathbb{E}[R(\theta)] \text{ subject to } \epsilon_{total} \leq \epsilon_{max}$$

where $R(\theta)$ is the expected return under policy parameters $\theta$.
