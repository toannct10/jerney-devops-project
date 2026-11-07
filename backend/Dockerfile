# ==============================================================
# Stage 1: Build & Dependencies
# ==============================================================
FROM node:20-alpine AS build

WORKDIR /app

# Copy package files để tận dụng Docker layer caching
COPY package.json package-lock.json* ./

# Cài đặt chỉ production dependencies và xóa cache để tối ưu kích thước
RUN npm ci --only=production && npm cache clean --force

# ==============================================================
# Stage 2: Production Runtime
# ==============================================================
FROM node:20-alpine AS production

# Bảo mật: Tạo group và user non-root để không chạy container bằng quyền root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Cài đặt dumb-init để xử lý tín hiệu hệ thống (SIGTERM, SIGINT) khi container chạy PID 1
RUN apk --no-cache add dumb-init

WORKDIR /app

# Sao chép node_modules từ build stage và mã nguồn
COPY --from=build /app/node_modules ./node_modules
COPY src/ ./src/

# Phân quyền thư mục ứng dụng cho non-root user
RUN chown -R appuser:appgroup /app

# Chuyển sang non-root user
USER appuser

EXPOSE 5000

# Khởi chạy ứng dụng qua dumb-init để hỗ trợ graceful shutdown
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "src/index.js"]
