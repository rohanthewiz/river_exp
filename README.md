# Exercising the River Job Processor

(Fast and reliable background jobs in Go)
[River Docs](https://riverqueue.com/docs)

## Setup

- Download and resolve dependencies: In the project root run `go mod tidy`
- Get the `river` command line tool `go install github.com/riverqueue/river/cmd/river@latest`

## DB Migration for River

- `river migrate-up --database-url "postgres://{user}:{password}@localhost:5432/postgres?sslmode=disable"`

## Running

- `go build && PG_CONN_STR="postgres://{user}:{password}@localhost:5432/postgres?sslmode=disable" ./river_exp`

### Results ()

```text
Waiting for some jobs to complete...
Sorted strings: [cat dog goat horse mouse whale]
Sorted strings: [bear tiger whale]
Period job called: 2025-04-17 19:36:16.837697 -0500 CDT m=+0.201824709
Period job called: 2025-04-17 19:36:26.8466 -0500 CDT m=+10.210929917
Adding a new periodic job...
Waiting for more jobs to complete...
Period job called: 2025-04-17 19:36:32.01759 -0500 CDT m=+15.382024167
Period job called: 2025-04-17 19:36:37.021141 -0500 CDT m=+20.385676209
Period job called: 2025-04-17 19:36:42.025014 -0500 CDT m=+25.389649876
```
