# BÀI 3: ĐỌC HIỂU & DÒ LỖI - LẬP TRÌNH PHÒNG THỦ CHỐNG ẢO TƯỞNG THAM SỐ
---

## I. PHÂN TÍCH CÁC LỖI LOGIC VÀ ĐIỂM YẾU TRONG MÃ NGUỒN CŨ

Đoạn mã nguồn ban đầu của `BookingService`:

```java
@Service
public class BookingService {
 
    @Tool(description = "Kiểm tra phòng trống khách sạn")
    public String getRoomAvailability(String checkIn, String checkOut, String roomType) {
        // Thực thi kiểm tra database
        LocalDate start = LocalDate.parse(checkIn);
        LocalDate end = LocalDate.parse(checkOut);
 
        if (start.isAfter(end)) {
            throw new IllegalArgumentException("Ngày nhận phòng không thể sau ngày trả phòng.");
        }
 
        // Logic giả lập truy vấn database
        boolean isAvailable = "Deluxe".equalsIgnoreCase(roomType);
        return isAvailable ? "Còn phòng trống" : "Hết phòng";
    }
}
```

Dưới góc nhìn **Defensive Programming (Lập trình phòng thủ)** và kiến trúc tích hợp **LLM / Spring AI Tool Calling**, đoạn mã trên gặp phải 4 nhóm lỗ hổng nghiêm trọng sau:

### 1.1 Lỗi `NullPointerException` (NPE) khi LLM trích xuất thiếu thông tin
- **Nguyên nhân gốc rễ:** Mô hình ngôn ngữ lớn (LLM) trích xuất dữ liệu từ câu chat tự do của người dùng. Khi người dùng không cung cấp đủ thông tin (ví dụ: *"Kiểm tra giúp tôi phòng Deluxe vào ngày 2026-07-15"* - thiếu `checkOut`), LLM sẽ truyền giá trị `null` vào các tham số của phương thức Java.
- **Điểm yếu mã nguồn:** 
  - `LocalDate.parse(checkIn)` sẽ lập tức ném ra `NullPointerException` nếu `checkIn` hoặc `checkOut` là `null`.
  - `"Deluxe".equalsIgnoreCase(roomType)` sẽ an toàn hơn `roomType.equalsIgnoreCase("Deluxe")`, nhưng nếu `roomType` bị `null` thì các thao tác xử lý sau đó (như ghi log hoặc format) vẫn có nguy cơ nổ NPE.
- **Hậu quả:** Ứng dụng dừng đột ngột mà không kịp bắt lỗi.

### 1.2 Lỗi `DateTimeParseException` do ảo tưởng định dạng ngày (Format Hallucination)
- **Nguyên nhân gốc rễ:** LLM có thể trả về các chuỗi ngày tháng không tuân thủ chuẩn ISO-8601 (`YYYY-MM-DD`). Ví dụ: LLM truyền vào `"15-07-2026"`, `"15/07/2026"`, hoặc các chuỗi mơ hồ như `"hôm nay"`, `"ngày mai"`.
- **Điểm yếu mã nguồn:** Phương thức `LocalDate.parse(CharSequence text)` mặc định chỉ chấp nhận định dạng `YYYY-MM-DD` (ISO_LOCAL_DATE). Khi nhận giá trị lệch chuẩn, Java Runtime sẽ lập tức ném ra `DateTimeParseException`. Ngoài ra, nếu ngày không hợp lệ trên lịch (ví dụ: `"2026-02-30"`), hàm `parse()` cũng thất bại.

### 1.3 Lỗi kiến trúc: Ném Runtime Exception làm Crash luồng Spring AI Engine (HTTP 500)
- **Nguyên nhân gốc rễ:** Khi câu lệnh `throw new IllegalArgumentException(...)` kích hoạt, Exception này không được đóng gói lại ở tầng Tool Callback.
- **Điểm yếu mã nguồn:**
  - Trong luồng xử lý Tool Call của Spring AI Engine, khi một `@Tool` method ném ra Exception không được kiểm soát (Unhandled Runtime Exception), Spring AI sẽ coi Tool execution bị bẻ gãy (Aborted).
  - Exception lan truyền thẳng lên Controller, trả về mã lỗi **HTTP 500 Internal Server Error** cho người dùng cuối.
  - **Gián đoạn hội thoại (Conversation Breakdown):** LLM bị mất ngữ cảnh giữa chừng, không thể đọc được thông điệp lỗi để tiếp tục hội thoại hỏi lại người dùng (*"Bạn ơi, ngày trả phòng phải sau ngày nhận phòng, vui lòng chọn lại"*).

