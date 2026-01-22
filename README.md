# vuk3r.github.io (Hexo + Butterfly)

## Yêu cầu

- Node.js (khuyến nghị LTS)
- Git

## Cài đặt lần đầu

```bash
npm install
```

## Thêm bài viết mới

- Bài viết nằm ở: `source/_posts/`

Tạo bài mới (tự sinh front-matter):

```bash
npx hexo new post "tieu-de-bai-viet"
```

Sau đó sửa nội dung trong file mới tạo ở `source/_posts/`.

## Chạy local để xem trước

```bash
npm run clean
npm run server
```

Mở `http://localhost:4000/`.

## Build để deploy

```bash
npm run clean
npm run build
```

Output nằm ở thư mục `public/`.

## Deploy lên GitHub Pages

Đảm bảo `_config.yml` đã đúng:

- `url: https://vuk3r.github.io`
- `deploy.repo: https://github.com/vuk3r/vuk3r.github.io.git`
- `deploy.branch: main`

Chạy deploy:

```bash
npm run deploy
```

Nếu Git yêu cầu đăng nhập, dùng **GitHub Personal Access Token (PAT)** thay cho password:

- GitHub → Settings → Developer settings → Personal access tokens
- Tạo token và cấp quyền phù hợp (thường cần quyền push repo)
- Khi Git hỏi mật khẩu, dán token vào

## Theme

Theme đang dùng: `butterfly` (cấu hình chính ở `_config.butterfly.yml`).


Quy trình làm việc từ giờ:
Làm việc trên main (viết bài, sửa config, ...)
Commit và push lên main:
   git add .   git commit -m "Your message"   git push origin main
Workflow tự động build và deploy lên gh-pages
Site sẽ có tại https://vuk3r.github.io
Workflow đã sẵn sàng. Source code được giữ trên main, và site được deploy từ gh-pages.