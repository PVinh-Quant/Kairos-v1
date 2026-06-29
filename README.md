<div align="center">
 
<img width="124" height="124" alt="image" src="https://github.com/user-attachments/assets/bf450abd-f468-43fa-9750-464e3ef95651" />

# Kairos v1
### **End-to-End Data Analytics Pipeline for Financial Market Research**

[![Python](https://img.shields.io/badge/Python-3.12+-blue?style=for-the-badge&logo=python)](https://www.python.org/)
[![Domain](https://img.shields.io/badge/Domain-FinTech%20%2F%20Crypto-orange?style=for-the-badge)](https://www.binance.com/)
[![License](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)](https://opensource.org/licenses/MIT)

`Python` • `Pandas` • `Polars` • `ETL Pipeline` • `Time-Series` • `Feature Engineering`

</div>

-----

### Data Skills Highlights

* **End-to-End ETL Pipeline:** Thiết kế pipeline tự động thu thập dữ liệu OHLCV đa khung (1m–1d) từ sàn giao dịch (OKX, Binance, Bybit) sử dụng CCXT, chuẩn hóa timestamp và xử lý khoảng trống dữ liệu (missing candles).
* **Multi-Timeframe Feature Engineering (không look-ahead):** Trích xuất các chỉ báo kỹ thuật trên nhiều khung thời gian song song. Giải quyết triệt để bài toán time-alignment nghiêm ngặt để tránh rò rỉ dữ liệu (look-ahead bias).
* **High-Performance Processing:** Sử dụng Polars để tối ưu hóa hiệu năng tính toán chỉ báo kỹ thuật trên tập dữ liệu lịch sử lớn.
* **Statistical Backtesting Framework:** Thiết kế bộ kiểm thử lịch sử giả lập khớp lệnh từng nến (bar-to-bar backtest), hỗ trợ cả đơn luồng (để debug) và đa luồng (để tối ưu hiệu năng).
* **Interactive Analytics Dashboard (PyQt6):** Trực quan hóa kết quả giao dịch: Equity curve, Drawdown chart, Daily PnL calendar, Session heatmap, Trade scatter plot.

-----

## Minh họa Analytics Dashboard

<img width="1920" height="1080" alt="576968808-4883c4f4-e1ca-4e34-b806-220ae38faccc" src="https://github.com/user-attachments/assets/b19055d4-6000-4410-9c7f-6db71cbcead1" />

-----

## 5 chế độ vận hành — chọn qua CLI menu

Khởi động bằng `python main.py`, hệ thống sử dụng thư viện `rich` để hiển thị menu CLI tương tác trực quan:

```text
 ┌────────────────────────────────────────────────────────────────────────────────────────┐
 │                     ██╗  ██╗ █████╗ ██╗██████╗  ██████╗ ███████╗                       │
 │                     ██║ ██╔╝██╔══██╗██║██╔══██╗██╔═══██╗██╔════╝                       │
 │                     █████╔╝ ███████║██║██████╔╝██║   ██║███████╗                       │
 │                     ██╔═██╗ ██╔══██║██║██╔══██╗██║   ██║╚════██║                       │
 │                     ██║  ██╗██║  ██║██║██║  ██║╚██████╔╝███████║                       │
 │                     ╚═╝  ╚═╝╚═╝  ╚═╝╚═╝╚═╝  ╚═╝ ╚═════╝ ╚══════╝                       │
 │                                  Analytics System v1                                   │
 ├──────────────────────────────────────────────┬─────────────────────────────────────────┤
 │  Menu                                        │  Config hiện tại                        │
 │  ──────────────────────────────────────────  │  ────────────────────────────────────── │
 │  LIVE TRADING                                │  Symbols:   BTC/USDT                    │
 │  [1] Giao dịch Realtime   (Thật · CCXT)      │  Đòn bẩy:   7x                          │
 │  [2] Demo / Paper Trading (Không rủi ro)     │  Backtest:  2026-01-01  →  2026-06-01   │
 │                                              │  Vốn:       10,000 USDT                 │
 │  BACKTESTING                                 ├─────────────────────────────────────────┤
 │  [3] Backtest Đơn luồng   (Bar-to-bar 1-CPU) │  Tác giả                                │
 │  [4] Backtest Đa luồng    (Song song đa-CPU) │  P. Vinh - Quant Research & Dev         │
 │                                              │                                         │
 │  ANALYTICS                                   │  "Romain Rolland: 'There is only one    │
 │  [5] Dashboard Analytics  (PyQt6 GUI)        │   heroism in the world: to see the      │
 │                                              │   world as it is, and to love it.'"     │
 │                                              │  ────────────────────────────────────── │
 │  [0] Thoát                                   │  Kairos v1 · 2026                       │
 └──────────────────────────────────────────────┴─────────────────────────────────────────┘
```

> [!NOTE]
> Hệ thống áp dụng cơ chế **Lazy Import**: Chỉ nạp các thư viện nặng (như PyQt6, CCXT) khi chức năng tương ứng được chọn giúp menu khởi chạy ngay lập tức (< 100ms).

-----

## Cấu trúc thư mục

```text
Kairos-v1/
├── main.py                              # Entry point – menu CLI điều hướng
├── requirements.txt                     # Các thư viện phụ thuộc của dự án
├── README.md                            # Tổng quan dự án
├── tai_lieu_chi_tiet.md                 # Tài liệu hướng dẫn kỹ thuật chuyên sâu
│
├── config/                              # Cấu hình hệ thống
│   ├── cau_hinh_giao_dich.yaml          # Tài sản, đòn bẩy, các tham số quản lý vốn
│   ├── cau_hinh_ao_config.json          # Tham số môi trường backtest/paper trading
│   ├── tai_khoan_api.json               # API credentials các sàn (đã gitignore)
│   ├── tai_khoan_api.json.example       # Template mẫu cấu hình API keys
│   └── thong_tin_san.yaml               # Phí và giới hạn lot/tick size của sàn
│
├── lay_du_lieu/                         # ETL Layer – Thu thập & Chuẩn hóa dữ liệu
│   ├── lay_ohlcv.py                     # Lấy nến lịch sử và gộp nến đa khung thời gian
│   ├── lay_thong_tin_tai_khoan.py       # Lấy số dư ví, vị thế mở từ sàn qua CCXT
│   ├── lay_marketsnapshot.py            # WebSocket: Dữ liệu lệnh (CVD, orderbook - Stub)
│   └── lay_macro.py                     # Macro: Fear & Greed Index, Open Interest (Stub)
│
├── logic_bar_to_bar/                    # Feature Engineering & Signal Layer
│   ├── phan_tich_ky_thuat/              # Các chỉ báo biến động, xu hướng, khối lượng
│   ├── chien_luoc/                      # 5 chiến lược giao dịch bar-to-bar chấm điểm độc lập
│   ├── quan_ly_chien_luoc.py            # Tổng hợp điểm tín hiệu và định tuyến lệnh
│   ├── chien_luoc_don_bay.py            # Bộ tính toán đòn bẩy động đa nhân tố
│   ├── stoploss_takeprofit.py           # Tính SL/TP động theo ATR
│   └── chien_luoc_trang_thai_thi_truong.py # Bộ lọc khung giờ và trạng thái thị trường
│
├── chuc_nang/                           # Vận hành hệ thống (Runners)
│   ├── chay_realtime.py                 # Bot giao dịch tài khoản thật (Live Trading)
│   ├── chay_demo.py                     # Chạy bot giả lập trên thị trường thực (Paper Trading)
│   ├── backtest_donluong.py             # Backtest dữ liệu lịch sử đơn luồng (debug)
│   └── backtest_daluong.py              # Backtest dữ liệu lịch sử đa luồng (song song)
│
├── thuc_thi_lenh/                       # Order Execution Layer
│   ├── bo_may_thuc_thi.py               # Quản lý kết nối sàn (CCXT wrapper)
│   ├── chon_san_giao_dich.py            # Xác định sàn giao dịch chính
│   ├── mo_lenh.py / dong_lenh.py        # Gửi lệnh Market/Limit mở/đóng vị thế
│   ├── mo_lenh_da_san.py                # Xử lý mở lệnh song song trên nhiều sàn
│   ├── quan_ly_danh_muc.py              # Kiểm tra quy mô và phân bổ vốn danh mục
│   ├── quan_ly_lenh.py                  # Quản lý trạng thái các lệnh đang mở
│   ├── theo_doi_lenh.py                 # Giám sát vị thế và kiểm tra SL/TP
│   └── ket_noi_san/                     # Connectors chi tiết của sàn Binance, OKX, Bybit
│
├── hien_thi/                            # UI Dashboard (PyQt6)
│   ├── dashboard_backtest.py            # Biểu đồ phân tích hiệu năng backtest
│   ├── dashboard_realtime.py            # Giám sát bot live trading thời gian thực
│   └── dashboard_demo.py                # Giám sát bot paper trading
│
├── thong_bao/                           # Hệ thống cảnh báo (Email, Telegram alerts)
│   ├── gui_email.py
│   └── gui_telegram.py
│
├── utils/                               # Các hàm helper và tiện ích bổ trợ
│   ├── chuyen_doi_don_vi.py             # Tính toán khối lượng coin, làm tròn số thập phân
│   ├── doc_cau_hinh.py                  # Load cấu hình từ YAML/JSON an toàn
│   ├── ham_tien_ich.py                  # Tính PnL, chuẩn hóa tên symbol, tạo run ID
│   ├── log.py                           # Logger ghi nhật ký hoạt động hệ thống
│   └── thoi_gian.py                     # Helper chuyển đổi múi giờ, sleep an toàn
│
└── du_lieu/                             # Lưu trữ dữ liệu cục bộ (CSV, JSON, Logs)
```

-----

## Hướng dẫn cài đặt & Khởi chạy

### 1. Cài đặt môi trường
Yêu cầu Python từ phiên bản **3.12** trở lên.

```bash
# Clone dự án về máy
git clone <repo>
cd kairos-v1

# Cài đặt các thư viện cần thiết
pip install -r requirements.txt
```

### 2. Thiết lập cấu hình
Sao chép template và tùy chỉnh API Keys của sàn giao dịch cùng cấu hình trading:

```bash
# Sao chép file mẫu API key
cp config/tai_khoan_api.json.example config/tai_khoan_api.json

# Cập nhật thông số API Key trong config/tai_khoan_api.json
# Tùy chỉnh danh sách symbol, đòn bẩy, cắt lỗ/chốt lời tại config/cau_hinh_giao_dich.yaml
```

### 3. Khởi chạy Menu chính
Chạy lệnh khởi động giao diện điều hướng:
```bash
python main.py
```

-----

## Tài liệu kỹ thuật chuyên sâu

Mọi giải thích về thuật toán gộp nến không look-ahead, cơ chế chấm điểm chiến lược tích hợp (Ensemble Scoring), công thức đòn bẩy động, quản lý rủi ro và các kịch bản vận hành chi tiết được tài liệu hóa đầy đủ tại:

📖 **[Tài liệu tham chiếu kỹ thuật chi tiết — Technical Reference Manual](tai_lieu_chi_tiet.md)**

-----

## Thông tin tác giả & Liên hệ

* **Tác giả:** P Vinh
* **Vai trò:** Financial Data Analyst · Quant Developer
* **Stack:** Python · Pandas · Polars · PyQt6 · CCXT
* **Phương pháp:** Data-driven design · Statistical validation · Human logic + AI-assisted development
* **Liên hệ:** ppvinh1513@gmail.com

*"Romain Rolland: 'There is only one heroism in the world: to see the world as it is, and to love it.'"*
