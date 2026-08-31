# Oracle SQL 練習筆記｜員工與部門查詢

來源：三小時全端上機模擬考練習題（Oracle SQL 部分）。

## 資料表結構

```sql
CREATE TABLE DEPARTMENT (
    DEPARTMENT_ID NUMBER(10) PRIMARY KEY,
    DEPARTMENT_NAME VARCHAR2(100) NOT NULL,
    LOCATION VARCHAR2(100)
);

CREATE TABLE EMPLOYEE (
    EMPLOYEE_ID NUMBER(10) PRIMARY KEY,
    EMPLOYEE_NO VARCHAR2(20) NOT NULL UNIQUE,
    EMPLOYEE_NAME VARCHAR2(100) NOT NULL,
    EMAIL VARCHAR2(150) NOT NULL,
    SALARY NUMBER(12,2),
    HIRE_DATE DATE NOT NULL,
    STATUS VARCHAR2(20) DEFAULT 'ACTIVE' NOT NULL,
    DEPARTMENT_ID NUMBER(10),
    UPDATED_AT DATE,
    CONSTRAINT FK_EMP_DEPT FOREIGN KEY (DEPARTMENT_ID)
        REFERENCES DEPARTMENT(DEPARTMENT_ID)
);
```

---

## 第 1 題｜有效員工與部門

**需求**
- 查詢所有 `STATUS = 'ACTIVE'` 的員工。
- 列出 `EMPLOYEE_NO`、`EMPLOYEE_NAME`、`EMAIL`、`SALARY`、`HIRE_DATE`、`DEPARTMENT_NAME`。
- 沒有部門的員工也必須顯示。
- 依 `HIRE_DATE` 由新到舊；相同時依 `EMPLOYEE_NO` 由小到大。

```sql
SELECT
    e.EMPLOYEE_NO,
    e.EMPLOYEE_NAME,
    e.EMAIL,
    e.SALARY,
    e.HIRE_DATE,
    d.DEPARTMENT_NAME
FROM EMPLOYEE e
LEFT JOIN DEPARTMENT d ON e.DEPARTMENT_ID = d.DEPARTMENT_ID
WHERE e.STATUS = 'ACTIVE'
ORDER BY e.HIRE_DATE DESC, e.EMPLOYEE_NO ASC;
```

**重點**：`LEFT JOIN` 保留沒有部門的員工；`WHERE` 一定要寫在 `ORDER BY` 之前。

---

## 第 2 題｜部門統計

**需求**
- 列出每個部門的有效員工人數、平均薪資、最高薪資與最低薪資。
- 沒有有效員工的部門仍要顯示，員工人數為 0。
- 平均薪資四捨五入至小數點後兩位。

```sql
SELECT
    d.DEPARTMENT_NAME,
    COUNT(e.EMPLOYEE_NO) TOTAL_EMP_COUNT,
    ROUND(AVG(e.SALARY), 2) AVG_SAL,
    MAX(e.SALARY) MAX_SAL,
    MIN(e.SALARY) MIN_SAL
FROM DEPARTMENT d
LEFT JOIN EMPLOYEE e
    ON d.DEPARTMENT_ID = e.DEPARTMENT_ID
    AND e.STATUS = 'ACTIVE'
GROUP BY d.DEPARTMENT_ID, d.DEPARTMENT_NAME;
```

**重點**：篩選條件（`e.STATUS = 'ACTIVE'`）要放在 `LEFT JOIN ... ON` 裡，而不是 `WHERE`，否則會把「沒有有效員工的部門」整列濾掉，違反題目要求。

---

## 第 3 題｜高於部門平均

**需求**
- 找出薪資高於自己部門平均薪資的有效員工。
- 列出 `EMPLOYEE_NAME`、`DEPARTMENT_NAME`、`SALARY`、`DEPT_AVG_SALARY`、`SALARY_DIFF`。
- 部門平均只計算有效員工。

```sql
SELECT
    EMPLOYEE_NAME,
    DEPARTMENT_NAME,
    SALARY,
    DEPT_AVG_SALARY,
    SALARY - DEPT_AVG_SALARY AS SALARY_DIFF
FROM (
    SELECT
        e.EMPLOYEE_NAME,
        d.DEPARTMENT_NAME,
        e.SALARY,
        ROUND(AVG(e.SALARY) OVER (PARTITION BY e.DEPARTMENT_ID), 2) AS DEPT_AVG_SALARY
    FROM EMPLOYEE e
    JOIN DEPARTMENT d ON e.DEPARTMENT_ID = d.DEPARTMENT_ID
    WHERE e.STATUS = 'ACTIVE'
)
WHERE SALARY > DEPT_AVG_SALARY;
```

