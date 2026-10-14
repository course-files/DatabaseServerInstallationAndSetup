# Relational Algebra

## Overview

Relational algebra is the formal theoretical language that underpins SQL. It consists of a set of operations that take one or more relations as input and produce a new relation as output. By combining these operations, we can express complex queries and transformations on relational data.

This lab will enable you to practise writing relational algebra expressions and translating them into SQL queries. You will work with the **Siwaka Dishes** database, which models a multi-branch Kenyan restaurant serving African food. The database includes tables for branches, employees, products, customers, customer orders, order details, payments, and customer feedback.

For each question, you will:

1. Write the **relational algebra expression**
2. Translate it into **SQL** and run it in pgAdmin 4 or DataGrip or DBeaver or `psql` (CLI)
3. Verify your results

---

You can connect to PostgreSQL using either pgAdmin 4 or the `psql` command-line tool or DataGrip or DBeaver or any other database tool.

We will use pgAdmin 4 in this case.

---

## Database Schema Reference

Before writing any relational algebra, familiarize yourself with the relations and their attributes.

Textual database schema:

```text
order_status      (order_status_id, status)

customer          (customer_number, customer_name, contact_first_name,
                   contact_last_name, phone, address_line1, address_line2,
                   postal_code, county, sub_county, status)

customer_order    (order_number, order_date, required_date, dispatch_date,
                   order_status_id, customer_number, branch_code)

order_detail      (order_detail_number, order_number, product_code,
                   quantity_ordered, price_each)

product           (product_code, product_name, product_description,
                   quantity_in_stock, cost_of_production, selling_price,
                   product_category_id)

product_category  (product_category_id, category_name, category_description)

payment           (payment_number, order_number, payment_date,
                   amount, payment_method_id)

payment_method    (payment_method_id, payment_method)

branch            (branch_code, phone, address_line1, address_line2,
                   postal_code, county, sub_county)

employee          (employee_number, first_name, last_name, email,
                   branch_code, job_title, reports_to)

customer_feedback (customer_feedback_id, food_quality, service_quality,
                   price_to_value, ambiance, order_number, comment)
```

**Foreign keys:**

| Attribute | References |
| --- | --- |
| `employee.branch_code` | `branch.branch_code` |
| `employee.reports_to` | `employee.employee_number` |
| `customer_order.order_status_id` | `order_status.order_status_id` |
| `customer_order.customer_number` | `customer.customer_number` |
| `customer_order.branch_code` | `branch.branch_code` |
| `product.product_category_id` | `product_category.product_category_id` |
| `payment.order_number` | `customer_order.order_number` |
| `payment.payment_method_id` | `payment_method.payment_method_id` |
| `order_detail.order_number` | `customer_order.order_number` |
| `order_detail.product_code` | `product.product_code` |
| `customer_feedback.order_number` | `customer_order.order_number` |

Graphical database schema:

![ERD of Siwaka Dishes](assets/images/ERD_of_siwaka_dishes_From_DBeaver.png)

---

## Relational Algebra Notation Reference

| Symbol | Operation |
| --- | --- |
| σ | Selection |
| Π | Projection |
| ∪ | Union |
| − | Set Difference |
| ∩ | Intersection |
| × | Cartesian Product |
| ⋈_F | Theta Join |
| ⋈ | Natural Join / Equijoin |
| ⟕ | Left Outer Join |
| ⊳_F | Semi-Join |
| ⊳⊲_F | Anti-Join |
| ℑ | Aggregate / Grouping |
| ρ | Rename |

---

## Part 1 — Selection (σ)

**Concept:** Selection filters *rows* from a single relation based on a predicate. It is a *unary* operation.

**Notation:** σ_predicate(R)

---

### Question 1a

List all employees whose job title is **'Manager'**.

**Relational Algebra:**

```text
σ_job_title = 'Manager' (employee)
```

**SQL:**

```sql
SELECT *
FROM   employee
WHERE  employee.job_title = 'Manager';
```

---

