# Setup & Publishing Guide

Step-by-step instructions to set up the project locally, run the tests, and publish everything to GitHub.

---

## Prerequisites

Make sure the following are installed before starting:

| Tool | Minimum version | Download |
|---|---|---|
| Git | Any recent | https://git-scm.com |
| Node.js | 18.0 or higher | https://nodejs.org |
| A GitHub account | — | https://github.com |

---

## Part 1 — Local Project Setup

### Step 1 · Create the project folder

```bash
mkdir loanlens
cd loanlens
```

Place the following files inside this folder:

- `emi-calculator.html`
- `README.md`
- `SETUP_STEPS.md`

---

### Step 2 · Initialise Git

```bash
git init
git add .
git commit -m "feat: initial commit — LoanLens EMI calculator (A1)"
```

---

### Step 3 · Initialise Node and install dependencies

```bash
npm init -y

npm install --save-dev \
  @playwright/test \
  @cucumber/cucumber \
  ts-node \
  typescript \
  dotenv

npx playwright install chromium
```

---

### Step 4 · Create the folder structure

```bash
mkdir -p tests/features
mkdir -p tests/step-definitions
mkdir -p tests/pages
mkdir -p api-tests
mkdir -p sql/screenshots
mkdir -p self-healing
mkdir -p test-results/screenshots
```

---

### Step 5 · Create `playwright.config.ts`

Create this file in the project root:

```ts
// playwright.config.ts
import { defineConfig } from '@playwright/test';
import * as path from 'path';
import * as dotenv from 'dotenv';
dotenv.config();

export default defineConfig({
  testDir: './tests',
  timeout: 30_000,
  reporter: [
    ['html', { outputFolder: 'test-results' }],
    ['list'],
  ],
  use: {
    baseURL: process.env.BASE_URL ??
      `file://${path.resolve(__dirname, 'emi-calculator.html')}`,
    screenshot: 'only-on-failure',
    video: 'off',
  },
});
```

---

### Step 6 · Create `.env` and `.env.example`

```bash
# Create .env with your actual path
echo 'BASE_URL=file:///absolute/path/to/loanlens/emi-calculator.html' > .env

# Create a safe template to commit
echo 'BASE_URL=file:///absolute/path/to/loanlens/emi-calculator.html' > .env.example

# Make sure .env is never committed
echo '.env' >> .gitignore

git add .gitignore .env.example
git commit -m "chore: add gitignore and env template"
```

> For a deployed version on GitHub Pages, use:  
> `BASE_URL=https://<username>.github.io/loanlens/emi-calculator.html`

---

## Part 2 — Writing the Tests

### Step 7 · Page Object Model

Create `tests/pages/EmiCalculatorPage.ts`:

```ts
import { Page, expect } from '@playwright/test';

export class EmiCalculatorPage {
  constructor(private page: Page) {}

  // --- Locators (all data-testid — stable across styling changes) ---
  sliderPrincipal = () => this.page.getByTestId('slider-principal');
  sliderRate      = () => this.page.getByTestId('slider-rate');
  sliderTenure    = () => this.page.getByTestId('slider-tenure');
  btnCalculate    = () => this.page.getByTestId('btn-calculate');
  emiResult       = () => this.page.getByTestId('emi-result');
  statPrincipal   = () => this.page.getByTestId('stat-principal');
  statInterest    = () => this.page.getByTestId('stat-interest');
  statTotal       = () => this.page.getByTestId('stat-total');
  donutChart      = () => this.page.getByTestId('donut-chart');
  donutPct        = () => this.page.getByTestId('donut-interest-pct');
  bdPrincipal     = () => this.page.getByTestId('bd-principal');
  bdInterest      = () => this.page.getByTestId('bd-interest');
  amortTable      = () => this.page.getByTestId('amortization-table');
  tableMeta       = () => this.page.getByTestId('table-meta');
  tableRow        = (n: number) => this.page.getByTestId(`row-month-${n}`);

  // --- Actions ---
  async goto(url: string) {
    await this.page.goto(url);
  }

  async setSlider(testId: string, value: number) {
    await this.page.getByTestId(testId).evaluate(
      (el: HTMLInputElement, val) => {
        el.value = String(val);
        el.dispatchEvent(new Event('input', { bubbles: true }));
      },
      value
    );
  }

  async calculate() {
    await this.btnCalculate().click();
  }

  // --- Helper: independent EMI formula for assertion comparison ---
  computeEMI(principal: number, annualRate: number, years: number): number {
    const r = annualRate / 12 / 100;
    const n = years * 12;
    if (r === 0) return principal / n;
    return (principal * r * Math.pow(1 + r, n)) / (Math.pow(1 + r, n) - 1);
  }
}
```

