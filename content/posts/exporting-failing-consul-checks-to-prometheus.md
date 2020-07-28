---
title: "Exporting Failing Consul Checks to Prometheus"
date: 2019-05-04T12:21:51-08:00
---

This weekend I hacked on a quick project to teach myself a bit more Go and maybe do something useful. I wrote a quick daemon to scrape failed Consul checks and make the data available for Prometheus scrape (this could tie in nicely with Alertmanager).

<!--more-->

So, I played around a bit and made a simple tool with some nice CLI arg parsing via kingpin:

```go
var (
	consul_addr = kingpin.Flag("consul-addr",
		"HTTP addresses of the Consul server").
		Default("127.0.0.1:8500").
		String()

	consul_scheme = kingpin.Flag("consul-scheme",
		"Scheme by which to connect to Consul").
		Default("http").
		String()

	consul_token = kingpin.Flag("consul-token",
		"Consul ACL token").
		OverrideDefaultFromEnvar("CONSUL_HTTP_TOKEN").
		String()

	metrics_endpoint = kingpin.Flag("metrics-endpoint",
		"Path endpoint to handle Prometheus scrapes").
		Default("/metrics").
		String()

	metrics_listen = kingpin.Flag("metrics-listen",
		"Listen address for Prometheus").
		Default(":8080").
		String()

	check_states = kingpin.Flag("check-state",
		"Healthcheck states to export").
		Default("warning", "critical").
		Strings()
)
```

And a nice little struct to hold the Prometheus metrics and Consul client handler:

```go
type Exporter struct {
	mutex sync.RWMutex

	consulFailures *prometheus.GaugeVec
	scrapeFailures prometheus.Counter

	failures map[string]string
	client   *api.Client
	states   []string
}
```

Of course, what good is a struct is without a constructor?

```go
func NewExporter() *Exporter {
	return &Exporter{
		consulFailures: prometheus.NewGaugeVec(
			prometheus.GaugeOpts{
				Name: "consul_check",
				Help: "Failing Consul service",
			},
			[]string{"check_id", "state"},
		),
		scrapeFailures: prometheus.NewCounter(
			prometheus.CounterOpts{
				Name: "consul_scrape_failures",
				Help: "Consul health API scrape failures",
			},
		),
		failures: make(map[string]string),
	}
}
```

Our Exporter even implements prometheus.Collector!

```go
func (e *Exporter) Describe(ch chan<- *prometheus.Desc) {
	e.scrapeFailures.Describe(ch)
	e.consulFailures.Describe(ch)
}

func (e *Exporter) Collect(ch chan<- prometheus.Metric) {
	e.mutex.Lock()
	defer e.mutex.Unlock()

	if err := e.collect(); err != nil {
		log.Errorf("Error scraping Consul: %s", err)
		e.scrapeFailures.Inc()
		e.scrapeFailures.Collect(ch)
	}

	e.consulFailures.Collect(ch)
}
```

And when it comes to the actual collection implementation, Consul’s Client provides wonderful retry logic and transport handling, all abstracted away, so we can focus on clean, simple code:

```go
func (e *Exporter) collect() error {
	health := e.client.Health()

	m := make(map[string]string)

	for _, state := range e.states {
		healthchecks, _, err := health.State(state, nil)
		if err != nil {
			return err
		}

		for _, h := range healthchecks {
			k := h.Node + ":" + h.ServiceName

			e.failures[k] = h.Status
			m[k] = h.Status
		}
	}

	for k := range e.failures {
		// prune old failures
		if _, ok := m[k]; ok {
			e.setFailure(k, e.failures[k])
		} else {
			e.clearFailure(k, e.failures[k])
			delete(e.failures, k)
		}
	}

	return nil
}

func (e *Exporter) setFailure(check_id, state string) {
	e.consulFailures.With(prometheus.Labels{
		"check_id": check_id,
		"state":    state,
	}).Set(1)
}

func (e *Exporter) clearFailure(check_id, state string) {
	e.consulFailures.Delete(prometheus.Labels{
		"check_id": check_id,
		"state":    state,
	})
}
```

And now we need to tie it all together!

```go
package main

import (
	"net/http"
	"sync"

	"github.com/hashicorp/consul/api"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/common/log"

	"gopkg.in/alecthomas/kingpin.v2"
)

const (
	_VERSION = "0.0.1"
)

<...snip...>

func main() {
	kingpin.Version(_VERSION)
	kingpin.Parse()

	e := NewExporter()
	prometheus.MustRegister(e)

	c := &api.Config{
		Address: *consul_addr,
		Scheme:  *consul_scheme,
		Token:   *consul_token,
	}

	client, err := api.NewClient(c)
	if err != nil {
		panic(err)
	}

	e.client = client
	e.states = *check_states

	http.Handle(*metrics_endpoint, prometheus.Handler())
	log.Fatal(http.ListenAndServe(*metrics_listen, nil))
}
```

... of course, I put all this together and realize that the official Consul Prometheus exporter can do all this, and a lot more.

But no matter! This was a fun learning experience. Coming from Perl/Bash/OpenResty where everything is super concrete and there’s no baked in logic for grace (retries, keepalive, etc), writing this felt wonderfully simple, if not a bit odd- if the Consul server goes away and magically reappears, the code above gracefully handles it. Normally, writing sane OpenResty code to handle that needs a few day’s worth of testable boilerplate.

I also feel like I’m getting comfortable enough with Go that I can use godocs as meaningful learning material and not be completely overwhelmed, which made hacking on this super pleasant – I spent all my time in reference material and studying implementation a bit, not scratching my head over syntax and idiom.
