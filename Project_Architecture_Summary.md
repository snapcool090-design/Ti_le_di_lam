# PROJECT ARCHITECTURE SUMMARY
## Ứng dụng "Khảo Sát Đi Làm" — CNC PQC 3P (Google Apps Script Web App)

> ## 🟢 CURRENT BASELINE — v1.1 (Verified)
> Tài liệu này đã trải qua 1 vòng **Architecture Verification** (đối chiếu trực tiếp với `code.gs`, `index.html`, `css.html`). Mọi phát hiện từ verification đã được hợp nhất vào bản dưới đây. **Đây là baseline chính thức, dùng làm cơ sở cho mọi phân tích và thay đổi code tiếp theo.**

**Phiên bản tài liệu:** Baseline v1.1 (đã qua Architecture Verification)
**Phạm vi:** `code.gs`, `index.html`, `css.html`
**Mục đích:** Tài liệu tham chiếu chính thức cho mọi thay đổi trong tương lai.

### Changelog v1.0 → v1.1
| # | Thay đổi | Nguồn |
|---|---|---|
| 1 | Bổ sung `getPendingEmployeesByLeader()` vào function inventory (mục 2.4) — orphan function bị bỏ sót ở v1.0 | Verification #1 |
| 2 | Nâng cấp `getEmployeeByGen()` từ UNKNOWN → **FACT — Confirmed Bug/Technical Debt** (mục 10, 12) | Verification #2 |
| 3 | Ghi nhận biến `shiftDisplay` (index.html) là dead local variable (mục 10) | Verification #3 |
| 4 | Ghi nhận `.question-card.hidden` (css.html) là duplicate + dead CSS (mục 4.2, 10) | Verification #4 |
| 5 | Ghi nhận selector không nhất quán trong `resetUIAfterLogout()` (mục 6, 10) | Verification #5 |
| 6 | Ghi nhận rủi ro nghiệp vụ của `processSupport()` — ghi đè toàn cột (mục 8, 10) | Verification #6 |
| 7 | Bổ sung độ chính xác: `checkGenCode()` trả cả `adminRole` thô lẫn `role` tính toán (mục 9) | Verification (Nhóm 1, mục 7) |

**Nguyên tắc:** Không có nội dung nào trong v1.0 bị xóa; chỉ bổ sung/nâng cấp mức độ chắc chắn. Không có thay đổi code nào được thực hiện trong quá trình cập nhật tài liệu này.

> Quy ước nhãn: **[FACT]** = có bằng chứng trực tiếp trong code · **[INFERENCE]** = suy luận hợp lý nhưng chưa xác nhận · **[UNKNOWN]** = không thể xác định từ code hiện có.

---

## 1. PROJECT OVERVIEW

### 1.1 Mục đích ứng dụng
**[FACT]** Đây là hệ thống khảo sát tình trạng đi làm hàng ngày của nhân viên (đi làm bình thường / nghỉ phép / nghỉ không lương / đi muộn / nghỉ nửa buổi / đi đào tạo...), phân theo ca (ngày/đêm) và theo leader phụ trách. Hệ thống có 3 nhóm quyền: `user` (nhân viên thường), `leader` (trưởng nhóm), `admin` (quản trị).

**[FACT]** Tiêu đề hiển thị: "CNC PQC 3P — KHẢO SÁT ĐI LÀM".

### 1.2 Kiến trúc tổng thể
**[FACT]** Kiến trúc 3 lớp chuẩn của Google Apps Script Web App:

```
┌─────────────────────────────────────────┐
│  Google Sheets (data store)              │
│  Employee_Master / Master_Questions /    │
│  Survey_Results / Di_support             │
└───────────────▲───────────────────────────┘
                │ SpreadsheetApp API
┌───────────────┴───────────────────────────┐
│  code.gs (server-side, chạy trên GAS)     │
└───────────────▲───────────────────────────┘
                │ google.script.run (async RPC)
┌───────────────┴───────────────────────────┐
│  index.html (UI + client-side JS)         │
│  css.html (included inline vào index.html)│
└─────────────────────────────────────────┘
```

**[FACT]** `doGet()` render `index.html` qua `HtmlService.createTemplateFromFile(...).evaluate()`, cho phép nhúng iframe toàn quyền (`XFrameOptionsMode.ALLOWALL`).

**[FACT]** Ứng dụng là SPA giả lập: chỉ có 1 trang HTML, chuyển "tab" bằng show/hide `div.tab-content`, không có route URL thực.

**[FACT]** Xác thực dựa trên mã GEN (8 chữ số) tra trong `Employee_Master`, không dùng OAuth người dùng cuối cho mục đích phân quyền nghiệp vụ.

### 1.3 Vai trò từng file
| File | Vai trò |
|---|---|
| `code.gs` | Toàn bộ logic server-side: đọc/ghi Google Sheets, xử lý nghiệp vụ, xuất CSV, xác thực, trigger. |
| `index.html` | Toàn bộ UI (login, khảo sát, thống kê, tra cứu, quản trị) + toàn bộ JavaScript client-side + gọi RPC tới `code.gs`. |
| `css.html` | Toàn bộ style, được include vào `<head>` của `index.html` qua `include('css.html')`. |

---

## 2. BACKEND — `code.gs`

### 2.1 Configuration (đọc thêm mục 7 & 8)
**[FACT]**
```js
CONFIG.SHEETS         = { EMPLOYEE_MASTER, MASTER_QUESTIONS, SURVEY_RESULTS, DI_SUPPORT }
CONFIG.USER_ROLES     = { ADMIN:'admin', LEADER:'leader', USER:'user' }
CONFIG.ANSWER_TYPES   = { NORMAL:'', SPECIAL:'Y' }
CONFIG.COLUMNS.EMPLOYEE_MASTER = { NAME:0, GEN:1, LEADER_ID:2, LEADER_NAME:3, EMPLOYEE_TYPE:4, RESIGNED:5, ADMIN_ROLE:6 }
SR_COL = { TIMESTAMP:0, LAST_EDITED:1, GEN:2, NAME:3, LEADER:4, EMPLOYEE_TYPE:5, SURVEY_DATE:6, Q3:7, Q4_ANSWER:8 }
```
Cột động (Q4 trở lên): `getAnswerColForQuestionIndex(i) = 8 + i*2`, `getReasonColForQuestionIndex(i) = 8 + i*2 + 1`.

---

### 2.2 Entry Points

| Function | Input | Output | Chức năng | Gọi bởi | Gọi tới | Tác động Sheets |
|---|---|---|---|---|---|---|
| `doGet()` | HTTP GET request (implicit) | `HtmlOutput` | Render `index.html`, set title + viewport meta | GAS runtime | `HtmlService`, `include()` (gián tiếp qua template) | Không |
| `include(filename)` | `filename: string` | HTML content string | Nhúng nội dung file khác (dùng để include `css.html`) | Template `index.html` | `HtmlService.createHtmlOutputFromFile` | Không |
| `onOpen()` | none (simple trigger) | none | Tạo custom menu trong Spreadsheet UI | GAS (khi mở Spreadsheet) | `SpreadsheetApp.getUi()` | Không |

---

### 2.3 API/Server Functions được Frontend gọi (qua `google.script.run`)

