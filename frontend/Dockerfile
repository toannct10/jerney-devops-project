# ==============================================================
# Stage 1: Build Frontend (Vite + React)
# ==============================================================
FROM node:20-alpine AS build

WORKDIR /app

# Copy package files để tận dụng Docker layer cache
COPY package.json package-lock.json* ./

RUN npm ci && npm cache clean --force

COPY . .

# Build ứng dụng React thành static files (HTML/CSS/JS) trong thư mục dist/
RUN npm run build

# ==============================================================
# Stage 2: Production Web Server (Nginx Alpine)
# ==============================================================
FROM nginx:1.27-alpine AS production

# Dọn dẹp cấu hình và static files mặc định của Nginx
RUN rm -rf /etc/nginx/conf.d/default.conf /usr/share/nginx/html/*

# Sao chép cấu hình Nginx tùy chỉnh (reverse proxy /api sang backend, listen port 8080)
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Sao chép bản build tĩnh từ Stage 1 sang thư mục web root của Nginx
COPY --from=build /app/dist /usr/share/nginx/html

# Bảo mật: Cấu hình phân quyền để Nginx có thể chạy dưới user không phải root (user 'nginx')
RUN chown -R nginx:nginx /usr/share/nginx/html && \
    chown -R nginx:nginx /var/cache/nginx && \
    chown -R nginx:nginx /var/log/nginx && \
    touch /var/run/nginx.pid && \
    chown -R nginx:nginx /var/run/nginx.pid

# Chuyển sang non-root user 'nginx' (UID 101)
USER nginx

# Mở port 8080 (vì non-root user không được bind port < 1024)
EXPOSE 8080

CMD ["nginx", "-g", "daemon off;"]
