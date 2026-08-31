# Spring Boot 練習筆記｜員工管理 API（完整版）

來源：三小時全端上機模擬考練習題（Spring Boot 部分，35 分／65 分鐘），實作於 `jpaCrudExercise` 專案。

## 情境與需求總覽

半成品已能啟動並連到資料庫，要完成一支「員工管理 API」。

**資料格式**

```json
{
  "id": 1,
  "employeeNo": "E0001",
  "employeeName": "Amy Chen",
  "email": "amy@example.com",
  "salary": 65000,
  "hireDate": "2023-03-15",
  "status": "ACTIVE",
  "departmentId": 10,
  "departmentName": "IT"
}
```

**必做 API**

| Method | Path | 要求 |
|---|---|---|
| GET | `/api/employees` | `keyword`、`status`、`page`、`size`；回傳分頁結果 |
| GET | `/api/employees/{id}` | 不存在回傳 404 |
| POST | `/api/employees` | 成功回傳 201 Created |
| PUT | `/api/employees/{id}` | 以 path id 為準；成功回傳 200 |
| DELETE | `/api/employees/{id}` | 成功回傳 204 No Content |
| GET | `/api/departments` | 供前端部門下拉選單使用 |

**列表查詢規則**
- `keyword`：不區分大小寫，同時搜尋員工姓名、員工編號或 Email。
- `status`：選填；僅允許 `ACTIVE` 或 `INACTIVE`。
- `page` 預設 0；`size` 預設 10；預設依 `HIRE_DATE` 降冪。
- 回傳至少包含 `content`、`page`、`size`、`totalElements`、`totalPages`。

**新增／修改驗證**
- `employeeNo`：必填，最多 20 字；不可重複。
- `employeeName`：必填，最多 100 字。
- `email`：必填、格式正確；不可重複且不區分大小寫。
- `salary`：不得小於 0。`hireDate`：必填且不得晚於今天。
- `status`：`ACTIVE` 或 `INACTIVE`。`departmentId`：必須指向存在的部門。

**錯誤與結構要求**
- 驗證失敗回 400；找不到員工或部門回 404；員工編號或 Email 重複回 409。
- 至少包含 `controller`、`service`、`repository`、`entity`、`dto`、`exception` 分層。
- Request／Response 不直接暴露 Entity。錯誤格式需一致，至少包含 `status`、`message`、`fieldErrors`。

**評分優先順序**：可啟動與 HTTP 行為 > 資料正確性 > 驗證／例外 > 程式美化。

---

## 專案分層

```
entity/       Employee, Department                  資料表對應
dto/          CreateEmployeeRequest, UpdateEmployeeRequest,
              EmployeeResponse, DepartmentResponse,
              PageResponse<T>, ErrorResponse          進出的資料格式，不暴露 Entity
repository/   EmployeeRepository, DepartmentRepository JPA 查詢
service/      EmployeeService, DepartmentService       業務邏輯（含驗證）
controller/   EmployeeController, DepartmentController REST 入口
exception/    ResourceNotFoundException,
              DuplicateResourceException,
              GlobalExceptionHandler                   統一例外處理
```

---

## 1. 列表查詢：`GET /api/employees`

`keyword`、`status` 都是選填，用 JPQL 的 `:param IS NULL OR ...` 讓同一條查詢同時支援「有帶」跟「沒帶」：

```java
@Query("SELECT e FROM Employee e WHERE " +
        "(:status IS NULL OR e.status = :status) AND " +
        "(:keyword IS NULL OR " +
        " LOWER(e.employeeName) LIKE LOWER(CONCAT('%', :keyword, '%')) OR " +
        " LOWER(e.employeeNo) LIKE LOWER(CONCAT('%', :keyword, '%')) OR " +
        " LOWER(e.email) LIKE LOWER(CONCAT('%', :keyword, '%')))")
Page<Employee> search(@Param("keyword") String keyword,
                       @Param("status") EmployeeStatus status,
                       Pageable pageable);
```

Controller 接住 4 個 query 參數，page/size 給預設值：

```java
@GetMapping
public ResponseEntity<PageResponse<EmployeeResponse>> getEmployeeList(
        @RequestParam(required = false) String keyword,
        @RequestParam(required = false) String status,
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size) {
    return ResponseEntity.ok(employeeService.getEmployeeList(keyword, status, page, size));
}
```

Service 把查詢條件正規化（空字串視為不篩選、`status` 白名單檢查），排序寫死在 `Pageable` 裡：

```java
Pageable pageable = PageRequest.of(page, size, Sort.by(Sort.Direction.DESC, "hireDate"));
Page<Employee> result = employeeRepository.search(normalizedKeyword, normalizedStatus, pageable);
return new PageResponse<>(result.map(EmployeeResponse::from));
```

