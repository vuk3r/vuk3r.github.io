# vuk3r.github.io (Hexo + Butterfly)

## Yêu cầu

- Node.js (khuyến nghị LTS)
- Git

## Cài đặt lần đầu

```bash
npm install
```

## Quy trình viết post mới

### Bước 1: Tạo file post mới

Tạo file Markdown mới với front-matter tự động:

```bash
npx hexo new post "tieu-de-bai-viet"
```

File sẽ được tạo tại: `source/_posts/tieu-de-bai-viet.md`

### Bước 2: Viết nội dung

Mở file vừa tạo và chỉnh sửa:

- Sửa front-matter (title, date, tags, categories, ...)
- Viết nội dung Markdown bên dưới

### Bước 3: Xem trước (tùy chọn)

Chạy server local để xem trước:

```bash
npm run clean
npm run server
```

Mở `http://localhost:4000/` để xem.

### Bước 4: Push lên GitHub để cập nhật

Commit và push lên `main`:

```bash
git add source/_posts/tieu-de-bai-viet.md
git commit -m "Add post: tieu-de-bai-viet"
git push origin main
```

**GitHub Actions sẽ tự động:**
- Build site từ source code
- Deploy lên branch `gh-pages`
- Site sẽ cập nhật tại `https://vuk3r.github.io`

## Xóa post

Xóa file Markdown trong `source/_posts/`, sau đó:

```bash
git add source/_posts/
git commit -m "Remove post: ten-bai"
git push origin main
```

## Theme

Theme đang dùng: `butterfly` (cấu hình chính ở `_config.butterfly.yml`).

## Cấu trúc branch

- **`main`**: Chứa source code (Markdown, config, themes)
- **`gh-pages`**: Chứa built files (HTML, CSS, JS) - tự động tạo bởi workflow