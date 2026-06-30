# BÀI 3: Thực hành Refactor Clean Code - Refinement Process

### 1. Phân tích các vi phạm Clean Code của mã nguồn ban đầu
Đoạn mã `LedgerBalanceCalculator` cũ vi phạm nghiêm trọng các tiêu chuẩn lập trình sạch (Clean Code), khiến việc bảo trì trở thành cơn ác mộng:

* **Arrow Anti-Pattern (Lồng nhau quá sâu):** Có tới 6 cấp độ `if` lồng nhau. Điều này khiến logic bị đẩy sâu vào bên phải màn hình, làm giảm khả năng đọc hiểu và cực kỳ dễ gây ra lỗi khi cần sửa đổi logic.
* **Quy tắc đặt tên tệ (Bad Naming Conventions):** Hàm tên là `calc` (tính gì?), biến là `list`, `a`, `t`. Người đọc mã nguồn phải tự đoán ngữ nghĩa thay vì mã tự giải thích (self-documenting).
* **Lặp code (Code Duplication):** Đoạn logic kiểm tra số dư dương và cộng dồn (`if (a.getBalance() > 0) { total += a.getBalance(); }`) bị viết lặp lại ở cả hai nhánh của lệnh `if (activeOnly)`.
* **Nguy cơ NullPointerException (NPE) tiềm ẩn:** Biểu thức `a.getBranch().equals(branch)` có thể văng lỗi NPE nếu bản thân `a.getBranch()` trả về `null`. Ngoài ra, hàm không kiểm tra tính hợp lệ của chuỗi `branch`.
* **Không có Logging (Zero Observability):** Hoàn toàn "mù" thông tin khi chạy trên Production. Nếu tính ra kết quả bất thường, developer không biết hệ thống đã duyệt qua bao nhiêu tài khoản.

---

### 2. Thiết kế chuỗi Prompt Cải tiến đầu ra (Iterative Prompting)

Dưới đây là chuỗi 3 vòng prompt được thiết kế theo đúng quy trình Refactoring từ cấu trúc đến tối ưu hiệu năng:

**Vòng 1 (Robustness & Clean Code - Cải trúc cấu trúc logic):**
> "Hãy đóng vai trò là một Senior Java Developer. Tôi cần refactor đoạn mã `LedgerBalanceCalculator` cũ. Hãy thực hiện 2 việc sau:
> 1. Đổi tên hàm và tên biến cho rõ nghĩa, tuân thủ Clean Code (ví dụ: `calculateTotalBalance`, `accounts`, `account`, `targetBranch`...).
> 2. Sử dụng kỹ thuật 'Guard Clauses' (Return Early) để loại bỏ hoàn toàn tình trạng `if-else` lồng nhau quá sâu (Arrow Anti-Pattern). Gộp logic kiểm tra số dư lớn hơn 0 để loại bỏ lặp code. Vui lòng xuất mã nguồn đã sửa."

**Vòng 2 (Maintainability & OOP - Tối ưu mã hóa khai báo):**
> "Rất tốt, cấu trúc đã phẳng hơn. Bây giờ, hãy nâng cấp đoạn code trên thành chuẩn Java 17 hiện đại:
> 1. Thay thế vòng lặp `for-each` truyền thống bằng **Java 17 Stream API** để bộ lọc dữ liệu trở nên gọn gàng và mang tính khai báo (Declarative) hơn.
> 2. Bổ sung annotation `@Transactional(readOnly = true)` của Spring Boot lên đầu phương thức để tối ưu hóa hiệu năng đọc dữ liệu từ Database."

**Vòng 3 (Logging & Context Tuning - Sẵn sàng cho Production):**
> "Hoàn hảo. Ở bước cuối cùng, hãy tích hợp cơ chế giám sát (Observability) để sẵn sàng đưa lên môi trường Production:
> 1. Tích hợp thư viện Lombok thông qua annotation `@Slf4j`.
> 2. Bổ sung ghi log ở cấp độ `INFO`. Hãy log ra thông điệp cho biết: Chi nhánh đang xử lý là gì, có bao nhiêu tài khoản hợp lệ đã được duyệt qua, và tổng số dư tính toán được là bao nhiêu.
     > Hãy xuất ra mã nguồn Java hoàn chỉnh cuối cùng."

---

### 3. Minh chứng chạy thực tế (Kết quả AI sinh ra ở vòng 3)

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Objects;

@Slf4j
public class LedgerBalanceCalculator {

    /**
     * Tính tổng số dư các tài khoản hợp lệ tại một chi nhánh cụ thể.
     * Sử dụng @Transactional(readOnly = true) để tối ưu hóa connection với Database trong Spring Boot.
     *
     * @param accounts     Danh sách tài khoản đầu vào
     * @param targetBranch Chi nhánh cần thống kê
     * @param activeOnly   Cờ kiểm tra chỉ lấy tài khoản đang hoạt động (ACTIVE)
     * @return Tổng số dư hợp lệ
     */
    @Transactional(readOnly = true)
    public double calculateTotalBalance(List<Account> accounts, String targetBranch, boolean activeOnly) {
        // Guard Clause: Trả về sớm nếu đầu vào không hợp lệ
        if (accounts == null || accounts.isEmpty()) {
            log.info("Danh sách tài khoản rỗng hoặc null. Trả về tổng số dư: 0.0");
            return 0.0;
        }

        // Lọc danh sách các tài khoản hợp lệ sử dụng Java 17 Stream API
        List<Account> validAccounts = accounts.stream()
                .filter(Objects::nonNull)
                .filter(account -> Objects.equals(targetBranch, account.getBranch()))
                .filter(account -> !activeOnly || "ACTIVE".equalsIgnoreCase(account.getStatus()))
                .filter(account -> account.getBalance() > 0)
                .toList(); // Tính năng mới của Java 16/17 thay cho collect(Collectors.toList())

        // Tính tổng số dư từ các tài khoản hợp lệ
        double totalBalance = validAccounts.stream()
                .mapToDouble(Account::getBalance)
                .sum();

        // Ghi log giám sát kết quả
        log.info("Hoàn tất tính toán chi nhánh [{}]. Số lượng TK hợp lệ: {}. Tổng số dư: {}", 
                 targetBranch, validAccounts.size(), totalBalance);

        return totalBalance;
    }
}