---

### Step 8 · Feature file (BDD scenarios)

Create `tests/features/emi-calculator.feature`:

```gherkin
Feature: EMI Calculator

  Background:
    Given I open the EMI calculator

  Scenario: Dashboard loads with all input controls visible
    Then the calculate button should be visible
    And the loan amount slider should be visible
    And the interest rate slider should be visible
    And the tenure slider should be visible

  Scenario: EMI output matches the expected formula result
    When I set the loan amount to 1000000
    And  I set the interest rate to 8.5
    And  I set the tenure to 10
    And  I click Calculate
    Then the monthly EMI should be approximately 12399
    And  the total interest stat should not be empty
    And  the total payable stat should not be empty

  Scenario: Donut chart is visible and shows non-zero interest percentage
    When I set the loan amount to 500000
    And  I set the interest rate to 10
    And  I set the tenure to 5
    And  I click Calculate
    Then the donut chart should be visible
    And  the interest percentage should be greater than 0

  Scenario: Amortization table row count matches the loan tenure
    When I set the loan amount to 200000
    And  I set the interest rate to 7
    And  I set the tenure to 2
    And  I click Calculate
    Then the amortization table should have 24 rows
    And  the table metadata should mention 24 monthly payments
```

---

### Step 9 · Step definitions

Create `tests/step-definitions/emi.steps.ts`:

```ts
import { Given, When, Then, Before, After } from '@cucumber/cucumber';
import { chromium, Browser, Page, expect } from '@playwright/test';
import { EmiCalculatorPage } from '../pages/EmiCalculatorPage';
import * as dotenv from 'dotenv';
import * as path from 'path';
dotenv.config();

let browser: Browser;
let page: Page;
let emiPage: EmiCalculatorPage;

const BASE_URL =
  process.env.BASE_URL ??
  `file://${path.resolve(__dirname, '../../emi-calculator.html')}`;

Before(async () => {
  browser = await chromium.launch();
  page    = await browser.newPage();
  emiPage = new EmiCalculatorPage(page);
});

After(async () => {
  await browser.close();
});

// ── Given ──────────────────────────────────────────────────────────────────
Given('I open the EMI calculator', async () => {
  await emiPage.goto(BASE_URL);
});

// ── When ───────────────────────────────────────────────────────────────────
When('I set the loan amount to {int}', async (val: number) => {
  await emiPage.setSlider('slider-principal', val);
});
When('I set the interest rate to {float}', async (val: number) => {
  await emiPage.setSlider('slider-rate', val);
});
When('I set the tenure to {int}', async (val: number) => {
  await emiPage.setSlider('slider-tenure', val);
});
When('I click Calculate', async () => {
  await emiPage.calculate();
});

// ── Then ───────────────────────────────────────────────────────────────────
Then('the calculate button should be visible', async () => {
  await expect(emiPage.btnCalculate()).toBeVisible();
});
Then('the loan amount slider should be visible', async () => {
  await expect(emiPage.sliderPrincipal()).toBeVisible();
});
Then('the interest rate slider should be visible', async () => {
  await expect(emiPage.sliderRate()).toBeVisible();
});
Then('the tenure slider should be visible', async () => {
  await expect(emiPage.sliderTenure()).toBeVisible();
});

Then('the monthly EMI should be approximately {int}', async (expected: number) => {
  const text      = (await emiPage.emiResult().textContent()) ?? '';
  const actual    = parseInt(text.replace(/[^\d]/g, ''), 10);
  const tolerance = expected * 0.01; // allow ±1%
  expect(Math.abs(actual - expected)).toBeLessThan(tolerance);
});

Then('the total interest stat should not be empty', async () => {
  const text = await emiPage.statInterest().textContent();
  expect(text?.trim()).not.toBe('—');
});
Then('the total payable stat should not be empty', async () => {
  const text = await emiPage.statTotal().textContent();
  expect(text?.trim()).not.toBe('—');
});

Then('the donut chart should be visible', async () => {
  await expect(emiPage.donutChart()).toBeVisible();
});
Then('the interest percentage should be greater than {int}', async (min: number) => {
  const text = (await emiPage.donutPct().textContent()) ?? '0';
  const pct  = parseInt(text.replace('%', ''), 10);
  expect(pct).toBeGreaterThan(min);
});

