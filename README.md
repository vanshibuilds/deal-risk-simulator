# deal-risk-simulator
Interactive appraisal engine for recurring-revenue deals — NPV, scenarios, Monte Carlo risk simulation

An algorithmic, client-side financial engineering tool designed to stress-test
commercial investment contracts and quantify down-tail risk.


🔗 [**Live Interactive Demo**](https://vanshibuilds.github.io/index/)

## 🚀 Key Architectural Features
* **Asymmetric Monte Carlo Engine:** Runs 4,000 parallel iterations utilizing skewed triangular and normal distributions to isolate value-destructive risks.
* **Numerical Bisection Solver:** Implements client-side bisection search algorithms to calculate annualized internal rate of return (IRR) and multi-variable break-evens dynamically.
* **State Preservation:** Features complete JSON deal state serialization allowing full session exports and re-imports.

## 🛠️ Tech Stack
* Vanilla JS (ES6+) for numerical calculation engines.
* Chart.js for data visualization mapping.
* Pure client-side execution for zero-latency slider updates.
