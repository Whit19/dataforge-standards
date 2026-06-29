# AFAS Project — Category Taxonomy Reference
**Update this file when any category, subcategory, or type assignment changes.**
Last updated: 2026-06-01 (subcategory violations fixed, in_budget corrections, Travel cleanup)

---

## Taxonomy Stats
| Metric | Value |
|--------|-------|
| Total categories | 29 |
| Total subcategories | 136 |
| Last structural change | 2026-06-01 |

---

## Taxonomy Version History
| Date | Change | Rows Affected |
|------|--------|---------------|
| 2026-03-09 | Dining Out subcategory `Dining Out` → `General` | 4,056 |
| 2026-03-09 | Car: removed `Insurance` subcat → moved to Bills & Utilities/Car Insurance | 11 |
| 2026-03-09 | Car: removed `Rental` subcat → moved to Travel/Transportation | 3 |
| 2026-03-09 | Medical: removed `Insurance-Life` subcat → moved to Bills & Utilities/Life Insurance | 11 |
| 2026-03-09 | Travel: `Airlines` → `Flights` | 133 |
| 2026-03-09 | Travel: `Hotels` → `Lodging` | 24 |
| 2026-03-09 | Travel: `Activities` → `Travel Activities` | 10 |
| 2026-03-09 | Pay: `Salary` → `Amy` | 56 |
| 2026-03-09 | Sports/Clubs: `U-club` → `Club Dues` | 54 |
| 2026-03-09 | Car: `Wash` → `Car Wash`; `Supercharger` → `Charging` | 56 |
| 2026-03-10 | ATM / Cash Spending: `ATM` → `General` | 123 |
| 2026-03-10 | Payment: truncated subcategories cleaned | 124 |
| 2026-03-10 | Fees: 6 truncated subcategories renamed | — |
| 2026-03-10 | Other Income: 5 truncated subcategories cleaned | — |
| 2026-03-10 | Transfer: `WEB TO DDA` → `Transfer Out`; `WEB FR DDA` → `Transfer In` | — |
| 2026-03-11 | TX 2222 reclassified → Other Income/Rewards Points/Income | 1 |
| 2026-03-12 | Bonus: `Salary` → `Amy` | TBD |
| 2026-03-12 | Cash Adj: `Daily Cash` → `General` | TBD |
| 2026-03-12 | Children: `Birthday Party`, `Photos`, `Pictures` → `General` | TBD |
| 2026-03-12 | Gifts / Charity: `Gifts` → `General` | TBD |
| 2026-03-12 | Groceries: `Groceries` → `General` | TBD |
| 2026-03-12 | Property Tax: `General` → `WFB-Belle` | TBD |
| 2026-03-12 | Cash Adj + Uncategorized: type corrected Other → Expense | 192 |
| 2026-03-12 | Children: `Babysit` added — 25 existing rows | 25 |
| 2026-03-12 | Children: `Rides` added — Venmo payments to SAM for rides | — |
| 2026-03-12 | Large Purchases: `Amy 50th` added | 1 |
| 2026-03-12 | Large Purchases: `Tom 50th` added | 2 |
| 2026-03-12 | Travel: `Spain` confirmed as trip-specific subcategory | — |
| 2026-06-01 | ATM / Cash Spending: `ATM` → `General` (stragglers missed in March) | 2 |
| 2026-06-01 | Clothing: `Clothing` → `General` (subcategory mirrored category name) | 27 |
| 2026-06-01 | Dining Out: `Dining Out` → `General` (subcategory mirrored category name) | 1 |
| 2026-06-01 | Gifts / Charity: `Gifts` → `General` (subcategory mirrored category name) | 7 |
| 2026-06-01 | Groceries: `Groceries` → `General` (subcategory mirrored category name) | 59 |
| 2026-06-01 | Payment: `Payment` → `General` (subcategory mirrored category name) | 13 |
| 2026-06-01 | Travel: `Airlines` → `Flights` (stragglers missed in March) | 3 |
| 2026-06-01 | Travel: `Hotels` → `Lodging` (stragglers missed in March) | 10 |
| 2026-06-01 | Large Purchases: in_budget → 1 (normalized — was split 0/1) | 1 |
| 2026-06-01 | HSA Deposit: in_budget → 0 (not budgetable — employer contribution) | 60 |
| 2026-06-01 | Payment: in_budget → 0 (stragglers — double counts underlying spend) | 3 |
| 2026-06-01 | category_map: Large Purchases in_budget → 1 (6 entries synced) | 6 |
| 2026-06-01 | category_map: HSA Deposit subcategory → EFT (was HSA Deposit — mirrored category name) | 2 |

