# Lightweight Nginx image serving the static HTML page
FROM nginx:alpine

# Replace the default Nginx page with our application
COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