List all orders that were placed between **2026-04-01** and **2026-04-30** (inclusive).

**Relational Algebra:**

### Question 1b

```text
σ_order_date ≥ '2026-04-01' AND order_date < '2026-05-01' (customer_order)
```

**SQL:**

```sql
SELECT *
FROM customer_order
WHERE order_date >= DATE '2026-04-01'
  AND order_date < DATE '2026-05-01';
```

---

### ✏️ Exercise 1

Write the relational algebra expression and the corresponding SQL query to list all **products** with a `selling_price` of **less than KES. 500** and a `quantity_in_stock` greater than **50**.

> Write your relational algebra expression here:
>
> ```text
> σ_??? (???)
> ```
>
> Write your SQL query here:
>
> ```sql
>
> ```

---

## Part 2 — Projection (Π)

**Concept:** Projection extracts *columns* from a relation and eliminates duplicates. It is a *unary* operation.

**Notation:** Π_a1, a2, ..., an (R)

---

### Question 2a

Create a list of all employees showing only their employee number, full name, and job title.

**Relational Algebra:**

```text
Π_employee_number, first_name, last_name, job_title (employee)
```

**SQL:**

```sql
SELECT employee.employee_number,
       employee.first_name,
       employee.last_name,
       employee.job_title
FROM   employee;
```

---

### Question 2b

List the distinct product category names available on the Siwaka Dishes menu.

**Relational Algebra:**

```text
Π_category_name (product_category)
```

**SQL:**

```sql
SELECT DISTINCT product_category.category_name
FROM product_category;
```

---

### ✏️ Exercise 2

Write the relational algebra and SQL to list the distinct **counties** in which Siwaka Dishes has a branch.

> Write your relational algebra expression here:
>
> ```text
>
> ```
>
> Write your SQL query here:
>
> ```sql
>
> ```

---

## Part 3 — Union (∪)

**Concept:** Union returns all tuples from two *union-compatible* relations (same number of attributes, compatible domains), eliminating duplicates.

**Notation:** R ∪ S

---

### Question 3

List all counties where Siwaka Dishes either has a **branch** or has at least one **customer**.

**Relational Algebra:**

```text
Π_county(branch) ∪ Π_county(customer)
```

**SQL:**

```sql
SELECT branch.county
FROM   branch

UNION

SELECT customer.county
FROM   customer;
```

---

### ✏️ Exercise 3

List all counties where Siwaka Dishes either has a **branch** or has processed at least one **customer order**.

> *Hint:* Since `customer_order` does not directly contain `county`, you need to join with `branch` first.
>
> Write your relational algebra expression here:
>
> ```text
>
> ```
>
> Write your SQL query here:
>
> ```sql
>
> ```

---

## Part 4 — Set Difference (−)

**Concept:** Set difference returns tuples that are in R but **not** in S.

**Notation:** R − S  →  SQL: `EXCEPT`

---

### Question 4

List all counties where Siwaka Dishes has a branch but where **no customer orders** were placed between **2026-04-01** and **2026-04-14** (inclusive).

**Relational Algebra:**

```text
Π_county(branch) − Π_county(σ_order_date ≥ '2026-04-01' AND order_date ≤ '2026-04-14' (customer_order ⋈ branch))
```

**SQL:**

```sql
SELECT branch.county
FROM   branch

EXCEPT

SELECT branch.county
FROM   customer_order
JOIN   branch ON customer_order.branch_code = branch.branch_code
WHERE customer_order.order_date >= DATE '2026-04-01'
  AND customer_order.order_date <= DATE '2026-04-14';
```

---

### ✏️ Exercise 4

**Question:** List the product codes of all products that have **never been sold for more than KES. 200 per unit**.

> *Hint:* Start with all product codes in `product` and subtract those that appear in `order_detail`.
>
> Write your relational algebra expression here:
>
> ```text
>
> ```
>
> Write your SQL query here:
>
> ```sql
>
> ```

---

## Part 5 — Intersection (∩)

