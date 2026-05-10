+++
date = '2025-09-22'
title = 'Raft Consensus Explained'
series = ['distributed systems']
+++

This is a placeholder post about the Raft consensus algorithm. Raft is designed to be understandable while providing the same guarantees as Paxos.

## Leader Election

Raft uses a leader-based approach. One server is elected leader and is responsible for managing the replicated log.

## Log Replication

The leader accepts log entries from clients and replicates them across the cluster, telling servers when it is safe to apply entries to their state machines.

## Safety

Raft guarantees that committed entries are durable and will eventually be executed by all available state machines.