---

## Rules for Adding New Subcategories
1. Check if an existing subcategory covers the case before adding a new one
2. New subcategory must be approved and dated in this version history
3. Add to `category_map` and `merchant_patterns` as needed
4. Never add subcategories to `Uncategorized` — use it only as a holding pen
5. Trip-specific subcategories (France, Spain, Amy Europe, Turks) are acceptable under Travel and Large Purchases
6. Subcategory name must never mirror the category name — use `General` instead

---

## Type Assignments and Budget Policy by Category
| Category | Default Type | In Budget | Notes |
|----------|-------------|-----------|-------|
| ATM / Cash Spending | Expense | ✅ Yes | |
| Bills & Utilities | Expense | ✅ Yes | |
| Bonus | Income | ✅ Yes | Tracked against income budget target |
| Car | Expense | ✅ Yes | |
| Cash Adj | Expense | ✅ Yes | |
| Children | Expense | ✅ Yes | |
| Clothing | Expense | ✅ Yes | |
| Dining Out | Expense | ✅ Yes | |
| Entertainment | Expense | ✅ Yes | |
| Fees | Expense | ✅ Yes | |
| Gifts / Charity | Expense | ✅ Yes | |
| Groceries | Expense | ✅ Yes | |
| HSA Deposit | Income | ❌ No | Employer contribution — not budgetable |
| Housing | Expense | ✅ Yes | |
| Large Purchases | Expense | ✅ Yes | Tracked separately but visible in budget |
| Medical / Health | Expense | ✅ Yes | |
| Other Income | Income | ✅ Yes | Tracked against income budget target |
| Pay | Income | ✅ Yes | Tracked against income budget target |
| Payment | Other | ❌ No | Credit card payments — double counts underlying spend |
| Personal Care | Expense | ✅ Yes | |
| Property Tax | Expense | ✅ Yes | Predictable — unlike income taxes |
| Sports / Clubs | Expense | ✅ Yes | |
| Subscriptions | Expense | ✅ Yes | |
| Taxes | Expense | ❌ No | Baird dividend-driven — unpredictable, not budgetable |
| Transfer | Other | ❌ No | Internal moves — not real spending or income |
| Travel | Expense | ✅ Yes | |
| Uncategorized | Expense | ✅ Yes | Default until reviewed |
| Work - Expense | Expense | ✅ Yes | Tracked against spending budget |
| Work - Reimb | Income | ✅ Yes | Tracked against income budget target |

---

## Full Taxonomy (29 categories, 136 subcategories)

