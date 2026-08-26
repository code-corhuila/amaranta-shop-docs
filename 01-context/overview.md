# System Overview

---

## What is Amaranta?

**Amaranta** is a unified digital platform for a real-world pet business that operates two
lines under one brand: **Amaranta la Clínica** (veterinary clinic — medical consultations,
vaccination, grooming, surgery) and **Amaranta Shop** (pet products store — food, accessories,
hygiene, toys, medication). The platform gives pet owners one place to browse and buy products
and to book clinic appointments, and gives Amaranta's own staff a point-of-sale and
receptionist panel to serve walk-in customers at the physical location with the exact same
system.

## Problem it solves

**Before the system:** Amaranta manages appointments over the phone or on a paper agenda, with
no enforced business-hours rule, so double-bookings and forgotten confirmations happen.
Product sales at the counter are tracked manually, with no shared view of stock between what
is sold in-store and what could be sold online. There is no single record connecting a pet, its
owner, its medical history, and its purchases.

**With the system:** Customers self-serve — browsing products, adding them to a cart, and
booking an appointment within the clinic's real operating hours — from home or from a
self-service kiosk in-store. Receptionists use the same screens to help walk-in customers.
Every pet has one identity across both the shop and the clinic, and every order or appointment
is backed by a domain event other parts of the system can react to (stock updates, email
confirmations).

## Main users

| Role | Description | What they do in the system |
|------|-------------|--------------------------|
| Customer (pet owner) | Registered end user, online or at the physical store | Browses the catalog, buys products, registers their pets, books/cancels appointments, views their pet's medical history |
| Receptionist | Front-desk staff at the Amaranta location | Assists walk-in customers: creates their account, adds products to their cart, books appointments on their behalf |
| Veterinarian | Clinical staff | Manages their appointment agenda, records diagnoses and treatments in the pet's medical history |
| Groomer | Clinical staff (bathing/grooming service) | Manages their appointment agenda for grooming/bathing services |
| Admin | Business owner / manager | Full visibility over catalog, orders, appointments, and staff accounts; the only role that can change other users' roles |

## Technology stack

| Layer | Technology | Justification |
|-------|-----------|---------------|
| Frontend | React 18 + TypeScript + Vite, Tailwind CSS, Framer Motion | React was the required frontend framework; Tailwind gives full control over the vivid, exclusive brand palette without fighting a component library's defaults; Framer Motion implements the "product flies into the cart" micro-interaction with shared-element transitions |
| Backend | Node.js 20 LTS + NestJS + TypeScript | NestJS's module/dependency-injection system maps directly onto the Hexagonal Architecture (Ports & Adapters) this repo standardizes on (`05-architecture/hexagonal-architecture.md`); it ships a native RabbitMQ microservice transporter and OpenAPI generation out of the box |
| Database | PostgreSQL 16, one schema/instance per microservice | Relational integrity fits invoices, appointment slots, and stock counts better than a document store; see ADR-002 for how it stays swappable to MySQL/SQL Server |
| Message broker | RabbitMQ | AMQP topic exchanges + native Dead Letter Queues fit Amaranta's moderate, routing-heavy event volume better than a log-streaming platform like Kafka; see ADR-003 |
| Infrastructure | Docker Compose (local) → Kubernetes + Helm (target), Kong API Gateway, Consul service discovery | Matches the portability requirement (NFR-007) and the course's requirement for explicit service discovery; see ADR-004 and ADR-005 |

## Current status

- **Phase:** In documentation / design (Software Design Documentation phase — no code has been written yet)
- **Current version:** v0.1.0 (documentation only)
- **Last release:** N/A
- **Next milestone:** Complete phases 00–05 (governance through architecture), then start `09-microservices` detailed design and the first implementation sprint

## Project contacts

| Role | Name | Contact |
|------|------|---------|
| Tech Lead / Product Owner / Full-stack developer | Francy Claros | francyclarosc@gmail.com |
| Course | Sistemas Distribuidos — Octavo Semestre | — |
