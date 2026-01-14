.PHONY: help build test lint run clean docker-build

help:
	@echo "Available targets:"
	@echo "  build        - Build the application"
	@echo "  test         - Run tests with race detector"
	@echo "  lint         - Run linters"
	@echo "  run          - Run the application"
	@echo "  clean        - Clean build artifacts"
	@echo "  docker-build - Build Docker image"

build:
	go build -v -o bin/api ./cmd/api

test:
	go test -v -race -coverprofile=coverage.out ./...

lint:
	golangci-lint run --timeout=5m

run:
	go run ./cmd/api

clean:
	rm -rf bin/
	rm -f coverage.out

docker-build:
	docker build -t api:latest .