| Function | Input | Output | Chức năng | Gọi tới | Sheets |
|---|---|---|---|---|---|
| `checkGenCode(genCode)` | `genCode: string` | object user data hoặc throw Error | Login: validate GEN, check nghỉ việc, xác định role | `findUserInMaster`, `getUserData`, `determineUserRole` | Đọc `Employee_Master` |
| `verifyAdminPassword(genCode, passwordPlain)` | 2 string | `{success, matched, hasPassword}` | Xác thực mật khẩu admin (hỗ trợ plaintext cũ + hash SHA-256 mới) | `hashPassword` | Đọc `Employee_Master` |
| `getSurveyDataForUI()` | none | `{success, questions[]}` | Trả câu hỏi khảo sát động cho form | `getSurveyQuestions` | Đọc `Master_Questions` |
| `getLeadersList()` | none | `{success, data[]}` | Danh sách leader (cột B của Master_Questions) | — | Đọc `Master_Questions` |
| `getEmployeeByGen(gen)` | `gen: string` | `{success, data:{gen,name,leaderName,row}}` | Lấy thông tin nhân viên để set leader mặc định | — | Đọc `Employee_Master` |
| `updateEmployeeLeader(gen, newLeaderName)` | 2 string | `{success, newLeaderName, newLeaderCode}` | Đổi leader của 1 nhân viên | — | Đọc + Ghi `Employee_Master` |
| `submitSurveyWithDate(genCode, leader, answers, userName, employeeType)` | gen, leader, `answers: object`, userName, employeeType | `{success, message}` | Ghi/update 1 dòng khảo sát | `initializeSurveyResultsSheet`, `getSurveyQuestions`, `prepareSurveyRowDataWithDate`, `findSurveyResultByGen`, `normalizeToDateOnly` | Đọc + Ghi `Survey_Results` (dùng LockService) |
| `getStatisticsForUI(surveyDate, shift)` | 2 string (optional) | `{success, statistics:{leaders[], questions[], totals{}, filterDate, filterShift}}` | Tính bảng thống kê Results tab | `parseDateString`, `getSheetData`, `loadMasterQuestionsData`, `getQuestion4GroupedOptions`, `buildSurveyLookupMap`, `hasSubmittedRequiredQuestions` | Đọc `Employee_Master`, `Survey_Results`, `Master_Questions` |
| `getEmployeesDetailedList(leader, employeeType, answerType, surveyDate, shift)` | 5 tham số | `{success, employees[], leader, employeeType, answerType, surveyDate, shift}` | Danh sách chi tiết cho modal (total/pending/onLeave/abnormal/q{n}_value) | `getSheetData`, `getSurveyQuestions`, `getQuestion4Mapping`, `buildSurveyLookupMap`, `hasSubmittedRequiredQuestions`, `checkMatchingAnswerDetailed` | Đọc `Employee_Master`, `Survey_Results`, `Master_Questions` |
| `processSupport(genCodes)` | `genCodes: array` | `{success, unprocessed[]}` | Đánh dấu nhân viên là "Support" (EMPLOYEE_TYPE='Y') | — | Đọc + Ghi (batch) `Employee_Master` |

> ⚠️ **[FACT — xác nhận qua Architecture Verification]** `processSupport()` có hành vi **"ghi đè toàn cột"**: với mỗi dòng trong `Employee_Master`, nếu GEN nằm trong danh sách truyền vào thì set `'Y'`, **ngược lại reset về `''`** — không phải "chỉ thêm mới". Hệ quả: nếu admin gọi hàm này với danh sách GEN **không đầy đủ** (thiếu một vài GEN đang là support), hàm sẽ **tự động xóa trạng thái support của những nhân viên không có trong danh sách**. Đây không phải lỗi kỹ thuật (code hoạt động đúng như logic được viết) mà là **rủi ro nghiệp vụ cần người vận hành admin hiểu rõ** trước khi dùng. Xem thêm Technical Debt #18.
| `processResign(genCodes)` | `genCodes: array` | `{success, unprocessed[]}` | Xóa dòng nhân viên nghỉ việc | — | Đọc + Xóa dòng `Employee_Master` |
| `processUpdate(employeeData)` | `employeeData: array` chuỗi `"gen_name_leaderCode"` | `{success, unprocessed[]}` | Update hoặc insert nhân viên | — | Đọc + Ghi `Employee_Master`, đọc `Master_Questions` |
| `getAdminReport(surveyDate)` | `surveyDate: string` | `{success, report:{pendingReport, attendanceReport}}` | Sinh báo cáo text (pending theo leader + tỉ lệ đi làm theo ca) | `parseDateString`, `getSheetData`, `getSurveyQuestions`, `getQuestion4Mapping`, `buildSurveyLookupMap`, `getLeadersList`, `hasSubmittedRequiredQuestions` | Đọc `Employee_Master`, `Survey_Results`, `Master_Questions` |
| `getSurveyHistoryByGen(genCode, dateFrom, dateTo)` | 3 string | `{success, data[], userName, genCode}` | Lịch sử khảo sát 1 GEN trong khoảng ngày (loại trừ "Đi làm bình thường") | `parseDateString`, `getSheetData`, `getQuestion4Mapping`, `normalizeToDateOnly` | Đọc `Survey_Results`, `Master_Questions` |
| `exportAllEmployeesCSV()` | none | `{success, csv, filename, count}` | Xuất CSV toàn bộ nhân viên đang làm | `getSheetData`, `csvEscape` | Đọc `Employee_Master` |
| `exportSurveyResultsCSV(dateFrom, dateTo)` | 2 string | `{success, csv, filename, count}` | Xuất CSV kết quả khảo sát theo khoảng ngày | `parseDateString`, `getSheetData`, `normalizeToDateOnly`, `csvEscape` | Đọc `Survey_Results` |

**[FACT]** 16 hàm trên là **toàn bộ bề mặt API** giữa frontend và backend — đây là "contract" quan trọng nhất cần bảo toàn.

---

### 2.4 Google Sheets / Data Processing (internal helper)

| Function | Input | Output | Chức năng | Sheets |
|---|---|---|---|---|
| `getSheetData(sheetName)` | string | `values[][]` | Đọc toàn bộ data range 1 sheet (1 lần `getValues()`) | Đọc (bất kỳ sheet nào) |
| `getSheetByName(sheetName)` | string | Sheet object | Lấy sheet, throw nếu không tồn tại | Không (chỉ metadata) |
| `getSurveyQuestions()` | none | `questions[]` | Đọc câu hỏi động từ cột G+ của `Master_Questions`, kèm placeholder cột F | Đọc `Master_Questions` |
| `loadMasterQuestionsData()` | none | `{questions, mapping, leaders}` | Gộp 3 lần đọc `Master_Questions` thành 1 lần (tối ưu) | Đọc `Master_Questions` |
| `buildSurveyLookupMap(surveyData, surveyDate)` | array + string | `Map{gen: row}` | Build Map tra cứu O(1) theo GEN, có filter theo ngày | Không (xử lý in-memory) |
| `getQuestion4Mapping()` | none | `{detail: {group, placeholder}}` | Mapping chi tiết Q4 → nhóm | Đọc `Master_Questions` |
| `getQuestion4GroupedOptions()` | none | `string[]` (hardcode 7 giá trị) | Danh sách nhóm cố định cho Q4 | Không |
| `findSurveyResultByGen(genCode, surveyData, surveyDate)` | 3 tham số | `row hoặc null` | Tìm tuyến tính 1 row theo GEN (dùng khi cần đơn lẻ) | Không (xử lý mảng đã đọc sẵn) |
| `initializeSurveyResultsSheet()` | none | `{success, headers}` | Tạo/mở rộng header `Survey_Results` dựa trên số câu hỏi động | Đọc + Ghi `Survey_Results` |
| `getPendingEmployeesByLeader(surveyDate)` | `surveyDate: string (optional)` | `{success, data:[{leaderName, pendingCount, totalNormalEmployees}], filterDate}` | Đếm nhân viên normal (employeeType !== 'Y') chưa hoàn thành khảo sát, theo từng leader | Đọc `Employee_Master`, `Survey_Results` |