**重點**：用分析函數 `AVG(...) OVER (PARTITION BY ...)` 一次算出每個部門的平均薪資，再包一層外層查詢篩選「高於平均」，比自己寫子查詢 JOIN 更簡潔。

---

## 第 4 題｜各部門最高薪資

**需求**
- 找出每個部門薪資最高的有效員工。
- 同一部門若並列最高，全部顯示。
- 沒有員工的部門不必顯示。

```sql
WITH RANK_TABLE AS (
    SELECT
        e.*,
        RANK() OVER (PARTITION BY e.DEPARTMENT_ID ORDER BY e.SALARY DESC NULLS LAST) AS RK
    FROM EMPLOYEE e
    WHERE e.STATUS = 'ACTIVE'
)
SELECT
    d.DEPARTMENT_NAME,
    r.EMPLOYEE_NO,
    r.EMPLOYEE_NAME,
    r.SALARY
FROM RANK_TABLE r
JOIN DEPARTMENT d ON d.DEPARTMENT_ID = r.DEPARTMENT_ID
WHERE r.RK = 1
ORDER BY d.DEPARTMENT_NAME, r.EMPLOYEE_NO;
```

**重點**：用 `RANK()`（而非 `ROW_NUMBER()`）讓並列最高薪資的員工都能顯示；最後一定要在外層加 `WHERE r.RK = 1` 篩選，否則整張排名表都會被列出來。

---

## 第 5 題｜分頁與更新

### (a) 薪資排名第 11–20 筆

**需求**：薪資高到低，相同時 `EMPLOYEE_NO` 小到大；不可在程式中分頁。

```sql
SELECT e.EMPLOYEE_NO, e.EMPLOYEE_NAME, e.SALARY
FROM EMPLOYEE e
WHERE e.STATUS = 'ACTIVE'
ORDER BY e.SALARY DESC NULLS LAST, e.EMPLOYEE_NO ASC
OFFSET 10 ROWS FETCH NEXT 10 ROWS ONLY;
```

**重點**：Oracle 12c 以後用 `OFFSET ... ROWS FETCH NEXT ... ROWS ONLY` 做資料庫端分頁；`LIMIT/OFFSET` 是 MySQL/PostgreSQL 語法，Oracle 不支援。

### (b) 加薪 5%

**需求**：`ACTIVE`、到職滿 3 年、薪資低於所屬部門有效員工平均的員工，加薪 5%。

```sql
UPDATE EMPLOYEE e
SET e.SALARY = e.SALARY * 1.05,
    e.UPDATED_AT = SYSDATE
WHERE e.STATUS = 'ACTIVE'
    AND e.HIRE_DATE <= ADD_MONTHS(SYSDATE, -36)
    AND e.SALARY < (
        SELECT AVG(e2.SALARY)
        FROM EMPLOYEE e2
        WHERE e2.DEPARTMENT_ID = e.DEPARTMENT_ID
          AND e2.STATUS = 'ACTIVE'
    );
```

**重點**：這題是 `UPDATE`，不是 `SELECT`；到職年資用 `ADD_MONTHS(SYSDATE, -36)` 判斷，Oracle 沒有 `TIMESTAMPDIFF`／`CURRENT()` 這兩個 MySQL 函數。

---

## 常見錯誤整理

- **子句順序寫反**：正確順序固定是 `SELECT → FROM → JOIN → WHERE → GROUP BY → HAVING → ORDER BY`，`WHERE` 絕不能寫在 `GROUP BY` 或 `ORDER BY` 之後。
- **LEFT JOIN + WHERE 的陷阱**：想保留右表沒有匹配的列時，過濾條件要寫進 `ON`；寫進 `WHERE` 會把這些列整個濾掉。
- **混用 MySQL / Oracle 語法**：`LIMIT/OFFSET`、`TIMESTAMPDIFF`、`CURRENT()` 是 MySQL 語法，Oracle 要用 `OFFSET...FETCH`、`MONTHS_BETWEEN`/`ADD_MONTHS`、`SYSDATE`。
- **視窗函數後忘記篩選**：`RANK()`/`ROW_NUMBER()` 算完排名後，通常還需要外層 `WHERE` 篩選排名結果，否則整張表都會被列出來。
