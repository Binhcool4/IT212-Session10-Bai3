[Vai trò]: Đóng vai trò là một System Analyst kiêm Backend Technical Lead.

[Ngữ cảnh]: Trong workspace hiện tại, file `CartService.java` thuộc module Giỏ hàng (thuộc Guai-api) đang thực hiện logic thêm sản phẩm vào giỏ thông qua hàm `addToCart(Long userId, Long productId, int quantity)`. Tuy nhiên, đoạn code này đang được viết ở mức "Happy Path" (chỉ xử lý luồng lý tưởng) và thiếu hụt các ràng buộc toàn vẹn dữ liệu quan trọng.

[Nhiệm vụ 1 - Phân tích lỗ hổng]: Hãy đọc hiểu ngữ cảnh của hàm `addToCart` và chỉ ra chi tiết các lỗ hổng logic nghiệp vụ (Logic Flaws) có thể gây hỏng hóc dữ liệu, thất thoát tài sản hoặc lỗi hệ thống. (Gợi ý: chú ý đến các giá trị biên của biến `quantity` và trạng thái tồn kho thực tế).

[Nhiệm vụ 2 - Xây dựng Đặc tả]: Dựa trên các lỗ hổng vừa tìm được, hãy xuất ra một danh sách "Quy tắc nghiệp vụ (Business Rules)" theo chuẩn tài liệu SRS để team Dev (cụ thể là các kỹ sư Spring Boot) có căn cứ tiến hành refactor và vá lỗi.

[Định dạng]: Trình bày rõ ràng bằng Markdown. Tách biệt phần "Phân tích lỗ hổng" và "Danh sách Business Rules". Các quy tắc nghiệp vụ cần được đánh mã số (VD: BR-CART-01) và có mô tả ràng buộc chặt chẽ để đảm bảo tính nhất quán của dữ liệu giỏ hàng qua các request.
## PHẦN 1: PHÂN TÍCH LỖ HỔNG LOGIC (LOGIC FLAWS) TRONG CART SERVICE

Sau khi phân tích hàm `addToCart`, phát hiện 4 lỗ hổng nghiêm trọng về mặt nghiệp vụ và toàn vẹn dữ liệu:

1.  **Lỗ hổng Giá trị âm/bằng không (Negative/Zero Quantity):**
    Hệ thống không kiểm tra tham số `quantity` đầu vào. Kẻ tấn công hoặc lỗi UI có thể truyền vào giá trị `quantity = -10`. Lúc này, dòng lệnh `item.setQuantity(item.getQuantity() + quantity)` sẽ làm giảm số lượng giỏ hàng sai quy tắc, hoặc tạo ra một bản ghi giỏ hàng mang số lượng âm, dẫn đến lỗi tính toán tổng tiền ở bước thanh toán.
2.  **Lỗ hổng Vượt quá tồn kho (Inventory Bypass):**
    Code hoàn toàn không gọi đến thuộc tính stock (tồn kho) của đối tượng `Product`. Người dùng có thể thêm 1,000 sản phẩm vào giỏ trong khi kho chỉ còn 5 sản phẩm.
3.  **Lỗ hổng Thiếu kiểm tra trạng thái sản phẩm (Product Status):**
    Sản phẩm có thể tồn tại trong database (`findById` thành công) nhưng đang ở trạng thái "Ngừng kinh doanh" hoặc "Ẩn". Đưa sản phẩm này vào giỏ hàng sẽ gây lỗi ở luồng Checkout.
4.  **Lỗ hổng Tranh chấp dữ liệu (Thiếu Transactional):**
    Do thiếu annotation `@Transactional`, nếu có lỗi xảy ra ngay sau khi cập nhật số lượng nhưng chưa kịp `save`, hoặc có nhiều request bắn tới cùng một lúc (Race Condition), dữ liệu giỏ hàng sẽ bị sai lệch và không nhất quán.

---

## PHẦN 2: DANH SÁCH QUY TẮC NGHIỆP VỤ (BUSINESS RULES) ĐỂ VÁ LỖI

Team Dev cần áp dụng ngay các quy tắc sau vào quá trình refactor `CartService.java`:

*   **BR-CART-01: Ràng buộc số lượng đầu vào tối thiểu (Minimum Quantity)**
    *   Hệ thống bắt buộc phải kiểm tra tham số `quantity`.
    *   **Điều kiện:** `quantity` phải là một số nguyên dương (> 0).
    *   **Ngoại lệ:** Trả về lỗi `400 Bad Request` với thông báo "Số lượng thêm vào giỏ hàng phải lớn hơn 0" nếu vi phạm.

*   **BR-CART-02: Ràng buộc trạng thái kinh doanh của sản phẩm (Product Availability)**
    *   Hệ thống chỉ cho phép thêm sản phẩm đang ở trạng thái hoạt động.
    *   **Điều kiện:** Thuộc tính trạng thái của `Product` (VD: `product.isActive()`) phải bằng `true`.
    *   **Ngoại lệ:** Trả về lỗi `400 Bad Request` với thông báo "Sản phẩm này hiện không khả dụng để mua".

*   **BR-CART-03: Ràng buộc giới hạn tồn kho (Inventory Limit Check)**
    *   Tổng số lượng của một sản phẩm trong giỏ hàng không được vượt quá số lượng tồn kho thực tế của sản phẩm đó.
    *   **Công thức kiểm tra:** `(Số lượng đang có trong giỏ + quantity truyền vào) <= product.getStock()`.
    *   **Ngoại lệ:** Trả về lỗi `409 Conflict` (hoặc `422 Unprocessable Entity`) với thông báo: "Số lượng vượt quá giới hạn tồn kho hiện tại. Kho còn: [X] sản phẩm".

*   **BR-CART-04: Ràng buộc giới hạn số lượng tối đa trên mỗi sản phẩm (Max Per Item Limit - Tùy chọn bảo vệ)**
    *   Để tránh spam hoặc lỗi số quá lớn (Integer Overflow), thiết lập một ngưỡng tối đa cho mỗi mặt hàng trong giỏ.
    *   **Điều kiện:** `(Số lượng đang có trong giỏ + quantity) <= MAX_ALLOWED_PER_ITEM` (VD: 99).
    *   **Ngoại lệ:** Trả về thông báo "Bạn chỉ được phép mua tối đa 99 sản phẩm loại này trong một đơn hàng".

*   **BR-CART-05: Đảm bảo tính toàn vẹn Giao dịch (Transaction Integrity)**
    *   Hàm `addToCart` phải được bọc trong một Database Transaction (sử dụng `@Transactional` của Spring).
    *   **Mục đích:** Đảm bảo tính ACID, giúp dữ liệu giỏ hàng (Cart) luôn được rollback sạch sẽ nếu có bất kỳ RuntimeException nào xảy ra trong quá trình kiểm tra kho hoặc lưu vào DB, giữ cho phiên làm việc luôn nhất quán.