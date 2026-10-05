# CarbonTrack: A Database-Driven Personal Carbon Footprint Calculator

**Project Check-in 1: Scope, Schema, and Strategy**

| Detail | Information |
|--------|-------------|
| Course | Introduction to Database Systems, Fall 2026 |
| Student | Nitinsai Karuturi |
| WSU ID | X596C589 |
| Check-in | Check-in 1: Scope, Schema, and Strategy |
| Platform | Android mobile application |

---

## 1. Problem Definition and Mobile Scope

### Problem

Most people have no clear idea how much CO₂ their everyday choices produce. Commuting, meals, and home energy use all add up, but the numbers are hidden. Without data, it is hard to know which habits matter most or whether things are improving over time.

CarbonTrack solves this by letting a user log everyday activities, automatically converting each one into kilograms of CO₂ using a stored emission factor, and showing the results as simple totals and summaries.

### Target Platform

Android mobile application.

**Platform constraints:** The app runs on Android and stores its data in a local, on-device SQLite database. This keeps the app usable offline for a single user. It also means the schema stays small and simple, with no server and no multi-user accounts this semester.

### Scope for the Semester

The scope emphasizes personal tracking and clear calculation over social or gamified features, which keeps it manageable for one semester.

| In Scope | Out of Scope |
|----------|--------------|
| Log daily activities in three categories: Transport, Food, Energy | Social features, sharing, or leaderboards |
| Store an emission factor (kg CO₂ per unit) for each activity type | Automatic tracking through GPS or smart meters |
| Calculate CO₂ per log entry as quantity × emission factor | Carbon offset purchases |
| View weekly CO₂ totals and see which category produces the most CO₂ | Multi-user accounts and login (single user on one device) |

---

## 2. Initial Database Design and Mechanics

### Tables, Primary Keys, and Foreign Key

| Table | Columns | Primary Key | Foreign Key |
|-------|---------|-------------|-------------|
| ActivityType (parent) | type_id, name, category, unit, kg_co2_per_unit | type_id | None |
| ActivityLog (child) | log_id, type_id, log_date, quantity | log_id | type_id → ActivityType(type_id) |

### Logical Relationship

One ActivityType can appear in many ActivityLog entries (for example, "Car commute" is logged on many days). Each ActivityLog entry refers to exactly one ActivityType through `type_id`, forming a one-to-many relationship. Storing emission factors in ActivityType means each factor is stored once, and the CO₂ for any log entry is derived from it instead of being typed in repeatedly.

### SQL: Schema

```sql
CREATE TABLE ActivityType (
    type_id          INTEGER PRIMARY KEY,
    name             VARCHAR(50) NOT NULL,
    category         VARCHAR(20) NOT NULL,
    unit             VARCHAR(20) NOT NULL,
    kg_co2_per_unit  DECIMAL(6,3) NOT NULL
                     CHECK (kg_co2_per_unit >= 0)
);

CREATE TABLE ActivityLog (
    log_id    INTEGER PRIMARY KEY,
    type_id   INTEGER NOT NULL,
    log_date  DATE NOT NULL,
    quantity  DECIMAL(8,2) NOT NULL
              CHECK (quantity > 0),
    FOREIGN KEY (type_id)
        REFERENCES ActivityType(type_id)
);
```

### SQL: Sample Data

*Emission factors below are illustrative placeholders and will be replaced with sourced values (for example, from the EPA) in later check-ins.*

```sql
INSERT INTO ActivityType VALUES
(1, 'Car commute',      'Transport', 'km',   0.250),
(2, 'Bus ride',         'Transport', 'km',   0.100),
(3, 'Beef meal',        'Food',      'meal', 6.000),
(4, 'Veggie meal',      'Food',      'meal', 0.800),
(5, 'Home electricity', 'Energy',    'kWh',  0.400);

INSERT INTO ActivityLog VALUES
(1, 1, '2026-09-28', 20),
(2, 3, '2026-09-28', 1),
(3, 5, '2026-09-29', 12),
(4, 2, '2026-09-30', 15),
(5, 4, '2026-10-01', 2);
```

### Five Example Queries in Relational Algebra

In every join below, the join condition is `ActivityType.type_id = ActivityLog.type_id`. Results are shown for the sample data above. Each query is paired with the SQL the app would run to execute it.

**Query 1: List all food activity types.**

$$\sigma_{category = \text{'Food'}}(\text{ActivityType})$$

**SQL equivalent:**

```sql
SELECT * FROM ActivityType WHERE category = 'Food';
```

**Supports:** choosing an activity by category. **Result:** Beef meal, Veggie meal.

**Query 2: Show log entries after September 29, 2026.**

$$\sigma_{log\_date > \text{'2026-09-29'}}(\text{ActivityLog})$$

**SQL equivalent:**

```sql
SELECT * FROM ActivityLog WHERE log_date > '2026-09-29';
```

**Supports:** the recent activity view. **Result:** Log 4 (Bus ride, 09-30) and log 5 (Veggie meal, 10-01).

**Query 3: Show full log history with activity names and units.**