**Concept:** Intersection returns only the tuples that exist in **both** R and S.

**Notation:** R ∩ S  →  SQL: `INTERSECT`

---

### Question 5

List the counties where Siwaka Dishes has both a **branch** and at least one registered **customer**.

**Relational Algebra:**

```text
Π_county(branch) ∩ Π_county(customer)
```

**SQL:**

```sql
SELECT branch.county
FROM   branch

INTERSECT

SELECT customer.county
FROM   customer;
```

---

### ✏️ Exercise 5

List the `customer_number` values of customers who have placed orders at branches in **both Nairobi county** and **Mombasa county**.

> *Hint:* Use `INTERSECT` between two queries — each joining `customer_order` with `branch` and filtering by county.
>
> Write your SQL query here:
>
> ```sql
>
> ```

---

## Part 6 — Cartesian Product (×)

**Concept:** Cartesian product concatenates every tuple of R with every tuple of S, producing |R| × |S| rows. In practice it is almost always filtered with a predicate (which gives you a theta join).

**Notation:** R × S  →  SQL: `CROSS JOIN`

---

### Question 6

Using a Cartesian product, list all customers alongside all customer orders (unfiltered, to illustrate the concept).

**Relational Algebra:**

```text
Π_customer_number, contact_first_name, contact_last_name (customer)
×
Π_customer_number, order_number, order_date (customer_order)
```

**SQL:**

```sql
SELECT customer.customer_number     AS cust_customer_number,
       customer.contact_first_name,
       customer.contact_last_name,
       customer_order.customer_number AS ord_customer_number,
       customer_order.order_number,
       customer_order.order_date
FROM   customer
CROSS JOIN customer_order
LIMIT  30;   -- limit output; the full cross product is 750,000 rows (300 × 2,500)
```

> **Observe:** Without a WHERE clause, this returns `300 × 2,500 = 750,000` rows — a large result set. This is why the Cartesian product is almost always combined with a predicate.

---

## Part 7 — Theta Join (⋈_F)

**Concept:** A theta join is a Cartesian product filtered by a predicate F, which may use any comparison operator (<, >, ≤, ≥, =, ≠).

**Notation:** R ⋈_F S ≡ σ_F (R × S)

---

### Question 7

List all products with a `selling_price` **greater than** the average selling price of products in the **'Meat-Based Dishes'** category. Use a theta join.

**Relational Algebra:**

```text
Let T = ρ_R(avg_meat_price) ℑ_AVG selling_price (
            σ_category_name = 'Meat-Based Dishes' (product ⋈ product_category))

Π_product_code, product_name, selling_price (product) ⋈_(selling_price > avg_meat_price) T
```

**SQL (theta join expressed with CROSS JOIN + WHERE):**

```sql
SELECT p.product_code,
       p.product_name,
       p.selling_price,
       avg_prices.avg_meat_price
FROM   product AS p
CROSS JOIN (
    SELECT AVG(product.selling_price) AS avg_meat_price
    FROM   product
    JOIN   product_category
        ON product.product_category_id = product_category.product_category_id
    WHERE  product_category.category_name = 'Meat-Based Dishes'
) AS avg_prices
WHERE  p.selling_price > avg_prices.avg_meat_price;
```

**SQL (equivalent, using JOIN):**

```sql
SELECT p.product_code,
       p.product_name,
       p.selling_price
FROM   product AS p
JOIN  (
    SELECT AVG(product.selling_price) AS avg_meat_price
    FROM   product
    JOIN   product_category
        ON product.product_category_id = product_category.product_category_id
    WHERE  product_category.category_name = 'Meat-Based Dishes'
) AS avg_prices ON p.selling_price > avg_prices.avg_meat_price;
```

---

### ✏️ Exercise 7

List all customer orders that were **dispatched after their required date** (i.e., late orders), along with the **county** of the branch where the order was placed. Use a theta join — express using `CROSS JOIN` + `WHERE` first, then rewrite using `JOIN`.

