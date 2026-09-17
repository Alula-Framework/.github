# Flight

A modular server-side framework for Swift.

Composition and application lifecycle at the bottom, HTTP and WebSockets above
it, real-time layers on top of those, and persistence beside them. Six
packages, each usable on its own.

```bash
flight new MyService
cd MyService && swift run MyService
# → MyService is flying, on http://127.0.0.1:8080
```

That is the skeleton: configuration, dependency injection, HTTP, and health
endpoints, with nothing else running. `--tier basics` adds a database and
`--tier demo` adds the rest.

## The packages

| Repository | What it is |
| --- | --- |
| [flight](https://github.com/Flight-Framework/flight) | The framework: compile-time composition and lifecycle, configuration, HTTP, WebSockets, PubSub, Channels, Presence, actuator endpoints, and token authentication |
| [flight-data](https://github.com/Flight-Framework/flight-data) | Persistence and caching: data-source protocols, an in-memory cache, migrations, and the PostgreSQL and Valkey drivers |
| [flight-cli](https://github.com/Flight-Framework/flight-cli) | The `flight` command, the starter templates it generates, and the tutorial |
| [flight-school](https://github.com/Flight-Framework/flight-school) | The interactive tutorial and documentation site for Flight, Hangar, and Changeset |
| [hangar](https://github.com/Flight-Framework/hangar) | A typed query builder and repository for PostgreSQL. Usable outside Flight |
| [hangar-vapor](https://github.com/Flight-Framework/hangar-vapor) | Hangar in a Vapor application: a pooled `Repo` per request, transactions that bind the ambient repo, and no Fluent to give up |
| [swift-changeset](https://github.com/Flight-Framework/swift-changeset) | Ecto-style changesets: collect changes, validate, apply only what is valid and only what changed. No Flight dependency |
| [flight-channels-js](https://github.com/Flight-Framework/flight-channels-js) | The browser and Node client for the Channels wire protocol |

## Start here

**[The tutorial](https://github.com/Flight-Framework/flight-cli/blob/main/TUTORIAL.md)**
builds one application in three parts, each ending at a project you can
download and run: configuration and HTTP, then a database, then a real-time
chat room with presence and authentication. Every stage ends with a command
and what you should see.

If you would rather read code than prose, the
[demo](https://github.com/Flight-Framework/flight-cli/tree/main/templates/demo)
is the finished application.

## What it looks like

Wiring happens at build time. A plugin scans your target for `@Controller`,
`@Service`, `@Repository` and `@Component` and generates the composition root
— adding a controller does not mean editing a list, and a misspelled
configuration key is a compile error rather than a page at 3am.

```swift
@Controller
struct UserController {

    /// The seam, not the concrete repository. Exactly one type in this target
    /// conforms to it, so the generated composition root binds it — nothing
    /// registers it by hand, and a test passes its own fake instead.
    @Inject var users: (any UserRepositoryProtocol)

    @GetRoute("/users/:id")
    func get(_ context: RequestContext) async throws -> User {
        guard let id = context.pathParam("id").flatMap({ UUID(uuidString: $0) }) else {
            throw HTTPError(.badRequest, "user id must be a UUID")
        }
        guard let user = try await users.find(byID: id) else {
            throw HTTPError(.notFound, "no user \(id)")
        }
        return user
    }
}
```

There is no runtime container. Modules are values holding what they provide,
and the generated `flightComposeModules` constructs them in the order the
values themselves imply — so a missing dependency is a build error, not a
resolution failure on the first request that needs it:

```swift
await Flight.run(
    configuration: try Configuration.load(),
    modules: [FlightWebModule<FlightTransport>.self, AppModule.self],
    composedBy: flightComposeModules)
```

## Design

A few decisions worth knowing before you invest in it:

**Composition, not resolution.** Everything is built once, at startup, from
values. Nothing is looked up by type at request time, so the wiring either
compiles or it doesn't — and the route handler holds its dependencies
directly rather than reaching for a container.

**Bring your own auth.** Flight validates tokens your identity provider
issued. There is no password hashing, no session store, and no token issuance
anywhere in it — any OIDC-compliant provider is configuration, not a fork. The
one security-critical primitive, signature verification, is delegated to
JWTKit.

**No hand-rolled HTTP.** The default transport wraps HummingbirdCore rather
than reimplementing HTTP/1.1 correctness, request-smuggling mitigations, and
WebSocket framing. Flight owns routing and dispatch; a transport is a module,
and any conforming one is a peer.

**Take only what you use.** Products sit behind traits, so an application that
wants only composition and lifecycle resolves 8 packages rather than 29, and
one using the in-memory cache never resolves a Postgres driver.

**Testing is the point of the injection.** A controller is a struct and a
route is one of its methods, so most tests construct the type with fakes and
call the method — no container, no router, no HTTP. A smaller tier drives
whole requests through the real routing, middleware and encoding when the
question is about the wiring itself.

## Status

**v0.18.x — early.** The APIs work and are tested, and they are not yet stable
across versions; 0.17 replaced the runtime container with compile-time
composition. Requires Swift 6.3 or later.

## License

MIT.
