## 8) Movie Recommendation Flow (hien tai)

Chuc nang recommend hien tai uu tien theo du lieu hanh vi user va fallback theo thu tu:

1. `INTERACTION_BASED` (neu co du lieu trong `user_movie_interactions`)
2. `CONTENT_BASED_HISTORY` (neu khong co interactions nhung co `watch_history`)
3. `GENRE_FALLBACK` (neu khong co interactions/history)

### 8.1 Nguon du lieu

- `user_movie_interactions`:
  - `MOVIE_SELECTED`, score = 2 (khi user mo chi tiet phim qua `POST /api/movies/select`)
  - `BOOKING_CONFIRMED`, score = 5 (khi booking thanh toan thanh cong)
- `watch_history`
- `user_genre_preferences`
- `movie_genres`
- `movies`
- `movie_recommendations` (cache snapshot async, khong phai nguon tinh truc tiep cho API sync)

### 8.2 Cong thuc tinh score

#### A) Interaction-Based (`INTERACTION_BASED`)

Voi moi genre `g`:

`weight(g) = sum(score_interaction(m))` voi moi phim tuong tac `m` co chua genre `g`.

Voi phim ung vien `c`:

`score(c) = sum(weight(g))` voi moi genre `g` cua phim `c`.

Loai bo cac phim da co trong tap phim user da tuong tac.

#### B) Content-Based tu Watch History (`CONTENT_BASED_HISTORY`)

Voi moi genre `g`:

`weight(g) = so lan g xuat hien trong cac phim da co trong watch_history`.

Voi phim ung vien `c`:

`score(c) = sum(weight(g))` voi moi genre `g` cua phim `c`.

Loai bo cac phim da xuat hien trong `watch_history`.

#### C) Genre Fallback (`GENRE_FALLBACK`)

`score(c) = so genre cua phim c trung voi user_genre_preferences`.

Moi genre trung tinh +1 diem.

### 8.3 Luong API sync cho client

1. App goi `GET /api/recommendations/genres` de lay danh sach genre.
2. App goi `POST /api/recommendations/select/genres` de luu preference.
3. Khi user mo chi tiet phim, app goi `POST /api/movies/select`:
   - luu `watch_history` (neu chua ton tai),
   - luu interaction `MOVIE_SELECTED`.
4. Khi booking thanh cong:
   - luu interaction `BOOKING_CONFIRMED`.
5. App goi `GET /api/recommendations/movies?limit=...`:
   - backend tu dong chon strategy theo thu tu uu tien o tren,
   - tra ve `movieId`, `title`, `posterUrl`, `status`, `score`, `strategy`.

### 8.4 Luong async qua RabbitMQ

- Producer gui event recommend khi:
  - user chon genre (`GENRES_SELECTED`)
  - user chon phim (`MOVIE_SELECTED`)
- Consumer (`RecommendationConsumer`) nhan event:
  - goi `processRecommendationEvent(userId, limit)`,
  - xoa recommendation cache cu cua user trong `movie_recommendations`,
  - luu snapshot recommendation moi.

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
[
\text{TF}(t,d) = \frac{\text{Số lần từ t xuất hiện trong phim}}{\text{Tổng số từ trong phim}}
]
Ví dụ phim A: genres = `Action, Drama`

- TF(Action) = 1/2 = 0.5
- TF(Drama) = 1/2 = 0.5
- TF(Comedy) = 0/2 = 0

#### Bước 2: Tính IDF (Inverse Document Frequency)

Công thức:
[
\text{IDF}(t) = \log \frac{N}{1 + n_t}
]

- N = tổng số phim = 3
- n_t = số phim có chứa từ t
- Action xuất hiện ở A,C → IDF(Action) ≈ 0.176
- Drama xuất hiện ở A,B → IDF(Drama) ≈ 0.176
- Comedy xuất hiện ở C → IDF(Comedy) ≈ 0.477

#### Bước 3: Tính TF-IDF

[
\text{TF-IDF}(t,d) = \text{TF}(t,d) \times \text{IDF}(t)
]

- Phim A → [0.088, 0.088, 0]
- Phim B → [0, 0.176, 0]
- Phim C → [0.088, 0, 0.238]

#### Bước 4: Chuẩn hóa vector

Để dùng cosine similarity, vector được normalize:
[
v_{normalized} = \frac{v}{\sqrt{\sum_i v_i^2}}
]

- Phim A → [0.6, 0.6, 0]
- Phim B → [0, 1.0, 0]
- Phim C → [0.34, 0, 0.94]

### 2. Tính độ tương đồng cosine

Cosine similarity = (A · B) / (||A|| \* ||B||)

- Cosine(A,B) = 0.6 × 0 + 0.6 × 1 + 0 × 0 = 0.6
- Cosine(A,C) = 0.6 × 0.34 + 0.6 × 0 + 0 × 0.94 = 0.204

### 3. Kết quả gợi ý

- Nếu user thích phim A, hệ thống sẽ gợi ý phim có cosine similarity cao nhất → B (0.6) trước C (0.204).

---

## 8.2.A Interaction-Based (`INTERACTION_BASED`)

### Ý tưởng

Sử dụng các **interaction của user với phim** (click, booking) để suy ra mức độ quan tâm của user đối với từng **genre**.

---

### Bước 1: Tính trọng số cho từng genre

Với mỗi genre `g`:

\[
weight(g) = \sum score(m)
\]

Trong đó:

- `m`: các phim user đã tương tác
- `score(m)`:
  - `MOVIE_SELECTED` = 2
  - `BOOKING_CONFIRMED` = 5

👉 Nếu một phim có nhiều genre → cộng điểm cho **tất cả genre đó**

---

### Ví dụ

User có interaction:

| Movie | Genres        | Interaction       | Score |
| ----- | ------------- | ----------------- | ----- |
| A     | Action, Drama | BOOKING_CONFIRMED | 5     |
| B     | Drama         | MOVIE_SELECTED    | 2     |

→ Tính weight:

- Action = 5
- Drama = 5 + 2 = 7

---

### Bước 2: Tính score cho phim candidate

Với mỗi phim ứng viên `c`:

\[
score(c) = \sum weight(g)
\]

với `g` là các genre của phim `c`.

---

### Ví dụ

Phim C có genres: `Action, Comedy`

- Action = 5
- Comedy = 0

→  
\[
score(C) = 5
\]

---

### (Optional – nên dùng cho demo đẹp hơn)

Chuẩn hóa theo số genre:

\[
score(c) = \frac{\sum weight(g)}{\text{số genre của c}}
\]

---

### Bước 3: Lọc dữ liệu

- Loại bỏ các phim user đã tương tác
- Sắp xếp theo `score` giảm dần
- Lấy top N phim

---

### Kết quả

Hệ thống sẽ recommend các phim có nhiều **genre trùng với sở thích đã thể hiện qua interaction**, với trọng số ưu tiên:

- Booking > Click
- Tương tác nhiều → weight cao hơn
