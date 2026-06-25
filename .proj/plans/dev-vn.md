# Kế hoạch: Chuyển đổi lunar-go sang lịch Việt Nam (UTC+7)

## Bối cảnh kỹ thuật

Lịch âm Việt Nam và Trung Quốc dùng cùng thuật toán thiên văn, chỉ khác **múi giờ**:
- Trung Quốc: UTC+8 → `ONE_THIRD = 1/3` (8/24 ngày)
- Việt Nam: UTC+7 → cần `7.0/24.0`

Sự chênh lệch 1 giờ này dẫn đến **tháng mới (ngày sóc) và tiết khí có thể sai 1 ngày** khi ngày sóc/tiết khí rơi vào khoảng 23:00–24:00 giờ Hà Nội.

### Các thành phần cần thay đổi trong `ShouXingUtil/ShouXingUtil.go`

| Thành phần | Loại | Hiện tại (UTC+8) | Cần đổi (UTC+7) |
|---|---|---|---|
| `ONE_THIRD` | Hằng số dùng trong tính toán cao độ | `1/3` (8h) | `7.0/24.0` |
| `SHUO_KB` | Bảng tra cứu ngày sóc (float64) | Baked UTC+8 | Cần recompute UTC+7 |
| `QI_KB` | Bảng tra cứu tiết khí (float64) | Baked UTC+8 | Cần recompute UTC+7 |
| `SB` | Bảng điều chỉnh +1/-1 ngày sóc | Baked UTC+8 | Cần recompute UTC+7 |
| `QB` | Bảng điều chỉnh +1/-1 tiết khí | Baked UTC+8 | Cần recompute UTC+7 |

---

## Kế hoạch thực hiện

### Bước 1 — Viết script Go tính lại bảng dữ liệu UTC+7

Tạo file `tools/gen_vn_tables/main.go` để tính lại `SB` và `QB` cho UTC+7. Các bảng `SHUO_KB` và `QI_KB` là interval tables (Julian Day + step), không phụ thuộc timezone trực tiếp — chúng được dùng để tìm khoảng thời gian gần đúng, rồi kết quả được làm tròn bằng `ONE_THIRD`. Vì vậy:

- `SHUO_KB` và `QI_KB`: **giữ nguyên** (chỉ dùng để lookup khoảng thời gian gần đúng)
- `SB` và `QB`: **cần recompute** vì chúng là bảng điều chỉnh ±1 ngày đã được encode theo UTC+8

**Cách verify:** So sánh kết quả ngày sóc UTC+7 với UTC+8 — những tháng nào khác nhau → cập nhật SB/QB.

```go
// tools/gen_vn_tables/main.go
package main

import (
    "fmt"
    "math"
    "strings"
)

const ONE_THIRD_VN = 7.0 / 24.0  // UTC+7

// Copy toàn bộ hàm shuoHigh, shuoLow, qiHigh, qiLow từ ShouXingUtil
// Thay ONE_THIRD -> ONE_THIRD_VN
// Chạy cho toàn bộ range của SB/QB
// In ra bảng SB_VN, QB_VN
func main() {
    // range SB: từ f2 của SHUO_KB đến f3=2436935
    // range QB: từ f2 của QI_KB đến f3=2436935
    generateSB()
    generateQB()
}
```

### Bước 2 — Tạo package `ShouXingUtilVN` (hoặc thêm tham số vào hàm hiện có)

**Cách tiếp cận được khuyến nghị: Tạo package riêng** để không làm hỏng code hiện tại.

```
ShouXingUtilVN/
  ShouXingUtilVN.go   ← copy ShouXingUtil.go, đổi ONE_THIRD và SB/QB
```

```go
// ShouXingUtilVN/ShouXingUtilVN.go
package ShouXingUtilVN

const ONE_THIRD = 7.0 / 24.0  // UTC+7 thay vì UTC+8

var SB = "..."  // bảng mới tính từ Bước 1
var QB = "..."  // bảng mới tính từ Bước 1

// Các hàm CalcShuo, CalcQi, QiAccurate, QiAccurate2 giữ nguyên logic
// (chúng sẽ tự động dùng ONE_THIRD mới của package này)
```

### Bước 3 — Tạo `calendar/LunarYearVN.go`