> *Hint:* The predicate is `dispatch_date > required_date`, which is a non-equality comparison — this is a theta join.
>
> Write your SQL (CROSS JOIN version) here:
>
> ```sql
>
> ```
>
> Write your SQL (JOIN version) here:
>
> ```sql
>
> ```

---

## Part 8 — Equijoin

**Concept:** An equijoin is a special theta join where the predicate uses only the **equality** operator (=).

---

### Question 8

List the full name of each customer alongside the details of every order they have placed. Use an equijoin.

**Relational Algebra:**

```text
(Π_customer_number, contact_first_name, contact_last_name (customer))
⋈_(customer.customer_number = customer_order.customer_number)
(Π_customer_number, order_number, order_date, order_status_id (customer_order))
```

**SQL (equijoin via CROSS JOIN + WHERE):**

```sql
SELECT co.c_customer_number,
       co.contact_first_name,
       co.contact_last_name,
       co.o_customer_number,
       co.order_number,
       co.order_date
FROM (
    SELECT customer.customer_number     AS c_customer_number,
           customer.contact_first_name,
           customer.contact_last_name,
           customer_order.customer_number AS o_customer_number,
           customer_order.order_number,
           customer_order.order_date
    FROM   customer
    CROSS JOIN customer_order
) AS co
WHERE co.c_customer_number = co.o_customer_number;
```

**SQL (cleaner equijoin using INNER JOIN):**

```sql
SELECT customer.customer_number,
       customer.contact_first_name,
       customer.contact_last_name,
       customer_order.order_number,
       customer_order.order_date,
       customer_order.order_status_id
FROM   customer
INNER JOIN customer_order
    ON customer.customer_number = customer_order.customer_number;
```

---

### ✏️ Exercise 8

List the full name of each employee alongside the **county** of the branch they work at. Use an equijoin (write both the CROSS JOIN version and the INNER JOIN version).

> Write your relational algebra expression here:
>
> ```text
>
> ```
>
> Write your SQL (CROSS JOIN version) here:
>
> ```sql
>
> ```
>
> Write your SQL (INNER JOIN version) here:
>
> ```sql
>
> ```

---

## Part 9 — Natural Join (⋈)

**Concept:** A natural join is an equijoin over **all common attributes**, with one copy of each common attribute in the result.

---

### Question 9

List customers alongside the orders they placed, using a natural join.

**Relational Algebra:**

```text
(Π_customer_number, contact_first_name, contact_last_name (customer))
⋈
(Π_customer_number, order_number, order_date, required_date (customer_order))
```

**SQL:**

```sql
SELECT *
FROM (
    SELECT customer_number,
           contact_first_name,
           contact_last_name
    FROM   customer
) AS c
NATURAL JOIN (
    SELECT customer_number,
           order_number,
           order_date,
           required_date
    FROM   customer_order
) AS o;
```

> **Note:** In PostgreSQL, `NATURAL JOIN` matches on columns with the **same name** in both relations. Always project down to only the columns you need before applying NATURAL JOIN — unexpected column name matches (e.g., two relations both having a `status` column) can lead to unintended results.

---

## Part 10 — Left Outer Join (⟕)

**Concept:** A left outer join returns all tuples from the left relation R. Where there is no matching tuple in S, the attributes from S are set to `NULL`.

**Notation:** R ⟕ S  →  SQL: `LEFT JOIN` / `LEFT OUTER JOIN`

---

### Question 10

Produce a full report of all products, including those that have **never appeared in any order detail**. Show the product code, name, selling price, and order detail information where available.

**Relational Algebra:**

```text
Π_product_code, product_name, selling_price (product) ⟕ order_detail
```

**SQL:**

```sql
SELECT product.product_code,
       product.product_name,
       product.selling_price,
       order_detail.order_number,
       order_detail.quantity_ordered,
       order_detail.price_each
FROM   product
LEFT OUTER JOIN order_detail
    ON product.product_code = order_detail.product_code
ORDER BY product.product_code;
```

