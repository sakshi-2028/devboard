FROM golang:1.26
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o backend .
EXPOSE 8081
CMD ["./backend"]