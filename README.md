# LoanLens — EMI Calculator

> **Streamhub QA Automation Assessment — Section A**  
> Built by: [Vikram Kumar] | [vikramakumar21@gmail.com] | [https://github.com/Vikrama0103IT]

---

## Overview

LoanLens is a client-side EMI (Equated Monthly Instalment) calculator that helps users plan loan repayments. It requires no backend, no build step, and no installation — open `emi-calculator.html` in any browser and it works immediately.

---

## Table of Contents

1. [Features](#features)
2. [Tech Stack](#tech-stack)
3. [Repository Structure](#repository-structure)
4. [How to Run the Web App](#how-to-run-the-web-app)
5. [How to Run the Tests](#how-to-run-the-tests)
6. [Architecture Notes](#architecture-notes)
7. [data-testid Reference](#data-testid-reference)

---

## Features

| # | Feature | Description |
|---|---|---|
| 1 | **Dashboard hero** | Monthly EMI displayed prominently in serif font with principal, total interest, and total payable as side stats |
| 2 | **Interactive sliders** | Loan amount (₹0–₹1 Cr), interest rate (1–30%), tenure (1–30 yrs) — labels update live as you drag |
| 3 | **Tick markers** | Dot marks every ₹5L on amount, every 1% on rate, every 2 yrs on tenure — turn blue as you drag past them |
| 4 | **Animated donut chart** | SVG donut: blue = principal, amber = interest; interest % shown in centre; animates on every calculation |
| 5 | **Breakdown bars** | Proportional progress bars showing principal vs interest amounts |
| 6 | **Amortization table** | Full month-by-month schedule — EMI, principal component, interest component, remaining balance |
| 7 | **Responsive layout** | Two-column desktop grid → single-column mobile stack (≤ 800 px) |
| 8 | **Reduced motion** | All CSS transitions disabled when `prefers-reduced-motion` is set |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Web app | Vanilla HTML, CSS, JavaScript — single file |
| Fonts | Google Fonts: DM Serif Display + Inter |
| Chart | Raw SVG (`stroke-dasharray` / `stroke-dashoffset`) — no library |
| UI tests | Playwright + Cucumber (BDD) + TypeScript |
| API tests | Playwright Test |
| Language | TypeScript |

---

## Repository Structure

```
loanlens/
│
├── emi-calculator.html          # ← Web application (A1)
├── README.md                    # ← This file
├── SETUP_STEPS.md               # ← Step-by-step GitHub publishing guide
├── .env.example                 # ← Environment variable template
├── playwright.config.ts         # ← Playwright configuration
├── package.json
│
├── tests/                       # ← UI test suite (A2)
│   ├── features/
│   │   └── emi-calculator.feature
│   ├── step-definitions/
│   │   └── emi.steps.ts
│   └── pages/
│       └── EmiCalculatorPage.ts
│
├── api-tests/                   # ← API test suite (A3)
│   └── jsonplaceholder.spec.ts
│
├── sql/                         # ← SQL queries + screenshots (A4)
│   ├── schema.sql
│   ├── round_trip_transfers.sql
│   ├── ipl_streaks.sql
│   └── screenshots/
│       ├── round_trip_result.png
│       └── ipl_streak_result.png
│
├── self-healing/                # ← AI self-healing exercise
│   └── SELF_HEALING.md
│
└── test-results/                # ← Committed test execution results
    ├── index.html               # Playwright HTML report
    └── screenshots/
```

---

## How to Run the Web App

No installation needed — just open the file:

```bash
# Clone the repository
git clone https://github.com/<your-username>/loanlens.git
cd loanlens

# macOS
open emi-calculator.html

# Linux
xdg-open emi-calculator.html

# Windows
start emi-calculator.html
```

Or drag `emi-calculator.html` directly into any browser tab.

---

## How to Run the Tests

### Prerequisites

- Node.js 18+
- npm

### Install dependencies

```bash
npm install
npx playwright install chromium
```

### Set environment variable

```bash
cp .env.example .env
# Edit .env and set BASE_URL to the absolute path of emi-calculator.html
```

### Run UI tests (A2)

```bash
npx playwright test tests/
```

### Run API tests (A3)

```bash
npx playwright test api-tests/
```

### Run all tests with HTML report

```bash
npx playwright test --reporter=html
npx playwright show-report
```

> Test results (HTML report + screenshots) are committed to `test-results/` so reviewers can inspect them without running the suite.

---

## Architecture Notes

### EMI Calculation Formula

Standard reducing-balance method:

```
EMI = P × r × (1 + r)ⁿ / ((1 + r)ⁿ – 1)

Where:
  P = Principal loan amount
  r = Monthly interest rate (annual rate / 12 / 100)
  n = Total months (years × 12)
```

Edge case handled: when `r = 0`, formula reduces to `EMI = P / n` to avoid division by zero.

### Donut Chart

Built with raw SVG — no charting library. Two `<circle>` elements use `stroke-dasharray` to render the principal arc (blue) and interest arc (amber). The second arc is offset using `stroke-dashoffset` so it begins exactly where the first ends. Both animate via CSS `transition`.

### Tick Marks

`renderTicks()` builds absolutely-positioned `<div>` elements inside a `.ticks` container. Positioning uses `offsetWidth` to map slider values to pixel coordinates. Deferred to `requestAnimationFrame` to ensure layout is painted before reading `offsetWidth`.