Then('the amortization table should have {int} rows', async (expected: number) => {
  const count = await emiPage
    .amortTable()
    .locator('tbody tr[data-testid]')
    .count();
  expect(count).toBe(expected);
});
Then('the table metadata should mention {int} monthly payments', async (months: number) => {
  const text = await emiPage.tableMeta().textContent();
  expect(text).toContain(`${months} monthly payments`);
});
```

---

### Step 10 · API test (A3)

Create `api-tests/jsonplaceholder.spec.ts`:

```ts
import { test, expect } from '@playwright/test';

const ENDPOINT = 'https://jsonplaceholder.typicode.com/posts';

test.describe('POST /posts — boundary and invalid input validation', () => {

  test('TC-01: excessively long title — mock accepts, real API should return 400', async ({ request }) => {
    const response = await request.post(ENDPOINT, {
      data: { userId: 1, title: 'A'.repeat(10_000), body: 'test body' },
    });
    expect(response.status()).toBe(201);
    const body = await response.json();
    expect(body).toHaveProperty('id');
  });

  test('TC-02: special characters in title — mock accepts, real API should sanitize or reject', async ({ request }) => {
    const response = await request.post(ENDPOINT, {
      data: { userId: 1, title: '🔥 <script>alert(1)</script> \x00\xFF', body: 'body' },
    });
    expect(response.status()).toBe(201);
    expect(await response.json()).toHaveProperty('id');
  });

  test('TC-03: missing required userId — mock returns 201, real API should return 422', async ({ request }) => {
    const response = await request.post(ENDPOINT, {
      data: { title: 'Post without userId', body: 'body content' },
    });
    expect(response.status()).toBe(201);
    const body = await response.json();
    expect(body).not.toHaveProperty('userId');
  });

  test('TC-04: completely empty request body — mock returns 201, real API should return 400', async ({ request }) => {
    const response = await request.post(ENDPOINT, { data: {} });
    expect(response.status()).toBe(201);
  });

  test('TC-05: null values for all fields — mock accepts gracefully without server error', async ({ request }) => {
    const response = await request.post(ENDPOINT, {
      data: { userId: null, title: null, body: null },
    });
    expect(response.status()).toBe(201);
    expect(await response.json()).toHaveProperty('id');
  });

});
```

---

## Part 3 — Run Tests and Capture Results

### Step 11 · Run the full test suite

```bash
# Run UI tests
npx playwright test tests/ --reporter=html

# Run API tests
npx playwright test api-tests/ --reporter=html

# Open the HTML report in your browser
npx playwright show-report test-results/
```

---

### Step 12 · Commit the test results

```bash
git add test-results/
git commit -m "test: add Playwright HTML report and execution results"
```

> **Important:** The assessment requires test results to be committed — not just the test code.

---

## Part 4 — Publish to GitHub

### Step 13 · Create a new GitHub repository

1. Go to https://github.com and sign in
2. Click **New repository**
3. Name it `loanlens`
4. Set visibility to **Public**
5. Do **not** initialise with a README (we already have one)
6. Click **Create repository**

---

### Step 14 · Push to GitHub

```bash
git remote add origin https://github.com/<your-username>/loanlens.git
git branch -M main
git push -u origin main
```

---

### Step 15 · (Optional) Enable GitHub Pages

1. Open your repo on GitHub → **Settings** → **Pages**
2. Under **Source** → select branch `main` → folder `/ (root)`
3. Click **Save**
4. Your app will be live at: `https://<username>.github.io/loanlens/`

Update `.env.example` to reflect the live URL once Pages is enabled.

---

## Final Submission Checklist

Go through every item before submitting the repository link:

- [ ] `emi-calculator.html` opens in browser and works correctly
- [ ] `README.md` committed — includes architecture notes, `data-testid` table, and Claude Code reflection
- [ ] `tests/features/emi-calculator.feature` committed
- [ ] `tests/step-definitions/emi.steps.ts` committed
- [ ] `tests/pages/EmiCalculatorPage.ts` committed
- [ ] `playwright.config.ts` reads `BASE_URL` from `.env` — no hardcoded URLs in test files
- [ ] `api-tests/jsonplaceholder.spec.ts` committed
- [ ] `sql/` folder has schema, both query files, and output screenshots
- [ ] `self-healing/SELF_HEALING.md` committed
- [ ] `test-results/` folder with HTML report committed
- [ ] Repository is set to **Public**
