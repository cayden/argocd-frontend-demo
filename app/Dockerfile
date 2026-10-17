# 极简静态前端镜像：基于 nginx:alpine
FROM nginx:1.27-alpine

# 清空默认页面，放入我们的 index.html
RUN rm -rf /usr/share/nginx/html/*
COPY index.html /usr/share/nginx/html/

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