`PageResponse<T>` 包住 Spring Data 的 `Page<T>`，固定輸出 `content`／`page`／`size`／`totalElements`／`totalPages`：

```java
public class PageResponse<T> {
    private final List<T> content;
    private final int page;
    private final int size;
    private final long totalElements;
    private final int totalPages;

    public PageResponse(Page<T> page) {
        this.content = page.getContent();
        this.page = page.getNumber();
        this.size = page.getSize();
        this.totalElements = page.getTotalElements();
        this.totalPages = page.getTotalPages();
    }
}
```

**重點**：`status` 傳入不合法字串（如 `FOO`）直接丟 400，不是靜默忽略或當成查無資料。

---

## 2. 單筆查詢：`GET /api/employees/{id}`

```java
@GetMapping("/{id}")
public ResponseEntity<EmployeeResponse> getEmployeeById(@PathVariable Long id) {
    return ResponseEntity.ok(employeeService.getEmployeeById(id));
}
```

不存在時 Service 丟 `ResourceNotFoundException`（`@ResponseStatus(HttpStatus.NOT_FOUND)`），交給全域例外處理器轉成統一格式的 404。

---

## 3. 新增／修改驗證：`POST` / `PUT /api/employees/{id}`

**Bean Validation 能處理的**（欄位必填、長度、格式、範圍、日期）寫在 DTO：

```java
@NotBlank(message = "employeeNo 為必填")
@Size(max = 20, message = "employeeNo 最多 20 字")
private String employeeNo;

@NotBlank(message = "email 為必填")
@Email(message = "email 格式不正確")
private String email;

// @Min 對 null 不生效（null 視為合法），符合「選填，但填了不得小於 0」
@Min(value = 0, message = "salary 不得小於 0")
private Integer salary;

@NotNull(message = "hireDate 為必填")
@PastOrPresent(message = "hireDate 不得晚於今天")
private Date hireDate;

// Entity 的 EmployeeStatus 其實還有第三個值 FREEZE（其他模組在用）。
// 若直接用 @RequestBody 綁 Enum，FREEZE 會被 Jackson 直接反序列化成功、繞過檢查，
// 所以改用 String + @Pattern 白名單，才能真正擋掉 ACTIVE/INACTIVE 以外的值。
@Pattern(regexp = "ACTIVE|INACTIVE", message = "status 僅允許 ACTIVE 或 INACTIVE")
private String status;
```

**要查資料庫才知道的規則**放在 Service 層：

```java
// employeeNo 不可重複；修改時要排除自己，避免「不改 employeeNo 直接存」被誤判成重複
private void validateEmployeeNoNotDuplicated(String employeeNo, Long excludeId) {
    boolean duplicated = (excludeId == null)
            ? employeeRepository.existsByEmployeeNo(employeeNo)
            : employeeRepository.existsByEmployeeNoAndIdNot(employeeNo, excludeId);
    if (duplicated) {
        throw new DuplicateResourceException("employeeNo 已存在：" + employeeNo);
    }
}

// departmentId 選填；有填的話必須指向存在的部門，順便把 departmentName 存回去
private void applyDepartment(Employee employee, Long departmentId) {
    if (departmentId == null) {
        employee.setDepartmentId(null);
        employee.setDepartmentName(null);
        return;
    }
    Department department = departmentRepository.findById(departmentId)
            .orElseThrow(() -> new ResourceNotFoundException("查無此部門，departmentId=" + departmentId));
    employee.setDepartmentId(department.getId());
    employee.setDepartmentName(department.getDepartmentName());
}
```

Controller 用 `@Valid` 觸發驗證，新增回 201、修改以 path id 為準回 200：

```java
@PostMapping
public ResponseEntity<EmployeeResponse> createEmployee(@Valid @RequestBody CreateEmployeeRequest request) {
    return ResponseEntity.status(HttpStatus.CREATED).body(employeeService.createEmployee(request));
}

@PutMapping("/{id}")
public ResponseEntity<EmployeeResponse> updateEmployee(@PathVariable Long id,
                                                        @Valid @RequestBody UpdateEmployeeRequest request) {
    return ResponseEntity.ok(employeeService.updateEmployee(id, request));
}
```

---

## 4. 刪除：`DELETE /api/employees/{id}`

```java
@DeleteMapping("/{id}")
public ResponseEntity<Void> deleteEmployee(@PathVariable Long id) {
    boolean isDeleted = employeeService.deleteEmployee(id);
    if (isDeleted) {
        return ResponseEntity.noContent().build();   // 204
    }
    return ResponseEntity.notFound().build();          // 404
}
```

---

## 5. 部門下拉選單：`GET /api/departments`

資料量小、給前端下拉選單用，不做分頁，直接回傳陣列：