### 1.4 Hạn chế về Schema: Truyền các tham số đơn lẻ kiểu String thô (`String checkIn, String checkOut...`)
- **Nguyên nhân gốc rễ:** Phương thức nhận các biến đơn lẻ làm cho Spring AI chỉ sinh ra JSON Schema ở mức cơ bản, thiếu các mô tả ngữ cảnh (descriptions) và các ràng buộc dữ liệu chi tiết cho từng trường.
- **Điểm yếu mã nguồn:** LLM không nhận được chỉ dẫn rõ ràng về format mong muốn (`YYYY-MM-DD`), dẫn đến tỉ lệ hallucination (ảo tưởng tham số) cao hơn nhiều so với khi có JSON Schema chuẩn hóa từ DTO/Record.

---

## II. GIẢI TRÌNH GIẢI PHÁP VALIDATE DỮ LIỆU PHÒNG THỦ (DEFENSIVE STRATEGY)

Để khắc phục triệt để các rủi ro trên và đạt tiêu chuẩn sản xuất (Production-ready), giải pháp lập trình phòng thủ được xây dựng qua 4 trụ cột chính:

```
[User Chat Prompt] 
       │
       ▼
[Spring AI Engine (LLM Tool Calling)]
       │ (JSON Schema từ RoomCheckRequest)
       ▼
┌─────────────────────────────────────────────────────────┐
│              PIPELINE VALIDATE 4 LỚP                    │
│ 1. Null & Blank Check  ──> Trả về Response (isSuccess=false)
│ 2. Regex Format Check ──> Trả về Response (isSuccess=false)
│ 3. Parse Date Check   ──> Trả về Response (isSuccess=false)
│ 4. Business Logic     ──> Trả về Response (isSuccess=false)
└──────────────────────────┬──────────────────────────────┘
                           │ (Nếu vượt qua 4 lớp)
                           ▼
              [Thực thi Logic Booking DB]
                           │
                           ▼
            [RoomCheckResponse (isSuccess=true)]
                           │
                           ▼
          [LLM đọc Response & Phản hồi mượt mà]
```

### 2.1 Đóng gói DTO với Java Record (`RoomCheckRequest` & `RoomCheckResponse`)
- **Lợi ích đối với Spring AI:** Java Record tự động sinh ra các getter immutable và hỗ trợ annotation `@JsonPropertyDescription`. Khi Spring AI phân tích `RoomCheckRequest`, nó sẽ tạo ra JSON Schema với các chú thích rõ ràng giúp LLM hiểu chính xác định dạng cần trích xuất (ví dụ: `YYYY-MM-DD`).
- **Phân tách phản hồi (`RoomCheckResponse`):** Chứa các trường thông tin chuẩn hóa:
  - `boolean isSuccess`: Đánh giá xem quá trình kiểm tra (Tool call) có thực thi thành công về mặt kỹ thuật & nghiệp vụ tham số hay không.
  - `boolean isAvailable`: Kết quả phòng trống.
  - `double pricePerNight`: Đơn giá phòng theo đêm.
  - `String message`: Thông điệp phản hồi chi tiết cho cả AI và người dùng.

### 2.2 Quy trình Kiểm chứng Dữ liệu 4 Lớp (4-Layer Defensive Validation Pipeline)
Trước khi gọi bất kỳ hàm xử lý logic nào, dữ liệu phải đi qua 4 bộ lọc phòng thủ:

1. **Bộ lọc 1 - Null & Blank Check (Chống NPE):**
   - Kiểm tra `request == null`, cũng như từng trường `checkIn`, `checkOut`, `roomType` xem có `null` hoặc rỗng (`.isBlank()`) hay không.
