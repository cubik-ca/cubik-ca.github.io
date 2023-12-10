---
layout: post
title: A New Application Architecture
description: Cloud-Native Event-Driven Architecture
date: 2023-12-09T21:51:00Z-07
categories: architecture
published: true
---
# A New Application Architecture
## Cloud-Native Event-Driven Architecture

You would be forgiven for thinking that the subtitle is nothing more than buzzwords and
vaporware. But I assure you it is not. This article will present my vision for a new (and
affordable!) application architecture that will give great returns in resiliency, scalability,
performance, responsiveness, maintainability, and user satisfaction. What I am selling is
not pie in the sky; it really works!

### PaaS and SaaS, not IaaS

The journey to Cloud-Native has been a long one. This is not something that will happen
overnight. For our organization, it has been a journey that has taken more than 6 years now,
and is only now starting to change from the straight "lift-and-shift" approach that gets
into the cloud. Now that most of our infrastructure is there, we can start replacing servers
with special-purpose services. For example, this article will make reference to Memphis.dev,
a SaaS message broker that replaces an entire RabbitMQ cluster. That cluster currently
requires 3 VMs that must be maintained on an ongoing basis: Erlang releases, RabbitMQ releases,
Linux patching. So the savings of replacing that cluster with a SaaS offering goes beyond
just the monetary savings (of which there is a modest amount), but also frees up a little
bit of sysadmin time too every month.

### Scalability

As an architect, perhaps my biggest challenge is preparing for large releases and new business.
I worry that when the business comes to me and asks about new business opportunities that I
will be limited by the existing infrastructure and architecture, and that no amount of
throwing money at the problem will solve it faster. There is one very good solution, and
that is to build scalable applications from the beginning.

### Resiliency

How many of your applications run with little to no redundancy? Are there databases in your
organization that would cause irreparable harm if they were to be corrupted or damaged?
How frequent are your planned maintenance outages? How about your unplanned ones? All of these
questions refer to resiliency, the ability to withstand difficult conditions, such as network
outages, hardware failures, viruses and malware, etc.

## Architecture Diagram