> ⚠️ **[FACT — xác nhận qua Architecture Verification]** `getPendingEmployeesByLeader()` **không được gọi từ `index.html` (không có `google.script.run.getPendingEmployeesByLeader`) và cũng không được gọi bởi bất kỳ hàm nào khác trong `code.gs`** — kể cả `getAdminReport()` và `getStatisticsForUI()`, dù 2 hàm này có logic đếm "pending" tương tự nhưng được viết lại độc lập, không tái sử dụng hàm này. Đây là **orphan function hoàn toàn** — bị bỏ sót hoàn toàn khỏi bản Architecture Summary v1.0, đã được bổ sung ở v1.1. Xem thêm mục 10 (Technical Debt #13).

---

### 2.5 Authentication / Authorization

| Function | Chức năng |
|---|---|
| `checkGenCode(genCode)` | Validate GEN không rỗng, tồn tại, chưa nghỉ việc, gán role |
| `determineUserRole(userData)` | Map cột `ADMIN_ROLE` (chuẩn hóa uppercase) → `admin`/`leader`/`user` |
| `hashPassword(plain)` | SHA-256 hash qua `Utilities.computeDigest` |
| `verifyAdminPassword(genCode, passwordPlain)` | So khớp password — hỗ trợ cả plaintext cũ và hash 64-hex mới (backward compatible) |
| `migratePasswordsToHash()` | Chạy 1 lần (thủ công trong GAS Editor) để convert toàn bộ password cũ sang hash |

**[FACT]** Không có cơ chế session/token nào khác ngoài việc client tự lưu `user_data` vào `localStorage` và gửi lại `gen` trong mỗi lệnh gọi RPC tiếp theo. Server **không** verify lại quyền hạn ở mỗi lệnh gọi (xem mục 9 — Security).

**[FACT — bổ sung qua Architecture Verification]** `checkGenCode()` trả về **toàn bộ object `userData`**, bao gồm cả field thô `adminRole` (giá trị gốc chưa xử lý từ cột G sheet `Employee_Master`) **và** field đã tính toán `role` (`admin`/`leader`/`user`). Tức là bất kỳ client nào gọi `checkGenCode(gen)` hợp lệ đều nhận được cả 2 dạng thông tin phân quyền của nhân viên đó — mức độ lộ thông tin cao hơn một chút so với mô tả ngắn gọn ở bản v1.0 ("trả về `.role`").

---

### 2.6 Utility Functions

| Function | Chức năng |
|---|---|
| `cellToString(row, col)` | Đọc an toàn 1 cell, trả string đã trim, tránh lỗi null/undefined |
| `normalizeToDateOnly(value)` | Chuẩn hóa Date/string/number → Date chỉ có phần ngày; hỗ trợ dd/mm/yyyy, yyyy-mm-dd, ISO fallback |
| `parseDateString(dateStr)` | Wrapper gọi `normalizeToDateOnly` |
| `getAnswerColForQuestionIndex(qIndex)` / `getReasonColForQuestionIndex(qIndex)` | Tính vị trí cột động cho câu trả lời/lý do |
| `hasSubmittedRequiredQuestions(surveyResult)` | Kiểm tra đã có ngày khảo sát hợp lệ (điều kiện coi là "đã nộp") |
| `checkMatchingAnswerDetailed(surveyResult, answerType, questions, question4Mapping)` | Xác định 1 kết quả khảo sát có khớp filter `answerType` không |
| `csvEscape(value)` | Escape CSV, ép kiểu text bằng `="..."` để giữ số 0 đầu (ví dụ GEN `01234567`) |

---

### 2.7 Trigger-related Functions

| Function | Loại | Chức năng |
|---|---|---|
| `setupMonthlyCleanupTrigger()` | Thiết lập trigger (chạy thủ công 1 lần) | Xóa trigger cũ trùng tên rồi tạo trigger time-based mới: chạy `deleteOldSurveyRecords` vào 02h00 ngày 10 hàng tháng |
| `deleteOldSurveyRecords()` | Trigger handler | Xóa các bản ghi `Survey_Results` cũ hơn ~2 tháng (dựa trên cutoff = cuối tháng hiện tại - 2), giữ nguyên format |
| `syncToFileB_daily()` | **[UNKNOWN]** không rõ cách được trigger | Đồng bộ `Employee_Master` (spreadsheet nguồn theo ID cứng) sang spreadsheet đích khác (`DS Leader`, `UserAuth`), dùng MD5 hash lưu trong PropertiesService để bỏ qua nếu không đổi |
| `onOpen()` | Simple trigger | Tạo menu Spreadsheet UI |

**[UNKNOWN]** Không có bằng chứng trong code về cách `syncToFileB_daily` được kích hoạt định kỳ — không có hàm `setupXXXTrigger` tương ứng trong phạm vi file được cung cấp.

---

### 2.8 Diagnostics / Admin-only Manual Functions (không gọi từ frontend)

| Function | Chức năng |
|---|---|
| `initializeApplication()` | Kiểm tra tồn tại các sheet cấu hình + gọi `initializeSurveyResultsSheet` |
| `checkSheetsStatus()` | Alert hiển thị trạng thái các sheet (dùng trong Spreadsheet UI qua menu) |

---

## 3. FRONTEND — `index.html`

### 3.1 Khu vực UI chính
**[FACT]**
1. **Login section** (`#login-section`) — form nhập GEN, hiển thị tên tự động.
2. **App section** (`#app-section`) — hiển thị sau khi login, gồm:
   - Header + User info bar (`.user-info`, nút Đăng Xuất)
   - Tabs navigation (`.tabs`)
   - Tab **Khảo Sát** (`#survey`)
   - Tab **Kết Quả** (`#results`) — chỉ leader/admin
   - Tab **Tra Cứu** (`#history`) — mọi user
   - Tab **Quản Trị** (`#admin`) — chỉ admin
3. **Modals**: xác nhận đổi leader (`#confirm-leader-modal`), chi tiết nhân viên (`#employee-detailed-modal`), password admin (tạo động bằng JS).

### 3.2 Element / ID / Class quan trọng

| Khu vực | ID/Class chính |
|---|---|
| Login | `#login-form`, `#gen-code`, `#user-name-display`, `#login-btn`, `#login-text`, `#login-loading`, `#login-message` |
| Tabs | `.tab-btn[data-tab="survey/results/history/admin"]`, `.tab-content` |
| Survey | `#questions-container`, `#survey-form`, `#submit-survey-btn`, `#survey-message`, `#leader-dropdown`, radio `name="q2"` (`survey-date-today/tomorrow/custom`, `#custom-date-input`), `name="q3"`, `#question-4-container`, radio động `q{n}_opt{i}`, `reason-input-q{n}-{i}` |
| Results | `#stats-date`, `#stats-shift`, `#stats-loading`, `#stats-table-container` (`.stats-table`, `.sticky-leader`, `.clickable-number`, `.total-row`) |
| Modal chi tiết | `#employee-detailed-modal`, `#employee-detailed-headers`, `#employee-detailed-content`, `#employee-detailed-table-container`, `#modal-copy-btn`, `#modal-close-btn` |
| Modal đổi leader | `#confirm-leader-modal`, `#confirm-leader-message`, `#confirm-leader-yes`, `#confirm-leader-no` |
| Modal password admin | `#admin-password-modal`, `#admin-password-input`, `#admin-password-confirm`, `#admin-password-skip`, `#admin-password-error` — **tạo động bằng `innerHTML`, không có sẵn trong HTML tĩnh** |
| History | `#history-date-from`, `#history-date-to`, `#history-gen-wrapper`, `#history-gen-input`, `#history-loading`, `#history-results`, `#export-survey-btn`, `#history-export-message` |
| Admin | `#admin-gen-input`, `.admin-actions` (nút Support/Resign/Update/Clear), `#admin-action-message`, `#export-employees-btn`, `#export-employees-message`, `#admin-date`, `#admin-report-container` (`#pending-report-content`, `#attendance-report-content`) |

**[FACT]** Tồn tại reference tới các ID **không có trong HTML**: `#edit-modal`, `#password-modal`, `#employee-modal` (dùng bởi `closeEditModal()`, `closePasswordModal()`, `closeEmployeeModal()`) — orphan/dead code, xem mục 10.

### 3.3 Event Handlers chính

| Sự kiện | Handler |
|---|---|
| Submit `#login-form` | `handleLogin(event)` |
| Input `#gen-code` | `restrictToNumbers(this); checkGenCodeRealTime()` |
| Submit `#survey-form` | `handleSurveySubmit(event)` |
| Click tab button | `showTab(tabName)` |
| Change radio Q2/Q3/Q4+ | `validateCustomDate()`, `handleWorkShiftChange()`, `toggleReasonInput()` |
| Click nút thống kê ô số | `showEmployeeDetailedModal(...)` (inline onclick sinh động trong HTML) |
| Click "Đăng Xuất" | `logout()` |
| Click các nút Admin | `handleSupport/handleResign/handleUpdate(button)` |
| Click Export | `exportAllEmployees(button)`, `exportSurveyResults(button)` |

### 3.4 Các nhóm function JavaScript chính (client-side)

- **Init/Auth**: `checkAutoLogin`, `handleLogin`, `loginSuccess`, `showAdminPasswordModal`, `closeAdminPasswordModal`, `demoteToLeaderAndProceed`, `finalizeLoginAndEnterApp`, `loginFailed`, `logout`, `resetUIAfterLogout`, `clearFormsAndMessages`
- **UI/Tab management**: `showLoginForm`, `showMainApp`, `showTab`, `hasPermissionForTab`, `getRoleDisplayName`
- **Survey rendering & logic**: `loadSurveyData`, `displaySurveyQuestions`, `loadLeadersAndDefault`, `buildLeaderDropdown`, `showConfirmLeaderModal`, `clearQuestion4Data`, `toggleReasonInput`, `toggleCustomDateInput`, `initializeCustomDateInput`, `validateCustomDate`, `handleWorkShiftChange`, `setQuestion4Required`, `removeQuestion4Required`, `validateQuestion4Visibility`, `validateSurveyForm`, `clearDependentQuestions`, `collectSurveyAnswers`, `handleSurveySubmit`, `resetSurveyForm`, `resetSubmitButton`, `shouldCollectAnswer`
- **Statistics (Results tab)**: `loadStatistics`, `displayStatisticsTable`, `groupLeadersByLeaderName`
- **Modal chi tiết**: `showEmployeeDetailedModal`, `displayEmployeeDetailedList`, `copyModalContent`, `fallbackCopyTextToClipboardModal`, `showModalCopyNote`, `closeEmployeeDetailedModal`, `formatTimestampModal`
- **History tab**: `initializeHistoryDatePickers`, `searchHistory`, `displayHistoryResults`, `copyHistoryResults`, `fallbackCopyHistory`, `showHistoryCopyMessage`
- **Admin tab**: `setButtonLoading`, `handleSupport`, `handleResign`, `handleUpdate`, `clearAdminInput`, `showAdminMessage`, `loadAdminReport`, `displayAdminReport`, `copyPendingReport`, `copyAttendanceReport`, `copyTextWithNotification`, `fallbackCopyTextWithNotification`, `showInlineCopyMessage`, `copyToClipboard`, `initializeAdminDatePicker`
- **Export**: `downloadCSV`, `exportAllEmployees`, `exportSurveyResults`
- **Date utilities (client)**: `formatDate`, `getWeekdayName`, `formatDateForPicker`, `formatDateFromPicker`, `initializeDatePickers`, `initializeDatePicker`, `getDefaultShift`, `initializeShiftSelector`, `getShiftDisplayName`
- **Message/copy helpers**: `showMessage`, `createDetailedSuccessMessage`, `escapeHtml`, `copySurveyMessage`, `fallbackCopyTextToClipboard`
- **Orphan/chưa triển khai đầy đủ**: `handlePasswordSubmit`, `handleEditSubmit`, `closeEditModal`, `closePasswordModal`, `closeEmployeeModal`, `showEmployeeList`, `shouldCollectAnswer` *(không thấy nơi gọi)*

### 3.5 Toàn bộ lời gọi `google.script.run`
Xem bảng đầy đủ tại mục 2.3 (16 hàm). Danh sách vị trí gọi trong `index.html`:
1. `checkGenCode` — `handleLogin`, `checkGenCodeRealTime`
2. `verifyAdminPassword` — `showAdminPasswordModal`
3. `getSurveyDataForUI` — `loadSurveyData`
4. `getLeadersList` — `loadLeadersAndDefault`
5. `getEmployeeByGen` — `loadLeadersAndDefault`
6. `updateEmployeeLeader` — `showConfirmLeaderModal`
7. `submitSurveyWithDate` — `handleSurveySubmit`
8. `getStatisticsForUI` — `loadStatistics`
9. `getEmployeesDetailedList` — `showEmployeeDetailedModal`
10. `processSupport` — `handleSupport`
11. `processResign` — `handleResign`
12. `processUpdate` — `handleUpdate`
13. `getAdminReport` — `loadAdminReport`
14. `getSurveyHistoryByGen` — `searchHistory`
15. `exportAllEmployeesCSV` — `exportAllEmployees`
16. `exportSurveyResultsCSV` — `exportSurveyResults`

### 3.6 Dữ liệu Frontend nhận từ Backend (cấu trúc field quan trọng)

| Backend function | Field frontend đang dùng |
|---|---|
| `checkGenCode` | `.gen, .name, .leaderName, .employeeType, .role` |
| `getSurveyDataForUI` | `.success, .questions[].id/.text/.options[].value/.placeholder` |
| `getStatisticsForUI` | `.statistics.leaders[]` (`.leaderName, .employeeType, .total, .pending, .onLeaveCount, .abnormalCount, .q{n}_{value}`), `.statistics.questions[].text/.options[]`, `.statistics.totals.normal/.special` |
| `getEmployeesDetailedList` | `.employees[].gen/.name/.leaveType/.reason/.timestamp` |
| `getAdminReport` | `.report.pendingReport, .report.attendanceReport` (plain text) |
| `getSurveyHistoryByGen` | `.data[].surveyDate/.shift/.leaveType/.reason/.timestamp`, `.userName, .genCode` |
| Export functions | `.csv, .filename, .count` |

**[FACT]** Đây là các "hợp đồng dữ liệu" (data contracts) mà nếu backend đổi tên field hoặc cấu trúc, frontend sẽ vỡ **âm thầm** (GAS không có type-checking).

### 3.7 Cách Frontend cập nhật UI
**[FACT]** Chủ yếu bằng cách build chuỗi HTML string rồi gán `element.innerHTML = html` (ví dụ `displaySurveyQuestions`, `displayStatisticsTable`, `displayEmployeeDetailedList`, `displayHistoryResults`, `displayAdminReport`). Không dùng framework, không dùng virtual DOM.

---

## 4. CSS — `css.html`

### 4.1 Nhóm style chính
- CSS variables (`:root`) — màu sắc, spacing, border-radius, shadow.
- Layout: `.container`, `.header`, `.user-info`.
- Login: `.login-container`, `.login-header`, `.login-form`, `.form-group`, `.form-control`.
- Buttons: `.btn`, `.btn-primary`, `.btn-success`, `.btn-danger`.
- Tabs: `.tabs`, `.tab-btn`, `.tab-content`.
- Survey form: `.survey-container`, `.question-card`, `.options-container`, `.option-row`, `.reason-input`.
- Statistics table: `.stats-container`, `.stats-table` + rule sticky riêng cho `#stats-table-container`, `#employee-detailed-table-container`, `#history-results`.
- Admin panel: `.admin-panel`, `.admin-textarea`, `.admin-actions`, `.report-section`.
- Modal: `.modal-overlay`, `.modal`, `body.modal-open`.
- Loading/status: `.loading`, `.btn-loading`, `.status-message` (+ `-success/-error/-info`).
- Date/select filter: `.stats-filter-control`, `input[type="date"]`.
- Responsive: `@media (max-width:790px)`, `@media (max-width:400px)`.

### 4.2 Component UI đang được style và dependency với `index.html`

| CSS selector | HTML element phụ thuộc |
|---|---|
| `.sticky-leader` | `<th>`/`<td>` cột Leader trong `.stats-table` (sinh động trong `displayStatisticsTable`) |
| `.clickable-number` | `<span>` số liệu trong bảng thống kê (onclick mở modal) |
| `.total-row` | `<tr>` dòng TOTAL |
| `#stats-table-container`, `#employee-detailed-table-container`, `#history-results` | Container riêng cho từng bảng — mỗi container có sticky-header riêng, **không dùng chung 1 class** → nếu đổi ID này phải sửa CSS tương ứng |
| `.question-card.hidden` | Class động thêm/xóa khi ẩn câu hỏi — **[FACT — xác nhận qua Architecture Verification]** rule này được khai báo **trùng lặp 2 lần** trong `css.html` (1 lần ở khu vực "SURVEY FORM" gần đầu file, 1 lần ở khu vực cuối file gần comment "Add to CSS file"), nội dung giống hệt nhau. Đã xác nhận bằng `grep` toàn bộ `index.html`: **không có bất kỳ đoạn JS nào toggle class `hidden` này** (`classList.add('hidden')`/`.remove('hidden')` liên quan `question-card` không tồn tại) → đây là **dead CSS đã xác nhận**, không chỉ là "có thể chưa dùng" như nhận định ban đầu. |
| `.msg-line`, `.msg-line-{idx}` | Sinh động trong `createDetailedSuccessMessage()` — **không có rule CSS riêng cho các class này** (chỉ style inline) |

**[FACT]** `@keyframes spin` khai báo **2 lần** trong file (dư thừa, không gây lỗi).

**[FACT]** Nhiều đoạn có comment tiếng Việt giải thích mục đích style (ví dụ `.input-group label` — icon + text).

---

## 5. DATA FLOW

### Luồng 1 — Login
```
User nhập GEN (index.html: #gen-code)
→ handleLogin() → google.script.run.checkGenCode(genCode)
→ code.gs: checkGenCode() → findUserInMaster() → getUserData() [đọc Employee_Master]
→ determineUserRole()
→ trả về {gen, name, leaderName, employeeType, role, ...}
→ index.html: loginSuccess(userData)
    → nếu role==='admin': showAdminPasswordModal() → verifyAdminPassword() (RPC riêng, đọc Employee_Master)
    → lưu localStorage.user_data
    → finalizeLoginAndEnterApp() → showMainApp() + loadSurveyData()
```

### Luồng 2 — Nộp khảo sát
```
collectSurveyAnswers() [đọc DOM: leader, q2, q3, q4+ answers]
→ handleSurveySubmit() → google.script.run.submitSurveyWithDate(gen, leader, answers, userName, employeeType)
→ code.gs: LockService.getScriptLock()
    → getSurveyQuestions() [đọc Master_Questions]
    → prepareSurveyRowDataWithDate() [build row array]
    → sheet.getDataRange().getValues() [đọc Survey_Results] → findSurveyResultByGen()
    → update (setValues) hoặc append (appendRow) [ghi Survey_Results]
    → release lock
→ trả {success, message}
→ index.html: createDetailedSuccessMessage() → showMessage('survey-message', ...)
```

### Luồng 3 — Xem thống kê (Results tab)
```
loadStatistics() [đọc #stats-date, #stats-shift]
→ google.script.run.getStatisticsForUI(date, shift)
→ code.gs: getSheetData(Employee_Master) + getSheetData(Survey_Results) + loadMasterQuestionsData(Master_Questions)
    → buildSurveyLookupMap() x2 (theo ngày, theo ngày+ca)
    → group nhân viên theo leader+type
    → tính pending/onLeave/abnormal + đếm option từng câu hỏi
→ trả object statistics{leaders[], questions[], totals{}}
→ index.html: displayStatisticsTable() → render bảng + gắn onclick showEmployeeDetailedModal(...)
```

### Luồng 4 — Modal chi tiết nhân viên
```
Click ô số (.clickable-number) → showEmployeeDetailedModal(leader, employeeType, answerType, surveyDate, titleText)
→ google.script.run.getEmployeesDetailedList(...)
→ code.gs: lọc Employee_Master + Survey_Results theo answerType (total/pending/onLeave/abnormal/q{n}_value)
→ trả {employees:[{gen,name,leaveType,reason,timestamp}]}
→ index.html: displayEmployeeDetailedList() → render bảng trong modal + enable copy button
```

### Luồng 5 — Tra cứu lịch sử
```
searchHistory() [đọc #history-date-from/to, #history-gen-input]
→ google.script.run.getSurveyHistoryByGen(gen, dateFrom, dateTo)
→ code.gs: đọc Survey_Results + Master_Questions (Q4 mapping), lọc theo GEN + khoảng ngày + loại trừ "Đi làm bình thường"
→ trả {data[], userName, genCode}
→ index.html: displayHistoryResults() → render bảng + text để copy
```

### Luồng 6 — Admin: xử lý GEN (Support/Resign/Update)
```
Nhập textarea #admin-gen-input → handleSupport/handleResign/handleUpdate(button)
→ google.script.run.processSupport/processResign/processUpdate(genCodes/employeeData)
→ code.gs: đọc Employee_Master, batch write (Support) / deleteRow theo vòng lặp (Resign) / update-or-insert (Update)
→ trả {success, unprocessed[]}
→ index.html: showAdminMessage(...)
```

### Luồng 7 — Export CSV
```
Click nút Export → exportAllEmployees()/exportSurveyResults()
→ google.script.run.exportAllEmployeesCSV()/exportSurveyResultsCSV(dateFrom, dateTo)
→ code.gs: đọc sheet tương ứng, build chuỗi CSV (với BOM UTF-8)
→ trả {csv, filename, count}
→ index.html: downloadCSV() → tạo Blob + trigger download trên trình duyệt
```

---

## 6. DEPENDENCY MAP

| Component | Depends on | Used by | Risk if changed |
|---|---|---|---|
| `checkGenCode()` | `Employee_Master` structure, `findUserInMaster`, `getUserData`, `determineUserRole` | `handleLogin`, `checkGenCodeRealTime` (index.html) | **CAO** — đổi field trả về sẽ vỡ toàn bộ luồng login + phân quyền tab |
| `getSurveyDataForUI()` | `getSurveyQuestions`, `Master_Questions` cột G+ | `loadSurveyData` | **CAO** — đổi cấu trúc `questions[]` vỡ toàn bộ form khảo sát động |
| `submitSurveyWithDate()` | `SR_COL`, `getSurveyQuestions`, LockService | `handleSurveySubmit` | **CAO** — sai lệch cột ghi sẽ làm hỏng dữ liệu `Survey_Results` |
| `getStatisticsForUI()` | `Employee_Master`, `Survey_Results`, `Master_Questions`, `buildSurveyLookupMap` | `loadStatistics`, `displayStatisticsTable` | **CAO** — field `statistics.leaders[]`/`questions[]` dùng trực tiếp để build key `q{n}_{value}` trong HTML onclick |
| `getEmployeesDetailedList()` | `answerType` string format (`total`/`pending`/`onLeave`/`abnormal`/`q{n}_{value}`) | `showEmployeeDetailedModal` | **CAO** — format `answerType` được tạo động trong `displayStatisticsTable` phải khớp tuyệt đối với logic parse trong `getEmployeesDetailedList`/`checkMatchingAnswerDetailed` |
| `#gen-code`, `#login-btn` | `checkGenCodeRealTime`, `handleLogin` | login flow | **CAO** |
| `#question-4-container`, `q4_opt{i}` | `handleWorkShiftChange`, `toggleReasonInput`, `displaySurveyQuestions` | Survey submission | **CAO** — logic ẩn/hiện Câu 4 gắn chặt với ID này |
| `.sticky-leader`, `.clickable-number` | CSS rule riêng cho `#stats-table-container` | `displayStatisticsTable` | **TRUNG BÌNH** — đổi class sẽ mất hiệu ứng sticky/hover nhưng không vỡ chức năng |
| `SR_COL` (code.gs) | Cấu trúc thực tế sheet `Survey_Results` | `submitSurveyWithDate`, `getStatisticsForUI`, `getEmployeesDetailedList`, `getAdminReport`, `getSurveyHistoryByGen`, `exportSurveyResultsCSV` | **RẤT CAO** — đây là hằng số trung tâm, sai 1 index sẽ ảnh hưởng dây chuyền tới ~6 hàm |
| `CONFIG.COLUMNS.EMPLOYEE_MASTER` | Cấu trúc thực tế sheet `Employee_Master` | Gần như mọi hàm liên quan nhân viên | **RẤT CAO** |
| `localStorage.user_data` key | `checkAutoLogin`, `loginSuccess`, `logout` | Toàn bộ session client | **TRUNG BÌNH** — đổi key sẽ làm mất session cũ của user đang dùng |
| `escapeHtml()` | Dùng trong hầu hết hàm render | Mọi nơi hiển thị dữ liệu động | **CAO** — nếu xóa/đổi có thể mở lỗ hổng XSS |
| `resetUIAfterLogout()` selector `.tab-btn[onclick="showTab('survey')"]` | Nội dung chuỗi `onclick` chính xác trên nút tab Khảo Sát trong HTML | Chạy khi `logout()` | **TRUNG BÌNH** — **[FACT — xác nhận qua Architecture Verification]** hàm này dùng attribute selector dựa trên chuỗi `onclick` literal, trong khi `showTab()` (nơi khác trong cùng file) dùng `data-tab` attribute selector cho cùng mục đích tìm nút tab. Đây là 2 chiến lược selector không nhất quán trong cùng file — nếu sau này đổi cú pháp `onclick` của nút Khảo Sát (kể cả chỉ đổi dấu nháy/khoảng trắng), selector trong `resetUIAfterLogout()` sẽ không tìm thấy phần tử, trong khi `showTab()` không bị ảnh hưởng. |

---

## 7. GOOGLE SHEETS / DATA STRUCTURE

### 7.1 Spreadsheet
**[FACT]** Spreadsheet chính: truy cập qua `SpreadsheetApp.getActiveSpreadsheet()` — **[UNKNOWN]** ID cụ thể không được nêu trong code (vì dùng active spreadsheet, tức Apps Script được bind trực tiếp vào 1 file Sheets cụ thể).

**[FACT]** 2 spreadsheet khác được truy cập bằng ID cứng (chỉ trong `syncToFileB_daily`):
- Nguồn: `1NyopLvnwHltt_HUwFxXmBu2M1WOLZQe_IU_tM6CtlI8` (sheet `Employee_Master`)
- Đích: `1vEBgTjHTSc6F5Th-6b3pMvMCChSTp8hiSkVKL0e6Khw` (sheet `DS Leader`, `UserAuth`)

**[UNKNOWN]** Không rõ "spreadsheet nguồn" trong `syncToFileB_daily` có phải cùng 1 file với active spreadsheet hay không (dùng `openById` thay vì `getActiveSpreadsheet`).

### 7.2 Sheets & cấu trúc cột (theo code tham chiếu)

**`Employee_Master`** — Column mapping (0-based, theo `CONFIG.COLUMNS.EMPLOYEE_MASTER`):
| Index | Tên field code | Diễn giải |
|---|---|---|
| 0 | NAME | Tên nhân viên |
| 1 | GEN | Mã GEN (8 số) |
| 2 | LEADER_ID | Mã Leader |
| 3 | LEADER_NAME | Tên Leader |
| 4 | EMPLOYEE_TYPE | Loại NV ('' = normal, 'Y' = support) |
| 5 | RESIGNED | Đã nghỉ việc ('Y' hoặc rỗng) |
| 6 | ADMIN_ROLE | Cột G — role (ADMIN/LEADER/rỗng) |
| 7 *(không có tên trong CONFIG)* | Password (đọc trực tiếp `row[7]` trong `verifyAdminPassword`/`migratePasswordsToHash`) — **[INFERENCE]** cột H |

**`Master_Questions`** — Column mapping (theo code, **[INFERENCE]** vì không có tên cột chính thức):
| Index | Diễn giải |
|---|---|
| 0 | Mã Leader (dùng trong `processUpdate` leaderMap) |
| 1 | Tên Leader (cột B — dùng bởi `getLeadersList`, `loadMasterQuestionsData`) |
| 3 | Chi tiết Q4 (detail) |
| 4 | Nhóm Q4 (group) |
| 5 | Placeholder (lý do) |
| 6+ | Header = text câu hỏi động; các dòng bên dưới = option |

**`Survey_Results`** — Column mapping (theo `SR_COL`):
| Index | Tên | Header (từ `initializeSurveyResultsSheet`) |
|---|---|---|
| 0 | TIMESTAMP | 'Timestamp' |
| 1 | LAST_EDITED | 'Last_Edited_By' *(khai báo header nhưng **[FACT]** không thấy code nào gán giá trị khi ghi row — luôn để trống)* |
| 2 | GEN | 'GEN' |
| 3 | NAME | 'Name' |
| 4 | LEADER | 'Leader' |
| 5 | EMPLOYEE_TYPE | 'Loại NV' |
| 6 | SURVEY_DATE | 'Ngày khảo sát' |
| 7 | Q3 | 'Bạn làm việc:' |
| 8+ | Q4_ANSWER, Q4_REASON, Q5_ANSWER, Q5_REASON... | Tên câu hỏi + `'Q{n}_Phụ'` |

**`Di_support`** — **[FACT]** khai báo trong `CONFIG.SHEETS` nhưng **không được sử dụng** ở bất kỳ hàm nào khác trong `code.gs`.

### 7.3 Data relationship
**[INFERENCE]**
- `Employee_Master.GEN` ↔ `Survey_Results.GEN` (quan hệ 1-nhiều: 1 nhân viên có nhiều bản ghi khảo sát theo ngày)
- `Employee_Master.LEADER_NAME` ↔ `Master_Questions` cột B (danh sách leader hợp lệ)
- `Survey_Results.Q4_ANSWER` ↔ `Master_Questions` cột D (detail) → cột E (group) — dùng để nhóm hiển thị thống kê

**[UNKNOWN]** Không rõ ràng buộc unique/index nào được áp dụng ở tầng Sheet (Google Sheets không có constraint DB thực sự) — mọi validation chỉ nằm trong code.

---

## 8. PERFORMANCE

### HIGH — Rủi ro chậm/timeout rõ rệt khi dữ liệu lớn
1. **`getStatisticsForUI()`** — đọc toàn bộ `Employee_Master` + `Survey_Results` + `Master_Questions` (3 lần `getSheetData`/`loadMasterQuestionsData`), sau đó filter mảng 2-3 lần, build Map 2 lần, và lặp lồng `leaders × employeeType × employees × questions × options` để đếm — với sheet lớn (hàng nghìn dòng), đây là hàm nặng nhất trong hệ thống, chạy mỗi lần bấm "Tìm kiếm" ở tab Kết Quả.
2. **`processUpdate()` / `updateEmployeeLeader()`** — dùng **linear search** (vòng lặp `for` quét toàn bộ `Employee_Master`) để tìm dòng theo GEN/leaderName; với `processUpdate` còn lồng thêm vòng lặp theo từng phần tử `employeeData` nhập vào → độ phức tạp O(N×M). Admin nhập nhiều GEN cùng lúc + sheet lớn sẽ chậm.
3. **`submitSurveyWithDate()`** — mỗi lần submit đọc lại toàn bộ `Survey_Results` (`sheet.getDataRange().getValues()`) để tìm existing row bằng `findSurveyResultByGen` (linear search) — với sheet có nhiều bản ghi lịch sử, đây là điểm nghẽn tiềm ẩn, đặc biệt khi nhiều user submit đồng thời (đã có LockService nhưng lock chỉ đảm bảo an toàn ghi, không giảm thời gian đọc).

### MEDIUM
4. **`getEmployeesDetailedList()`** — build `surveyMap` và `pendingMap` riêng biệt (có thể trùng lặp công việc khi `shift==='all'`), cộng với vòng lặp qua toàn bộ `Employee_Master` mỗi lần mở modal.
5. **`getAdminReport()`** — đọc `Employee_Master` + `Survey_Results` + `Master_Questions`, có vòng lặp lồng `leaders × normalEmployees` bằng `.filter()` (chạy filter cho mỗi leader thay vì group 1 lần) — chưa dùng kỹ thuật group giống `getStatisticsForUI`.
6. **`processResign()`** — `sheet.deleteRow()` gọi trong vòng lặp (N lần gọi API riêng cho N GEN) — không thể batch dễ dàng trong GAS API, nhưng vẫn là N lệnh gọi Spreadsheet Service.
7. **`deleteOldSurveyRecords()`** — đọc + ghi lại toàn bộ sheet (`clearContent` + `setValues`) — chấp nhận được vì chạy 1 lần/tháng qua trigger, nhưng nếu sheet rất lớn (>10.000 dòng) có thể chạm giới hạn thời gian thực thi 6 phút của GAS.

### LOW
8. Các hàm export CSV (`exportAllEmployeesCSV`, `exportSurveyResultsCSV`) — đọc 1 lần, xử lý in-memory, chấp nhận được.
9. `getSurveyQuestions()`, `loadMasterQuestionsData()` — đã tối ưu đọc 1 lần bằng `getValues()`.
10. `buildSurveyLookupMap()` — O(n) 1 lần, hiệu quả.

### Frontend/Backend calls
**[FACT]** Mỗi thao tác chính (login, load survey, load stats, mở modal, submit, export) đều là 1 lệnh `google.script.run` riêng biệt — không có gộp request. Với tab Results, `loadLeadersAndDefault()` khi load survey gọi **2 RPC nối tiếp** (`getLeadersList` → `getEmployeeByGen`), tăng độ trễ cảm nhận khi mở tab Khảo Sát.

### Khả năng timeout
**[INFERENCE]** GAS có giới hạn 6 phút/execution (script chạy dưới quyền user) hoặc 30 phút (trigger dưới quyền owner). Các hàm rủi ro timeout nhất nếu sheet phát triển lớn: `getStatisticsForUI`, `processUpdate` (nhiều GEN), `deleteOldSurveyRecords`, `syncToFileB_daily`.

---

## 9. SECURITY

| Hạng mục | Đánh giá |
|---|---|
| **Authorization** | **[FACT]** Không có kiểm tra quyền ở tầng server cho hầu hết hàm — ví dụ `getStatisticsForUI`, `getEmployeesDetailedList`, `processSupport/Resign/Update`, `getAdminReport`, export CSV **đều có thể gọi trực tiếp từ console trình duyệt** mà không cần đúng role, vì phân quyền hiện tại chỉ nằm ở **frontend** (`hasPermissionForTab`, ẩn/hiện tab). Đây là rủi ro bảo mật cấu trúc (không phải bug mới phát sinh, mà là đặc điểm kiến trúc hiện tại). |
| **User validation** | `checkGenCode` có validate GEN tồn tại + chưa nghỉ việc, nhưng không có giới hạn rate-limit hay chống brute-force GEN 8 số. |
| **Data exposure** | `checkGenCode` trả về toàn bộ thông tin nhân viên (kể cả `adminRole`) cho **bất kỳ ai** biết GEN hợp lệ, kể cả khi gọi hàm này với GEN của người khác. |
| **Client-side trust** | Frontend tự quyết định hiển thị tab admin dựa trên `currentUser.role` lưu trong `localStorage` — có thể bị sửa thủ công trong DevTools để hiện tab admin, nhưng các hàm admin thực sự (`processSupport` v.v.) vẫn gọi được server bất kể — do đó **rủi ro nằm ở server, không phải ở việc ẩn/hiện tab**. |
| **XSS** | `escapeHtml()` được áp dụng khá nhất quán khi render dữ liệu động (tên nhân viên, câu hỏi, lý do...) — điểm tốt. Cần rà soát thêm các chỗ nối chuỗi HTML trực tiếp không qua `escapeHtml` nếu phát hiện sau này. |
| **Spreadsheet access** | Dùng `getActiveSpreadsheet()` — chạy dưới quyền deploy của Web App (`Execute as`), **[UNKNOWN]** cấu hình deploy thực tế (Me / User accessing) không nằm trong code. |
| **Sensitive information** | Password admin được hash SHA-256 (tốt), nhưng **không có salt** — với password ngắn/đơn giản, có thể bị tấn công rainbow-table nếu cột H trong `Employee_Master` bị lộ. 2 Spreadsheet ID hardcode trong `syncToFileB_daily` — không phải bí mật tuyệt đối nhưng vẫn là thông tin nhạy cảm nằm trong source code. |

---

## 10. TECHNICAL DEBT

**[FACT]** Liệt kê nguyên trạng — KHÔNG đề xuất sửa ở bước này:

1. Các hàm/modal orphan không có DOM tương ứng: `closeEditModal()`, `closePasswordModal()`, `closeEmployeeModal()`, `handlePasswordSubmit()`, `handleEditSubmit()`, `showEmployeeList()` (dùng `alert()` demo).
2. Biến global `currentQuestions` khai báo nhưng không thấy được gán/đọc ở đâu khác.
3. `CONFIG.SHEETS.DI_SUPPORT = 'Di_support'` khai báo nhưng không dùng trong bất kỳ hàm nào.
4. `@keyframes spin` khai báo trùng lặp 2 lần trong `css.html`.
5. Cột `LAST_EDITED` (SR_COL index 1) có header nhưng không thấy code nào gán giá trị khi ghi dữ liệu — luôn để trống.
6. `getEmployeeByGen(gen)` đọc `row[2]` gán vào biến `name` — cần đối chiếu lại với `CONFIG.COLUMNS.EMPLOYEE_MASTER` (NAME=0) để xác nhận đây có phải là điểm không nhất quán hay không (chưa đủ căn cứ để khẳng định là bug — xem mục 12, UNKNOWN #6).
7. Không có cơ chế kiểm tra quyền (authorization) ở tầng server cho các hàm nhạy cảm (đã nêu ở mục 9).
8. `syncToFileB_daily` không có bằng chứng về cơ chế trigger đi kèm trong code hiện tại.
9. Linear search còn tồn tại trong `processUpdate`, `updateEmployeeLeader`, `getEmployeeByGen`, `findSurveyResultByGen` — chưa được thay bằng Map như các hàm thống kê khác.
10. Không dùng `CacheService` ở bất kỳ đâu — mọi lần load dữ liệu tĩnh (câu hỏi, danh sách leader) đều đọc lại Sheet.
11. Password hash không có salt.
12. Comment tiêu đề file `code.gs` liệt kê "15 thay đổi tối ưu" nhưng một số vùng (mục 8, 9 ở trên) cho thấy tối ưu chưa đồng đều giữa các hàm.

---

## 11. CHANGE IMPACT MAP

### Nếu thay đổi 1 backend function
| Thay đổi | Ảnh hưởng |
|---|---|
| Đổi tên function trong nhóm mục 2.3 (16 hàm API) | **Vỡ ngay lập tức** lời gọi `google.script.run.<tên cũ>` tương ứng trong `index.html` — phải cập nhật đồng bộ cả 2 file |
| Đổi cấu trúc object trả về (thêm/xóa/đổi tên field) | Vỡ mọi chỗ frontend truy cập field đó — không có lỗi biên dịch, chỉ lỗi runtime `undefined` |
| Đổi `SR_COL` index | Ảnh hưởng dây chuyền: `submitSurveyWithDate`, `getStatisticsForUI`, `getEmployeesDetailedList`, `getAdminReport`, `getSurveyHistoryByGen`, `exportSurveyResultsCSV`, `initializeSurveyResultsSheet` — cần đối chiếu với header thực tế trên Sheet |
| Đổi `CONFIG.COLUMNS.EMPLOYEE_MASTER` index | Ảnh hưởng mọi hàm đọc/ghi `Employee_Master` (gần như toàn bộ hệ thống) |
| Đổi logic `answerType` string format (`q{n}_{value}`) trong `getEmployeesDetailedList`/`checkMatchingAnswerDetailed` | Phải đồng bộ với cách `displayStatisticsTable` (frontend) build chuỗi `key` khi gắn `onclick` |

### Nếu thay đổi 1 frontend function
| Thay đổi | Ảnh hưởng |
|---|---|
| Đổi tên hàm được gọi bằng `onclick="..."` sinh động trong HTML string (VD: `showEmployeeDetailedModal`) | Vỡ ngay vì chuỗi `onclick` được build tại runtime, không có type-check |
| Đổi signature (số tham số/thứ tự) của `showEmployeeDetailedModal`, `handleWorkShiftChange`, `toggleReasonInput` | Ảnh hưởng tới các đoạn HTML sinh động gọi hàm này với tham số theo thứ tự cũ |
| Đổi `collectSurveyAnswers()` cấu trúc `answers` object | Ảnh hưởng `submitSurveyWithDate` (backend) và `createDetailedSuccessMessage` |

### Nếu thay đổi 1 HTML ID
| Thay đổi | Ảnh hưởng |
|---|---|
| Đổi ID input/form trong Survey (`#leader-dropdown`, `q4_opt{i}`, `reason-input-q{n}-{i}`) | Vỡ `collectSurveyAnswers`, `toggleReasonInput`, `displaySurveyQuestions` |
| Đổi ID modal (`#employee-detailed-*`) | Vỡ toàn bộ luồng modal chi tiết nhân viên |
| Đổi ID `#stats-date`, `#stats-shift` | Vỡ `loadStatistics` |
| Đổi ID liên quan CSS sticky (`#stats-table-container`, `#employee-detailed-table-container`) | Mất hiệu ứng sticky header (không vỡ chức năng nhưng ảnh hưởng UX) |

### Nếu thay đổi 1 CSS class
| Thay đổi | Ảnh hưởng |
|---|---|
| Đổi `.clickable-number`, `.sticky-leader`, `.total-row` | Mất style nhưng không vỡ logic JS (các class này không được JS query bằng `querySelector`, chỉ dùng cho CSS) — **rủi ro thấp về chức năng, rủi ro UX trung bình** |
| Đổi `.tab-btn`, `.tab-content` | **CAO** — các class này được JS dùng để add/remove `active` (`classList.add/remove('active')`) — đổi tên sẽ vỡ logic chuyển tab |
| Đổi `.modal-overlay`, `.active` (modal) | **CAO** — dùng trực tiếp trong JS để mở/đóng modal |

### Nếu thay đổi 1 return value từ backend
Xem chi tiết bảng "Nếu thay đổi 1 backend function" ở trên — nguyên tắc chung: **mọi thay đổi cấu trúc return value đều cần rà soát toàn bộ nơi gọi tương ứng trong `index.html` trước khi triển khai**, vì không có cơ chế kiểm tra kiểu dữ liệu tự động giữa 2 lớp.

---

## 12. FACT / INFERENCE / UNKNOWN — Tổng hợp các điểm chưa chắc chắn quan trọng nhất

1. **[UNKNOWN]** Cấu trúc cột chính xác của `Master_Questions` ngoài các cột được code tham chiếu trực tiếp — chưa có dữ liệu mẫu để verify toàn bộ.
2. **[UNKNOWN]** Cột password thực tế trong `Employee_Master` (index 7, suy luận là cột H) — chưa xác nhận từ dữ liệu thật.
3. **[UNKNOWN]** Vai trò thực tế của sheet `Di_support` (khai báo nhưng không dùng) — có thể dùng ở phần code/Apps Script khác ngoài phạm vi được cung cấp.
4. **[UNKNOWN]** Cơ chế trigger cho `syncToFileB_daily()`.
5. **[UNKNOWN]** Có tồn tại file `.gs` khác ngoài `code.gs` trong project Apps Script hay không.
6. **[UNKNOWN]** `getEmployeeByGen()` đọc `row[2]` gán biến `name` — cần xác nhận đây là hành vi cố ý hay không nhất quán so với `CONFIG.COLUMNS.EMPLOYEE_MASTER.NAME=0`; hiện tại trường này dường như không được frontend sử dụng trực tiếp (`loadLeadersAndDefault` chỉ đọc `.leaderName`) nên chưa gây lỗi quan sát được.
7. **[UNKNOWN]** Cấu hình deploy Web App thực tế ("Execute as" / "Who has access").
8. **[UNKNOWN]** Spreadsheet ID của "spreadsheet chính" (dùng active spreadsheet, không hardcode).
9. **[FACT]** Mọi cấu trúc, tên hàm, tên field khác được liệt kê trong tài liệu này đều trích trực tiếp từ code — có thể coi là nguồn xác thực (source of truth) cho các thay đổi sau này.

---

## 13. BASELINE RULES — Rules for Future Changes

Các nguyên tắc bắt buộc tuân thủ khi có yêu cầu thay đổi project trong tương lai:

1. **Không phá vỡ API giữa frontend và backend** — 16 hàm liệt kê ở mục 2.3 là hợp đồng cố định; mọi thay đổi tên/tham số/cấu trúc trả về phải được đánh giá tác động 2 chiều trước khi thực hiện.
2. **Không đổi tên function đang được gọi** (cả server-side lẫn client-side) nếu không thực sự cần thiết — nếu bắt buộc phải đổi, cập nhật đồng bộ tất cả nơi gọi trong cùng 1 lần thay đổi.
3. **Không đổi HTML ID/class đang được JavaScript hoặc CSS sử dụng** nếu không cần thiết — đặc biệt các ID liên quan Survey form động, modal, và class dùng để toggle trạng thái (`active`, `hidden`).
4. **Không thay đổi cấu trúc dữ liệu Sheet** (`SR_COL`, `CONFIG.COLUMNS.EMPLOYEE_MASTER`, cột `Master_Questions`) nếu chưa xác nhận với dữ liệu thực tế trên Spreadsheet — vì các cấu trúc này chỉ được suy ra từ code, chưa verify bằng dữ liệu thật.
5. **Không refactor ngoài phạm vi yêu cầu** — kể cả khi phát hiện dead code hay pattern chưa tối ưu, chỉ sửa khi được yêu cầu rõ ràng.
6. **Không xóa code (kể cả nhìn có vẻ "chết")** nếu chưa xác định chắc chắn không có dependency ẩn — ví dụ các hàm modal orphan (`closeEditModal` v.v.) có thể được gọi từ nơi khác ngoài phạm vi 3 file này.
7. **Luôn đối chiếu 2 chiều** trước khi sửa: sửa `code.gs` → kiểm tra `index.html` có dùng return value không; sửa ID/class trong `index.html` → kiểm tra `css.html` và các hàm JS có tham chiếu không.
8. **Ưu tiên thay đổi tối thiểu (minimal diff)** — không viết lại toàn bộ function nếu chỉ cần sửa 1 dòng logic.
9. **Bất kỳ thay đổi nào liên quan tới bảo mật/phân quyền** (mục 9) cần được thảo luận rõ ràng về trade-off trước khi triển khai, vì đây là thay đổi kiến trúc, không phải bug fix đơn thuần.
10. **Trước khi sửa bất kỳ hàm nào trong danh sách "RẤT CAO" ở mục 6 (Dependency Map)**, bắt buộc phải nêu rõ root cause, phạm vi ảnh hưởng, và có xác nhận trước khi implement.

---

## CURRENT ARCHITECTURE HEALTH

| Tiêu chí | Điểm (0–10) | Lý do |
|---|---|---|
| **Architecture** | 6/10 | Kiến trúc 3 lớp rõ ràng, tách bạch backend/frontend hợp lý cho quy mô GAS Web App. Tuy nhiên thiếu tầng authorization ở server, phụ thuộc hoàn toàn vào "hợp đồng ngầm" giữa 2 lớp (không type-safe), và có 1 sheet (`Di_support`) + 1 hàm sync (`syncToFileB_daily`) nằm ngoài luồng chính không rõ vai trò. |
| **Code quality** | 6.5/10 | Code có comment giải thích tốt, đặt tên hàm rõ ràng, đã qua 1 vòng tối ưu có ghi chú đầy đủ (15 điểm cải tiến). Điểm trừ: còn dead code (modal orphan, biến không dùng), một số hàm (`processUpdate`, `updateEmployeeLeader`) chưa được tối ưu đồng đều so với phần còn lại, và có điểm cần xác minh (`getEmployeeByGen`). |
| **Performance** | 6/10 | Các hàm thống kê chính (`getStatisticsForUI`, `buildSurveyLookupMap`) đã dùng Map O(1) và batch read/write — tốt cho quy mô hiện tại. Tuy nhiên vẫn còn linear search trong `processUpdate`/`updateEmployeeLeader`/`findSurveyResultByGen`, và `getStatisticsForUI` xử lý khá nặng (nhiều lớp filter/loop lồng nhau) — sẽ giảm hiệu năng rõ rệt khi dữ liệu tăng lên hàng nghìn dòng. |
| **Maintainability** | 6/10 | Cấu trúc rõ ràng giúp dễ đọc, nhưng việc gắn `onclick="..."` với tham số string được build động trong HTML string (thay vì event delegation) làm tăng rủi ro lỗi khi refactor. Thiếu tài liệu chính thức về cấu trúc Sheet trước khi có tài liệu này. Không có unit test. |
| **Security** | 4/10 | Điểm yếu rõ rệt nhất: không có kiểm tra authorization ở server cho các hàm nhạy cảm (thống kê, xuất báo cáo, quản lý nhân viên) — chỉ dựa vào ẩn/hiện UI ở client. Password có hash nhưng không salt. Đây là điểm cần ưu tiên xem xét nếu ứng dụng mở rộng phạm vi sử dụng hoặc dữ liệu trở nên nhạy cảm hơn. |

---

*Tài liệu này là baseline tại thời điểm phân tích. Mọi thay đổi code trong tương lai nên được đối chiếu ngược lại với tài liệu này, và tài liệu nên được cập nhật song song khi kiến trúc thay đổi.*