```
ATM / Cash Spending
  └─ General

Bills & Utilities
  └─ Car Insurance
     Electric
     Gas & Water
     Home Insurance
     Internet
     Life Insurance
     Mortgage
     Phone

Bonus
  └─ Amy
     Tom

Car
  └─ Accessory
     Car Wash
     Charging
     Gas
     General
     Maintenance
     Parking
     Registration
     Tires
     Tolls

Cash Adj
  └─ General

Children
  └─ Allowance
     Babysit
     Driving Instructions
     Education
     General
     Rides
     School Lunch

Clothing
  └─ General

Dining Out
  └─ Bars & Alcohol
     Coffee
     Delivery
     General

Entertainment
  └─ Activities
     Concerts & Events
     Gaming
     General
     Movies
     Sporting Events

Fees
  └─ Amex Membership
     ATM Inquiry Fee
     ATM Transaction Fee
     General
     Interest Charged
     International Fee
     Non-Associated Bank Fee
     PayPal Fee
     Rocket Money
     Stop Payment
     Tax Prep

Gifts / Charity
  └─ Donation
     General

Groceries
  └─ Alcohol
     General

HSA Deposit
  └─ EFT

Housing
  └─ Amazon
     Cleaning
     Electronics
     General
     Hardware
     Housewares
     Landscape
     Maintenance
     Target
     Walmart
     Wood

Large Purchases
  └─ 20th Anniversary
     Amy 50th
     Camper
     Chimney
     DC Trip
     Electronics
     Fence
     Furniture
     Garage
     Gutters
     House Painting
     Lawyer
     Postal
     Shuffle Board
     Soffit - Facia
     Tesla
     Tesla Model Y
     Tom 50th
     Turks
     Wall-Pavers

Medical / Health
  └─ Dental
     Doctor
     General
     Hospital
     Pharmacy
     Vision

Other Income
  └─ Apple CC Cash
     Check
     Customer Deposit
     Deposit
     General
     PayPal
     Rewards Points
     Venmo

Pay
  └─ Amy
     Tom

Payment
  └─ ACH Deposit
     Amex
     Apple Card
     Autopay
     Chase
     Credit Card
     DataForge
     General
     Mobile Payment

Personal Care
  └─ Eyecare
     Fitness
     General
     Hair
     Health & Wellness
     Massage
     Nails
     Waxing

Property Tax
  └─ WFB-Belle

Sports / Clubs
  └─ Camp
     Climbing
     Club Dues
     General
     Golf
     Hockey
     Lacrosse
     Skiing
     Soccer
     Sporting Goods

Subscriptions
  └─ Apps & Software
     Auto Subscription
     Books & Audible
     General
     News & Media
     Smart Home
     Streaming

Taxes
  └─ Federal
     State

Transfer
  └─ Baird
     General
     Investment Transfer
     PayPal
     Security Insurance
     Transfer In
     Transfer Out
     Venmo

Travel
  └─ Amy Europe
     Entertainment
     Flights
     France
     General
     Lodging
     Other
     Parking
     Spain
     Transportation
     Travel Activities
     Vacation Dining

Uncategorized
  └─ General
     VENMO - Review

Work - Expense
  └─ Amy
     Amy - London
     DataForge
     General
     Lodging

Work - Reimb
  └─ Amy
     Credit
     General
```

---

## Notable Subcategory Decisions (quick reference)

| Subcategory | Category | Rationale |
|-------------|----------|-----------|
| WFB-Belle | Property Tax | Property-specific label for Belle property — supports future additional properties |
| France, Spain, Amy Europe, Turks | Travel | Trip-specific labels — kept intentionally for travel analysis |
| Auto Subscription | Subscriptions | Tesla FSD monthly — confirmed correct, not Car |
| Lodging | Work - Expense | Business lodging — kept separate from Travel/Lodging |
| Club Dues | Sports / Clubs | Replaces U-club (University Club) |
| VENMO - Review | Uncategorized | 46 rows pending manual review (ISSUE-004) |
| Transfer In / Transfer Out | Transfer | Replaces WEB FR DDA / WEB TO DDA bank descriptions |
| Car Wash | Car | Replaces Wash — NOT a subcategory/category mirror (Car Wash ≠ Car) |
| Charging | Car | Tesla Supercharger — replaces Supercharger |
| Rewards Points | Other Income | Cashback and credit card rewards deposits |
| Rides | Children | Venmo payments to SAM for rides — distinct from Driving Instructions |
| Amy 50th | Large Purchases | Amy's 50th birthday dinner (Zarletti) |
| Tom 50th | Large Purchases | Tom's 50th birthday trip (Cosmopolitan Las Vegas) |
| Travel Activities | Travel | Retained — distinct enough from Travel to not be considered a mirror |
| Transfer In / Transfer Out | Transfer | Retained — directional qualifiers make these distinct from Transfer |
