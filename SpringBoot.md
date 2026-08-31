# Spring Boot 練習筆記｜員工管理 API

來源：三小時全端上機模擬考練習題（Spring Boot 部分），實作於 `jpaCrudExercise` 專案的 Employee 模組。

## 需求

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

---

## 分頁查詢：Repository + 統一分頁回應

`keyword`、`status` 都是選填，用 JPQL 的 `:param IS NULL OR ...` 寫法讓同一條查詢同時支援「有帶」跟「沒帶」兩種情況：

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

Service 端把查詢條件正規化（空字串視為不篩選、`status` 白名單檢查），排序寫死在 `Pageable` 裡：

```java
Pageable pageable = PageRequest.of(page, size, Sort.by(Sort.Direction.DESC, "hireDate"));
Page<Employee> result = employeeRepository.search(normalizedKeyword, normalizedStatus, pageable);
return new PageResponse<>(result.map(EmployeeResponse::from));
```

`PageResponse<T>` 是包住 Spring Data `Page<T>` 的固定格式，確保回傳一定有 `content`／`page`／`size`／`totalElements`／`totalPages` 五個欄位：

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
    // getters ...
}
```

**重點**：`status` 傳入不合法字串（例如 `FOO`）要直接丟 400，而不是靜默忽略或當成查無資料；這裡選擇在 Service 丟 `IllegalArgumentException`，交給全域例外處理器轉成統一格式。

---

## 新增／修改驗證：Bean Validation + Service 層業務規則

Bean Validation（`jakarta.validation`）能處理的：欄位必填、長度上限、Email 格式、數字範圍、日期不得晚於今天：

```java
@NotBlank(message = "employeeNo 為必填")
@Size(max = 20, message = "employeeNo 最多 20 字")
private String employeeNo;

@NotBlank(message = "email 為必填")
@Email(message = "email 格式不正確")
private String email;

// @Min 對 null 不生效（null 視為合法），剛好符合「選填，但填了不得小於 0」
@Min(value = 0, message = "salary 不得小於 0")
private Integer salary;

@NotNull(message = "hireDate 為必填")
@PastOrPresent(message = "hireDate 不得晚於今天")
private Date hireDate;

// 用 String 接住 status 而不是直接綁 Enum：
// Entity 的 EmployeeStatus 其實還有第三個值 FREEZE（其他功能在用），
// 如果直接用 @RequestBody 綁 Enum，FREEZE 會被 Jackson 直接反序列化成功、繞過檢查。
// 改用 String + @Pattern 白名單，才能真正擋掉 ACTIVE/INACTIVE 以外的值。
@Pattern(regexp = "ACTIVE|INACTIVE", message = "status 僅允許 ACTIVE 或 INACTIVE")
private String status;
```

Bean Validation 沒辦法處理的（要查資料庫才知道）：`employeeNo`／`email` 是否重複、`departmentId` 是否存在，這兩類放在 Service 層：

```java
// employeeNo 不可重複；修改時要排除自己，不然「不改 employeeNo 直接存」會誤判成重複
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

---

## 統一錯誤格式

用 `@RestControllerAdvice` 把三種錯誤都轉成同一種 JSON 結構（`status`、`message`、`fieldErrors`），Controller／Service 不用各自組錯誤訊息：

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // @Valid 驗證失敗 -> 400，附上每個欄位各自的錯誤訊息
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) { ... }

    // 找不到員工／部門 -> 404
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) { ... }

    // employeeNo / email 重複 -> 409
    @ExceptionHandler(DuplicateResourceException.class)
    public ResponseEntity<ErrorResponse> handleDuplicate(DuplicateResourceException ex) { ... }
}
```

實測結果對照：

| 情境 | HTTP | 回應 |
|---|---|---|
| 缺必填欄位 | 400 | `fieldErrors` 列出每個缺的欄位 |
| `departmentId` 不存在 | 404 | `查無此部門，departmentId=999` |
| `employeeNo` 重複 | 409 | `employeeNo 已存在：E0001` |
| `email` 重複（大小寫不同） | 409 | `email 已存在：AMY@example.com` |
| `hireDate` 晚於今天 | 400 | `hireDate 不得晚於今天` |
| `salary` 為負數 | 400 | `salary 不得小於 0` |
| `status` 傳 `FREEZE` | 400 | `status 僅允許 ACTIVE 或 INACTIVE` |

---

## 順手修掉的既有 Bug

補上功能的過程中發現 `EmployeeController` 原本完全沒有 `@RestController`／`@RequestMapping` 標註（等於這個 Controller 沒有被註冊成真正的 API），`updateEmployee` 也漏了 `@PathVariable`／`@RequestBody`，而且回傳的是「傳入的參數」而不是「更新後存進 DB 的結果」；`EmployeeServiceImpl` 也漏了 `@Service`，找不到員工時丟的是 `RuntimeException("查無此書")`（複製 Book 模組時忘記改訊息）。這些都是讓 API 完全跑不起來或行為不對的重要 bug，一併修正。

## 常見誤區整理

- **選填欄位用 `@NotBlank`／`@NotNull` 硬擋**：像 `salary`、`status`、`departmentId` 都是選填但有值時才驗證，混用會誤把合法的「沒填」擋掉。
- **重複值驗證要排除自己**：更新既有資料時，如果查重複沒有排除自己的 id，會導致「不改該欄位、直接存原值」被誤判成重複。
- **直接把 Request Body 綁進 Enum**：會讓 Enum 裡「規格不允許但技術上合法」的值（如本例的 `FREEZE`）繞過白名單檢查，應該先用 `String` 接住，再自己做規則檢查。
- **DB 相關的驗證不能只靠 Bean Validation**：唯一值、外鍵是否存在這類要查資料庫才能確定的規則，必須在 Service 層做，Annotation 做不到。
