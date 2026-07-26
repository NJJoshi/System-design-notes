# Spring Transaction Management — Interview Notes

Source: https://www.javainuse.com/spring/transaction-interview

## Diagrams on the page

The article has 6 illustrative diagrams (images, not reproduced here — click through to view on the source site):

1. [Database Transactions](https://www.javainuse.com/static/trans-min.JPG) — under Q1, visualizing a transaction as one logical unit of work over a DB.
2. [OrganizationService Exit](https://www.javainuse.com/static/boot-66-19.JPG) — under Q2, showing the `joinOrganization` flow: Employee insert → HealthInsurance insert as one application transaction.
3. [Transaction Management Concern](https://www.javainuse.com/static/boot-66-9-min.JPG) — under Q4, showing transaction management sitting as a cross-cutting concern alongside business logic (the AOP framing).
4. [Transaction Management Proxy](https://www.javainuse.com/static/boot-66-7-min.JPG) — under Q4, showing the proxy Spring generates around `@Transactional`-annotated methods (calls go through the proxy, which begins/commits/rolls back the transaction).
5. [Transaction without Rollback](https://www.javainuse.com/static/roll2-min.JPG) — under the checked-exceptions Q, illustrating a case where an exception is thrown but the transaction does **not** roll back (i.e., the checked-exception pitfall).
6. [Transaction without Rollback — DB view](https://www.javainuse.com/static/roll1-min.JPG) — companion diagram showing the resulting (inconsistent) database state in that same scenario.

Note: diagrams #5/#6 correspond to a Q&A pair ("How to handle Transactions for checked Exceptions?") that wasn't in the original extract below — added as its own section.


## 1. What is a Database Transaction?

A database transaction is a single logical unit of work which accesses and possibly modifies the contents of a database. It's a sequence of queries that run together, performing one logical function (all succeed, or none do).

## 2. What is an Application Transaction?

An application transaction is a sequence of application actions treated as a single logical unit by the application — even if it spans multiple services/tables.

Example: a `joinOrganization` operation is really two steps that must succeed or fail together:
1. Persist Employee Information
2. Persist HealthInsurance Information

## 3. What is Transaction Management?

A **transaction manager** coordinates transactions across one or more resources (e.g., databases) and manages their **atomicity** and **durability** — ensuring all steps commit together, or all roll back together.

## 4. How to implement Transaction Management in Spring Boot?

Use the `@Transactional` annotation on a method (or class). Spring implements transactions as a **cross-cutting concern using AOP** — it wraps the annotated method in a proxy that starts a transaction before the method runs and commits/rolls back after, depending on whether an exception was thrown.

```java
@Service
public class OrganzationServiceImpl implements OrganizationService {

    @Autowired
    EmployeeService employeeService;

    @Autowired
    HealthInsuranceService healthInsuranceService;

    @Override
    @Transactional
    public void joinOrganization(Employee employee,
        EmployeeHealthInsurance employeeHealthInsurance) {

        employeeService.insertEmployee(employee);

        if (employee.getEmpId().equals("emp1")) {
            throw new RuntimeException(
                "thowing exception to test transaction rollback");
        }

        healthInsuranceService.registerEmployeeHealthInsurance(
            employeeHealthInsurance);
    }
}
```

**How rollback works here:** if an exception is thrown after `insertEmployee` but before `registerEmployeeHealthInsurance`, Spring rolls back the *entire* transaction — so the employee insert is undone too. This is the whole point of wrapping both calls in one `@Transactional` boundary: it's all-or-nothing, even though two different services/tables are involved.

> Note: by default, `@Transactional` only rolls back on **unchecked** exceptions (`RuntimeException`/`Error`). Checked exceptions require `@Transactional(rollbackFor = Exception.class)` to trigger a rollback.

## 5. How to handle Transactions for checked Exceptions?

By default, Spring's declarative `@Transactional` only triggers a rollback for **unchecked** exceptions (`RuntimeException` and `Error`). If a checked exception is thrown (e.g., `Exception`, `IOException`), the transaction **commits anyway** — this is the pitfall shown in the "Transaction without Rollback" diagrams on the source page (partial data gets persisted even though a checked exception was thrown).

Fix: explicitly tell Spring which exceptions should also trigger a rollback via `rollbackFor`:

```java
@Transactional(rollbackFor = Exception.class)
public void joinOrganization(Employee employee,
        EmployeeHealthInsurance employeeHealthInsurance) throws Exception {
    employeeService.insertEmployee(employee);

    if (employee.getEmpId().equals("emp1")) {
        throw new Exception("checked exception - without rollbackFor this would still commit");
    }

    healthInsuranceService.registerEmployeeHealthInsurance(employeeHealthInsurance);
}
```

## 6. Transaction Propagation Types

Propagation defines how a transactional method behaves when it's called from within another transaction (or with no transaction present).

| Propagation | Behaviour |
|---|---|
| **REQUIRED** | Always executes in a transaction. If there is any existing transaction it uses it. If none exists then only a new one is created. |
| **SUPPORTS** | It may or may not run in a transaction. If current transaction exists then it is supported. If none exists then gets executed without transaction. |
| **NOT_SUPPORTED** | Always executes without a transaction. If there is any existing transaction it gets suspended. |
| **REQUIRES_NEW** | Always executes in a new transaction. If there is any existing transaction it gets suspended. |
| **NEVER** | Always executes without any transaction. It throws an exception if there is an existing transaction. |
| **MANDATORY** | Always executes in a transaction. If there is any existing transaction it is used. If there is no existing transaction it will throw an exception. |

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void someMethod() {
    // always runs in its own new transaction
}
```

## 7. Transaction Isolation Levels

Isolation controls how/whether concurrent transactions see each other's uncommitted or in-progress changes. The differences between levels are best understood through three "read anomalies" they may or may not allow:

- **Dirty Read** — Transaction B reads a row that Transaction A has changed but not yet committed. If A rolls back, B acted on data that never really existed.
- **Non-Repeatable Read** — Transaction A reads the same row twice, and gets a *different value* the second time because Transaction B updated + committed it in between.
- **Phantom Read** — Transaction A re-runs the same range query twice, and gets a *different set of rows* the second time because Transaction B inserted/deleted a matching row in between.

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|---|---|---|
| **READ_UNCOMMITTED** | ✅ Possible | ✅ Possible | ✅ Possible |
| **READ_COMMITTED** | ❌ Prevented | ✅ Possible | ✅ Possible |
| **REPEATABLE_READ** | ❌ Prevented | ❌ Prevented | ✅ Possible |
| **SERIALIZABLE** | ❌ Prevented | ❌ Prevented | ❌ Prevented |
| **DEFAULT** | Uses whatever the underlying database's default isolation level is (e.g. `READ_COMMITTED` on Oracle/PostgreSQL, `REPEATABLE_READ` on MySQL InnoDB). |

### READ_UNCOMMITTED — example (dirty read)

Bank balance row starts at `1000`.

1. **Txn A** debits 500: `UPDATE account SET balance = 500 WHERE id = 1;` — **not yet committed**.
2. **Txn B** (running at READ_UNCOMMITTED) reads the row and sees `balance = 500`, and shows it to the user.
3. **Txn A** hits an error and **rolls back** — balance reverts to `1000`.
4. Txn B acted on `500`, a value that never actually existed in committed history. That's the dirty read.

### READ_COMMITTED — example (non-repeatable read)

Balance row starts at `1000`.

1. **Txn B** reads the row: `balance = 1000`.
2. **Txn A** updates it to `800` and **commits**.
3. **Txn B**, still in the same transaction, reads the row again: now sees `balance = 800`.
4. Txn B got two different values for the same row in one transaction — a non-repeatable read. (No dirty read happened, because Txn B only ever saw *committed* values.)

### REPEATABLE_READ — example (phantom read)

1. **Txn A** runs `SELECT * FROM employees WHERE dept = 'HR';` → returns 5 rows.
2. **Txn B** inserts a new HR employee and **commits**.
3. **Txn A** re-runs the *exact same query* → now returns 6 rows.
4. The existing 5 rows Txn A read are still consistent (REPEATABLE_READ guarantees that), but a new "phantom" row appeared — the guarantee doesn't cover rows that didn't exist yet at the first read.

### SERIALIZABLE — example (phantom prevented)

Same scenario as above, but at SERIALIZABLE: Txn B's `INSERT` into the HR department would be **blocked until Txn A commits/completes** (or one of them fails with a serialization conflict). The two transactions behave as if run one strictly after the other — so Txn A's second read of the HR query is guaranteed to still return exactly 5 rows.

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void someMethod() {
    // ...
}

@Transactional(isolation = Isolation.SERIALIZABLE)
public void transferFunds(Long fromId, Long toId, BigDecimal amount) {
    // strictest isolation — good for money transfers, bad for throughput
}
```

## Quick combined example

```java
@Transactional(
    propagation = Propagation.REQUIRED,
    isolation = Isolation.READ_COMMITTED,
    rollbackFor = Exception.class
)
public void joinOrganization(Employee employee,
        EmployeeHealthInsurance employeeHealthInsurance) {
    // business logic
}
```