2. **Bộ lọc 2 - Định dạng Regex (Chống DateTimeParseException):**
   - Sử dụng Pattern Regex `^\d{4}-\d{2}-\d{2}$` để đảm bảo chuỗi ngày tuân theo đúng khuôn dạng 4 chữ số năm - 2 chữ số tháng - 2 chữ số ngày.
3. **Bộ lọc 3 - Parse thử nghiệm (Strict Date Parse Check):**
   - Đặt `LocalDate.parse()` trong khối `try-catch (DateTimeParseException)`. Việc này giúp ngăn chặn các ngày "ma" không tồn tại trên lịch (ví dụ `"2026-02-30"` hay `"2026-04-31"`).
4. **Bộ lọc 4 - Ràng buộc Nghiệp vụ (Business Rule Check):**
   - Ngày nhận phòng phải trước hoặc bằng ngày trả phòng (`!start.isAfter(end)` hoặc `start.isBefore(end)`).
   - Ngày nhận phòng không được nằm trong quá khứ so với ngày hiện tại (`!start.isBefore(LocalDate.now())`).

### 2.3 Nguyên tắc Fail-Safe & Degradation (Tuyệt đối không ném Exception)
- Thay vì sử dụng `throw new IllegalArgumentException(...)`, tất cả các điểm thất bại trong validation đều **bắt giữ lỗi và đóng gói trả về** đối tượng `RoomCheckResponse` với:
  - `isSuccess = false`
  - `isAvailable = false`
  - `pricePerNight = 0.0`
  - `message = "Mô tả chi tiết lý do sai sót"`
- **Tác dụng:** Giữ cho phương thức Java luôn chạy hoàn tất thành công (Exit Code 0), ngăn chặn HTTP 500, bảo vệ Spring AI Engine không bị crash luồng.

### 2.4 Vòng phản hồi tự sửa lỗi của AI (AI Self-Correction Loop)
- Khi `RoomCheckResponse` trả về `isSuccess = false` kèm thông điệp lỗi chi tiết (ví dụ: *"Ngày nhận phòng (2026-07-20) không thể sau ngày trả phòng (2026-07-15). Vui lòng chọn lại ngày checkout."*), LLM sẽ đọc thông điệp này từ Tool Output.
- LLM tự động chuyển hướng hội thoại thông minh: *"Tôi thấy ngày trả phòng của bạn đang bị đặt trước ngày nhận phòng. Bạn có thể cho tôi biết lại ngày trả phòng chính xác không?"* thay vì báo lỗi hệ thống 500.

---

## III. MÃ NGUỒN JAVA SAU KHU REFACTOR THÀNH CÔNG

### 3.1 DTO Đầu vào: `RoomCheckRequest.java`

```java
package com.rhotels.booking.dto;

import com.fasterxml.jackson.annotation.JsonPropertyDescription;

/**
 * Record đóng gói yêu cầu kiểm tra phòng trống.
 * Hỗ trợ Spring AI tự động sinh JSON Schema chặt chẽ cho LLM.
 */
public record RoomCheckRequest(
    @JsonPropertyDescription("Ngày nhận phòng (Check-in date), bắt buộc đúng định dạng YYYY-MM-DD. Ví dụ: 2026-07-15")
    String checkIn,

    @JsonPropertyDescription("Ngày trả phòng (Check-out date), bắt buộc đúng định dạng YYYY-MM-DD. Ví dụ: 2026-07-18")
    String checkOut,

    @JsonPropertyDescription("Loại phòng khách sạn mong muốn. Ví dụ: Standard, Deluxe, Suite")
    String roomType
) {}
```

### 3.2 DTO Đầu ra: `RoomCheckResponse.java`

```java
package com.rhotels.booking.dto;

/**
 * Record đóng gói kết quả phản hồi từ Tool kiểm tra phòng trống.
 * Đảm bảo không ném Exception ra ngoài Spring AI Engine.
 */
public record RoomCheckResponse(
    boolean isSuccess,      // Cờ báo hiệu yêu cầu kiểm tra có hợp lệ hay không
    boolean isAvailable,    // Cờ báo hiệu còn phòng trống hay hết phòng
    double pricePerNight,   // Đơn giá phòng mỗi đêm (VNĐ/Đêm hoặc USD)
    String message          // Thông điệp chi tiết phản hồi người dùng hoặc hướng dẫn AI hỏi lại
) {
    /**
     * Static Factory Method tạo Response thất bại (Dữ liệu đầu vào không hợp lệ)
     */
    public static RoomCheckResponse failure(String errorMessage) {
        return new RoomCheckResponse(false, false, 0.0, errorMessage);
    }

    /**
     * Static Factory Method tạo Response thành công (Đã kiểm tra nghiệp vụ xong)
     */
    public static RoomCheckResponse success(boolean isAvailable, double pricePerNight, String message) {
        return new RoomCheckResponse(true, isAvailable, pricePerNight, message);
    }
}
```

