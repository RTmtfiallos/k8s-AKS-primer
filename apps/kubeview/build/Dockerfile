# ================================================================================================
# === Stage 1: Build and bundle the Vue.js app with Vue CLI 3 ====================================
# ================================================================================================
FROM node:10-alpine as vue-build
ARG sourceDir="web/client"

WORKDIR /build

# Install all the Vue.js dev tools & CLI, and our app dependencies 
COPY ${sourceDir}/package*.json ./
RUN npm install --silent

# Copy in the Vue.js app source
COPY ${sourceDir}/.env.production .
COPY ${sourceDir}/.eslintrc.js .
COPY ${sourceDir}/public ./public
COPY ${sourceDir}/src ./src

# Run Vue CLI build & bundle, and output to ./dist
RUN npm run build

# ================================================================================================
# === Stage 2: Build Golang API server and host for Vue app ======================================
# ================================================================================================
FROM golang:1.12-alpine as go-build
WORKDIR /build
ARG goPackage="github.com/benc-uk/kubeview/cmd/server"
ARG version="0.1.7"
ARG buildInfo="Local Docker build"

ENV PORT 8000

# Install system dependencies
RUN apk update && apk add git gcc musl-dev

# Fetch and cache Go modules
COPY go.mod .
COPY go.sum .
RUN go mod download

# Copy in Go source files
COPY cmd/ ./cmd
#COPY pkg/ ./pkg

# Now run the build
# Disabling cgo results in a fully static binary that can run without C libs
# Also inject version and build details 
RUN GO111MODULE=on CGO_ENABLED=0 GOOS=linux go build \
    -ldflags "-X main.version=$version -X 'main.buildInfo=$buildInfo'" \
    -o server \
    $goPackage

# ================================================================================================
# === Stage 3: Bundle server exe and Vue dist in runtime image ===================================
# ================================================================================================
FROM scratch
WORKDIR /app 
EXPOSE 8000

# Copy in output from Vue bundle (the dist)
# Copy the server binary
COPY --from=vue-build /build/dist ./frontend
COPY --from=go-build /build/server . 

# That's it! Just run the server with incluster mode enabled
ENV IN_CLUSTER=true
CMD [ "./server"]