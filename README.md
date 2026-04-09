# Content-Based Filtering Example

## Ví dụ Content-Based Filtering

Giả sử bạn có 3 phim với genres:

| Film | Genres         |
| ---- | -------------- |
| A    | Action, Drama  |
| B    | Drama          |
| C    | Action, Comedy |

### 1. Tính TF-IDF từ genres

#### Bước 1: Tính TF (Term Frequency)

Công thức:

```
TF(t,d) = (số lần t xuất hiện trong phim) / (tổng số từ trong phim)
```

Ví dụ phim A: genres = `Action, Drama`

* TF(Action) = 1/2 = 0.5
* TF(Drama) = 1/2 = 0.5
* TF(Comedy) = 0/2 = 0

#### Bước 2: Tính IDF (Inverse Document Frequency)

Công thức:

```
IDF(t) = log(N / (1 + n_t))
```

* N = tổng số phim = 3
* n_t = số phim có chứa từ t
* Action xuất hiện ở A,C → IDF(Action) ≈ 0.176
* Drama xuất hiện ở A,B → IDF(Drama) ≈ 0.176
* Comedy xuất hiện ở C → IDF(Comedy) ≈ 0.477

#### Bước 3: Tính TF-IDF

```
TF-IDF(t,d) = TF(t,d) * IDF(t)
```

* Phim A → [0.088, 0.088, 0]
* Phim B → [0, 0.176, 0]
* Phim C → [0.088, 0, 0.238]

#### Bước 4: Chuẩn hóa vector

Để dùng cosine similarity, vector được normalize:

```
v_normalized = v / sqrt(sum(v_i^2))
```

* Phim A → [0.6, 0.6, 0]
* Phim B → [0, 1.0, 0]
* Phim C → [0.34, 0, 0.94]

### 2. Tính độ tương đồng cosine

```
cosine(A,B) = (A · B) / (||A|| * ||B||)
```

* Cosine(A,B) = 0.6 × 0 + 0.6 × 1 + 0 × 0 = 0.6
* Cosine(A,C) = 0.6 × 0.34 + 0.6 × 0 + 0 × 0.94 = 0.204

### 3. Kết quả gợi ý

* Nếu user thích phim A, hệ thống sẽ gợi ý phim có cosine similarity cao nhất → B (0.6) trước C (0.204)

---

## 8.2.A Interaction-Based (`INTERACTION_BASED`)

### Ý tưởng

Sử dụng các **interaction của user với phim** (click, booking) để suy ra mức độ quan tâm của user đối với từng **genre**.

---

### Bước 1: Tính trọng số cho từng genre

Với mỗi genre `g`:

```
weight(g) = sum(score(m))
```

Trong đó:

* `m`: các phim user đã tương tác
* `score(m)`:

  * `MOVIE_SELECTED` = 2
  * `BOOKING_CONFIRMED` = 5

👉 Nếu một phim có nhiều genre → cộng điểm cho tất cả genre đó

---

### Ví dụ

| Movie | Genres        | Interaction       | Score |
| ----- | ------------- | ----------------- | ----- |
| A     | Action, Drama | BOOKING_CONFIRMED | 5     |
| B     | Drama         | MOVIE_SELECTED    | 2     |

→ Weight:

* Action = 5
* Drama = 5 + 2 = 7

---

### Bước 2: Tính score cho phim candidate

```
score(c) = sum(weight(g))
```

---

### Ví dụ

Phim C: `Action, Comedy`

* Action = 5
* Comedy = 0

→ score(C) = 5

---

### (Optional)

```
score(c) = sum(weight(g)) / số_genre
```

---

### Bước 3: Lọc

* Loại phim đã tương tác
* Sort giảm dần theo score
* Lấy top N

---

### Kết quả

* Ưu tiên phim cùng genre user thích
* Booking > Click
* Interaction nhiều → score cao hơn
