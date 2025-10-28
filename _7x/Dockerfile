# 运行阶段
FROM nginx:latest  AS client-server

WORKDIR /usr/share/nginx/html

COPY  ./zh/dist ./
COPY  ./nginx.conf /etc/nginx/nginx.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
