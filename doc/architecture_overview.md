# ERP Gateway & Modular Architecture Overview

This document summarizes how the core Odoo-based ERP in this repository is organized, how
its modules communicate, and the options you have for introducing an API gateway or other
integration layers on top of the existing modular-monolith foundation.

## 1. High-Level Structure

The project follows Odoo's modular monolith pattern: every business capability is packaged
as an addon module that advertises its metadata, dependencies, and data files in a
`__manifest__.py` file. Module discovery is centralized in `odoo.modules.module`, which
validates manifests and builds an import path for each addon by extending the
`odoo.addons` namespace with the configured addon directories and optional upgrade paths.
This allows modules to remain isolated but still run in the same process and database
connection pool.【F:odoo/modules/module.py†L1-L172】

Because all addons share the same Python interpreter and database connection pool, they can
call each other's models or services directly through the Odoo ORM once the registry has
been initialized. This keeps cross-module calls fast while letting you control load order
and dependencies via manifest `depends` lists.

## 2. Request Routing Pipeline

Incoming HTTP requests enter the WSGI application defined in `odoo.http`. The module's
built-in call graph illustrates the full lifecycle: static assets are served directly,
unauthenticated endpoints (`auth="none"`) are routed without opening a database, and all
other requests go through `_serve_db`, which opens a registry cursor, matches the URL to a
controller route, authenticates the user, and dispatches to the endpoint. The dispatcher
handles serialization/deserialization and runs pre/post hooks so modules can inject
behaviour at each stage.【F:odoo/http.py†L1-L120】

Controllers are defined inside addon modules by subclassing `odoo.http.Controller` and
annotating methods with `@route`. Once the module is installed, these endpoints become part
of the global routing table, so other modules or your gateway layer can forward requests to
them without additional wiring.

## 3. Application Servers and RPC Endpoints

`odoo.service.server` wraps Werkzeug's WSGI server implementations and manages worker
threads or greenlets. Each worker stores per-request metadata (such as the current RPC
model/method) in `thread_local`, which is later reused for logging and tracing. The server
also centralizes process supervision concerns like memory limits, signal handling, hot
reloading, and WebSocket upgrades.【F:odoo/service/server.py†L1-L200】

For programmatic access, the XML-RPC/JSON-RPC endpoints reuse the same registry and ORM.
The RPC dispatcher validates the requested model method, opens a cursor against the target
database, checks user credentials, and then invokes the method via the ORM. Results are
normalized (e.g., recordsets become ID lists) before being returned to the caller. The
`retrying` helper automatically retries calls that hit serialization or concurrency errors
with exponential backoff, ensuring that gateway-triggered operations remain consistent even
under load.【F:odoo/service/model.py†L1-L220】

These RPC services give you a ready-made gateway surface: you can expose module methods to
external consumers via JSON-RPC without rewriting business logic. Alternatively, you can
create dedicated HTTP controllers that orchestrate calls to multiple modules and serialize
responses in a REST style.

## 4. Designing an API Gateway Layer

To introduce a formal gateway on top of this architecture:

1. **Choose the entry protocol.** If your consumers already integrate with Odoo's native
   XML-RPC/JSON-RPC, reuse `odoo.service.model.dispatch`. For REST/GraphQL, add a thin
   gateway module with controllers that call into the ORM.
2. **Aggregate module logic.** Inside your gateway endpoints, obtain an environment via
   `request.env` (for HTTP controllers) or a registry cursor (for background jobs) and call
   the desired models. Because modules run in the same process, you can compose flows across
   sales, inventory, accounting, etc., without network overhead.
3. **Handle cross-cutting concerns.** Use dispatcher pre/post hooks or middleware in your
   gateway module for authentication, rate limiting, or auditing. The HTTP stack already
   exposes hook points (`_pre_dispatch`, `_post_dispatch`) that you can extend.
4. **Manage dependencies.** Declare each module your gateway relies on in the manifest's
   `depends` list so Odoo loads them first. This ensures model classes and controllers are
   registered before the gateway starts handling traffic.

## 5. Getting Started for a Custom ERP

* Create a new addon (e.g., under `addons/custom_gateway`) with a `__manifest__.py` that
  depends on the core modules you need (such as `base`, `sale`, or custom apps).
* Implement controllers for your gateway API. Use `request.env['model']` to access ORM
  models and compose responses. Reuse the retry logic provided by the environment when
  performing writes to handle database contention gracefully.
* If you need to expose RPC to external systems, register your module's methods with
  `@api.model`/`@api.model_create_multi` and call them via the existing `execute_kw` RPC
  signature.

By leveraging the existing HTTP pipeline, module registry, and RPC layer, you can layer a
gateway in front of the modular monolith without forking the framework. This keeps your ERP
extensible and aligned with standard Odoo deployment practices while giving external
systems a consistent integration surface.
