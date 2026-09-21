Phần 1 – Đề xuất 2 giải pháp lưu trữ
Giải pháp 1: Bảng giá tĩnh dạng Ma trận (Static Price Matrix)

Mỗi tổ hợp (SeatType, TimeSlot, DayType) là một dòng có sẵn giá.

PRICING_MATRIX
- PricingID        PK
- SeatType         (Standard / VIP / Sweetbox)
- TimeSlot         (Before17 / After17)
- DayType          (Weekday / Weekend / Holiday)
- Price            DECIMAL
  UNIQUE(SeatType, TimeSlot, DayType)

SPECIAL_DATE   -- bắt buộc để xử lý bẫy Lễ/Tết
- SpecialDate      PK  (ngày cụ thể, vd 2026-12-25)
- DayTypeOverride  ('Holiday')
- Description

Logic: khi tính giá, trước tiên tra SPECIAL_DATE theo ngày cụ thể; nếu có → dùng DayTypeOverride; nếu không → tính DayType từ thứ trong tuần (Sat/Sun = Weekend, còn lại = Weekday). Sau đó lookup thẳng trong PRICING_MATRIX bằng 3 khóa.

Giải pháp 2: Rule Engine (Base Price + Adjustment Rules)

Giá gốc theo loại ghế, cộng thêm các "lớp" điều chỉnh theo điều kiện.

BASE_PRICE
- SeatType        PK
- BasePrice

PRICING_RULE
- RuleID              PK
- AppliesToSeatType   FK (nullable = áp dụng mọi loại ghế)
- AppliesToTimeSlot   FK (nullable)
- AppliesToDayType    FK (nullable)
- AdjustmentType      (% hoặc Fixed)
- AdjustmentValue
- Priority
- EffectiveFrom / EffectiveTo

SPECIAL_DATE  -- giống giải pháp 1

Logic: lấy BasePrice theo SeatType, sau đó áp lần lượt các PRICING_RULE đang hiệu lực (theo Priority) để cộng/nhân điều chỉnh ra giá cuối.

Phần 2 – Bảng so sánh Trade-off
Tiêu chí	GP1: Ma trận tĩnh	GP2: Rule Engine
Linh hoạt marketing	Thấp – đổi giá dịp Tết phải sửa/insert lại từng dòng cố định; không hỗ trợ khuyến mãi %/thời hạn ngắn tự nhiên	Cao – chỉ cần thêm 1 rule mới (vd "giảm 20% VIP dịp Tết 1-7/2") mà không đụng dữ liệu giá gốc
Tốc độ query	Rất nhanh – 1 lần lookup theo khóa (SeatType, TimeSlot, DayType), có thể index trực tiếp	Chậm hơn – phải join nhiều rule, tính toán runtime (lọc theo ngày hiệu lực, sắp theo priority, cộng dồn)
Xử lý bẫy ngày Lễ	Tốt nếu có bảng SPECIAL_DATE riêng (đã tách khỏi thứ trong tuần)	Tốt tương tự, vì SPECIAL_DATE dùng chung; rule engine còn dễ mở rộng thêm case đặc biệt khác (vd giờ vàng, combo)
Độ phức tạp cài đặt/bảo trì	Đơn giản, dễ hiểu, dễ audit	Phức tạp hơn – cần cơ chế resolve rule, xử lý xung đột priority, dễ bug logic nếu nhiều rule chồng nhau
Khả năng mở rộng thêm yếu tố mới (vd thêm Phòng chiếu 4DX)	Kém – matrix phình theo cấp số nhân số tổ hợp	Tốt – chỉ thêm 1 chiều điều kiện mới trong rule
Phần 3 – Giải pháp chọn: Hybrid (Ma trận nền + lớp Adjustment)

Chọn kết hợp cả hai: dùng Giải pháp 1 làm bảng giá nền (PRICING_RULE = base matrix, tra cứu nhanh, ổn định cho vận hành hằng ngày), cộng thêm bảng PRICE_ADJUSTMENT (từ tinh thần Giải pháp 2) chỉ dùng cho các đợt khuyến mãi/marketing ngắn hạn có thời hạn (StartDate–EndDate). Bẫy ngày Lễ được xử lý độc lập bằng SPECIAL_DATE dùng chung cho cả hai lớp.

Luồng tính giá (Showtime → Seat → Price):

Từ SHOWTIME.ShowDate → tra SPECIAL_DATE; có thì lấy DayTypeID override, không thì suy ra từ thứ trong tuần.
Từ SHOWTIME.ShowTime → xác định TimeSlot.
Từ SEAT.SeatTypeID → có SeatType.
Lookup PRICING_RULE(SeatType, TimeSlot, DayType) → BasePrice.
Kiểm tra PRICE_ADJUSTMENT đang hiệu lực (theo ngày chiếu, theo SeatType nếu có) → cộng/nhân điều chỉnh → ghi FinalPrice vào TICKET.
