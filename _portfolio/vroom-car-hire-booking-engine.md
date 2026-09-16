---
title: "VroomVroomVroom | Car Hire Aggregator Booking Engine"
start_date: 2016-06-01
end_date: 2026-06-01
website: "https://vroomvroomvroom.com.au?utm_source=gusdecool.github.io"
excerpt: "Car hire aggregator booking engine, serving millions visitors a day."
---

[Visit the website]({{ page.website }})

![landing page](/image/portfolio/vroom-landing.png)
![landing page](/image/portfolio/vroom-search-page.png)

## Tech stacks used
PHP, TypeScript, MySQL, Laravel (REST API), React (Widget & SPA), AWS Cloud, Docker, Event-Driven Architecture,
MJML & EJS for email templating, Symfony Component & Doctrine ORM.

## REST API & 3rd Parties Integration
Developed the REST API that will be consumed by our web, mobile and 3rd parties B2B. As aggregator, I also integrated
API from 3rd party as the provider of the services.

## Payment Gateway
Implemented payment gateway integration with Stripe, PayPal & Braintree. Stripe capable of multi platform connection.
Managed the auto recovery/refund when failure occur and idempotency to avoid duplicate charge. 

## Docker for local development
I was at Prosura team before assigned to Vroom team. When I started at Vroom team, it took me a week to setup
the local dev environment due to it has multiple repositories: admin, web, API it has all their own repo and connect to
each others. I also noticed when a new team member joined, they even could take longer for this, sometime even 2 weeks.
It also harder to support the team as each of them has their own dev environment, sometime different `php.ini` could 
result an unexpected behavior.

To solve this issue, I created a docker image with compose file to sync all of our infrastructure. When a new team member
want to join the development, they only need to run `docker-compose up` and it will sync all of our infrastructure. This
successfully reduce the onboarding time for new team members down to 1 day.

## Improving API capability
We have API v1 where it data structure is not flexible enough to support our business needs as it only support supplier API
that using XML/SOAP. I have implemented API v2 which is more flexible to support any kinds of API format and can 
be easily extended in the future with abstraction & SOLID principle.

To save time refactoring the whole API v1 into API v2, I built a bridge system to make v2 communicate to v1. This greatly
reduce the time needed to refactor the whole API v1 into API v2 and mitigating the risk of breaking the existing API v1.

## GIS Spatial coordinate
Originally when searching for car depot, we use database that have plain lat & lng in floating. This is not efficient, I
migrated it to use SQL GIS spatial data type which is more efficient and accurate. Improved the latency by 50%.

## Technical Lead Engineering
Lead team consist of 8 developers and intersect to mentor QA team on how to testing the application, which part need focused.
Pionereed how to write testing notes for QA to rise the standard operation procedure.

## Owned the full SDLC CI/CD
Manage the task from management, distributed the task with team, hands-on development, responsible for production deployment
and report the deliverability to stakeholders.

## Infrastructure as Code
Implemented IaC to automate AWS SAM deployment where previously we do it manually via CDK or AWS Console.

## Architected React Migration
Architected gradual migration approach for the frontend website from Laravel Blade to React. Mitigated the risk breaking
the business core functionality.

## AI-first development
Mentored the team on how to use AI code assistant while still maintaining the code quality and best security practices.

## Event-Driven System & Data ETL
Architected event-driven system with AWS Event Bridge to API Destination or Lambda. Reduced API latency down to 75%. 
Added data ETL pipeline into CRM to improve the conversion.

## User registration
Implemented hybrid user registration with AWS Cognito and internal user RBAC. 

## AWS Cloud & Google Cloud
Managed the AWS Cloud & Google Cloud infrastructure. Secured the parameter secrets, data privacy & IAM least privilege.

----

## Client reviews
![upwork review](/image/upwork-review-1.png)
