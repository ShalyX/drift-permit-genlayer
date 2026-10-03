# Acme Payments API policy — revision 1

This public document is a controlled fixture for DriftPermit's live GenLayer demonstration.

## Authentication

Every payment request must use OAuth. Anonymous requests and static API-key-only requests are rejected.

## Transaction limit

A single charge cannot exceed 100 USD.

## Customer data

Customer data is processed only for the requested transaction and is not retained by the provider after settlement.
