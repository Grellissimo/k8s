FROM nginx:alpine

# Copy custom landing page into default NGINX HTML root
COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
