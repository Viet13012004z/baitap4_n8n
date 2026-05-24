# baitap4_n8n
# Nguyễn Hoàng Việt - k225480106074 
# Bài tập đăng bài tự động wp-n8n tele
# 1. Cấu hình file docker-compose.yml ( bổ sung n8n) 

```
version: '3.8'

services:
  mariadb:
    image: mariadb:latest
    container_name: mariadb_db
    environment:
      TZ: "Asia/Ho_Chi_Minh"
      MARIADB_ROOT_PASSWORD: "root_password_cua_ban"
      MARIADB_DATABASE: "wordpress_db"
      MARIADB_USER: "wp_user"
      MARIADB_PASSWORD: "wp_password_cua_ban"
    volumes:
      - db_data:/var/lib/mysql
    restart: always

  phpmyadmin:
    image: phpmyadmin:latest
    container_name: phpmyadmin_web
    environment:
      PMA_HOST: mariadb
      PMA_ARBITRARY: 1
    ports:
      - "8080:80"
    restart: always
    depends_on:
      - mariadb

  wordpress:
    image: wordpress:latest
    container_name: wordpress_web
    environment:
      WORDPRESS_DB_HOST: mariadb
      WORDPRESS_DB_NAME: wordpress_db
      WORDPRESS_DB_USER: wp_user
      WORDPRESS_DB_PASSWORD: wp_password_cua_ban
    ports:
      - "8000:80"
    volumes:
      - wp_data:/var/www/html
    restart: always
    depends_on:
      - mariadb

  n8n:
    image: n8nio/n8n:latest
    container_name: n8n_automation
    environment:
      - TZ=Asia/Ho_Chi_Minh
      - WEBHOOK_URL=https://sub-domain3.cua-ban.com/ # THAY BẰNG SUB-DOMAIN 3 CỦA BẠN
    ports:
      - "5678:5678"
    volumes:
      - n8n_data:/home/node/.n8n
    restart: always

  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflare_tunnel
    restart: always
    command: tunnel run --token <TOKEN_TUNNEL_CỦA_BẠN>

volumes:
  db_data:
  wp_data:
  n8n_data:
  ```
# 2. Cấu hình cloudfare tunnel 

<img width="1485" height="820" alt="image" src="https://github.com/user-attachments/assets/b1a69e7b-6664-4c2d-9a5d-879df1230bd8" />

# 3. Kiểm tra php, wp, n8n xem đã truy cập được chưa 

<img width="1912" height="1079" alt="image" src="https://github.com/user-attachments/assets/22dbdad0-e4db-4bee-a743-77135f72219b" />

<img width="1907" height="1079" alt="image" src="https://github.com/user-attachments/assets/4e5e3c65-ce6f-49ad-8e92-569121cb2ec3" />

<img width="1908" height="1076" alt="image" src="https://github.com/user-attachments/assets/1b3f08fe-3836-44a9-a3ce-299ff42eb404" />

# 4. Chuẩn bị API và thông tin kết nối 

# telegram bot token: 

<img width="1755" height="1079" alt="image" src="https://github.com/user-attachments/assets/10a9989b-8cab-4ebd-9f93-031f5aa26d7e" />

# Gemini api key: Truy cập vào Google AI Studio, nhấn Create API Key và lưu lại chuỗi khóa.

<img width="1918" height="1031" alt="image" src="https://github.com/user-attachments/assets/cfa5f170-3455-4bf2-a962-2f10fb9ea98a" />

# WordPress Application Password:

<img width="1913" height="1077" alt="image" src="https://github.com/user-attachments/assets/1c42dbbb-567a-43cf-9aa0-05172dca3062" />

# Xây dựng Workflow trong n8n 

# Node 1: telegram trigger

- Dán api token lấy được từ @botfather

<img width="1917" height="1079" alt="image" src="https://github.com/user-attachments/assets/b89880b9-f979-4f85-bb4c-48287b4265e6" />

# Node 2: Message a model - Gemini 

- Lấy API key từ google ai studio dán vào 

<img width="1319" height="714" alt="image" src="https://github.com/user-attachments/assets/9158b66f-49a9-4bf6-8122-e74f0660bfb4" />

# Node 3: Code in JavaScript 

```
const items = $input.all();
// 1. Lấy dữ liệu văn bản thô từ Gemini trả về
const rawText = items[0].json.content?.parts?.[0]?.text || "";

try {
  // 2. Tìm kiếm điểm bắt đầu và kết thúc của chuỗi JSON thực tế ({ })
  const start = rawText.indexOf('{');
  const end = rawText.lastIndexOf('}');
  const jsonStr = rawText.substring(start, end + 1);
  
  // 3. Giải mã chuỗi JSON thành Object
  const data = JSON.parse(jsonStr);

  // 4. Trích xuất Tiêu đề bài viết
  const title = data.title || data.post_title || "Bài viết mới từ Bot";

  // 5. Tự động chuyển đổi dữ liệu cấu trúc thành định dạng HTML sạch
  let contentHtml = "";
  const bodyData = data.content || data.blog_post || data.blog_content;
  
  if (typeof bodyData === 'object' && bodyData !== null) {
    if (bodyData.introduction) {
      contentHtml += `<p style="font-size: 16px; line-height: 1.6; color: #333;"><i>${bodyData.introduction}</i></p>`;
    }
    if (bodyData.sections && Array.isArray(bodyData.sections)) {
      bodyData.sections.forEach(s => {
        contentHtml += `<h2 style="color: #1a1a1a; margin-top: 20px;">${s.heading || ""}</h2>`;
        contentHtml += `<p style="line-height: 1.6; color: #444;">${(s.body || "").replace(/\n/g, "<br>")}</p>`;
      });
    }
  } else {
    contentHtml = bodyData || rawText;
  }

  // 6. Trả ra 2 trường chuẩn chỉnh cho WordPress nhận
  return {
    title: title,
    content: contentHtml
  };

} catch (e) {
  // Nếu có lỗi parse hệ thống vẫn tự đăng text thô để không bị mất bài viết
  return {
    title: "Bài viết mới",
    content: rawText.replace(/```json/g, "").replace(/```/g, "")
  };
}
```
# Node 4: Wordpress - Create a post

- Dán mk lấy từ WordPress Application Password:

<img width="1917" height="1077" alt="image" src="https://github.com/user-attachments/assets/24aba2fd-e463-49dc-b1e1-1a7d7774d788" />

# Workflow n8n 

<img width="1918" height="1079" alt="image" src="https://github.com/user-attachments/assets/4ab13a4a-7107-48bb-b4ad-50f527668999" />

# Sau khi cấu hình xong hết để đăng bài tự động ấn Publish và nhắn tin với con bot tele mà mình tạo để tạo ra bài viết mới theo yêu cầu 

<img width="1912" height="1078" alt="image" src="https://github.com/user-attachments/assets/4bcaa871-6a9a-47f3-b99c-d48f56e2c571" />

<img width="1917" height="1078" alt="image" src="https://github.com/user-attachments/assets/da47dc44-70c9-48ef-ace9-149875571a38" />

<img width="1917" height="1079" alt="image" src="https://github.com/user-attachments/assets/81a8ab40-59f4-46d9-9c44-648e4e8168a7" />

<img width="870" height="1885" alt="image" src="https://github.com/user-attachments/assets/4a9f0323-0510-44b3-8c0b-7732ff190f55" />



