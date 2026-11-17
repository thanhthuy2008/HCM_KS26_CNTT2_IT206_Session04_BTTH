# BTTH Session 04: Chuỗi prompt xử lý đơn hàng

## 1. Bài toán

Tính tổng tiền từ đơn hàng viết tự do. Dữ liệu có thể dị biệt (thiếu giá, số lượng âm). Một đơn lỗi **không được làm crash** cả lô.

## 2. Thiết kế

```text
Đơn thô → P1 Trích xuất JSON → Validate (code) → P2 Tính (CoT) → P3 Tính lại → So khớp → Kết quả
Lỗi ở bước nào → ghi nhãn lỗi, bỏ qua đơn đó, chạy tiếp
```

| Bước | Vai trò | Kỹ thuật |
|---|---|---|
| P1 | Văn bản → JSON | Cấm đoán số |
| Validate | Chặn dữ liệu dị biệt | Code, 0 token |
| P2 | Tính từng bước | Chain-of-Thought |
| P3 | Tính lại cách khác | Kiểm chứng chéo |
| So khớp | Lệch → retry 1 lần, vẫn lệch → báo lỗi | Xử lý đứt gãy |

## 3. Prompt

**P1:**
```text
Trích xuất đơn hàng thành JSON, không giải thích. Thiếu trường thì để null, không tự đoán.
Dạng: {"items":[{"name":str,"qty":int,"price":int}],"discount_pct":int|null,"discount_min":int|null,"ship":int|null}
Quy đổi: 250k=250000, 1tr5=1500000.
Đơn: {order}
```

**P2:**
```text
Dữ liệu: {json}
Tính từng dòng: tạm tính → giảm giá (chỉ khi tạm tính >= discount_min) → cộng ship.
Dòng cuối: TONG=<số nguyên>
```

**P3:**
```text
Dữ liệu: {json}
Tính bằng cách khác: nhân hệ số (1 - discount_pct/100), rồi cộng ship.
Chỉ trả: TONG=<số nguyên>
```

## 4. Code điều phối

```python
import json, re

def total(t):
    m = re.search(r"TONG=(\d+)", t)
    if not m: raise ValueError("Thiếu TONG")
    return int(m.group(1))

def validate(d):
    if not d.get("items"): raise ValueError("Không có món")
    for i in d["items"]:
        if not isinstance(i.get("qty"), int) or i["qty"] <= 0: raise ValueError("Sai số lượng")
        if not isinstance(i.get("price"), int) or i["price"] <= 0: raise ValueError("Thiếu/sai giá")
    if d.get("ship") is None: raise ValueError("Thiếu ship")

def process(order):
    try:
        d = json.loads(call_llm(P1.format(order=order)))
        validate(d)
        for _ in range(2):  # retry 1 lần
            a, b = total(call_llm(P2.format(json=d))), total(call_llm(P3.format(json=d)))
            if a == b: return {"status": "OK", "total": a}
        return {"status": "LOI", "reason": "Không xác minh được"}
    except Exception as e:  # 1 đơn lỗi không làm sập lô
        return {"status": "LOI", "reason": str(e)}
```

## 5. Kết quả chạy thử

| # | Đầu vào | Kết quả |
|---|---|---|
| 1 | 3 áo 250k, 2 quần 400k, giảm 10% đơn từ 1tr5, ship 30k | OK: **1.425.000 VNĐ** |
| 2 | 2 áo, giảm 10% | LOI: thiếu giá |
| 3 | -1 áo 250k, ship 30k | LOI: sai số lượng |

Đơn 1: tạm tính 1.550.000 (đủ điều kiện) → giảm còn 1.395.000 → cộng ship = 1.425.000. P3 cho cùng kết quả ✔

