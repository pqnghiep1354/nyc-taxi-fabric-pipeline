# 📖 NYC TLC Data Dictionary

## Từ điển Dữ liệu - NYC Taxi & Limousine Commission

---

## 📋 Mục lục

1. [Giới thiệu](#-giới-thiệu)
2. [Bronze Layer - Raw Data](#-bronze-layer---raw-data)
   - [NYC Yellow Taxi Trip Records](#1-nyc-yellow-taxi-trip-records)
   - [Taxi Zone Lookup](#2-taxi-zone-lookup)
   - [Weather Data](#3-weather-data)
   - [US Holidays](#4-us-holidays)
3. [Silver Layer - Cleansed Data](#-silver-layer---cleansed-data)
   - [silver_taxi_trips](#1-silver_taxi_trips)
   - [silver_taxi_zones](#2-silver_taxi_zones)
   - [silver_weather](#3-silver_weather)
   - [silver_holidays](#4-silver_holidays)
   - [silver_calendar](#5-silver_calendar)
4. [Gold Layer - Business Data](#-gold-layer---business-data)
   - [Dimension Tables](#dimension-tables)
   - [Fact Tables](#fact-tables)
5. [Lookup Values & Reference Codes](#-lookup-values--reference-codes)
6. [Data Quality Rules](#-data-quality-rules)
7. [Business Rules & Calculations](#-business-rules--calculations)

---

## 📌 Giới thiệu

### Mục đích

Tài liệu này mô tả chi tiết tất cả các trường dữ liệu (fields/columns) được sử dụng trong dự án NYC Taxi Data Engineering, bao gồm:

- **Định nghĩa** của từng trường
- **Kiểu dữ liệu** (data type)
- **Giá trị hợp lệ** (valid values)
- **Ví dụ** cụ thể
- **Quy tắc nghiệp vụ** liên quan

### Quy ước ký hiệu

| Ký hiệu | Ý nghĩa |
|---------|---------|
| 🔑 | Primary Key |
| 🔗 | Foreign Key |
| 📊 | Measure (số liệu tính toán) |
| 🏷️ | Attribute (thuộc tính mô tả) |
| 🕐 | Datetime field |
| ✨ | Derived column (cột được tính toán) |
| ⚠️ | Nullable (có thể NULL) |

---

## 🥉 Bronze Layer - Raw Data

### 1. NYC Yellow Taxi Trip Records

**Nguồn:** Azure Open Datasets - NYC TLC Yellow Taxi  
**Định dạng:** Parquet  
**Số bản ghi:** ~194.5 triệu (2021-2024)  
**Cập nhật:** Hàng tháng bởi NYC TLC

#### Schema

| # | Tên cột | Kiểu dữ liệu | Nullable | Mô tả |
|---|---------|--------------|----------|-------|
| 1 | 🔑 VendorID | INTEGER | No | Mã nhà cung cấp LPEP (xem bảng lookup) |
| 2 | 🕐 tpep_pickup_datetime | TIMESTAMP | No | Ngày giờ bật đồng hồ tính tiền (meter engaged) |
| 3 | 🕐 tpep_dropoff_datetime | TIMESTAMP | No | Ngày giờ tắt đồng hồ tính tiền (meter disengaged) |
| 4 | 📊 passenger_count | INTEGER | Yes ⚠️ | Số hành khách (do tài xế nhập) |
| 5 | 📊 trip_distance | DOUBLE | No | Khoảng cách chuyến đi tính bằng miles (từ taximeter) |
| 6 | 🔗 PULocationID | INTEGER | No | TLC Taxi Zone ID nơi bật đồng hồ |
| 7 | 🔗 DOLocationID | INTEGER | No | TLC Taxi Zone ID nơi tắt đồng hồ |
| 8 | 🏷️ RatecodeID | INTEGER | Yes ⚠️ | Mã loại giá cuối cùng áp dụng (xem bảng lookup) |
| 9 | 🏷️ store_and_fwd_flag | STRING | Yes ⚠️ | Cờ lưu trữ và chuyển tiếp |
| 10 | 🏷️ payment_type | INTEGER | No | Mã phương thức thanh toán (xem bảng lookup) |
| 11 | 📊 fare_amount | DOUBLE | No | Giá cước theo thời gian và khoảng cách (USD) |
| 12 | 📊 extra | DOUBLE | No | Phụ phí misc (rush hour, overnight) |
| 13 | 📊 mta_tax | DOUBLE | No | Thuế MTA tự động ($0.50) |
| 14 | 📊 improvement_surcharge | DOUBLE | No | Phụ phí cải thiện ($0.30) |
| 15 | 📊 tip_amount | DOUBLE | No | Tiền tip (chỉ ghi nhận với thẻ tín dụng) |
| 16 | 📊 tolls_amount | DOUBLE | No | Tổng phí cầu đường |
| 17 | 📊 total_amount | DOUBLE | No | Tổng tiền khách trả (không bao gồm tip tiền mặt) |
| 18 | 📊 congestion_surcharge | DOUBLE | Yes ⚠️ | Phụ phí ùn tắc (từ 2019) |
| 19 | 📊 airport_fee | DOUBLE | Yes ⚠️ | Phí sân bay ($1.25 cho pickup tại JFK/LGA) |

#### Chi tiết từng trường

---

##### VendorID

**Định nghĩa:** Mã định danh nhà cung cấp LPEP (Livery Passenger Enhancement Program) đã cung cấp bản ghi.

| Giá trị | Nhà cung cấp | Mô tả |
|---------|--------------|-------|
| 1 | Creative Mobile Technologies, LLC (CMT) | Nhà cung cấp thiết bị taximeter truyền thống |
| 2 | VeriFone Inc. | Nhà cung cấp hệ thống thanh toán điện tử |

**Ví dụ:** `1`, `2`

**Lưu ý:** 
- Giá trị NULL hoặc ngoài {1, 2} là không hợp lệ
- Phân bố thường ~60% VeriFone, ~40% CMT

---

##### tpep_pickup_datetime

**Định nghĩa:** Ngày và giờ khi đồng hồ tính tiền được bật (meter engaged). Đây là thời điểm bắt đầu chuyến đi.

**Format:** `YYYY-MM-DD HH:MM:SS`

**Ví dụ:** `2023-06-15 14:30:25`

**Quy tắc nghiệp vụ:**
- Phải trước `tpep_dropoff_datetime`
- Trong dự án này, chỉ giữ lại records với năm 2021-2024
- Timezone: Eastern Time (ET)

**Anomalies phát hiện:**
- Một số records có năm < 2001 hoặc > 2025 (data entry error)
- Cần filter trong Silver layer

---

##### tpep_dropoff_datetime

**Định nghĩa:** Ngày và giờ khi đồng hồ tính tiền được tắt (meter disengaged). Đây là thời điểm kết thúc chuyến đi.

**Format:** `YYYY-MM-DD HH:MM:SS`

**Ví dụ:** `2023-06-15 14:52:10`

**Quy tắc nghiệp vụ:**
- Phải sau `tpep_pickup_datetime`
- Chênh lệch với pickup không quá 24 giờ (loại bỏ outliers)
- Timezone: Eastern Time (ET)

---

##### passenger_count

**Định nghĩa:** Số lượng hành khách trong xe. Giá trị này do tài xế nhập vào hệ thống.

**Range hợp lệ:** 0 - 9

| Giá trị | Ý nghĩa |
|---------|---------|
| 0 | Không có hành khách (có thể là chuyến đi rỗng hoặc lỗi nhập liệu) |
| 1-6 | Số hành khách thông thường |
| 7-9 | Xe van hoặc trường hợp đặc biệt |

**Ví dụ:** `1`, `2`, `3`

**Lưu ý:**
- Trường này có thể NULL
- Giá trị phổ biến nhất là 1 (~70% trips)
- Giá trị > 6 rất hiếm (<0.1%)

---

##### trip_distance

**Định nghĩa:** Khoảng cách di chuyển được ghi nhận bởi taximeter, tính bằng **miles**.

**Đơn vị:** Miles (1 mile ≈ 1.609 km)

**Range hợp lệ:** 0.0 - 500.0 miles

**Ví dụ:** `3.5`, `12.8`, `0.5`

**Quy tắc nghiệp vụ:**
- Giá trị 0 có thể là chuyến đi cancelled hoặc rất ngắn
- Giá trị > 100 miles thường là outliers (trừ chuyến đi sân bay xa)
- Giá trị âm là lỗi data và cần loại bỏ

**Thống kê tham khảo:**
- Trung bình: ~2.9 miles
- Median: ~1.6 miles
- 95th percentile: ~10 miles

---

##### PULocationID (Pickup Location ID)

**Định nghĩa:** Mã TLC Taxi Zone nơi đồng hồ tính tiền được bật (điểm đón khách).

**Range hợp lệ:** 1 - 265

**Ví dụ:** `161` (Midtown Center), `132` (JFK Airport), `138` (LaGuardia Airport)

**Liên kết:** Join với bảng `taxi_zone_lookup` để lấy tên Zone và Borough

**Top Pickup Locations:**
| LocationID | Zone | Borough |
|------------|------|---------|
| 161 | Midtown Center | Manhattan |
| 237 | Upper East Side South | Manhattan |
| 236 | Upper East Side North | Manhattan |
| 162 | Midtown East | Manhattan |
| 230 | Times Square/Theatre District | Manhattan |

---

##### DOLocationID (Dropoff Location ID)

**Định nghĩa:** Mã TLC Taxi Zone nơi đồng hồ tính tiền được tắt (điểm trả khách).

**Range hợp lệ:** 1 - 265

**Ví dụ:** `234` (Union Sq), `48` (Clinton East), `79` (East Village)

**Liên kết:** Join với bảng `taxi_zone_lookup` để lấy tên Zone và Borough

---

##### RatecodeID

**Định nghĩa:** Mã loại giá cước cuối cùng được áp dụng cho chuyến đi.

| Giá trị | Loại giá | Mô tả |
|---------|----------|-------|
| 1 | Standard rate | Giá tiêu chuẩn theo meter |
| 2 | JFK | Giá cố định $52 từ/đến JFK |
| 3 | Newark | Giá thương lượng đến Newark |
| 4 | Nassau or Westchester | Giá gấp đôi theo meter |
| 5 | Negotiated fare | Giá thương lượng |
| 6 | Group ride | Chuyến đi nhóm |

**Ví dụ:** `1`, `2`, `5`

**Lưu ý:**
- ~97% trips sử dụng Standard rate (1)
- JFK trips (~2%) có giá cố định
- Giá trị NULL hoặc ngoài range cần xử lý

---

##### store_and_fwd_flag

**Định nghĩa:** Cờ cho biết bản ghi trip có được lưu trữ trong bộ nhớ xe trước khi gửi đến vendor hay không (do không có kết nối server).

| Giá trị | Ý nghĩa |
|---------|---------|
| Y | Store and forward trip (lưu rồi gửi) |
| N | Not a store and forward trip (gửi trực tiếp) |

**Ví dụ:** `N`, `Y`

**Lưu ý:** 
- Đa số trips có giá trị `N` (>99%)
- Flag này ít ảnh hưởng đến phân tích nghiệp vụ

---

##### payment_type

**Định nghĩa:** Mã số cho biết hành khách thanh toán bằng phương thức nào.

| Giá trị | Phương thức | Mô tả |
|---------|-------------|-------|
| 1 | Credit card | Thẻ tín dụng/ghi nợ |
| 2 | Cash | Tiền mặt |
| 3 | No charge | Miễn phí |
| 4 | Dispute | Tranh chấp |
| 5 | Unknown | Không xác định |
| 6 | Voided trip | Chuyến đi bị hủy |

**Ví dụ:** `1`, `2`

**Quy tắc nghiệp vụ:**
- `tip_amount` chỉ được ghi nhận khi `payment_type = 1` (Credit card)
- Tips tiền mặt không được tracking trong dữ liệu

**Phân bố điển hình:**
- Credit card: ~70%
- Cash: ~28%
- Others: ~2%

---

##### fare_amount

**Định nghĩa:** Giá cước theo thời gian và khoảng cách được tính bởi taximeter.

**Đơn vị:** USD (Đô la Mỹ)

**Range hợp lệ:** $0.00 - $5,000.00

**Công thức tính (Standard rate):**
```
fare = $3.00 (initial charge)
     + $0.70 per 1/5 mile
     + $0.70 per 60 seconds in slow traffic
```

**Ví dụ:** `15.50`, `52.00` (JFK flat rate), `8.00`

**Lưu ý:**
- Giá trị âm là lỗi data
- Flat rate JFK = $52.00 (không bao gồm tolls, tips, surcharges)
- Giá có thể cao hơn bình thường vào rush hour hoặc overnight

---

##### extra

**Định nghĩa:** Các phụ phí miscellaneous. Hiện tại chỉ bao gồm rush hour và overnight charges.

**Đơn vị:** USD

| Loại | Giá trị | Thời gian áp dụng |
|------|---------|-------------------|
| Rush hour surcharge | $1.00 | Weekdays 4PM-8PM |
| Overnight surcharge | $0.50 | 8PM-6AM |
| Rush hour + overnight | $1.00 | Nếu overlap |

**Ví dụ:** `0.00`, `0.50`, `1.00`

---

##### mta_tax

**Định nghĩa:** Thuế MTA (Metropolitan Transportation Authority) tự động được trigger dựa trên metered rate đang sử dụng.

**Giá trị cố định:** $0.50

**Ví dụ:** `0.50`

**Lưu ý:** Thuế này được áp dụng cho hầu hết các chuyến đi trong NYC

---

##### improvement_surcharge

**Định nghĩa:** Phụ phí cải thiện $0.30 được áp dụng cho các chuyến đi được đánh dấu (flagged) tại stand hoặc street hail.

**Giá trị cố định:** $0.30

**Ví dụ:** `0.30`

**Lưu ý:** Áp dụng từ năm 2015

---

##### tip_amount

**Định nghĩa:** Tiền tip/tiền boa. Trường này được tự động điền cho các thanh toán bằng thẻ tín dụng. Tips tiền mặt không được ghi nhận.

**Đơn vị:** USD

**Range hợp lệ:** $0.00 trở lên

**Ví dụ:** `3.00`, `5.50`, `0.00`

**Quy tắc nghiệp vụ:**
- Chỉ có giá trị > 0 khi `payment_type = 1` (Credit card)
- Khi `payment_type = 2` (Cash), `tip_amount = 0` (nhưng thực tế có thể có tip)
- Tip percentage thường dao động 15-25% của fare

**Công thức tính tip_percentage:**
```
tip_percentage = (tip_amount / fare_amount) * 100
```

---

##### tolls_amount

**Định nghĩa:** Tổng số tiền phí cầu đường đã trả trong chuyến đi.

**Đơn vị:** USD

**Ví dụ:** `0.00`, `6.55` (Lincoln Tunnel), `9.50` (Verrazano Bridge)

**Các cầu/hầm phổ biến:**
| Cầu/Hầm | Phí ước tính |
|---------|--------------|
| RFK Bridge | $6.55 |
| Lincoln Tunnel | $16.00 |
| Holland Tunnel | $16.00 |
| Verrazano Bridge | $9.50 |
| Whitestone Bridge | $9.50 |

---

##### total_amount

**Định nghĩa:** Tổng số tiền khách hàng phải trả. Được tính trước khi thanh toán.

**Đơn vị:** USD

**Công thức:**
```
total_amount = fare_amount 
             + extra 
             + mta_tax 
             + improvement_surcharge 
             + tip_amount 
             + tolls_amount 
             + congestion_surcharge 
             + airport_fee
```

**Ví dụ:** `25.30`, `68.00`, `12.50`

**Lưu ý:** 
- Không bao gồm tips tiền mặt
- Đây là giá trị chính để tính revenue

---

##### congestion_surcharge

**Định nghĩa:** Phụ phí ùn tắc giao thông cho các chuyến đi trong vùng congestion zone của Manhattan.

**Giá trị:** $2.50 (Yellow taxi), $2.75 (For-hire vehicles)

**Áp dụng:** Chuyến đi bắt đầu, kết thúc, hoặc đi qua Manhattan dưới 96th Street

**Ví dụ:** `0.00`, `2.50`

**Lưu ý:**
- Bắt đầu áp dụng từ 1/1/2019
- Records trước 2019 có thể có giá trị NULL

---

##### airport_fee

**Định nghĩa:** Phí pickup tại sân bay.

**Giá trị:** $1.25 cho pickup tại JFK hoặc LaGuardia

**Ví dụ:** `0.00`, `1.25`

**Lưu ý:**
- Chỉ áp dụng cho pickups, không áp dụng cho dropoffs
- LocationID 132 (JFK) và 138 (LaGuardia)

---

### 2. Taxi Zone Lookup

**Nguồn:** NYC TLC  
**Định dạng:** CSV  
**Số bản ghi:** 265  
**Cập nhật:** Static (ít thay đổi)

#### Schema

| # | Tên cột | Kiểu dữ liệu | Nullable | Mô tả |
|---|---------|--------------|----------|-------|
| 1 | 🔑 LocationID | INTEGER | No | Mã định danh duy nhất cho taxi zone |
| 2 | 🏷️ Borough | STRING | No | Tên quận/borough |
| 3 | 🏷️ Zone | STRING | No | Tên khu vực/zone |
| 4 | 🏷️ service_zone | STRING | No | Loại vùng phục vụ |

#### Chi tiết từng trường

---

##### LocationID

**Định nghĩa:** Mã định danh duy nhất cho mỗi taxi zone trong NYC.

**Range:** 1 - 265

**Ví dụ:** `1`, `132`, `265`

**Zones đặc biệt:**
| LocationID | Zone | Ghi chú |
|------------|------|---------|
| 1 | Newark Airport | Ngoài NYC |
| 132 | JFK Airport | Queens |
| 138 | LaGuardia Airport | Queens |
| 264 | Unknown | Data không xác định |
| 265 | NA | Not Available |

---

##### Borough

**Định nghĩa:** Tên của borough (quận) chứa zone.

**Giá trị hợp lệ:**

| Borough | Số zones | Mô tả |
|---------|----------|-------|
| Manhattan | 69 | Trung tâm NYC, nhiều taxi trips nhất |
| Queens | 69 | Chứa 2 sân bay (JFK, LaGuardia) |
| Brooklyn | 61 | Borough đông dân nhất |
| Bronx | 43 | Borough cực bắc |
| Staten Island | 20 | Ít taxi trips nhất |
| EWR | 1 | Newark Airport (New Jersey) |
| Unknown | 2 | Zones không xác định |

**Ví dụ:** `Manhattan`, `Brooklyn`, `Queens`

---

##### Zone

**Định nghĩa:** Tên cụ thể của khu vực địa lý.

**Ví dụ:** 
- `Midtown Center`
- `Upper East Side South`
- `JFK Airport`
- `Times Square/Theatre District`
- `Financial District North`

**Lưu ý:** Tên zone có thể chứa ký tự đặc biệt như `/`, `'`

---

##### service_zone

**Định nghĩa:** Phân loại vùng phục vụ của taxi.

| Giá trị | Mô tả |
|---------|-------|
| Yellow Zone | Vùng được phép đón khách bằng street hail |
| Boro Zone | Vùng ngoại ô, cần booking trước |
| Airports | Khu vực sân bay |
| EWR | Newark Airport |
| N/A | Không áp dụng |

**Ví dụ:** `Yellow Zone`, `Boro Zone`, `Airports`

---

### 3. Weather Data

**Nguồn:** NOAA (National Oceanic and Atmospheric Administration)  
**Trạm:** Central Park, Manhattan  
**Định dạng:** CSV  
**Số bản ghi:** 1,155 (daily records)  
**Khoảng thời gian:** 2021-01-01 đến 2024-02-29

#### Schema

| # | Tên cột | Kiểu dữ liệu | Nullable | Mô tả |
|---|---------|--------------|----------|-------|
| 1 | 🔑 date | DATE | No | Ngày ghi nhận |
| 2 | 📊 tavg | DOUBLE | Yes ⚠️ | Nhiệt độ trung bình (°F) |
| 3 | 📊 tmax | DOUBLE | Yes ⚠️ | Nhiệt độ cao nhất (°F) |
| 4 | 📊 tmin | DOUBLE | Yes ⚠️ | Nhiệt độ thấp nhất (°F) |
| 5 | 📊 prcp | DOUBLE | Yes ⚠️ | Lượng mưa (inches) |
| 6 | 📊 snow | DOUBLE | Yes ⚠️ | Lượng tuyết rơi (inches) |
| 7 | 📊 snwd | DOUBLE | Yes ⚠️ | Độ dày tuyết phủ (inches) |
| 8 | 📊 wdir | DOUBLE | Yes ⚠️ | Hướng gió (độ) |
| 9 | 📊 wspd | DOUBLE | Yes ⚠️ | Tốc độ gió (mph) |

#### Chi tiết từng trường

---

##### date

**Định nghĩa:** Ngày ghi nhận dữ liệu thời tiết.

**Format:** `YYYY-MM-DD`

**Ví dụ:** `2023-06-15`, `2024-01-01`

---

##### tavg (Temperature Average)

**Định nghĩa:** Nhiệt độ trung bình trong ngày.

**Đơn vị:** Fahrenheit (°F)

**Công thức chuyển đổi:** `°C = (°F - 32) × 5/9`

**Ví dụ:** `55.0` (°F) = `12.8` (°C)

**Range điển hình NYC:**
- Mùa đông: 25-40°F (-4 đến 4°C)
- Mùa hè: 70-85°F (21 đến 29°C)

---

##### tmax (Temperature Maximum)

**Định nghĩa:** Nhiệt độ cao nhất trong ngày.

**Đơn vị:** Fahrenheit (°F)

**Ví dụ:** `78.0`, `32.0`

---

##### tmin (Temperature Minimum)

**Định nghĩa:** Nhiệt độ thấp nhất trong ngày.

**Đơn vị:** Fahrenheit (°F)

**Ví dụ:** `45.0`, `18.0`

---

##### prcp (Precipitation)

**Định nghĩa:** Tổng lượng mưa trong ngày.

**Đơn vị:** Inches (1 inch = 25.4 mm)

**Ví dụ:** `0.00`, `0.15`, `1.25`

**Phân loại:**
| Lượng mưa | Mô tả |
|-----------|-------|
| 0.00 | Không mưa |
| 0.01 - 0.10 | Mưa nhẹ (Light rain) |
| 0.11 - 0.50 | Mưa vừa (Moderate rain) |
| > 0.50 | Mưa nặng (Heavy rain) |

---

##### snow (Snowfall)

**Định nghĩa:** Lượng tuyết rơi trong ngày.

**Đơn vị:** Inches

**Ví dụ:** `0.0`, `3.5`, `12.0`

**Lưu ý:** Chỉ có giá trị > 0 trong mùa đông (Nov-Mar)

---

##### snwd (Snow Depth)

**Định nghĩa:** Độ dày lớp tuyết phủ trên mặt đất.

**Đơn vị:** Inches

**Ví dụ:** `0.0`, `5.0`, `15.0`

---

##### wdir (Wind Direction)

**Định nghĩa:** Hướng gió chính.

**Đơn vị:** Độ (0-360)

| Góc độ | Hướng |
|--------|-------|
| 0/360 | North (Bắc) |
| 90 | East (Đông) |
| 180 | South (Nam) |
| 270 | West (Tây) |

**Ví dụ:** `180`, `270`, `45`

---

##### wspd (Wind Speed)

**Định nghĩa:** Tốc độ gió trung bình.

**Đơn vị:** Miles per hour (mph)

**Ví dụ:** `8.5`, `15.0`, `25.0`

**Phân loại:**
| Tốc độ (mph) | Mô tả |
|--------------|-------|
| 0-7 | Gió nhẹ (Light) |
| 8-18 | Gió vừa (Moderate) |
| 19-31 | Gió mạnh (Fresh/Strong) |
| > 31 | Gió rất mạnh (Gale/Storm) |

---

### 4. US Holidays

**Nguồn:** US Federal Holidays  
**Định dạng:** CSV  
**Số bản ghi:** 75  
**Khoảng thời gian:** 2021-2026

#### Schema

| # | Tên cột | Kiểu dữ liệu | Nullable | Mô tả |
|---|---------|--------------|----------|-------|
| 1 | 🔑 date | DATE | No | Ngày lễ |
| 2 | 🏷️ holiday | STRING | No | Tên ngày lễ |
| 3 | 🏷️ type | STRING | No | Loại ngày lễ |

#### Chi tiết từng trường

---

##### date

**Định nghĩa:** Ngày diễn ra ngày lễ.

**Format:** `YYYY-MM-DD`

**Ví dụ:** `2023-12-25`, `2024-07-04`

---

##### holiday

**Định nghĩa:** Tên đầy đủ của ngày lễ.

**Danh sách US Federal Holidays:**

| Ngày lễ | Ngày (thường) | Mô tả |
|---------|---------------|-------|
| New Year's Day | Jan 1 | Năm mới |
| Martin Luther King Jr. Day | 3rd Mon of Jan | Ngày MLK |
| Presidents' Day | 3rd Mon of Feb | Ngày Tổng thống |
| Memorial Day | Last Mon of May | Ngày Tưởng niệm |
| Independence Day | Jul 4 | Ngày Độc lập |
| Labor Day | 1st Mon of Sep | Ngày Lao động |
| Columbus Day | 2nd Mon of Oct | Ngày Columbus |
| Veterans Day | Nov 11 | Ngày Cựu chiến binh |
| Thanksgiving | 4th Thu of Nov | Lễ Tạ ơn |
| Christmas Day | Dec 25 | Giáng sinh |

**Ví dụ:** `Christmas Day`, `Thanksgiving`, `Independence Day`

---

##### type

**Định nghĩa:** Phân loại ngày lễ.

| Giá trị | Mô tả |
|---------|-------|
| Federal | Ngày lễ liên bang (chính thức) |
| Observed | Ngày nghỉ bù (khi lễ rơi vào weekend) |

**Ví dụ:** `Federal`, `Observed`

---

## 🥈 Silver Layer - Cleansed Data

### 1. silver_taxi_trips

**Nguồn:** Bronze taxi trips sau khi làm sạch và enrich  
**Định dạng:** Delta Lake  
**Partitioning:** `pickup_year`, `pickup_month`

#### Inherited Columns (từ Bronze)

Tất cả columns từ Bronze layer được giữ nguyên với data quality filters đã áp dụng.

#### Derived Columns (✨ Mới tạo)

| # | Tên cột | Kiểu dữ liệu | Công thức | Mô tả |
|---|---------|--------------|-----------|-------|
| 1 | ✨ pickup_date | DATE | `DATE(tpep_pickup_datetime)` | Ngày đón khách |
| 2 | ✨ pickup_year | INTEGER | `YEAR(tpep_pickup_datetime)` | Năm đón khách (partition) |
| 3 | ✨ pickup_month | INTEGER | `MONTH(tpep_pickup_datetime)` | Tháng đón khách (partition) |
| 4 | ✨ pickup_day | INTEGER | `DAY(tpep_pickup_datetime)` | Ngày trong tháng |
| 5 | ✨ pickup_hour | INTEGER | `HOUR(tpep_pickup_datetime)` | Giờ đón khách (0-23) |
| 6 | ✨ pickup_dayofweek | INTEGER | `DAYOFWEEK(tpep_pickup_datetime)` | Ngày trong tuần (1=Sun, 7=Sat) |
| 7 | ✨ trip_duration_minutes | DOUBLE | `(dropoff - pickup) / 60` | Thời gian chuyến đi (phút) |
| 8 | ✨ avg_speed_mph | DOUBLE | `distance / (duration / 60)` | Tốc độ trung bình (mph) |
| 9 | ✨ time_of_day | STRING | CASE expression | Khoảng thời gian trong ngày |
| 10 | ✨ distance_category | STRING | CASE expression | Phân loại khoảng cách |
| 11 | ✨ tip_percentage | DOUBLE | `(tip / fare) * 100` | Phần trăm tip |
| 12 | ✨ is_weekend | BOOLEAN | `dayofweek IN (1, 7)` | Có phải cuối tuần |
| 13 | ✨ is_rush_hour | BOOLEAN | `hour IN (7-9, 17-19)` | Có phải giờ cao điểm |
| 14 | ✨ vendor_name | STRING | Lookup | Tên vendor |
| 15 | ✨ payment_type_name | STRING | Lookup | Tên payment type |
| 16 | ✨ rate_code_name | STRING | Lookup | Tên rate code |

#### Chi tiết Derived Columns

---

##### trip_duration_minutes

**Công thức:**
```python
trip_duration_minutes = (
    unix_timestamp(tpep_dropoff_datetime) - 
    unix_timestamp(tpep_pickup_datetime)
) / 60
```

**Ví dụ:** `15.5`, `32.0`, `8.2`

**Quy tắc nghiệp vụ:**
- Giá trị < 0 là không hợp lệ
- Giá trị > 1440 (24 giờ) được loại bỏ

---

##### avg_speed_mph

**Công thức:**
```python
avg_speed_mph = trip_distance / (trip_duration_minutes / 60)
```

**Ví dụ:** `12.5`, `25.0`, `8.0`

**Lưu ý:**
- NULL nếu duration = 0
- Giá trị > 100 mph có thể là outlier

---

##### time_of_day

**Công thức:**
```python
time_of_day = CASE
    WHEN hour >= 5 AND hour < 12 THEN 'Morning'
    WHEN hour >= 12 AND hour < 17 THEN 'Afternoon'
    WHEN hour >= 17 AND hour < 21 THEN 'Evening'
    ELSE 'Night'
END
```

| Giá trị | Giờ | Mô tả |
|---------|-----|-------|
| Morning | 5:00 - 11:59 | Buổi sáng |
| Afternoon | 12:00 - 16:59 | Buổi chiều |
| Evening | 17:00 - 20:59 | Buổi tối |
| Night | 21:00 - 4:59 | Ban đêm |

---

##### distance_category

**Công thức:**
```python
distance_category = CASE
    WHEN trip_distance <= 1 THEN 'Short'
    WHEN trip_distance <= 5 THEN 'Medium'
    WHEN trip_distance <= 15 THEN 'Long'
    ELSE 'Very Long'
END
```

| Giá trị | Khoảng cách | Mô tả |
|---------|-------------|-------|
| Short | 0 - 1 miles | Chuyến ngắn trong khu vực |
| Medium | 1 - 5 miles | Chuyến trung bình |
| Long | 5 - 15 miles | Chuyến dài (có thể đến sân bay) |
| Very Long | > 15 miles | Chuyến rất dài |

---

##### tip_percentage

**Công thức:**
```python
tip_percentage = CASE
    WHEN fare_amount > 0 AND payment_type = 1 
    THEN (tip_amount / fare_amount) * 100
    ELSE 0
END
```

**Ví dụ:** `18.5`, `20.0`, `0.0`

**Lưu ý:**
- Chỉ tính cho credit card payments
- Giá trị > 100% có thể là outlier hoặc lỗi

---

##### is_weekend

**Công thức:**
```python
is_weekend = dayofweek(tpep_pickup_datetime) IN (1, 7)  # 1=Sunday, 7=Saturday
```

| Giá trị | Ý nghĩa |
|---------|---------|
| TRUE | Saturday hoặc Sunday |
| FALSE | Monday - Friday |

---

##### is_rush_hour

**Công thức:**
```python
is_rush_hour = (
    dayofweek NOT IN (1, 7)  # Weekdays only
    AND (
        (hour >= 7 AND hour <= 9)   # Morning rush
        OR (hour >= 17 AND hour <= 19)  # Evening rush
    )
)
```

| Giá trị | Ý nghĩa |
|---------|---------|
| TRUE | Giờ cao điểm (7-9 AM hoặc 5-7 PM, weekdays) |
| FALSE | Giờ bình thường |

---

### 2. silver_taxi_zones

**Nguồn:** Bronze taxi_zones với enrichments  
**Định dạng:** Delta Lake

#### Schema

| # | Tên cột | Kiểu dữ liệu | Nguồn | Mô tả |
|---|---------|--------------|-------|-------|
| 1 | 🔑 location_id | INTEGER | Original | Mã zone |
| 2 | 🏷️ borough | STRING | Standardized | Borough đã chuẩn hóa |
| 3 | 🏷️ zone | STRING | Original | Tên zone |
| 4 | 🏷️ service_zone | STRING | Original | Loại vùng phục vụ |
| 5 | ✨ is_airport | BOOLEAN | Derived | Có phải sân bay |
| 6 | ✨ zone_type | STRING | Derived | Phân loại zone |

#### Derived Columns

##### is_airport

**Công thức:**
```python
is_airport = location_id IN (1, 132, 138)  # Newark, JFK, LaGuardia
```

##### zone_type

**Công thức:**
```python
zone_type = CASE
    WHEN is_airport THEN 'Airport'
    WHEN borough = 'Manhattan' THEN 'Manhattan'
    WHEN service_zone = 'Yellow Zone' THEN 'Yellow Zone'
    ELSE 'Outer Borough'
END
```

---

### 3. silver_weather

**Nguồn:** Bronze weather với enrichments  
**Định dạng:** Delta Lake

#### Schema

| # | Tên cột | Kiểu dữ liệu | Nguồn | Mô tả |
|---|---------|--------------|-------|-------|
| 1 | 🔑 date | DATE | Original | Ngày |
| 2-9 | (weather fields) | Various | Original | Các trường weather gốc |
| 10 | ✨ weather_condition | STRING | Derived | Điều kiện thời tiết |
| 11 | ✨ is_severe_weather | BOOLEAN | Derived | Thời tiết khắc nghiệt |
| 12 | ✨ temp_range | STRING | Derived | Khoảng nhiệt độ |

#### Derived Columns

##### weather_condition

**Công thức:**
```python
weather_condition = CASE
    WHEN snow > 0 THEN 'Snow'
    WHEN prcp > 0.5 THEN 'Heavy Rain'
    WHEN prcp > 0 THEN 'Rain'
    WHEN wspd > 25 THEN 'Windy'
    ELSE 'Clear'
END
```

| Giá trị | Điều kiện |
|---------|-----------|
| Snow | Có tuyết rơi |
| Heavy Rain | Mưa > 0.5 inch |
| Rain | Mưa <= 0.5 inch |
| Windy | Gió > 25 mph |
| Clear | Trời quang |

##### is_severe_weather

**Công thức:**
```python
is_severe_weather = (
    snow > 3  # > 3 inches snow
    OR prcp > 1  # > 1 inch rain
    OR wspd > 35  # > 35 mph wind
)
```

##### temp_range

**Công thức:**
```python
temp_range = CASE
    WHEN tavg < 32 THEN 'Freezing'
    WHEN tavg < 50 THEN 'Cold'
    WHEN tavg < 68 THEN 'Mild'
    WHEN tavg < 86 THEN 'Warm'
    ELSE 'Hot'
END
```

| Giá trị | Nhiệt độ (°F) | Nhiệt độ (°C) |
|---------|---------------|---------------|
| Freezing | < 32 | < 0 |
| Cold | 32 - 49 | 0 - 9 |
| Mild | 50 - 67 | 10 - 19 |
| Warm | 68 - 85 | 20 - 29 |
| Hot | >= 86 | >= 30 |

---

### 4. silver_holidays

**Nguồn:** Bronze holidays với enrichments  
**Định dạng:** Delta Lake

#### Schema

| # | Tên cột | Kiểu dữ liệu | Nguồn | Mô tả |
|---|---------|--------------|-------|-------|
| 1 | 🔑 date | DATE | Original | Ngày lễ |
| 2 | 🏷️ holiday | STRING | Original | Tên ngày lễ |
| 3 | 🏷️ type | STRING | Original | Loại (Federal/Observed) |
| 4 | ✨ holiday_category | STRING | Derived | Phân loại ngày lễ |
| 5 | ✨ is_major_holiday | BOOLEAN | Derived | Ngày lễ lớn |

#### Derived Columns

##### holiday_category

**Công thức:**
```python
holiday_category = CASE
    WHEN holiday IN ('Christmas Day', 'Thanksgiving', 'New Year''s Day') 
        THEN 'Major'
    WHEN holiday IN ('Independence Day', 'Memorial Day', 'Labor Day') 
        THEN 'Patriotic'
    WHEN holiday LIKE '%Day' AND holiday NOT LIKE '%Christmas%' 
        THEN 'Commemorative'
    ELSE 'Other'
END
```

##### is_major_holiday

**Công thức:**
```python
is_major_holiday = holiday IN (
    'New Year''s Day',
    'Independence Day', 
    'Thanksgiving',
    'Christmas Day'
)
```

---

### 5. silver_calendar

**Nguồn:** Generated date dimension với holidays joined  
**Định dạng:** Delta Lake  
**Khoảng thời gian:** 2021-01-01 đến 2024-12-31

#### Schema

| # | Tên cột | Kiểu dữ liệu | Mô tả |
|---|---------|--------------|-------|
| 1 | 🔑 date | DATE | Ngày |
| 2 | 🏷️ year | INTEGER | Năm |
| 3 | 🏷️ quarter | INTEGER | Quý (1-4) |
| 4 | 🏷️ month | INTEGER | Tháng (1-12) |
| 5 | 🏷️ month_name | STRING | Tên tháng |
| 6 | 🏷️ day | INTEGER | Ngày trong tháng |
| 7 | 🏷️ day_of_week | INTEGER | Ngày trong tuần |
| 8 | 🏷️ day_name | STRING | Tên ngày |
| 9 | 🏷️ week_of_year | INTEGER | Tuần trong năm |
| 10 | 🏷️ is_weekend | BOOLEAN | Có phải cuối tuần |
| 11 | 🏷️ is_holiday | BOOLEAN | Có phải ngày lễ |
| 12 | 🏷️ holiday_name | STRING | Tên ngày lễ (nếu có) |
| 13 | 🏷️ is_major_holiday | BOOLEAN | Ngày lễ lớn |

---

## 🥇 Gold Layer - Business Data

### Dimension Tables

#### gold_dim_date

**Mô tả:** Date dimension cho time intelligence  
**Grain:** One row per calendar date  
**Số bản ghi:** ~1,461 (4 năm)

| # | Tên cột | Kiểu dữ liệu | Mô tả |
|---|---------|--------------|-------|
| 1 | 🔑 date_sk | INTEGER | Surrogate key (YYYYMMDD) |
| 2 | 🏷️ full_date | DATE | Ngày đầy đủ |
| 3 | 🏷️ year | INTEGER | Năm |
| 4 | 🏷️ quarter | INTEGER | Quý |
| 5 | 🏷️ month | INTEGER | Tháng |
| 6 | 🏷️ month_name | STRING | Tên tháng |
| 7 | 🏷️ day | INTEGER | Ngày |
| 8 | 🏷️ day_of_week | INTEGER | Ngày trong tuần |
| 9 | 🏷️ day_name | STRING | Tên ngày |
| 10 | 🏷️ week_of_year | INTEGER | Tuần trong năm |
| 11 | 🏷️ is_weekend | BOOLEAN | Cuối tuần |
| 12 | 🏷️ is_holiday | BOOLEAN | Ngày lễ |
| 13 | 🏷️ holiday_name | STRING | Tên ngày lễ |

**Surrogate Key Format:**
```
date_sk = YEAR * 10000 + MONTH * 100 + DAY
Ví dụ: 2023-06-15 → 20230615
```

---

#### gold_dim_time

**Mô tả:** Time dimension với 15-minute intervals  
**Grain:** One row per 15-minute time slot  
**Số bản ghi:** 96

| # | Tên cột | Kiểu dữ liệu | Mô tả |
|---|---------|--------------|-------|
| 1 | 🔑 time_sk | INTEGER | Surrogate key (HHMM) |
| 2 | 🏷️ hour | INTEGER | Giờ (0-23) |
| 3 | 🏷️ minute | INTEGER | Phút (0, 15, 30, 45) |
| 4 | 🏷️ time_of_day | STRING | Khoảng thời gian |
| 5 | 🏷️ peak_period | STRING | Cao điểm/thấp điểm |
| 6 | 🏷️ hour_label | STRING | Nhãn giờ (12-hour format) |

**Surrogate Key Format:**
```
time_sk = HOUR * 100 + MINUTE
Ví dụ: 14:30 → 1430
```

**peak_period values:**
| Giá trị | Điều kiện |
|---------|-----------|
| Morning Rush | 7:00 - 9:59 |
| Midday | 10:00 - 15:59 |
| Evening Rush | 16:00 - 19:59 |
| Evening | 20:00 - 22:59 |
| Night | 23:00 - 6:59 |

---

#### gold_dim_location

**Mô tả:** Location dimension cho taxi zones  
**Grain:** One row per taxi zone  
**Số bản ghi:** 265

| # | Tên cột | Kiểu dữ liệu | Mô tả |
|---|---------|--------------|-------|
| 1 | 🔑 location_sk | INTEGER | Surrogate key (= location_id) |
| 2 | 🏷️ location_id | INTEGER | Original TLC zone ID |
| 3 | 🏷️ zone | STRING | Tên zone |
| 4 | 🏷️ borough | STRING | Borough |
| 5 | 🏷️ service_zone | STRING | Loại vùng phục vụ |
| 6 | 🏷️ is_airport | BOOLEAN | Có phải sân bay |
| 7 | 🏷️ zone_type | STRING | Phân loại zone |

---

#### gold_dim_vendor

**Mô tả:** Vendor dimension  
**Grain:** One row per vendor  
**Số bản ghi:** 2

| # | Tên cột | Kiểu dữ liệu | Mô tả |
|---|---------|--------------|-------|
| 1 | 🔑 vendor_sk | INTEGER | Surrogate key |
| 2 | 🏷️ vendor_id | INTEGER | Original vendor ID |
| 3 | 🏷️ vendor_name | STRING | Tên vendor |
| 4 | 🏷️ vendor_description | STRING | Mô tả |

**Data:**
| vendor_sk | vendor_id | vendor_name | vendor_description |
|-----------|-----------|-------------|-------------------|
| 1 | 1 | CMT | Creative Mobile Technologies, LLC |
| 2 | 2 | VeriFone | VeriFone Inc. |

---

#### gold_dim_payment_type

**Mô tả:** Payment type dimension  
**Grain:** One row per payment type  
**Số bản ghi:** 6

| # | Tên cột | Kiểu dữ liệu | Mô tả |
|---|---------|--------------|-------|
| 1 | 🔑 payment_type_sk | INTEGER | Surrogate key |
| 2 | 🏷️ payment_type_id | INTEGER | Original payment type ID |
| 3 | 🏷️ payment_type_name | STRING | Tên payment type |
| 4 | 🏷️ allows_tip | BOOLEAN | Có ghi nhận tip |

**Data:**
| payment_type_sk | payment_type_id | payment_type_name | allows_tip |
|-----------------|-----------------|-------------------|------------|
| 1 | 1 | Credit Card | TRUE |
| 2 | 2 | Cash | FALSE |
| 3 | 3 | No Charge | FALSE |
| 4 | 4 | Dispute | FALSE |
| 5 | 5 | Unknown | FALSE |
| 6 | 6 | Voided Trip | FALSE |

---

#### gold_dim_rate_code

**Mô tả:** Rate code dimension  
**Grain:** One row per rate code  
**Số bản ghi:** 7

| # | Tên cột | Kiểu dữ liệu | Mô tả |
|---|---------|--------------|-------|
| 1 | 🔑 rate_code_sk | INTEGER | Surrogate key |
| 2 | 🏷️ rate_code_id | INTEGER | Original rate code ID |
| 3 | 🏷️ rate_code_name | STRING | Tên rate code |
| 4 | 🏷️ rate_description | STRING | Mô tả chi tiết |
| 5 | 📊 rate_multiplier | DOUBLE | Hệ số giá |

**Data:**
| rate_code_sk | rate_code_id | rate_code_name | rate_multiplier |
|--------------|--------------|----------------|-----------------|
| 1 | 1 | Standard Rate | 1.0 |
| 2 | 2 | JFK | 1.0 (flat $52) |
| 3 | 3 | Newark | 1.5 |
| 4 | 4 | Nassau/Westchester | 2.0 |
| 5 | 5 | Negotiated Fare | 1.0 |
| 6 | 6 | Group Ride | 0.8 |
| 7 | 99 | Unknown | 1.0 |

---

#### gold_dim_weather

**Mô tả:** Daily weather dimension  
**Grain:** One row per day  
**Số bản ghi:** ~1,155

| # | Tên cột | Kiểu dữ liệu | Mô tả |
|---|---------|--------------|-------|
| 1 | 🔑 weather_sk | INTEGER | Surrogate key (YYYYMMDD) |
| 2 | 🏷️ date | DATE | Ngày |
| 3 | 📊 temp_avg | DOUBLE | Nhiệt độ TB (°F) |
| 4 | 📊 temp_max | DOUBLE | Nhiệt độ cao nhất |
| 5 | 📊 temp_min | DOUBLE | Nhiệt độ thấp nhất |
| 6 | 📊 precipitation | DOUBLE | Lượng mưa (inches) |
| 7 | 📊 snowfall | DOUBLE | Lượng tuyết (inches) |
| 8 | 📊 wind_speed | DOUBLE | Tốc độ gió (mph) |
| 9 | 🏷️ weather_condition | STRING | Điều kiện thời tiết |
| 10 | 🏷️ temp_range | STRING | Khoảng nhiệt độ |
| 11 | 🏷️ is_severe_weather | BOOLEAN | Thời tiết khắc nghiệt |

---

### Fact Tables

#### gold_fact_trips

**Mô tả:** Main fact table chứa chi tiết từng chuyến đi  
**Grain:** One row per taxi trip  
**Số bản ghi:** ~180+ triệu (sau data quality filters)  
**Partitioning:** `date_sk`

| # | Tên cột | Kiểu dữ liệu | Loại | Mô tả |
|---|---------|--------------|------|-------|
| 1 | 🔗 date_sk | INTEGER | FK | → gold_dim_date |
| 2 | 🔗 time_sk | INTEGER | FK | → gold_dim_time |
| 3 | 🔗 pickup_location_sk | INTEGER | FK | → gold_dim_location |
| 4 | 🔗 dropoff_location_sk | INTEGER | FK | → gold_dim_location |
| 5 | 🔗 vendor_sk | INTEGER | FK | → gold_dim_vendor |
| 6 | 🔗 payment_type_sk | INTEGER | FK | → gold_dim_payment_type |
| 7 | 🔗 rate_code_sk | INTEGER | FK | → gold_dim_rate_code |
| 8 | 🔗 weather_sk | INTEGER | FK | → gold_dim_weather |
| 9 | 📊 passenger_count | INTEGER | Measure | Số hành khách |
| 10 | 📊 trip_distance_miles | DOUBLE | Measure | Khoảng cách (miles) |
| 11 | 📊 trip_duration_minutes | DOUBLE | Measure | Thời gian (phút) |
| 12 | 📊 fare_amount | DOUBLE | Measure | Tiền cước (USD) |
| 13 | 📊 tip_amount | DOUBLE | Measure | Tiền tip (USD) |
| 14 | 📊 total_amount | DOUBLE | Measure | Tổng tiền (USD) |
| 15 | 📊 tip_percentage | DOUBLE | Measure | % tip |
| 16 | 🏷️ is_weekend | BOOLEAN | Flag | Cuối tuần |
| 17 | 🏷️ is_rush_hour | BOOLEAN | Flag | Giờ cao điểm |

---

#### gold_fact_trips_daily

**Mô tả:** Pre-aggregated fact table theo ngày  
**Grain:** One row per date + pickup_location + dropoff_location  
**Partitioning:** `date_sk`

| # | Tên cột | Kiểu dữ liệu | Loại | Mô tả |
|---|---------|--------------|------|-------|
| 1 | 🔗 date_sk | INTEGER | FK | → gold_dim_date |
| 2 | 🔗 pickup_location_sk | INTEGER | FK | → gold_dim_location |
| 3 | 🔗 dropoff_location_sk | INTEGER | FK | → gold_dim_location |
| 4 | 📊 trip_count | BIGINT | Measure | Số chuyến đi |
| 5 | 📊 total_passengers | BIGINT | Measure | Tổng hành khách |
| 6 | 📊 total_distance | DOUBLE | Measure | Tổng khoảng cách |
| 7 | 📊 total_duration | DOUBLE | Measure | Tổng thời gian |
| 8 | 📊 total_fare | DOUBLE | Measure | Tổng tiền cước |
| 9 | 📊 total_tips | DOUBLE | Measure | Tổng tips |
| 10 | 📊 total_revenue | DOUBLE | Measure | Tổng doanh thu |
| 11 | 📊 avg_fare | DOUBLE | Measure | Giá cước TB |
| 12 | 📊 avg_tip_pct | DOUBLE | Measure | % tip TB |
| 13 | 📊 avg_distance | DOUBLE | Measure | Khoảng cách TB |
| 14 | 📊 avg_duration | DOUBLE | Measure | Thời gian TB |

---

#### gold_fact_trips_hourly

**Mô tả:** Pre-aggregated fact table theo giờ  
**Grain:** One row per date + time_sk + pickup_location  
**Partitioning:** `date_sk`

| # | Tên cột | Kiểu dữ liệu | Loại | Mô tả |
|---|---------|--------------|------|-------|
| 1 | 🔗 date_sk | INTEGER | FK | → gold_dim_date |
| 2 | 🔗 time_sk | INTEGER | FK | → gold_dim_time |
| 3 | 🔗 pickup_location_sk | INTEGER | FK | → gold_dim_location |
| 4 | 📊 trip_count | BIGINT | Measure | Số chuyến đi |
| 5 | 📊 total_passengers | BIGINT | Measure | Tổng hành khách |
| 6 | 📊 total_revenue | DOUBLE | Measure | Tổng doanh thu |
| 7 | 📊 avg_fare | DOUBLE | Measure | Giá cước TB |

---

## 📋 Lookup Values & Reference Codes

### VendorID Lookup

| Code | Name | Description |
|------|------|-------------|
| 1 | CMT | Creative Mobile Technologies, LLC |
| 2 | VeriFone | VeriFone Inc. |

### RatecodeID Lookup

| Code | Name | Description | Fare Type |
|------|------|-------------|-----------|
| 1 | Standard rate | Giá tiêu chuẩn theo meter | Metered |
| 2 | JFK | JFK Airport flat fare | Flat ($52) |
| 3 | Newark | Newark Airport | Negotiated |
| 4 | Nassau/Westchester | Ngoài NYC | Double meter |
| 5 | Negotiated fare | Giá thỏa thuận | Negotiated |
| 6 | Group ride | Đi chung | Shared |
| 99 | Unknown | Không xác định | Unknown |

### Payment Type Lookup

| Code | Name | Description | Tips Recorded |
|------|------|-------------|---------------|
| 1 | Credit card | Thẻ tín dụng/ghi nợ | ✅ Yes |
| 2 | Cash | Tiền mặt | ❌ No |
| 3 | No charge | Miễn phí | ❌ No |
| 4 | Dispute | Tranh chấp | ❌ No |
| 5 | Unknown | Không xác định | ❌ No |
| 6 | Voided trip | Chuyến hủy | ❌ No |

### Borough Lookup

| Code | Name | Zones Count |
|------|------|-------------|
| Manhattan | Manhattan | 69 |
| Queens | Queens | 69 |
| Brooklyn | Brooklyn | 61 |
| Bronx | Bronx | 43 |
| Staten Island | Staten Island | 20 |
| EWR | Newark Airport | 1 |
| Unknown | Unknown | 2 |

### Time of Day Lookup

| Code | Hours | Description |
|------|-------|-------------|
| Morning | 05:00 - 11:59 | Buổi sáng |
| Afternoon | 12:00 - 16:59 | Buổi chiều |
| Evening | 17:00 - 20:59 | Buổi tối |
| Night | 21:00 - 04:59 | Ban đêm |

### Weather Condition Lookup

| Code | Condition | Priority |
|------|-----------|----------|
| Snow | Có tuyết rơi | 1 (highest) |
| Heavy Rain | Mưa > 0.5 inch | 2 |
| Rain | Mưa <= 0.5 inch | 3 |
| Windy | Gió > 25 mph | 4 |
| Clear | Trời quang | 5 (lowest) |

### Temperature Range Lookup

| Code | Range (°F) | Range (°C) |
|------|------------|------------|
| Freezing | < 32 | < 0 |
| Cold | 32 - 49 | 0 - 9 |
| Mild | 50 - 67 | 10 - 19 |
| Warm | 68 - 85 | 20 - 29 |
| Hot | >= 86 | >= 30 |

### Distance Category Lookup

| Code | Range (miles) | Range (km) |
|------|---------------|------------|
| Short | 0 - 1 | 0 - 1.6 |
| Medium | 1 - 5 | 1.6 - 8 |
| Long | 5 - 15 | 8 - 24 |
| Very Long | > 15 | > 24 |

---

## ✅ Data Quality Rules

### Bronze → Silver Transformation Rules

#### Taxi Trips Filters

| Rule ID | Field | Condition | Action |
|---------|-------|-----------|--------|
| DQ001 | tpep_pickup_datetime | year < 2021 OR year > 2024 | EXCLUDE |
| DQ002 | PULocationID | < 1 OR > 265 | EXCLUDE |
| DQ003 | DOLocationID | < 1 OR > 265 | EXCLUDE |
| DQ004 | trip_distance | < 0 | EXCLUDE |
| DQ005 | trip_distance | > 500 | EXCLUDE |
| DQ006 | fare_amount | < 0 | EXCLUDE |
| DQ007 | fare_amount | > 5000 | EXCLUDE |
| DQ008 | trip_duration | < 0 | EXCLUDE |
| DQ009 | trip_duration | > 1440 (24h) | EXCLUDE |
| DQ010 | passenger_count | < 0 OR > 9 | SET NULL |
| DQ011 | passenger_count | NULL | KEEP (nullable) |
| DQ012 | tip_amount | < 0 | SET 0 |
| DQ013 | total_amount | < fare_amount | RECALCULATE |

#### Null Handling

| Field | Strategy | Default Value |
|-------|----------|---------------|
| passenger_count | Keep NULL | - |
| RatecodeID | Replace | 99 (Unknown) |
| store_and_fwd_flag | Replace | 'N' |
| congestion_surcharge | Replace | 0.00 |
| airport_fee | Replace | 0.00 |

### Referential Integrity Checks

```sql
-- Check for orphan records in fact table
SELECT COUNT(*) as orphan_count
FROM gold_fact_trips f
LEFT JOIN gold_dim_location l ON f.pickup_location_sk = l.location_sk
WHERE l.location_sk IS NULL;

-- Check date range consistency
SELECT 
    MIN(full_date) as min_date,
    MAX(full_date) as max_date
FROM gold_dim_date;
```

---

## 📐 Business Rules & Calculations

### Revenue Calculations

```
Total Revenue = SUM(total_amount)

Fare Revenue = SUM(fare_amount)

Tips Revenue = SUM(tip_amount)
-- Lưu ý: Chỉ bao gồm tips từ credit card

Surcharges = SUM(extra + mta_tax + improvement_surcharge + congestion_surcharge + airport_fee)

Tolls = SUM(tolls_amount)
```

### Trip Metrics

```
Average Fare = SUM(fare_amount) / COUNT(trips)

Average Tip Percentage = 
    SUM(tip_amount) / SUM(fare_amount) * 100
    WHERE payment_type = 1

Average Trip Distance = SUM(trip_distance) / COUNT(trips)

Average Trip Duration = SUM(trip_duration_minutes) / COUNT(trips)

Average Speed = 
    SUM(trip_distance) / (SUM(trip_duration_minutes) / 60)
```

### Time-based Calculations

```
Rush Hour Trips = COUNT(trips) WHERE is_rush_hour = TRUE

Weekend Trips = COUNT(trips) WHERE is_weekend = TRUE

Night Trips = COUNT(trips) WHERE time_of_day = 'Night'
```

### Location Metrics

```
Airport Trips = COUNT(trips) 
    WHERE pickup_location_id IN (1, 132, 138)
    OR dropoff_location_id IN (1, 132, 138)

Manhattan Trips = COUNT(trips)
    WHERE pickup_borough = 'Manhattan'
    OR dropoff_borough = 'Manhattan'

Cross-Borough Trips = COUNT(trips)
    WHERE pickup_borough <> dropoff_borough
```

### Weather Impact

```
Weather Impact Factor = 
    (Avg Trips on Severe Weather Days / Avg Trips on Normal Days) - 1

-- Negative value = weather reduces trips
-- Positive value = weather increases trips (rare)
```

### YoY Growth

```
Revenue YoY Growth % = 
    (Revenue_Current_Year - Revenue_Previous_Year) 
    / Revenue_Previous_Year * 100
```

---

## 📚 Phụ lục

### A. Conversions

| From | To | Formula |
|------|----|---------|
| Miles | Kilometers | km = miles × 1.609 |
| Fahrenheit | Celsius | °C = (°F - 32) × 5/9 |
| Inches | Centimeters | cm = inches × 2.54 |
| MPH | KPH | kph = mph × 1.609 |

### B. NYC Fare Structure (2024)

| Component | Amount |
|-----------|--------|
| Initial charge | $3.00 |
| Per 1/5 mile | $0.70 |
| Per 60 seconds (slow traffic) | $0.70 |
| Rush hour surcharge (4-8 PM weekdays) | $1.00 |
| Overnight surcharge (8 PM - 6 AM) | $0.50 |
| MTA tax | $0.50 |
| Improvement surcharge | $0.30 |
| Congestion surcharge (below 96th St) | $2.50 |
| JFK flat fare | $52.00 |

### C. Glossary

| Term | Definition |
|------|------------|
| LPEP | Livery Passenger Enhancement Program |
| TLC | Taxi and Limousine Commission |
| MTA | Metropolitan Transportation Authority |
| Taximeter | Thiết bị đo khoảng cách và tính tiền |
| Street hail | Bắt taxi ngoài đường |
| Medallion | Giấy phép hoạt động taxi vàng NYC |
| Borough | Quận/khu vực hành chính của NYC |
| Surrogate key | Khóa nhân tạo (không có ý nghĩa nghiệp vụ) |
| Grain | Mức độ chi tiết của một bảng dữ liệu |

---

*Document Version: 1.0*  
*Last Updated: January 2026*  
*Author: NYC Taxi Data Engineering Project*
