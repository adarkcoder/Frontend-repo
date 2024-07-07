# Use a specific Node.js version for the build stage
FROM node:14 AS build

# Set the working directory
WORKDIR /app

# Copy package.json and package-lock.json
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy the rest of the application code
COPY . .

# Build the React app
RUN npm run build

# Use the official Nginx image to serve the app
FROM nginx:alpine

# Expose port 8080
EXPOSE 8080

# Ensure the necessary directories exist and have the right permissions
RUN mkdir -p /tmp/nginx && \
    chown -R nginx:nginx /usr/share/nginx /etc/nginx /var/cache/nginx  /tmp/nginx 

COPY nginx.conf /etc/nginx/nginx.conf

# Copy the build output to Nginx's html directory
COPY --from=build /app/build /usr/share/nginx/html

# Switch to the non-root user
USER nginx

# Start Nginx server
CMD ["nginx", "-g", "daemon off;"]