> Rows where `order_number` is `NULL` indicate products that have never been ordered.

---

### ✏️ Exercise 10

Produce a list of all branches, including those where **no employee** has been assigned. Show `branch_code`, `county`, and employee details where available.

> Write your relational algebra expression here:
>
> ```text
>
> ```
>
> Write your SQL query here:
>
> ```sql
>
> ```

---

## Part 11 — Semi-Join (⊳_F)

**Concept:** A semi-join returns only the tuples from R that have at least one matching tuple in S under predicate F, but does not include any attributes from S.

**Notation:** R ⊳_F S ≡ Π_A (R ⋈_F S), where A is the set of all attributes of R

**SQL equivalent:** `WHERE EXISTS (subquery)`

---

### Question 11

List all details of employees who work at a branch in **Langata** sub-county.

**Relational Algebra:**

```text
employee ⊳_(employee.branch_code = branch.branch_code) (σ_sub_county = 'Langata' (branch))
```

**SQL:**

```sql
SELECT employee.*
FROM   employee
WHERE  EXISTS (
    SELECT *
    FROM   branch
    WHERE  branch.sub_county = 'Langata'
    AND    employee.branch_code = branch.branch_code
);
```

---

### ✏️ Exercise 11

List all details of customers who have placed **at least one customer order**.

> Write your relational algebra expression here:
>
> ```text
>
> ```
>
> Write your SQL query here:
>
> ```sql
>
> ```

---

## Part 12 — Anti-Join (⊳⊲_F)

**Concept:** An anti-join returns tuples from R that have **no** matching tuple in S under predicate F. It is the complement of the semi-join.

**SQL equivalent:** `WHERE NOT EXISTS (subquery)`

---

### Question 12

List all details of employees who **do not** work at a branch in Kilimani sub-county.

**Relational Algebra:**

```text
employee ⊳⊲_(employee.branch_code = branch.branch_code) (σ_sub_county = 'Kilimani' (branch))
```

**SQL:**

```sql
SELECT employee.*
FROM   employee
WHERE  NOT EXISTS (
    SELECT *
    FROM   branch
    WHERE  branch.sub_county = 'Kilimani'
    AND    employee.branch_code = branch.branch_code
);
```

---

### ✏️ Exercise 12

List all customers who have **never placed a customer order**.

> Write your relational algebra expression here:
>
> ```text
>
> ```
>
> Write your SQL query here:
>
> ```sql
>
> ```

---

## Part 13 — Aggregate Operations (ℑ)

**Concept:** Aggregate functions are applied to a relation to compute a single summary value over one or more attributes.

**Notation:** ρ_R(alias) ℑ_AL (R), where AL contains (`<function> <attribute>`) pairs

**Common functions:** COUNT, SUM, AVG, MIN, MAX

You can also create User-Defined Functions (UDFs) in PostgreSQL for more complex calculations, but we will focus on built-in aggregate functions for now in this lab.

An example of a User-Defined Function in a business context is a function that calculates the customer lifetime value (CLV) based on the customer's order history and average spend.

---

### Question 13

Display the total number of products with a `selling_price` greater than KES. 300.

**Relational Algebra:**

```text
ρ_R(product_count) ℑ_COUNT product_code (σ_selling_price > 300 (product))
```

**SQL:**

```sql
SELECT COUNT(product.product_code) AS product_count
FROM   product
WHERE  product.selling_price > 300;
```

---

### ✏️ Exercise 13

Display the **total revenue** (sum of `payment.amount`) received for all orders with a status of **'Delivered'** during the month of April 2026.

> *Hint:* You need to join `payment` → `customer_order` → `order_status` to filter by status name.
>
> Write your relational algebra expression here:
>
> ```text
>
> ```
>
> Write your SQL query here:
>
> ```sql
>
> ```

---

## Part 14 — Grouping Operations (GA ℑ AL)

**Concept:** Grouping groups the tuples of R by the grouping attributes GA, then applies aggregate functions AL to each group.

