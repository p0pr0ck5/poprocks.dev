---
title: "Stateful Consul Watch Events Handling"
date: 2018-11-26T12:37:50-08:00
---

Recently I’ve been exploring Consul’s Events mechanism as a way to propagate broadcast data across our infrastructure. It’s a useful pipeline, given our existing use of Consul- message systems like NATS or similar might be a more purpose-built solution, but being able to leverage existing infrastructure and code lets us deploy new solutions quickly and cheaply.

<!--more-->

The Consul Events mechanism exhibits, to an uninitiated outsider, unexpected behavior. Unlike traditional pub/sub message queue systems, events are stored in a ring buffer in each agent, without a concept of ordering or time-based delivery. This has been noted before, and we’ve previously been able to work around it by being smart with existing tooling like OpenResty to deal with out-of-order message delivery. These solutions work well for working with the Consul Event HTTP API; the consul watch CLI command exhibits some odd behaviors when fetching event data, so we needed to write a bit of logic around sanely ordering and handling events.

For our use case, we only care about reacting to new events propagated throughout the cluster. Since each event has a distinct Lamport clock time, it’s easy enough to store long-lived state about seen events in a spawned handler. Order is not a strict concern in this case, since we’ll assume that each unique message is meaningful and requires a reaction.

For funsies, I hacked together some Perl to be able to manage state on disk. First, we need to be able to fetch and set the event state:

```perl
use constant STATE_FILE => q#/var/run/consul_watch.state#;

sub write_state {
  my ($state) = @_;
  my $data;

  open(my $fh, '>', STATE_FILE) or die $!;

  try {
    $data = encode_json($state);
  } catch {
    warn "Could not encode JSON:\n$state\n";
    $data = {};
  };

  print $fh $data;

  close $fh;
}

sub load_state {
  my $state;

  open(my $fh, '<', STATE_FILE) or die $!;

  my $row = <$fh>;

  try {
    $state = decode_json($row) if $row;;
  } catch {
    warn "Invalid state:\n$row\n";
    $state = {};
  };

  close $fh;

  return $state;
}
```

Once this in place, we can wrap event processing in stateful mechanisms:

```perl
sub process_event {
  my ($event) = @_;

  print "handling this event\n" . to_json($event, { pretty => 1 }) . "\n";
}

sub process {
  while (<>) {
    my $h;

    try {
      $h = decode_json($_);
    } catch {
      warn "Invalid event received\n";
      continue;
    };

    my $state = load_state();

    for my $event (@{$h}) {
      if (! $state->{$event->{LTime}}) {
        process_event($event);

        $state->{$event->{LTime}} = 1;
      }
    }

    write_state($state);
  }
}
```

Trivial stuff. Handling odd cases where both a single event and a series of events are received is supported, since with each payload received we test against our seen list of LTime values. Using JSON is a bit lazy but it works well enough (using something like Storable data might be a better solution). With this, we can write a consul watch -type events handler that will respond only to new events received by the cluster.

This is a bare-bones example, of course. Production-ready state management would need to prune old events (Consul maintains a list of up to 256 most recent events, stored in a ring buffer), and high-performance needs would likely best be served with an in-memory state management mechanism, relying on disk storage only for durability (though frankly I’d suspect at that scale the Consul Events mechanism is not an appropriate solution).
