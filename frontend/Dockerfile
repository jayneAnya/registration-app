# Build stage
FROM node:18-alpine as build

WORKDIR /app

ARG REACT_APP_API_URL
ENV REACT_APP_API_URL=$REACT_APP_API_URL

# Add network configurations
ENV NODE_OPTIONS=--max-old-space-size=4096
ENV npm_config_registry=https://registry.npmjs.org/

# Copy package files
COPY package*.json ./
COPY .npmrc ./

# Clean install dependencies
RUN npm cache clean --force && \
    rm -rf node_modules && \
    npm install --legacy-peer-deps

# Copy the rest of the application
COPY . .

# Build the app
RUN npm run build && ls -l build && test -f build/index.html

# Production stage
FROM nginx:alpine

# Copy the build output
COPY --from=build /app/build /usr/share/nginx/html

# Copy nginx configuration
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 3000

CMD ["nginx", "-g", "daemon off;"] 



# # Single-stage: build React and serve with Nginx in one image
# FROM nginx:alpine

# # Working directory for building
# WORKDIR /app

# # Build-time API URL
# ARG REACT_APP_API_URL
# ENV REACT_APP_API_URL=$REACT_APP_API_URL

# # Install Node + npm to build the app
# RUN apk add --no-cache nodejs npm

# # Copy only package files first for better caching
# COPY package*.json ./
# COPY .npmrc ./

# # Install dependencies
# RUN npm cache clean --force && \
#     rm -rf node_modules && \
#     npm install --legacy-peer-deps

# # Copy the rest of the source
# COPY . .

# # Build the production assets
# RUN npm run build && ls -l build && test -f build/index.html

# # Move built files to Nginx web root
# RUN rm -rf /usr/share/nginx/html && \
#     mv build /usr/share/nginx/html

# # Copy Nginx config
# COPY nginx.conf /etc/nginx/conf.d/default.conf

# # Expose Nginx port
# EXPOSE 3000

# # Run Nginx in foreground
# CMD ["nginx", "-g", "daemon off;"]