**Notation:** GA ℑ_AL (R)  →  SQL: `GROUP BY`

---

### Question 14

Display the number of employees assigned to each branch.

**Relational Algebra:**

```text
ρ_R(branch_code, employee_count)
branch_code ℑ_COUNT employee_number (employee)
```

**SQL:**

```sql
SELECT employee.branch_code,
       COUNT(employee.employee_number) AS employee_count
FROM   employee
GROUP BY employee.branch_code;
```

---

### ✏️ Exercise 14

Display the number of orders and the total payment amount received for each **order status** (e.g., Pending, Processing, In Transit, Delivered, and Cancelled).

> *Hint:* Join `customer_order` → `order_status` → `payment`. Use `COUNT(DISTINCT ...)` to avoid double-counting orders that have multiple payment rows.
>
> Write your relational algebra expression here:
>
> ```text
>
> ```
>
> Write your SQL query here:
>
> ```sql
>
> ```

---

## Part 15 — HAVING and ORDER BY

**Concept:** `HAVING` applies a filter to grouped rows (it is similar to `WHERE`, but for groups). `ORDER BY` sorts the final result.

---

### Question 15

List each branch that has **more than 5 employees**, showing the branch code, sub-county, and employee count, ordered by employee count in descending order.

**Relational Algebra:*

ORDER BY — Not representable in standard relational algebra
This is a fundamental theoretical point worth understanding clearly.

Relational algebra is defined on sets. A set, by definition, has no ordering. Every operation — join, selection, projection, union — produces an unordered set of tuples. There is simply no notion of "the first row" or "sorted output" in the formal model.

ORDER BY is a presentation concern, not a data concern. It describes how results are displayed to a user, not what data is returned.

Some literature and query optimizers introduce an extended sort operator τ (tau) for this purpose: `τ_{employee_count DESC} (expression)`

```text
ρ_R(branch_code, sub_county, employee_count) (
    σ_{COUNT employee_number > 5} (
        _{employee.branch_code, branch.sub_county} ℑ_{COUNT employee_number} (
            employee ⋈_(employee.branch_code = branch.branch_code) branch
        )
    )
)
```

**SQL:**

```sql
SELECT employee.branch_code,
       branch.sub_county,
       COUNT(employee.employee_number) AS employee_count
FROM   employee
JOIN   branch ON employee.branch_code = branch.branch_code
GROUP BY employee.branch_code,
         branch.sub_county
HAVING COUNT(employee.employee_number) > 5
ORDER BY employee_count DESC;
```

---

### ✏️ Exercise 15

List each **product category** that has **more than 2 products**, showing the category name, product count, and average selling price, ordered by average selling price ascending.

> Write your relational algebra expression here:
>
> ```text
>
> ```
>
> Write your SQL query here:
>
> ```sql
>
> ```

---

## Part 16 — LIMIT and OFFSET

**Concept:** `LIMIT` restricts the number of rows returned. `OFFSET` skips a given number of rows before returning results — useful for pagination.

---

### Question 16

Display the top 5 most expensive products.

**SQL:**

```sql
SELECT product.product_code,
       product.product_name,
       product.selling_price
FROM   product
ORDER BY product.selling_price DESC
LIMIT 5;
```

---

### Question 16b — Pagination

Display products ranked 6th to 10th by selling price (page 2 of 5 items per page).

```sql
SELECT product.product_code,
       product.product_name,
       product.selling_price
FROM   product
ORDER BY product.selling_price DESC
LIMIT  5
OFFSET 5;
```

---

### ✏️ Exercise 16

Display the **top 3 customers** who have paid the most money in total across all their orders.

> Write your SQL query here:
>
> ```sql
>
> ```

---

## Part 17 — Combining Multiple Operations

The power of relational algebra comes from *combining* multiple operations together. The following questions require you to combine two or more operations.

---

### Question 17a — Selection + Projection + Join

List the first name, last name, and branch county of all employees with the job title **'Manager'**.

