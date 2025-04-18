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

## Sample Code

```go
package main

import (
	"context"
	"fmt"
	"log/slog"
	"os"
	"sort"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"

	"github.com/riverqueue/river"
	"github.com/riverqueue/river/riverdriver/riverpgxv5"
	"github.com/riverqueue/river/rivershared/util/slogutil"
)

// pgConnStr can look something like this:
// `postgres://<user>:<password>@localhost:5432/postgres?sslmode=disable`
var pgConnStr = os.Getenv("PG_CONN_STR")

// ONE-TIME JOB DEFINITION
type SortArgs struct {
	// Strings is a slice of strings to sort.
	Strings []string `json:"strings"`
}

func (SortArgs) Kind() string { return "SortJob" }

type SortWorker struct {
	river.WorkerDefaults[SortArgs]
}

func (w *SortWorker) Work(ctx context.Context, job *river.Job[SortArgs]) error {
	sort.Strings(job.Args.Strings)
	fmt.Printf("Sorted strings: %+v\n", job.Args.Strings)
	return nil
}

// PERIODIC JOB DEFINITION
type PeriodicJobArgs struct{}

// Kind is the unique string name for this job.
func (PeriodicJobArgs) Kind() string { return "PeriodicJob" }

// PeriodicJobWorker is a job worker for sorting strings.
type PeriodicJobWorker struct {
	river.WorkerDefaults[PeriodicJobArgs]
}

func (w *PeriodicJobWorker) Work(ctx context.Context, job *river.Job[PeriodicJobArgs]) error {
	fmt.Printf("Period job called: %s\n", time.Now().String())
	return nil
}

// This example demonstrates how to register job types on workers, start a
// client, and insert jobs to be worked.
func main() {
	ctx := context.Background()

	dbPool, err := pgxpool.New(ctx, pgConnStr)
	if err != nil {
		handleCriticalError(err)
	}
	defer dbPool.Close()

	// Add worker (job) types to the workers pool
	workers := river.NewWorkers()
	river.AddWorker(workers, &SortWorker{})
	river.AddWorker(workers, &PeriodicJobWorker{})

	// A Client will run job instances
	riverClient, err := river.NewClient(riverpgxv5.New(dbPool), &river.Config{
		Logger: slog.New(&slogutil.SlogMessageOnlyHandler{Level: slog.LevelWarn}),
		PeriodicJobs: []*river.PeriodicJob{
			river.NewPeriodicJob(
				river.PeriodicInterval(10*time.Second),
				func() (river.JobArgs, *river.InsertOpts) {
					return PeriodicJobArgs{}, nil
				},
				&river.PeriodicJobOpts{RunOnStart: true},
			),
		},
		Queues: map[string]river.QueueConfig{
			river.QueueDefault: {MaxWorkers: 10},
		},
		TestOnly: true, // suitable only for use in tests; remove for live environments
		Workers:  workers,
	})
	if err != nil {
		handleCriticalError(err)
	}

	// Subscribe to completed jobs
	subscribeChan, subscribeCancel := riverClient.Subscribe(river.EventKindJobCompleted)
	defer subscribeCancel()

	if err := riverClient.Start(ctx); err != nil {
		handleCriticalError(err)
	}
	defer func() {
		if err := riverClient.Stop(ctx); err != nil {
			handleCriticalError(err)
		}
	}()

	// Start a transaction to insert a job. It's also possible to insert a job
	// outside a transaction, but this usage is recommended to ensure that all
	// data a job needs to run is available by the time it starts. Because of
	// snapshot visibility guarantees across transactions, the job will not be
	// worked until the transaction has committed.
	tx, err := dbPool.Begin(ctx)
	if err != nil {
		handleCriticalError(err)
	}
	defer tx.Rollback(ctx)

	// Insert a one-time job
	_, err = riverClient.InsertTx(ctx, tx, SortArgs{
		Strings: []string{
			"whale", "tiger", "bear",
		},
	}, nil)
	if err != nil {
		handleCriticalError(err)
	}

	// Insert another one-time job
	_, err = riverClient.InsertTx(ctx, tx, SortArgs{
		Strings: []string{
			"goat", "whale", "cat", "dog", "mouse", "horse",
		},
	}, nil)
	if err != nil {
		handleCriticalError(err)
	}

	// Note: Periodic Jobs don't need to be inserted -- they will start once the client starts

	if err := tx.Commit(ctx); err != nil {
		handleCriticalError(err)
	}

	fmt.Println("Waiting for some jobs to complete...")
	waitForNJobs(subscribeChan, 4)

	riverClient.PeriodicJobs().Clear()

	// Periodic jobs can also be configured dynamically after a client has
	// already started. Added jobs are scheduled for run immediately.
	fmt.Println("Adding a new periodic job...")
	riverClient.PeriodicJobs().Add(
		river.NewPeriodicJob(
			river.PeriodicInterval(5*time.Second),
			func() (river.JobArgs, *river.InsertOpts) {
				return PeriodicJobArgs{}, nil
			},
			nil,
		),
	)

	fmt.Println("Waiting for more jobs to complete...")
	waitForNJobs(subscribeChan, 3)
}

func waitForNJobs(subscribeChan <-chan *river.Event, n int) {
	for range n {
		<-subscribeChan
	}
}

func handleCriticalError(err error) {
	fmt.Println("Error:", err)
	os.Exit(1)
}
```

## Results

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