```java
@GetMapping
public ResponseEntity<List<DepartmentResponse>> getDepartmentList() {
    return ResponseEntity.ok(departmentService.getAllDepartments());
}
```

---

## 6. 統一錯誤格式

用 `@RestControllerAdvice` 把三種錯誤都轉成同一種 JSON 結構（`status`、`message`、`fieldErrors`）：

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // @Valid 驗證失敗 -> 400，附上每個欄位各自的錯誤訊息
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) { ... }

    // 查詢參數不合法（例如 status=FOO）-> 400
    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<ErrorResponse> handleBadRequest(IllegalArgumentException ex) { ... }

    // 找不到員工／部門 -> 404
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) { ... }

    // employeeNo / email 重複 -> 409
    @ExceptionHandler(DuplicateResourceException.class)
    public ResponseEntity<ErrorResponse> handleDuplicate(DuplicateResourceException ex) { ... }
}
```

## 實測結果對照（六支 API 皆已啟動實測）

| API | 情境 | HTTP | 回應重點 |
|---|---|---|---|
| GET 列表 | 預設查詢 | 200 | `content`/`page`/`size`/`totalElements`/`totalPages` 齊全 |
| GET 列表 | `keyword=amy` | 200 | 只回符合姓名/編號/Email 的員工 |
| GET 列表 | `status=INACTIVE` | 200 | 只回該狀態員工 |
| GET 列表 | `status=FOO` | 400 | `status 僅允許 ACTIVE 或 INACTIVE` |
| GET 單筆 | id 不存在 | 404 | `查無此員工，id=999` |
| POST | 缺必填欄位 | 400 | `fieldErrors` 列出每個缺的欄位 |
| POST | `departmentId` 不存在 | 404 | `查無此部門，departmentId=999` |
| POST | `employeeNo` 重複 | 409 | `employeeNo 已存在：E0001` |
| POST | `email` 重複（大小寫不同） | 409 | `email 已存在：AMY@example.com` |
| POST | `hireDate` 晚於今天 | 400 | `hireDate 不得晚於今天` |
| POST | `salary` 為負數 | 400 | `salary 不得小於 0` |
| POST | `status=FREEZE` | 400 | `status 僅允許 ACTIVE 或 INACTIVE` |
| POST | 全部合法 | 201 | 回傳含 `departmentName` 的員工資料 |
| PUT | id 存在 | 200 | 回傳更新後的資料 |
| PUT | id 不存在 | 404 | `查無此員工，id=999` |
| DELETE | id 存在 | 204 | 無內容 |
| DELETE | id 不存在 | 404 | — |
| GET 部門 | — | 200 | 回傳部門陣列供下拉選單使用 |

---

## 順手修掉的既有 Bug

補這支 API 的過程中發現幾個讓程式完全跑不起來或行為錯誤的既有問題：

- `EmployeeController` 原本沒有 `@RestController`／`@RequestMapping` 標註，等於沒有被註冊成真正的 API。
- `updateEmployee` 漏了 `@PathVariable`／`@RequestBody`，而且回傳的是「傳入的參數」而不是「存進 DB 後的結果」。
- `EmployeeServiceImpl` 漏了 `@Service`，Spring 不會把它組成 Bean。
- 找不到員工時丟的是 `RuntimeException("查無此書")`（從 Book 模組複製時忘記改訊息）。
- `BorrowBookDetail` 這個 Entity 缺少 `@Entity`／`@Table` 標註，會導致 Hibernate 直接讓整個 Spring Context 啟動失敗（連 Employee API 都連帶測不了）。
- `target/classes` 裡有一份搬移套件前的 `BookServiceImpl.class` 殘留檔案，跟現在 `service.impl` 底下同名的類別衝突，`mvn clean` 後才解決。

---

## 常見誤區整理

- **選填欄位用 `@NotBlank`／`@NotNull` 硬擋**：像 `salary`、`status`、`departmentId` 都是選填但有值時才驗證，混用會誤把合法的「沒填」擋掉。
- **重複值驗證要排除自己**：更新既有資料時，查重複沒排除自己的 id，會導致「不改該欄位、直接存原值」被誤判成重複。
- **直接把 Request Body 綁進 Enum**：會讓 Enum 裡「規格不允許但技術上合法」的值（如本例的 `FREEZE`）繞過白名單檢查，應該先用 `String` 接住，再自己做規則檢查。
- **DB 相關的驗證不能只靠 Bean Validation**：唯一值、外鍵是否存在這類要查資料庫才能確定的規則，必須在 Service 層做，Annotation 做不到。
- **`spring-boot-starter-validation` 容易忘記加**：沒有這個依賴，`@Valid`、`@NotBlank` 等註解不會生效，驗證會整個被跳過而不是報錯，容易誤以為程式碼寫對了。