### 3.3 Service xử lý Tool: `BookingService.java`

```java
package com.rhotels.booking.service;

import com.rhotels.booking.dto.RoomCheckRequest;
import com.rhotels.booking.dto.RoomCheckResponse;
import org.springframework.ai.tool.annotation.Tool;
import org.springframework.stereotype.Service;

import java.time.LocalDate;
import java.time.format.DateTimeParseException;
import java.util.regex.Pattern;

@Service
public class BookingService {

    // Regex kiểm tra định dạng ISO date YYYY-MM-DD
    private static final Pattern DATE_PATTERN = Pattern.compile("^\\d{4}-\\d{2}-\\d{2}$");

    @Tool(description = "Kiểm tra phòng trống và đơn giá của khách sạn dựa trên ngày nhận phòng, ngày trả phòng và loại phòng")
    public RoomCheckResponse getRoomAvailability(RoomCheckRequest request) {
        
        // =========================================================================
        // LỚP 1: DEFENSIVE VALIDATION - KIỂM TRA NULL & RỖNG (ANTI NULLPOINTEREXCEPTION)
        // =========================================================================
        if (request == null) {
            return RoomCheckResponse.failure("Lỗi dữ liệu: Yêu cầu kiểm tra phòng bị null.");
        }

        if (request.checkIn() == null || request.checkIn().isBlank()) {
            return RoomCheckResponse.failure("Thiếu thông tin ngày nhận phòng (checkIn). Vui lòng cung cấp ngày nhận phòng theo định dạng YYYY-MM-DD.");
        }

        if (request.checkOut() == null || request.checkOut().isBlank()) {
            return RoomCheckResponse.failure("Thiếu thông tin ngày trả phòng (checkOut). Vui lòng cung cấp ngày trả phòng theo định dạng YYYY-MM-DD.");
        }

        if (request.roomType() == null || request.roomType().isBlank()) {
            return RoomCheckResponse.failure("Thiếu thông tin loại phòng (roomType). Vui lòng chọn loại phòng mong muốn (Ví dụ: Standard, Deluxe, Suite).");
        }

        String checkInStr = request.checkIn().trim();
        String checkOutStr = request.checkOut().trim();
        String roomTypeStr = request.roomType().trim();

        // =========================================================================
        // LỚP 2: DEFENSIVE VALIDATION - KIỂM TRA ĐỊNH DẠNG REGEX (YYYY-MM-DD)
        // =========================================================================
        if (!DATE_PATTERN.matcher(checkInStr).matches()) {
            return RoomCheckResponse.failure(
                String.format("Ngày nhận phòng '%s' không đúng định dạng YYYY-MM-DD. Vui lòng đính chính lại (Ví dụ: 2026-07-15).", checkInStr)
            );
        }

        if (!DATE_PATTERN.matcher(checkOutStr).matches()) {
            return RoomCheckResponse.failure(
                String.format("Ngày trả phòng '%s' không đúng định dạng YYYY-MM-DD. Vui lòng đính chính lại (Ví dụ: 2026-07-18).", checkOutStr)
            );
        }

        // =========================================================================
        // LỚP 3: DEFENSIVE VALIDATION - PARSE THỬ NGHIỆM ĐỂ CHỐNG LỖI NGÀY KHÔNG TỒN TẠI
        // =========================================================================
        LocalDate start;
        LocalDate end;

        try {
            start = LocalDate.parse(checkInStr);
        } catch (DateTimeParseException e) {
            return RoomCheckResponse.failure(
                String.format("Ngày nhận phòng '%s' không tồn tại trên lịch thực tế. Vui lòng kiểm tra lại.", checkInStr)
            );
        }

        try {
            end = LocalDate.parse(checkOutStr);
        } catch (DateTimeParseException e) {
            return RoomCheckResponse.failure(
                String.format("Ngày trả phòng '%s' không tồn tại trên lịch thực tế. Vui lòng kiểm tra lại.", checkOutStr)
            );
        }

        // =========================================================================
        // LỚP 4: DEFENSIVE VALIDATION - RÀNG BUỘC NGHIỆP VỤ (BUSINESS LOGIC VALIDATION)
        // =========================================================================
        // Kiểm tra ngày nhận phòng không được sau ngày trả phòng
        if (start.isAfter(end)) {
            return RoomCheckResponse.failure(
                String.format("Ngày nhận phòng (%s) không thể diễn ra sau ngày trả phòng (%s). Vui lòng chọn lại khoảng thời gian phù hợp.", checkInStr, checkOutStr)
            );
        }

        // (Tùy chọn) Kiểm tra ngày nhận phòng không được ở trong quá khứ
        LocalDate today = LocalDate.now();
        if (start.isBefore(today)) {
            return RoomCheckResponse.failure(
                String.format("Ngày nhận phòng (%s) không thể là ngày trong quá khứ (Hôm nay là %s).", checkInStr, today)
            );
        }

        // =========================================================================
        // LOGIC XỬ LÝ TRUY VẤN DATABASE (GIẢ LẬP SAU KHI DỮ LIỆU ĐÃ AN TOÀN 100%)
        // =========================================================================
        double pricePerNight;
        boolean isAvailable;

        switch (roomTypeStr.toUpperCase()) {
            case "DELUXE":
                isAvailable = true;
                pricePerNight = 1500000.0; // 1,500,000 VNĐ
                break;
            case "SUITE":
                isAvailable = true;
                pricePerNight = 3000000.0; // 3,000,000 VNĐ
                break;
            case "STANDARD":
                isAvailable = false; // Giả sử loại Standard đã hết phòng
                pricePerNight = 800000.0;
                break;
            default:
                // Nếu loại phòng không nằm trong danh mục của khách sạn
                return RoomCheckResponse.failure(
                    String.format("Khách sạn R-Hotels hiện không có hạng phòng '%s'. Các hạng phòng hiện có: Standard, Deluxe, Suite.", roomTypeStr)
                );
        }

        // Trả về kết quả thành công kèm thông điệp đầy đủ
        if (isAvailable) {
            return RoomCheckResponse.success(
                true,
                pricePerNight,
                String.format("Hạng phòng %s còn trống từ ngày %s đến ngày %s. Giá niêm yết: %,.0f VNĐ/đêm.", 
                    roomTypeStr, checkInStr, checkOutStr, pricePerNight)
            );
        } else {
            return RoomCheckResponse.success(
                false,
                pricePerNight,
                String.format("Rất tiếc, hạng phòng %s đã HẾT PHÒNG trong khoảng thời gian từ %s đến %s.", 
                    roomTypeStr, checkInStr, checkOutStr)
            );
        }
    }
}
```

---

## IV. TỔNG KẾT VÀ BÀI HỌC RÚT RA

1. **Nguyên tắc "Do Not Trust LLM Input":** Trong các hệ thống AI Agents / Tool Calling, dữ liệu do LLM trích xuất từ câu thoại của người dùng luôn phải được coi là dữ liệu không an toàn (Untrusted Input). Lập trình viên bắt buộc phải đặt bộ lọc Validate chặt chẽ trước khi đưa vào logic xử lý hệ thống.
2. **Nguyên tắc "Fail Gracefully":** Tuyệt đối không ném Unhandled Exception ở tầng Tool Callback. Đóng gói lỗi thành kết quả phản hồi chứa thông điệp mô tả giúp hệ thống chạy liên tục, đồng thời nâng cao trải nghiệm người dùng thông qua khả năng tự phản hồi đính chính của AI.
3. **Chuẩn hóa JSON Schema:** Sử dụng DTO/Record kết hợp với Jackson Annotation `@JsonPropertyDescription` giúp định hình JSON Schema chuẩn xác cho Spring AI Engine, giảm tới 90% nguy cơ LLM ảo tưởng tham số đầu vào.
