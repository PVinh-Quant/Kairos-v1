# HƯỚNG DẪN KỸ THUẬT CHI TIẾT (TECHNICAL REFERENCE MANUAL)

Dự án Kairos-v1

> [!IMPORTANT]
> **LƯU Ý VỀ PHIÊN BẢN (PORTFOLIO VERSION)**
> 
> Đây là phiên bản phục vụ mục đích xây dựng **Portfolio cá nhân**.

---

## Mục lục tài liệu chuyên sâu

* [Cấu Hình Hệ Thống](#config)
* [Tầng Thu Thập Dữ Liệu (ETL & WebSockets)](#lay_du_lieu)
* [Engine Chiến Lược & Chỉ Báo Kỹ Thuật](#chien_luoc)
* [Thực Thi Lệnh & Quản Lý Vị Thế](#thuc_thi_lenh)
* [Kịch Bản Vận Hành (Runners)](#chuc_nang)
* [Giao Diện Dashboard PyQt6](#hien_thi)
* [Tiện Ích & Helpers](#utils)
* [Lưu Trữ Dữ Liệu](#du_lieu)
* [Hệ Thống Cảnh Báo & Alert](#thong_bao)

---

<a name="config"></a>

## Cấu Hình Hệ Thống

Thư mục chứa toàn bộ tham số cấu hình của hệ thống Kairos. Mọi module đều đọc cấu hình qua `utils/doc_cau_hinh.py` — không hardcode giá trị trực tiếp trong code.

---

## Các file cấu hình

### `cau_hinh_giao_dich.yaml` — Tham số giao dịch chính
Điểm kiểm soát trung tâm cho mọi chiến lược. Thay đổi file này để điều chỉnh hành vi giao dịch mà không cần sửa code.

| Tham số | Giá trị mặc định | Ý nghĩa |
|---|---|---|
| `san_giao_dich_chinh` | `"okx"` | Sàn giao dịch được dùng khi chạy realtime |
| `cap_giao_dich` | `["BTC/USDT"]` | Danh sách cặp giao dịch cần theo dõi |
| `don_bay` | `7` | Đòn bẩy sử dụng trong các lệnh |
| `max_lenh_cho_phep` | `20` | Giới hạn số vị thế mở đồng thời |
| `von_moi_lenh_usdt` | `100.0` | Số tiền phân bổ cho mỗi lệnh ($) |
| `cat_lo_percent` | `0.1` | Ngưỡng cắt lỗ (10%) |
| `chot_loi_percent` | `0.15` | Ngưỡng chốt lời (15%) |

### `thong_tin_san.yaml` — Thông số kỹ thuật từng sàn
Chứa thông số chi tiết của sàn giao dịch (như Binance, OKX, Bybit). Giúp tự động tính toán khối lượng tối thiểu và phí.

*   `min_notional`: Giá trị lệnh tối thiểu (ví dụ: `5.0` USDT).
*   `maker_fee` / `taker_fee`: Tỷ lệ phí giao dịch cho Maker/Taker (ví dụ: `0.0002` / `0.0004`).
*   `max_leverage`: Đòn bẩy tối đa cho phép trên sàn (ví dụ: `100`).

### `cau_hinh_ao_config.json` — Tham số paper trading / backtest
Được sử dụng bởi chế độ Demo và các bộ kiểm thử lịch sử (Backtest).

*   `so_du_ban_dau`: Vốn khởi tạo giả lập (ví dụ: `10000`).
*   `ngay_bat_dau`: Ngày bắt đầu backtest (`"2025-01-01"`).
*   `ngay_ket_thuc`: Ngày kết thúc backtest (`"2025-01-05"`).
*   `phi_giao_dich`: Tỷ lệ phí giao dịch áp dụng (`0.0004`).
*   `do_truot_gia`: Tỷ lệ trượt giá giả lập (`0.0001`).
*   `so_luong_luong`: Số lượng worker processes cho backtest đa luồng (nếu không khai báo sẽ tự động dùng `cpu_count - 1`).

### `tai_khoan_api.json` — API keys sàn giao dịch
> [!WARNING]
> File này chứa thông tin bảo mật nhạy cảm (API Key và Secret Key). **Tuyệt đối KHÔNG commit file này lên Git/GitHub.** File này đã được thêm vào hệ thống `.gitignore`.

### `tai_khoan_api.json.example` — Template API keys
Bản mẫu hướng dẫn người dùng thiết lập API Keys cho các sàn Binance, OKX, Bybit.

---

## Cách đọc cấu hình trong code

Các module không đọc trực tiếp file cấu hình mà import các hàm từ `utils/doc_cau_hinh.py`:

```python
from utils.doc_cau_hinh import (
    lay_cau_hinh_api,
    lay_cau_hinh_giao_dich,
    lay_thong_tin_san,
    lay_cau_hinh_ao
)

config_trade = lay_cau_hinh_giao_dich()
min_notional = lay_thong_tin_san().get('okx', {}).get('min_notional', 5.0)
```

---

## .gitignore

Cấu hình loại trừ các file rác sinh ra trong quá trình chạy, môi trường ảo và đặc biệt là file chứa thông tin bảo mật:
*   Môi trường ảo: `env/`, `venv/`
*   Cache Python: `**/__pycache__/`, `*.pyc`
*   Dữ liệu cục bộ: `du_lieu/nhat_ky_hoat_dong.log`
*   Cấu hình nhạy cảm: `config/tai_khoan_api.json`

---

<a name="lay_du_lieu"></a>

## Tầng Thu Thập Dữ Liệu (ETL & WebSockets)

Chịu trách nhiệm giao tiếp với sàn giao dịch thông qua API REST (CCXT) để lấy dữ liệu lịch sử hoặc duy trì kết nối WebSocket thời gian thực nhận dữ liệu thị trường trực tiếp.

---

## 1. `lay_ohlcv.py` — Nến OHLCV đa khung thời gian

Hỗ trợ 2 chế độ vận hành chính của hệ thống:

### Chế độ 1 — Realtime / Demo bot
*   Hàm `lay_du_lieu_nen(ten_san, symbol)`: Lấy dữ liệu 8 khung thời gian (1m, 3m, 5m, 15m, 30m, 1h, 4h, 1d) song song.
*   Cơ chế gộp nến: Chỉ fetch nến 1m, 5m, 30m, 4h thô từ sàn, sau đó dùng Polars gộp nến nội bộ (`gop_nen()`) thành các khung lớn hơn (1m -> 3m, 5m -> 15m, 30m -> 1h, 4h -> 1d) để giảm số lượt gọi API, tránh lỗi rate limit của sàn.

### Chế độ 2 — Tải lịch sử cho backtest
*   Hàm `tai_du_lieu_lich_su(symbol, start_str, end_str)`: Tải toàn bộ dữ liệu nến 1m từ Binance.
*   Tự động tính toán và thêm **30 ngày buffer** trước ngày bắt đầu thực tế. Điều này đảm bảo khi hệ thống chạy đến ngày đầu tiên của backtest, các chỉ báo kỹ thuật (như EMA50, ADX) đã tích lũy đủ dữ liệu lịch sử để tính toán giá trị chính xác (tránh lỗi null/warm-up period).

---

## 2. `lay_marketsnapshot.py` — Dữ liệu thị trường real-time (WebSocket)

Duy trì kết nối WebSocket thời gian thực siêu tốc đến sàn Binance Futures thông qua Singleton `KairosDataManager`.

*   Mỗi symbol đăng ký 5 luồng (streams) dữ liệu đồng thời:
    1.  `aggTrade`: Tích lũy khối lượng mua/bán chủ động, tính toán CVD (Cumulative Volume Delta).
    2.  `markPrice`: Lấy funding rate hiện tại và giá mark.
    3.  `depth5`: Độ sâu sổ lệnh (Order Book) top 5 dùng để tính imbalance.
    4.  `forceOrder`: Dữ liệu thanh lý (liquidation) bắt buộc, phân biệt rõ Long/Short.
    5.  `bookTicker`: Spread bid/ask tốt nhất.
*   `mo_theo_doi(cap_giao_dich)` / `dong_theo_doi(cap_giao_dich)`: Hàm tiện ích để bắt đầu hoặc kết thúc subscription cho một symbol.

---

## 3. `lay_thong_tin_tai_khoan.py` — Thông tin tài khoản sàn
*   `lay_so_du_kha_dung(ten_san, asset='USDT')`: Truy vấn tài khoản thật thông qua CCXT để lấy số dư khả dụng (Free balance) có thể dùng để ký quỹ mở lệnh.
*   `lay_vi_the_hien_tai(ten_san, symbol)`: Lấy thông tin vị thế thực tế đang có trên sàn (chiều LONG/SHORT, khối lượng, giá vào, PnL chưa thực hiện).

---

## 4. `lay_macro.py` — Dữ liệu vĩ mô & tâm lý thị trường
*   `lay_du_lieu_cam_xuc()`: Lấy chỉ số Fear & Greed Index từ API alternative.me. Trả về giá trị `50` (trung tính) nếu kết nối lỗi.
*   `lay_du_lieu_io(symbol, period, limit)`: Lấy lịch sử Open Interest (OI) từ Binance Futures REST API để phân tích sự tham gia của dòng tiền lớn.

---

## Cách dữ liệu chạy qua hệ thống

```text
SÀN GIAO DỊCH (REST API / CCXT)         SÀN GIAO DỊCH (WEBSOCKET)
        │                                       │
        ▼ (nến 1m thô)                          ▼ (CVD, Orderbook, Liquidation)
 ┌──────────────┐                     ┌─────────────────────────┐
 │ lay_ohlcv.py │                     │  lay_marketsnapshot.py  │
 └──────┬───────┘                     └────────────┬────────────┘
        │                                          │
        ▼ (Polars gộp nến)                         │ (cập nhật cache)
 ┌──────────────────────────────────────────────┐  │
 │ df_1m ... df_1d                              │  │
 └──────────────────────┬───────────────────────┘  │
                        │                          │
                        ▼                          ▼
                   ┌──────────────────────────────────┐
                   │ CHIẾN LƯỢC / ENGINE VÀO - RA LỆNH│
                   └──────────────────────────────────┘
```

---

## Thêm nguồn dữ liệu mới — Cách hook vào chiến lược

### Bước 1 — Tạo hàm lấy dữ liệu trong `lay_du_lieu/`
Ví dụ tạo file `lay_du_lieu/lay_kinh_te.py` lấy thông tin tin tức kinh tế:
```python
def lay_tin_tuc_moi():
    # Gọi API lấy tin tức vĩ mô
    return {"interest_rate_decision": "raise"}
```

### Bước 2 — Gọi trong `chuc_nang/chay_realtime.py` và đóng gói vào `MarketSnapshot`
Sửa file `chay_realtime.py` để bổ sung dữ liệu tin tức vào snapshot gửi sang chiến lược:
```python
from lay_du_lieu.lay_kinh_te import lay_tin_tuc_moi

# Trong luồng quét thị trường:
MarketSnapshot = mo_theo_doi(symbol)
MarketSnapshot['economic_news'] = lay_tin_tuc_moi()
```

### Bước 3 — Dùng trong hàm `phan_tich_*` của chiến lược
Sửa file chiến lược trong `logic_bar_to_bar/` để đọc tin tức:
```python
def phan_tich_entry(df, snapshot):
    news = snapshot.get('economic_news', {})
    if news.get("interest_rate_decision") == "raise":
        # Hạ bớt điểm mua hoặc đứng ngoài
        return 0
```

---

<a name="chien_luoc"></a>

## Engine Chiến Lược & Chỉ Báo Kỹ Thuật

Trọng tâm xử lý logic của dự án. Hệ thống hỗ trợ Engine **Bar-to-bar** (xử lý từng nến trong thời gian thực) để quét tín hiệu và thực hiện backtest.

---

## Cấu Trúc Thư Mục

```text
logic_bar_to_bar/                      # Engine dạng từng nến (Live / Demo / Bar-to-bar Backtest)
├── chien_luoc/                        # 5 chiến lược giao dịch cốt lõi
│   ├── chien_luoc_breakout.py
│   ├── chien_luoc_mean_reversion.py
│   ├── chien_luoc_scalping.py
│   ├── chien_luoc_squeeze.py
│   └── chien_luoc_theo_trend_following.py
├── phan_tich_ky_thuat/                # Bộ chỉ báo kỹ thuật theo thời gian thực (Polars)
│   ├── bien_dong.py
│   ├── cau_truc_gia.py
│   ├── chu_ky.py
│   ├── dong_luong_dao_chieu.py
│   ├── khoi_luong.py
│   ├── vi_the.py
│   └── xu_huong.py
├── quan_ly_chien_luoc.py              # Điều phối & định tuyến chiến lược
├── chien_luoc_don_bay.py              # Đòn bẩy động theo ATR
├── stoploss_takeprofit.py             # SL/TP động theo ATR
└── chien_luoc_trang_thai_thi_truong.py # Bộ lọc trạng thái thị trường (thời gian + số lượng dữ liệu)
```

---

## Kiến Trúc Hai Lớp

*   **Lớp Chiến lược riêng biệt**: Mỗi chiến lược (Breakout, Trend following,...) chỉ chứa logic toán học hoặc rule kích hoạt của chiến lược đó. Các hàm nhận vào dữ liệu nến đa khung thời gian và trả về điểm số/tín hiệu thô.
*   **Lớp Điều phối chung (`quan_ly_chien_luoc.py`)**: Nhận tín hiệu từ tất cả chiến lược, tổng hợp điểm số và đưa ra quyết định mở/đóng lệnh cuối cùng cho danh mục đầu tư.

---

## Nguyên Lý Điều Phối (`quan_ly_chien_luoc.py`)

### Dòng Chạy Bar-to-Bar
1.  Quét và tính toán các chỉ báo kỹ thuật trên 8 khung thời gian cho symbol.
2.  Kiểm tra điều kiện cấm giao dịch thông qua `chien_luoc_trang_thai_thi_truong.py`.
3.  Chạy kiểm tra song song 5 chiến lược cốt lõi theo tuần tự ưu tiên:
    1.  `Breakout` (`chien_luoc_breakout.py`)
    2.  `Mean Reversion` (`chien_luoc_mean_reversion.py`)
    3.  `Scalping` (`chien_luoc_scalping.py`)
    4.  `Squeeze` (`chien_luoc_squeeze.py`)
    5.  `Trend Following` (`chien_luoc_theo_trend_following.py`)
4.  Nếu chiến lược nào trả về tín hiệu giao dịch hợp lệ với điểm số tuyệt đối `abs(diem) >= 20` (ngưỡng xác nhận đa khung), chiến lược đó sẽ ngay lập tức được chọn để mở vị thế. Nếu không có chiến lược nào đạt yêu cầu, hệ thống đứng ngoài.

---

## Bộ Lọc Trạng Tai Thị Trường (`chien_luoc_trang_thai_thi_truong.py`)
Ngăn chặn bot mở lệnh trong những khoảng thời gian rủi ro cao (như ngày cuối tuần đối với thị trường truyền thống, giờ spread giãn mạnh lúc 22:00 UTC của phiên Mỹ, hoặc khoảng thời gian biến động mạnh ±15 phút quanh giờ settlement Funding Rate của sàn).

---

## Năm Chiến Lược Cốt Lõi

### 1. Breakout (`chien_luoc_breakout.py`)
*   **Nguyên lý**: Mua khi giá phá vỡ lên trên đỉnh cao nhất của N nến (Donchian Upper), bán khi giá thủng đáy thấp nhất N nến.
*   **Yêu cầu xác nhận**: Vol tăng đột biến (Volume Surge > 1.5) để loại bỏ breakout giả.

### 2. Mean Reversion (`chien_luoc_mean_reversion.py`)
*   **Nguyên lý**: Giao dịch ngược xu hướng khi giá đi quá xa khỏi dải giá trị trung bình (lệch ngoài Bollinger Bands 2.5 std hoặc RSI rơi vào vùng cực đoan < 15 hoặc > 85).
*   **Điểm ra**: Đóng vị thế khi giá hồi quy về đường trung vị (Mid band / SMA50).

### 3. Scalping (`chien_luoc_scalping.py`)
*   **Nguyên lý**: Ăn chênh lệch nhỏ trong khung thời gian cực ngắn (1m - 3m). Mua bán liên tục dựa trên vùng hỗ trợ/kháng cự tĩnh kết hợp với dao động Stoch/RSI trong phiên Á.

### 4. Squeeze (`chien_luoc_squeeze.py`)
*   **Nguyên lý**: Giao dịch dựa trên sự bứt phá của biên độ nén. Kích hoạt khi Bollinger Bands co lại nằm hoàn toàn bên trong dải Keltner Channel.
*   **Tín hiệu**: Mở vị thế theo hướng phá vỡ ngay khi dải băng mở rộng trở lại.

### 5. Trend Following (`chien_luoc_theo_trend_following.py`)
*   **Nguyên lý**: Giao dịch thuận xu hướng chính trên khung lớn.
*   **Bộ lọc**: Giá nằm trên EMA50 (LONG) hoặc dưới EMA50 (SHORT). Xác nhận hướng đi bởi độ dốc EMA và sức mạnh xu hướng ADX > 25.

---

## Quản Trị Rủi Ro Động

### Stop Loss & Take Profit (`stoploss_takeprofit.py`)
Hệ thống không sử dụng stop-loss hay take-profit cố định (hard-fixed percent) vì hành vi này sẽ giết chết tài khoản trong thị trường biến động mạnh hoặc bóp nghẹt lợi nhuận trong thị trường đi ngang. 
*   **Cơ chế**: Tính khoảng cách SL/TP động theo biên độ ATR (Average True Range) của khung 15m.
*   `sl_price = entry_price - ATR * base_sl` (LONG).
*   `tp_price = entry_price + ATR * base_sl * rr` (LONG).
*   Ngưỡng an toàn dự phòng: Đọc `cat_lo_percent` và `chot_loi_percent` từ cấu hình làm rào chắn cuối cùng (ROE Hard Cap).

### Đòn Bẩy Động (`chien_luoc_don_bay.py`)
*   **Cơ chế**: Tự động hạ đòn bẩy khi thị trường biến động cực đại (ATR tăng vọt) để bảo toàn vốn, và nâng đòn bẩy lên mức tối đa khi thị trường nén chặt đi ngang (ATR giảm mạnh) để tận dụng tối ưu hiệu suất vốn.

---

## Bộ Chỉ Báo Kỹ Thuật (`phan_tich_ky_thuat/`)

Mỗi chỉ báo tồn tại dưới phiên bản **bar-to-bar** (Polars, trả về dict từ nến cuối cùng, dùng cho realtime và backtest).

### `bien_dong.py` — Biến Động

| Chỉ báo | Bar-to-bar | Mô tả ngắn |
| --- | :---: | --- |
| ATR | ✅ | Biến động trung bình, nền tảng SL/TP/đòn bẩy |
| Bollinger Bands + Squeeze | ✅ | Phát hiện nén và kiệt sức |
| Keltner Channel | ✅ | Dải ATR-based, kết hợp với BB để xác định Squeeze |
| Donchian Channel | ✅ | Đỉnh/đáy N nến, dùng cho breakout |
| Historical Volatility | ✅ | HV annualized từ log-return, đo biến động thực tế |
| Chaikin Volatility | ✅ | ROC của EMA(H-L), phát hiện biến động đang mở rộng |
| ATR Bands | ✅ | SMA ± ATR×multiplier, rộng hơn Keltner |

### `xu_huong.py` — Xu Hướng

| Chỉ báo | Bar-to-bar | Mô tả ngắn |
| --- | :---: | --- |
| EMA Trend | ✅ | EMA so giá, xác định hướng xu hướng |
| SMA | ✅ | Đường trung bình đơn giản, ngưỡng xa/gần |
| ADX | ✅ | Độ mạnh xu hướng, ADX > 25 = đang có trend |
| Ichimoku | ✅ | 5 đường xác định xu hướng, hỗ trợ/kháng cự |
| SuperTrend | ✅ | ATR-based, đổi chiều khi giá cắt đường |
| MACD | ✅ | EMA(fast)-EMA(slow), phát hiện crossover và momentum |
| Parabolic SAR | ✅ | Điểm dừng-đảo chiều, theo sát xu hướng |
| Aroon | ✅ | Số nến từ đỉnh/đáy gần nhất, phát hiện trend mới |
| Vortex Indicator | ✅ | VI+ vs VI−, xác định hướng và lực xu hướng |

### `dong_luong_dao_chieu.py` — Động Lượng & Đảo Chiều

| Chỉ báo | Bar-to-bar | Mô tả ngắn |
| --- | :---: | --- |
| RSI | ✅ | Sức mạnh mua/bán 14 nến, quá mua/quá bán |
| Stochastic %K/%D | ✅ | Vị trí giá trong khoảng N nến |
| CCI | ✅ | Lệch chuẩn so trung bình, điểm đảo chiều cực đoan |
| Williams %R | ✅ | Vị trí close so với range N nến |
| Rate of Change (ROC) | ✅ | Tốc độ thay đổi giá, tăng/giảm tốc momentum |
| Money Flow Index (MFI) | ✅ | RSI nhân volume, tích hợp cả dòng tiền |
| Awesome Oscillator | ✅ | SMA(midprice,5) − SMA(midprice,34) |
| True Strength Index (TSI) | ✅ | Momentum double-smoothed, bộ lọc nhiễu tốt |
| Ultimate Oscillator | ✅ | Kết hợp 3 chu kỳ, giảm tín hiệu giả |

### `khoi_luong.py` — Khối Lượng

| Chỉ báo | Bar-to-bar | Mô tả ngắn |
| --- | :---: | --- |
| Volume MA | ✅ | So sánh volume hiện tại với trung bình N nến |
| Volume MA Dual (Fast/Slow) | ✅ | Phát hiện surge và dry-up theo hai đường MA |
| OBV | ✅ | Tích lũy volume theo chiều giá, xác nhận trend |
| VWAP | ✅ | Giá trung bình theo volume, tham chiếu smart money |
| Volume Profile | ✅ | POC / VAH / VAL — vùng giá có volume cao nhất |
| Chaikin Money Flow (CMF) | ✅ | Tỷ lệ volume mua/bán trong N nến |
| A/D Line | ✅ | Tích lũy/phân phối, phát hiện phân kỳ vs giá |
| MFI Volume | ✅ | MFI dạng volume-layer riêng |
| Ease of Movement (EOM) | ✅ | Di chuyển giá tương đối so với volume |

### `cau_truc_gia.py` — Cấu Trúc Giá

| Chỉ báo | Bar-to-bar | Mô tả ngắn |
| --- | :---: | --- |
| Breakout | ✅ | Phá đỉnh/đáy N nến |
| ZigZag | ✅ | Đỉnh/đáy cấu trúc theo biên độ tối thiểu |
| Fractals | ✅ | Đỉnh/đáy 5 nến kiểu Bill Williams |
| Pivot Points | ✅ | Swing H/L, phát hiện sweep và breakout cấu trúc |
| Fair Value Gap (FVG) | ✅ | Khoảng trống giá chưa lấp, vùng giá muốn quay lại |
| Heikin Ashi | ✅ | Nến làm mượt, nhìn rõ trend và doji |
| Market Structure (BOS/CHoCH) | ✅ | Phá cấu trúc (tiếp tục) vs đổi ký tự (đảo chiều) |
| Order Blocks | ✅ | Vùng nến nguồn gốc của impulse move lớn |
| Support / Resistance | ✅ | Cluster mức giá được test nhiều lần |

### `vi_the.py` — Vị Thế & Tâm Lý

| Chỉ báo | Bar-to-bar | Mô tả ngắn |
| --- | :---: | --- |
| CVD | ✅ (WebSocket) | Chênh lệch tích lũy mua/bán |
| Funding Rate | ✅ (WebSocket) | Lãi suất định kỳ long/short |
| Order Book Imbalance | ✅ (WebSocket) | Tỷ lệ bid/ask gần nhất |
| Liquidation Data | ✅ (WebSocket) | Quét thanh khoản bắt buộc |

### `chu_ky.py` — Chu Kỳ & Phiên

| Chỉ báo | Bar-to-bar | Mô tả ngắn |
| --- | :---: | --- |
| Phiên giao dịch | ✅ (lọc ngày/giờ) | Châu Á / London / New York / Overlap |
| Session Range | – | Đỉnh/đáy từng phiên, breakout phiên |
| Giờ spread cao | ✅ | Lọc khung giờ rủi ro (ví dụ 22h UTC) |

---


## Cảnh Báo Look-Ahead — Những Lỗi Phổ Biến

> [!CAUTION]
> **Look-ahead bias** là lỗi nguy hiểm nhất trong backtest đa khung thời gian: chiến lược được tính với dữ liệu nó không thể có ở thời điểm đó, khiến kết quả backtest giả tốt không phản ánh thực tế.

### Lỗi 1 — Dùng `merge_asof(direction='forward')`

```python
# SAI — lấy dữ liệu từ tương lai
pd.merge_asof(df_1m, df_htf, on='timestamp', direction='forward')

# ĐÚNG — chỉ lấy dữ liệu đã xảy ra trước hoặc tại thời điểm đó
pd.merge_asof(df_1m, df_htf, on='timestamp', direction='backward')
```

### Lỗi 2 — Quên Shift Index Sau Resample

```python
# SAI — nến 15m đóng lúc 14:30 được dùng ngay tại 14:30
htf_close = df['close'].resample('15min').last()

# ĐÚNG — nến 15m đóng lúc 14:30 chỉ được dùng từ 14:45
htf_close.index += pd.to_timedelta('15min')
```

### Lỗi 3 — Dùng `.last()` Thay Vì Tách Closed/Live

```python
# SAI — dùng giá đóng của nến 15m đang hình thành (chưa đóng)
htf_live = df.groupby(pd.Grouper(freq='15min'))['close'].last()

# ĐÚNG — tách riêng: closed candle dùng resample+shift, live dùng cumulative
htf_closed = df['close'].resample('15min').last()
htf_closed.index += pd.to_timedelta('15min') # anti-lookahead
live_close = df['close'] # close 1m hiện tại
```

### Lỗi 4 — Dùng `join` Thay `join_asof` Khi Timestamp Không Khớp Chính Xác

```python
# SAI — nếu timestamp HTF không khớp 1:1 với 1m, kết quả là NaN
df_1m.merge(df_htf, on='timestamp', how='left')

# ĐÚNG — lấy giá trị HTF gần nhất TRƯỚC hoặc bằng timestamp 1m
pd.merge_asof(df_1m, df_htf, on='timestamp', direction='backward')
```

---

## Hệ Thống Chấm Điểm Đa Khung Thời Gian

Mỗi chiến lược không chỉ phân tích một khung mà phân tích toàn bộ 8 khung đồng thời, sau đó cộng điểm có trọng số:

| Khung thời gian | Trọng số điển hình |
| --- | --- |
| 1 ngày (1d) | 10 — tín hiệu vĩ mô |
| 4 giờ (4h) | 8 — xu hướng trung hạn |
| 1 giờ (1h) | 5 — xác nhận chiến thuật |
| 15 phút (15m) | 3 — điểm vào |
| 30 phút (30m) | 2–4 |
| 5 phút (5m) | 2 |
| 3 phút (3m) | 1–3 |
| 1 phút (1m) | 1–4 |

Trọng số thay đổi theo từng chiến lược. Tín hiệu chỉ được phát ra khi điểm số tích lũy vượt ngưỡng xác nhận tối thiểu của từng chiến lược, giúp lọc bỏ các tín hiệu yếu từ một khung duy nhất.

### Ensemble Scoring — Hệ thống chấm điểm có trọng số

Thay vì dùng một quy tắc (rule) đơn lẻ, KAIROS kết hợp nhiều góc nhìn phân tích độc lập để đưa ra điểm số tổng hợp:

**Các nhóm phân tích chính:**

| Module | Chức năng phân tích | Trọng số điển hình |
|---|---|---|
| `xu_huong.py` | EMA alignment, ADX strength, trend structure | Cao (khung 1h) |
| `cau_truc_gia.py` | Breakout, Fractal, FVG, Support/Resistance | Cao (khung 4h) |
| `khoi_luong.py` | Volume surge, OBV, VWAP deviation | Trung bình |
| `dong_luong_dao_chieu.py` | RSI divergence, MACD, momentum exhaustion | Trung bình |
| `bien_dong.py` | ATR regime, Bollinger squeeze, Keltner | Thấp–Trung bình |
| `vi_the.py` | CVD, Funding Rate, Order Book Imbalance | Xác nhận |
| `chu_ky.py` | Session classification, funding hour filter | Lọc |

**Công thức tổng hợp tín hiệu (Formula):**
```text
Total Score = Σ (Feature_Score_i × Weight_i × Timeframe_Multiplier)

Tín hiệu phát ra:
 - BUY nếu Total Score ≥ Threshold
 - SELL nếu Total Score ≤ -Threshold
 - HOLD (Không làm gì) nếu nằm trong khoảng [-Threshold, Threshold]
```

> [!NOTE]
> Các chiến lược sử dụng các bộ chỉ báo có trọng số cố định được tối ưu hóa trước để thích nghi với các điều kiện thị trường khác nhau. Mỗi chiến lược có cơ chế chấm điểm và trọng số riêng cho từng nhóm chỉ báo kỹ thuật nhằm tối ưu hóa hiệu suất giao dịch.

---

## Kiểm Thử & Unit Tests

> [!IMPORTANT]
> **Lưu ý cấu trúc dự án**: File test `test_chien_luoc.py` được đề xuất phục vụ kiểm thử logic chiến lược thô trên 3000 nến tổng hợp. Hiện tại mã nguồn kiểm thử chưa được đưa vào nhánh chính.
> 
> Hướng phát triển kiểm thử chuẩn công nghiệp bằng PyTest:
> 1. Tạo thư mục `tests/` tại thư mục gốc dự án.
> 2. Viết file `tests/test_chien_luoc.py` kiểm định đầu ra của các chiến lược:
>    - `signal` phải thuộc tập `{-1, 0, 1}`.
>    - Phải chứa cột `entry_signal`.
>    - Các chỉ báo không phát sinh lỗi giá trị `NaN` hoặc chia cho 0.
> 3. Lệnh chạy kiểm thử: `pytest tests/ -v`

---

## Mở Rộng & Hướng Dẫn Thiết Kế Chiến Lược Chuyên Sâu

Hệ thống Kairos-v1 sử dụng Kiến trúc **Bar-to-bar** để quét tín hiệu thời gian thực và thực hiện backtest.

---

### Hướng Dẫn Phát Triển Chiến Lược Bar-to-Bar

Lớp Bar-to-Bar xử lý dữ liệu theo thời gian thực (realtime) trên nến cuối cùng vừa đóng cửa của mỗi khung thời gian.

#### Cấu Trúc Khai Báo Đầu Vào/Đầu Ra (Data Contract):
Mỗi file chiến lược phải xuất bản tối thiểu hai hàm chính:
1.  **`phan_tich_<ten_chien_luoc>()`**: Kiểm tra điều kiện và tính điểm để mở vị thế mới.
    - **Inputs**: `symbol`, 8 khung Polars DataFrame, và `MarketSnapshot` (dữ liệu WebSocket thời gian thực).
    - **Outputs**: `(tin_hieu, diem, ly_do)` trong đó `tin_hieu` thuộc `["buy", "sell", None]`, `diem` là float, và `ly_do` là chuỗi mô tả.
2.  **`thoat_<ten_chien_luoc>()`**: Kiểm tra điều kiện để đóng sớm vị thế đang mở (trước khi chạm SL/TP cứng).
    - **Inputs**: 8 khung Polars DataFrame, và `MarketSnapshot`.
    - **Outputs**: `(huong_thoat, diem, ly_do)` trong đó `huong_thoat` thuộc `["buy", "sell", None]`. Ví dụ: nếu đang nắm giữ vị thế LONG, hàm cần trả về `("sell", -20, "...")` để kích hoạt đóng lệnh.

#### Code Mẫu Chuẩn Hóa và Giải Thích (Bar-to-Bar Template):
```python
import polars as pl
from joblib import Parallel, delayed
from utils.log import logger
from chien_luoc.logic_bar_to_bar.phan_tich_ky_thuat.dong_luong_dao_chieu import pt_rsi
from chien_luoc.logic_bar_to_bar.phan_tich_ky_thuat.bien_dong import pt_bollinger_squeeze

def tinh_diem_rieng_le_khung(df, timeframe_name, weight):
    """Hàm phụ tính toán chỉ báo cho từng khung thời gian riêng biệt sử dụng Polars."""
    if df is None or df.height < 30:
        return 0, ""
        
    try:
        # Tính toán chỉ báo (trả về dict kết quả của nến cuối cùng)
        rsi_data = pt_rsi(df) # ví dụ: {"rsi_val": 25.4, "trang_thai": "QUÁ_BÁN"}
        bb_data = pt_bollinger_squeeze(df) # ví dụ: {"lower_band": 62500, "upper_band": 63500}
        
        last_n = df.tail(1).to_dicts()[0]
        close_now = last_n["close"]
        
        diem = 0
        ly_do = ""
        
        # Logic tính điểm Long
        if rsi_data["rsi_val"] < 30 and close_now < bb_data["lower_band"]:
            diem += 1 * weight
            ly_do = f"{timeframe_name}: RSI={rsi_data['rsi_val']:.1f} Quá bán + Thủng dải BB"
            
        # Logic tính điểm Short
        elif rsi_data["rsi_val"] > 70 and close_now > bb_data["upper_band"]:
            diem -= 1 * weight
            ly_do = f"{timeframe_name}: RSI={rsi_data['rsi_val']:.1f} Quá mua + Vượt dải BB"
            
        return diem, ly_do
    except Exception as e:
        logger.error(f"Lỗi tính toán chỉ báo khung {timeframe_name}: {e}")
        return 0, ""

def phan_tich_custom(symbol, df_1m, df_3m, df_5m, df_15m, df_30m, df_1h, df_4h, df_1d, MarketSnapshot=None):
    # A. Cấu hình đa khung thời gian để chạy song song
    cau_hinh_khung = [
        (df_1h, "1H", 5),   # Khung thời gian 1H có trọng số (weight) là 5
        (df_15m, "15M", 3), # Khung thời gian 15M có trọng số là 3
    ]
    
    # B. Sử dụng joblib.Parallel với cơ chế Multi-threading để tăng tốc độ tính toán
    ket_qua = Parallel(n_jobs=-1, prefer="threads")(
        delayed(tinh_diem_rieng_le_khung)(df, tf, weight) 
        for df, tf, weight in cau_hinh_khung
    )
    
    tong_diem = sum(res[0] for res in ket_qua)
    ly_do = [res[1] for res in ket_qua if res[1] != ""]
    
    # C. Tích hợp dữ liệu WebSocket thời gian thực (OrderFlow) làm bộ lọc xác nhận
    if MarketSnapshot:
        delta = MarketSnapshot.get("delta", 0) # Tích lũy mua/bán chủ động (CVD)
        imbalance = MarketSnapshot.get("imbalance", 0) # Mất cân bằng sổ lệnh
        
        if tong_diem >= 5 and delta > 500:
            tong_diem += 2 # Cộng điểm nếu OrderFlow đồng thuận mua
            ly_do.append("Flow xác nhận MUA")
        elif tong_diem <= -5 and delta < -500:
            tong_diem -= 2 # Trừ điểm nếu OrderFlow đồng thuận bán
            ly_do.append("Flow xác nhận BÁN")
            
    THRESHOLD_ENTRY = 8
    tin_hieu = "buy" if tong_diem >= THRESHOLD_ENTRY else "sell" if tong_diem <= -THRESHOLD_ENTRY else None
    
    return tin_hieu, round(tong_diem, 2), " | ".join(ly_do)

def thoat_custom(df_1m, df_3m, df_5m, df_15m, df_30m, df_1h, df_4h, df_1d, MarketSnapshot=None):
    """Tính toán tín hiệu thoát vị thế sớm."""
    if df_1h is None or df_1h.height < 20:
        return None, 0, "Thiếu dữ liệu"
        
    rsi_1h = pt_rsi(df_1h)
    
    # Cơ chế Hysteresis: Thoát vị thế LONG khi RSI hồi phục về vùng trung vị
    if 45 <= rsi_1h["rsi_val"] <= 55:
        return "sell", -20, f"RSI 1H hồi quy về điểm trung tính ({rsi_1h['rsi_val']:.1f})"
        
    return None, 0, "Giữ vị thế"
```

---

### Quy Trình 3 Bước Tích Hợp Chiến Lược Mới Vào Bot

Khi bạn phát triển một chiến lược mới hoàn toàn (ví dụ mang tên: `MACD_Trend`), hãy tuân thủ quy trình tích hợp 3 bước sau để đưa vào vận hành:

#### Bước 1: Tạo file chiến lược tương ứng trong thư mục `logic_bar_to_bar`
- Tạo file mới tại: `logic_bar_to_bar/chien_luoc/chien_luoc_macd_trend.py`.
- Định nghĩa 2 hàm `phan_tich_macd_trend()` và `thoat_macd_trend()` theo định dạng của mục hướng dẫn.

#### Bước 2: Đăng ký router thực thi thời gian thực
- Mở file [logic_bar_to_bar/quan_ly_chien_luoc.py](file:///d:/The%20V/Kairos-v1/logic_bar_to_bar/quan_ly_chien_luoc.py).
- Import các hàm vào/ra lệnh:
  ```python
  from logic_bar_to_bar.chien_luoc.chien_luoc_macd_trend import phan_tich_macd_trend, thoat_macd_trend
  ```
- Định cấu hình liên kết trong hàm `chien_luoc_vao_lenh()`:
  ```python
  elif chien_luoc == "MACD_Trend":
      tin_hieu, diem, ly_do = phan_tich_macd_trend(symbol, df_1m, df_3m, df_5m, df_15m, df_30m, df_1h, df_4h, df_1d, MarketSnapshot)
  ```
- Định cấu hình liên kết trong hàm `chien_luoc_thoat_lenh()`:
  ```python
  elif chien_luoc == "MACD_Trend":
      check_thoat, ly_do_thoat = thoat_macd_trend(df_1m, df_3m, df_5m, df_15m, df_30m, df_1h, df_4h, df_1d, MarketSnapshot)
  ```

#### Bước 3: Kiểm tra liên kết chiến lược
- Sử dụng bộ thông số cấu hình vào hệ thống và chạy Bar-to-Bar Backtest:
  ```powershell
  python chuc_nang/backtest_donluong.py
  ```

---

<a name="thuc_thi_lenh"></a>

## Thực Thi Lệnh & Quản Lý Vị Thế

Module chịu trách nhiệm giao tiếp với sàn giao dịch, đặt lệnh, quản lý rủi ro và quản lý trạng thái vị thế thời gian thực.

---

## Sơ đồ luồng dữ liệu

```text
              Chiến lược (realtime / demo)
                          │
                          ▼
 ┌──────────────────────────────────────────────────┐
 │ bo_may_thuc_thi.py (Singleton Pool kết nối CCXT) │
 └──────┬─────────────────┬───────────────────┬─────┘
        │                 │                   │
        ▼                 ▼                   ▼
┌──────────────┐   ┌───────────────┐   ┌──────────────────────┐
│  mo_lenh.py  │   │  dong_lenh.py │   │  mo_lenh_da_san.py   │
└──────┬───────┘   └──────┬────────┘   └──────────┬───────────┘
       │                  │                       │
       └──────────────────┼───────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────┐
│ quan_ly_lenh.py (State Persistence & GUI Signal) │
└──────┬──────────────────────────────────────┬────┘
       │                                      │
       ▼                                      ▼
┌────────────────┐                 ┌──────────────────────┐
│theo_doi_lenh.py│                 │ quan_ly_danh_muc.py  │
└────────────────┘                 └──────────────────────┘
```

---

## Các file chính

### `bo_may_thuc_thi.py` – Connection Pool (Singleton)
Quản lý các kết nối CCXT đến Binance, OKX, Bybit. Được khởi tạo một lần duy nhất (`quan_ly_san`) khi hệ thống start để tránh rate limit do tạo lại kết nối liên tục.

### `mo_lenh.py` & `dong_lenh.py` – Giao dịch an toàn
*   `thuc_hien_mo_lenh()`: Tự động gọi `set_leverage` trước khi đặt lệnh. Hỗ trợ cả lệnh Limit và Market.
*   `thuc_hien_dong_lenh()`: Luôn truyền tham số `reduceOnly=True` để đảm bảo lệnh chỉ đóng vị thế hiện tại chứ không bao giờ mở ngược.
*   **Hedge Mode Handling**: Tự động chèn thêm `posSide='long'|'short'` khi giao dịch trên các sàn sử dụng chế độ Hedge Mode (như OKX, Bybit).

### `theo_doi_lenh.py` – Heartbeat vị thế
Sử dụng `fetch_positions()` so sánh với state cục bộ để phát hiện các vị thế bị đóng ngoài hệ thống (bị thanh lý hoặc người dùng đóng bằng tay trên app điện thoại).

### `quan_ly_lenh.py` – Quản lý trạng thái và đồng bộ
*   **Persistence**: Lưu các lệnh đang chạy vào `trang_thai_lenh_realtime.json` / `trang_thai_lenh_demo.json` để khôi phục trạng thái an toàn sau khi bot hoặc máy tính bị crash đột ngột.
*   **Thread-safe Signals**: Emit signal `ui_signals.data_changed` để thông báo cho giao diện PyQt6 cập nhật đồ thị và bảng biểu, tránh polling gây nghẽn CPU.

### `quan_ly_danh_muc.py` – Quản trị rủi ro danh mục
*   `kiem_tra_phan_bo_von()`: Giới hạn số lượng vị thế mở đồng thời.
*   `kiem_tra_rui_ro_danh_muc()`: Tính toán tổng thiệt hại tối đa nếu toàn bộ các lệnh đang mở chạm mức SL (giới hạn rủi ro danh mục tối đa 10%).

---

<a name="chuc_nang"></a>

## Kịch Bản Vận Hành (Runners)

Chứa 5 file thực thi độc lập tương ứng với 5 chế độ chạy của hệ thống, được gọi từ menu launcher chính (`main.py`).

```text
chuc_nang/
├── chay_realtime.py         # Live trading — đặt lệnh thật qua CCXT
├── chay_demo.py             # Paper trading — giả lập không đặt lệnh thật
├── backtest_donluong.py     # Backtest bar-to-bar đơn luồng (tuần tự)
└── backtest_daluong.py      # Backtest bar-to-bar đa luồng (ProcessPoolExecutor)
```

---

## 1. `chay_realtime.py` — Live Trading Bot
*   **Kiến trúc**: Chạy song song 2 Thread Daemon:
    -   `Thread-Scanner`: Quét thị trường mỗi 5s, tính toán tín hiệu đa khung và thực hiện mở vị thế thật qua CCXT.
    -   `Thread-Manager`: Quét các vị thế đang giữ mỗi 5s, kiểm tra điều kiện thoát (chạm SL/TP hoặc tín hiệu đảo chiều), thực hiện đóng vị thế và ghi reward vào `trading_memory.csv`.

## 2. `chay_demo.py` — Paper Trading (Giả Lập)
*   **Kiến trúc**: Giống hoàn toàn `chay_realtime.py` nhưng **không đặt lệnh thật**.
*   **Giả lập khớp lệnh**: Tự động cộng/trừ tỷ lệ trượt giá (`do_truot_gia`) và phí giao dịch ảo (`phi_giao_dich`) trực tiếp vào biến số dư giả lập `VON_AO` đọc từ `cau_hinh_ao`.

## 3. `backtest_donluong.py` — Backtest Bar-to-Bar Đơn Luồng
*   Chạy giả lập trên dữ liệu lịch sử tuần tự từng symbol. Phù hợp để debug chi tiết dòng chảy dữ liệu nến của chiến lược.
*   Tự động lưu lịch sử lệnh ra CSV tại `du_lieu/thong_tin_lenh/`.

## 4. `backtest_daluong.py` — Backtest Bar-to-Bar Đa Luồng
*   Sử dụng `ProcessPoolExecutor` phân phối mỗi symbol chạy trên một tiến trình CPU riêng biệt để tăng tốc độ xử lý.
*   Kết quả trả về qua `Manager().Queue()` và được merge, sắp xếp theo thời gian đóng lệnh (`time_close`) để tính toán Equity Curve tổng hợp chính xác của danh mục.



<a name="hien_thi"></a>

## Giao Diện Dashboard PyQt6

Bản đồ giao diện trực quan hóa dữ liệu hiệu năng cao được xây dựng trên nền tảng PyQt6, PyQt6-Charts và pyqtgraph.

---

## Triết lý Thiết kế & Hệ thống Theme "Pro Terminal"

Giao diện được thiết kế theo phong cách Bloomberg Terminal tối giản, sang trọng:
*   **CARD_BG (`#141414`)** và **BORDER_COLOR (`#2a2a2a`)** tạo chiều sâu tốt cho các Panel.
*   **ACCENT_COLOR (`#C8AA6E`)** là màu vàng đồng đặc trưng dùng cho tiêu đề panel.
*   **Draggable Card**: Các Panel được thiết kế dạng Dockable (`QDockWidget`). Người dùng có thể kéo thả thay đổi bố cục hoặc tách rời panel thành cửa sổ riêng biệt để sử dụng trên nhiều màn hình.

---

## Chi Tiết Các Phân Hệ Giao Diện

### 1. Phân hệ cho HR / Nhà tuyển dụng (Trực quan & UX)
*   **Daily PnL Calendar (`CalendarWidget`)**: Lưới hiển thị kết quả giao dịch theo ngày dạng GitHub Contribution Graph. Tự động nội suy tuyến tính màu từ tối sang xanh lục bảo sáng (lợi nhuận cao) hoặc đỏ thẫm (thua lỗ lớn).
*   **Drawdown Curves (`TotalAssetChart`)**: Đồ thị Equity Curve kết hợp Underwater Chart (đo sụt giảm vốn tức thời). Sử dụng thuật toán downsampling tự động để giữ đồ thị chạy mượt mà ở mức 60 FPS khi dữ liệu vượt quá 10,000 điểm.

### 2. Phân hệ cho Tech Lead / Quant Lead (Toán học & Chẩn đoán)
*   **Trade Scatter Plot (`TradeScatterWidget`)**: Trực quan hóa mối quan hệ giữa thời gian nắm giữ vị thế (Hold Duration - Trục X thang Logarit) và biên lợi nhuận thực tế (PnL - Trục Y). Giúp chẩn đoán lỗi gồng lỗ lâu - chốt lời vội (Prospect Theory).
*   **Session Heatmap (`DistributionChart`)**: Ma trận phân bố Winrate và PnL theo 24 giờ trong ngày và các ngày trong tuần. Phục vụ việc xác định và lọc bỏ các khung giờ thanh khoản mỏng hoặc nhiễu động.
*   **QPainter Candlestick Engine (`CoreCandlestickChart`)**: Biểu đồ nến chuyên nghiệp được vẽ thủ công bằng API cấp thấp `QPainter` để tối đa hóa tốc độ render. Hiển thị điểm mua (mũi tên xanh), bán (mũi tên tím), điểm đóng lệnh (chữ X cam) và vẽ đường nét đứt kết nối để chẩn đoán visual validation.

---

<a name="utils"></a>

## Tiện Ích & Helpers

Tập hợp các module helper dùng chung giúp chuẩn hóa dữ liệu và vận hành hệ thống.

---

## Chi tiết từng file

### `ham_tien_ich.py`
*   `chuyen_doi_symbol_chuan()`: Chuẩn hóa tên symbol về BASE/QUOTE (ví dụ: `BTC-USDT-SWAP` của OKX -> `BTC/USDT`).
*   `tinh_pnl()`: Tính toán chính xác PnL tuyệt đối ($) và tương đối (%) cho cả lệnh LONG và SHORT.
*   `gop_va_dong_bo_data()`: Hàm alignment đa khung thời gian sử dụng `merge_asof(backward)` để loại bỏ triệt để look-ahead bias.
*   `tao_run_id()`: Tạo run_id duy nhất từ timestamp + UUID ngắn.

### `log.py`
*   Tích hợp ghi log ra console và file `nhat_ky_hoat_dong.log` sử dụng Rich format.
*   **Đồng hồ ảo (`TimeContext`)**: Khi backtest, log sẽ ghi nhận thời điểm giả lập của nến đang chạy thay vì giờ máy tính thực tế, giúp quá trình debug logic chiến lược diễn ra rất trực quan.

### `doc_cau_hinh.py`
*   Nạp cấu hình từ 4 file YAML/JSON trong thư mục `config/` và trả về dict. Tự động trả về giá trị mặc định an toàn nếu file cấu hình bị lỗi cú pháp.

### `chuyen_doi_don_vi.py`
*   Chuyển đổi tiền tệ, làm tròn số lượng coin theo đúng độ phân giải (precision amount) mà sàn quy định để tránh lỗi đặt lệnh "Invalid quantity".

---

<a name="du_lieu"></a>

## Lưu Trữ Dữ Liệu

Thư mục lưu trữ toàn bộ trạng thái vận hành của hệ thống Kairos.

```text
du_lieu/
├── nhat_ky_hoat_dong.log          # File log hoạt động hệ thống
└── thong_tin_lenh/
    ├── trang_thai_lenh_demo.json  # Vị thế mở của chế độ Demo
    ├── trang_thai_lenh_realtime.json # Vị thế mở của chế độ Realtime
    ├── lich_su_lenh_demo.json     # Lịch sử vị thế đã đóng (Demo - JSONL)
    └── lich_su_lenh_realtime.json # Lịch sử vị thế đã đóng (Realtime - JSONL)
```

*   **JSON/JSONL & CSV**: Dành cho việc lưu trữ lịch sử lệnh và runtime state gọn nhẹ, đảm bảo tốc độ đọc ghi tức thời ở tần suất cao của bot.

---

<a name="thong_bao"></a>

## Hệ Thống Cảnh Báo & Alert

Dùng để thông báo trạng thái hoạt động của bot thời gian thực đến người dùng thông qua các kênh truyền thông phổ biến.

---

## Chi Tiết Các Kênh Cảnh Báo

### `gui_telegram.py`
*   Gửi thông báo mở/đóng lệnh thời gian thực đến ứng dụng Telegram.
*   **Cấu hình**: Nhập `TOKEN` và `CHAT_ID` trực tiếp vào file.
*   **Safety Gate**: Nếu các hằng số này chứa chuỗi mặc định `"TOKEN_TELEGRAM_CUA_BAN"`, hàm sẽ tự động bỏ qua để hệ thống vận hành bình thường không bị lỗi crash.

### `gui_email.py`
*   Gửi email cảnh báo chi tiết qua giao thức SMTP Gmail (sử dụng mã hóa TLS cổng 587).
*   **Cấu hình**: Nhập `EMAIL_GUI` và mật khẩu ứng dụng Gmail `MAT_KHAU_APP`. Tự động bỏ qua nếu phát hiện hằng số chứa giá trị mặc định `"your_app_password"`.

---