**Relational Algebra:**

```text
Π_first_name, last_name, county (σ_job_title = 'Manager' (employee) ⋈ branch)
```

**SQL:**

```sql
SELECT employee.first_name,
       employee.last_name,
       branch.county
FROM   employee
JOIN   branch ON employee.branch_code = branch.branch_code
WHERE  employee.job_title = 'Manager';
```

---

### Question 17b — Grouping + Having + Join

For each branch county, show the total payment amount received for **'Delivered'** orders in 2026 Quarter 1 (1st January to 31st March 2026), but only for counties where this total exceeds KES. 10,000, ordered by total revenue descending.

**SQL:**

```sql
SELECT branch.county,
       SUM(payment.amount) AS total_revenue
FROM payment
         JOIN customer_order
              ON payment.order_number = customer_order.order_number
         JOIN order_status
              ON customer_order.order_status_id = order_status.order_status_id
         JOIN branch
              ON customer_order.branch_code = branch.branch_code
WHERE order_status.status = 'Delivered' AND
      customer_order.order_date BETWEEN '2026-01-01' AND '2026-03-31'
GROUP BY branch.county
HAVING SUM(payment.amount) > 10000
ORDER BY total_revenue DESC;
```

---

### ✏️ Exercise 17

List the product name and total quantity sold for the **top 5 best-selling products** (by total `quantity_ordered` across all order details).

> Write your SQL query here:
>
> ```sql
>
> ```

---

## Part 18 — Division

**Concept:** Division (R ÷ S) returns those tuples of R that are associated with **every** tuple in S. It has no direct SQL keyword and must be expressed using a combination of other operations — typically using `NOT EXISTS` with a double negation.

**Notation:** R ÷ S

---

### Question 18

Find all customers who have placed orders at **every branch in Kasarani sub-county or Starehe sub-county**.

By double negation, this resolves to:

>"There does not exist a branch in Kasarani or Starehe at which this customer has not placed an order"

This is logically identical to:

>"This customer has placed an order at every branch in Kasarani or Starehe"

This is precisely the semantics of relational division: CustomerOrders ÷ TargetBranches.

This is a division query: Customer ÷ Branch (filtered to branches in Kasarani or Starehe).

**Relational Algebra:**

```text
Π_{customer_number, contact_first_name, contact_last_name} (
    customer
    ⋈
    (Π_{customer_number, branch_code}(customer_order)
     ÷
     Π_branch_code(σ_{sub_county = 'Kasarani' ∨ sub_county = 'Starehe'}(branch)))
)
```

**SQL (expressed using double NOT EXISTS):**

```sql
SELECT DISTINCT c.customer_number,
                c.contact_first_name,
                c.contact_last_name
FROM customer c
WHERE EXISTS (
    -- Guard clause: if there are no branches in Kasarani or Starehe, return no customers
    SELECT 1 FROM branch WHERE (branch.sub_county = 'Kasarani' OR branch.sub_county = 'Starehe'))
  AND NOT EXISTS (
    -- For every branch in Kasarani or Starehe,
    -- check if there is no order from this customer at that branch
    SELECT *
    FROM branch b
    WHERE (b.sub_county = 'Kasarani' OR b.sub_county = 'Starehe')
      AND NOT EXISTS (
        -- And there is no order from this customer at that branch
        SELECT *
        FROM customer_order co
        WHERE co.customer_number = c.customer_number
          AND co.branch_code = b.branch_code));
```

---

### ✏️ Exercise 18

Find all customers who have ordered **every product in the 'Fried Dishes' category** at least once.

> Write your SQL query here:
>
> ```sql
>
> ```

---

## Lab Deliverable

For each **exercise** (1 through 18), type the relational algebra expression and the SQL query into the same markdown file and submit this file as your lab deliverable. Ensure that you have run each SQL query against the database and verified that it returns the expected results.

---

**Submission Details:**

Student ID: `__________`  
Student Name: `________________________`  
Date of Submission: `________________________`

---