$$\pi_{name,\ log\_date,\ quantity,\ unit}\left(\text{ActivityType} \bowtie_{\text{ActivityType.type\_id} = \text{ActivityLog.type\_id}} \text{ActivityLog}\right)$$

**SQL equivalent:**

```sql
SELECT t.name, l.log_date, l.quantity, t.unit
FROM ActivityType t
JOIN ActivityLog l ON t.type_id = l.type_id;
```

**Supports:** the main history screen. **Result:** All five log entries with readable names and units.

**Query 4: Show only transport logs.**

$$\pi_{name,\ log\_date,\ quantity}\left(\sigma_{category = \text{'Transport'}}(\text{ActivityType}) \bowtie_{\text{ActivityType.type\_id} = \text{ActivityLog.type\_id}} \text{ActivityLog}\right)$$

**SQL equivalent:**

```sql
SELECT t.name, l.log_date, l.quantity
FROM ActivityType t
JOIN ActivityLog l ON t.type_id = l.type_id
WHERE t.category = 'Transport';
```

**Supports:** the category filter on the history screen. **Result:** Car commute on 09-28 (20), Bus ride on 09-30 (15).

**Query 5: Find high-emission activity types (over 1 kg CO₂ per unit) that have been logged.**

$$\pi_{name,\ category}\left(\sigma_{kg\_co2\_per\_unit > 1}\left(\text{ActivityType} \bowtie_{\text{ActivityType.type\_id} = \text{ActivityLog.type\_id}} \text{ActivityLog}\right)\right)$$

**SQL equivalent:**

```sql
SELECT DISTINCT t.name, t.category
FROM ActivityType t
JOIN ActivityLog l ON t.type_id = l.type_id
WHERE t.kg_co2_per_unit > 1;
```

**Supports:** the "biggest impact habits" insight. **Result:** Beef meal (Food).

### How These Queries Support the App's Main Features

- Logging an activity inserts into ActivityLog, and the activity choices come from ActivityType (Query 1).
- The history screen joins the two tables so each log shows a readable name and unit (Queries 2, 3, and 4).
- The insights screen uses the emission factor in ActivityType to highlight the highest-impact habits (Query 5).

### Note on Weekly CO₂ Totals

The app's headline feature is the weekly CO₂ total, which needs quantity × kg_co2_per_unit summed per week. Standard relational algebra cannot express a sum, so this uses the extended aggregation operator:

$$\gamma_{WEEK(log\_date),\ SUM(quantity \times kg\_co2\_per\_unit)\ \rightarrow\ weekly\_co2}\left(\text{ActivityLog} \bowtie_{\text{ActivityLog.type\_id} = \text{ActivityType.type\_id}} \text{ActivityType}\right)$$

Using a Monday–Sunday week definition, all five sample records fall within the week of September 28–October 4, 2026.

For the sample data, CO₂ per entry is 5.0, 6.0, 4.8, 1.5, and 1.6 kg, for a total of 18.9 kg CO₂ for the week of September 28, 2026. In SQL this is a `SUM` with `GROUP BY` on the week, to be implemented in the app. In SQLite:

```sql
SELECT strftime('%Y-%W', l.log_date) AS week,
       SUM(l.quantity * t.kg_co2_per_unit) AS weekly_co2
FROM ActivityLog l
JOIN ActivityType t ON l.type_id = t.type_id
GROUP BY week;
```

---

## 3. AI Utilization Plan

### Goal

I am using AI in this project to build my own understanding of databases and problem-solving, not to have working code handed to me. AI is mainly a tutor and reviewer: I design the schema and write the queries myself first, then use AI to challenge and check my reasoning.

### Tools and Roles

| Tool | Role | Not used for |
|------|------|--------------|
| Claude (chat) | Tutor: explains concepts, checks my reasoning, helps diagnose SQL errors through guiding questions | Generating my full schema or final write-ups |
| SQLite or an online SQL playground | Verification: I run every schema and query myself on the sample data (not an AI tool; my source of truth) | N/A |
| GitHub Copilot (later, app development) | Boilerplate and syntax help in the mobile app code | Designing tables or deciding how data is related |

### Example Prompts

1. **Concept tutoring:** "Explain the difference between selection (σ) and projection (π) in relational algebra with a small example. Then give me three practice problems without the answers."
2. **Reasoning check:** "Here is my relational algebra for 'show only transport logs.' Tell me whether the join and filter order are correct, but do not rewrite it. Ask me questions that help me find any mistakes."
3. **Guided debugging:** "My INSERT into ActivityLog fails with a foreign key error. Do not fix it for me. Ask me guiding questions so I can find the cause."
4. **Self-testing:** "Quiz me on primary keys versus foreign keys using my ActivityType and ActivityLog tables. Wait for my answer before revealing the correct one."
5. **Design critique:** "Here is my two-table design and the reason for each column. What questions should I ask about whether any column depends on something other than the key? Do not give me the answer."

### How I Will Keep This Honest

- I attempt each task on my own first, then use AI to review it.
- I run every SQL statement and compare the results with what I worked out by hand.
- I keep notes on where AI helped me understand something and where I got stuck, for my later reflections.
- If I cannot explain a design decision without the AI in front of me, I study it until I can.