```go
// calendar/LunarYearVN.go
package calendar

// NewLunarYearVN tạo âm lịch theo múi giờ Việt Nam (UTC+7)
func NewLunarYearVN(lunarYear int) *LunarYear {
    // Giống NewLunarYear nhưng dùng ShouXingUtilVN thay ShouXingUtil
}
```

Vì `LunarYear.compute()` gọi trực tiếp `ShouXingUtil.CalcQi` và `ShouXingUtil.CalcShuo`, cần refactor để nhận interface hoặc tách hàm riêng.

**Cách refactor `LunarYear.compute()`:**

```go
type CalcFuncs struct {
    CalcQi      func(jd float64) float64
    CalcShuo    func(jd float64) float64
    QiAccurate2 func(jd float64) float64
}

var defaultCalcFuncs = CalcFuncs{
    CalcQi:      ShouXingUtil.CalcQi,
    CalcShuo:    ShouXingUtil.CalcShuo,
    QiAccurate2: ShouXingUtil.QiAccurate2,
}

var vnCalcFuncs = CalcFuncs{
    CalcQi:      ShouXingUtilVN.CalcQi,
    CalcShuo:    ShouXingUtilVN.CalcShuo,
    QiAccurate2: ShouXingUtilVN.QiAccurate2,
}

func (lunarYear *LunarYear) computeWith(funcs CalcFuncs) { ... }
```

### Bước 4 — Entry points cho người dùng

```go
// calendar/Solar.go — thêm method
func (solar *Solar) GetLunarVN() *Lunar {
    return NewLunarFromSolarVN(solar)
}

// calendar/Lunar.go — thêm constructor
func NewLunarFromSolarVN(solar *Solar) *Lunar {
    ly := NewLunarYearVN(solar.GetYear())
    // ... logic tương tự NewLunarFromSolar
}

func NewLunarVN(lunarYear, lunarMonth, lunarDay, hour, minute, second int) *Lunar {
    // dùng NewLunarYearVN thay NewLunarYear
}
```

### Bước 5 — Viết test so sánh VN vs CN

Tạo `test/LunarVN_test.go`:

```go
// Các ngày biên giới đã biết VN khác CN
// Ví dụ: tháng 11 âm 2014 — Trung Quốc ngày 22/11, Việt Nam ngày 23/11
var knownDiffs = []struct {
    solar        string // dương lịch
    lunarCN      string // âm CN
    lunarVN      string // âm VN
}{
    {"2014-11-22", "2014-10-01", "2014-09-30"}, // ví dụ minh họa
}

func TestLunarVNKnownDates(t *testing.T) {
    for _, tc := range knownDiffs {
        solar := parseSolar(tc.solar)
        cn := solar.GetLunar()
        vn := solar.GetLunarVN()
        // assert vn matches tc.lunarVN
    }
}
```

**Danh sách ngày cần test** (lấy từ tài liệu lịch sử so sánh VN/CN):
- 2014-11-22: sóc tháng 10 CN ↔ tháng 9 VN
- 2020-04-23: sóc tháng 4 CN ↔ tháng 3 VN  
- Các năm có tháng nhuận khác nhau giữa VN và CN

---

## Thứ tự thực hiện

```
[1] Script gen_vn_tables → tính SB_VN, QB_VN
[2] ShouXingUtilVN/     → package tính toán UTC+7
[3] Refactor LunarYear  → tách CalcFuncs thành injectable
[4] LunarYearVN         → dùng ShouXingUtilVN
[5] Entry points        → GetLunarVN(), NewLunarVN()
[6] Test                → verify các ngày biên giới VN/CN
```

## Rủi ro và lưu ý

- **Tháng nhuận khác nhau**: Một số năm VN và CN có tháng nhuận khác nhau hoàn toàn (không chỉ ngày sóc lệch 1 ngày) — đây là hệ quả tự nhiên của thuật toán khi dùng UTC+7, không phải lỗi.
- **Backward compatibility**: Không sửa API hiện tại. Tất cả method VN là method mới (`GetLunarVN`, `NewLunarVN`...).
- **HolidayUtil**: Hiện tại chỉ chứa dữ liệu lịch nghỉ Trung Quốc — cần thêm `HolidayUtilVN` riêng nếu muốn hỗ trợ ngày lễ Việt Nam.
