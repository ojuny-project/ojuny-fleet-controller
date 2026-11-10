# DRAFT - ojuny-fleet-controller

Status: draft scope. Nothing is built. This file describes what the repository
is supposed to contain, not what it contains today.

## Purpose

The central management server for an Ojuny fleet. This repository is OFC,
Ojuny Fleet Controller: the template flake for the server host itself. It
provides the headless NixOS system, the management stack, the deployment path
and the shared services that keep a set of Ojuny endpoints reachable, patched
and backed up.

OFM is the web GUI and lives in `ojuny-fleet-manager`. OFC owns the host, the
services, the network and the deployment engine.

The model is Entra ID and Intune style: users, groups, endpoints, policies,
applications and deployment from one control plane. The first scope is an
Ojuny-native implementation, not a compatibility layer for Microsoft products.
Central account control is declarative and is done through Nix, not through
SSO, LDAP or Kerberos.

Fleet does not manage Ojuny Pro, Console or Home Server installations. It
deploys its own integrated `ojuny-endpoint` template to enrolled machines.

## What the repository will contain

```text
ojuny-fleet-controller/
  flake.nix                  server host and service modules
  flake.lock
  modules/
    fleet/                   controller service and data paths
    ofm/                     web interface service
    vpn/                     endpoint connectivity
    cache/                   Hydra and nix-serve-ng
    storage/                 NFS and backup targets
    dns/                     internal DNS and reverse proxy
    deploy/                  deploy-rs and nixos-anywhere orchestration
    endpoint/                integrated ojuny-endpoint template
    auth/                    Nix-declared accounts and roles
    audit/                   action and access logs
    images/                  install media generation
  docs/                      architecture, network, security, operations
  tests/                     VM and service checks
```

### Host and services

- The fleet controller service that owns the managed configuration tree,
  policy evaluation and deployment queue.
- The integrated `ojuny-endpoint` template that enrolled machines run. It is
  part of this repository's managed configuration, not an external product.
- `ojuny-fleet-manager` served behind the internal reverse proxy.
- A VPN so endpoints remain reachable without inbound ports where the network
  allows it.
- Hydra and `nix-serve-ng` as the fleet binary cache, served over the VPN.
- NFS shares with device-level and user-level access.
- Centralized incremental backup of endpoint home directories, excluding
  hidden state directories, with retention and scope policies.
- Internal DNS and an HTTP(S) proxy so services have stable names.
- An optional Git host for the managed flake and endpoint repositories.
- A deployment engine that builds and pushes with deploy-rs, and provisions
  new machines with nixos-anywhere.
- An install-media builder that produces pre-addressed live ISOs, one per
  concurrent deployment target, without reflashing the USB for each machine.

### Managed configuration

- One multi-system flake, or one flake per endpoint, stored in Git and edited
  by OFM.
- A stable data model for users, groups, endpoints, servers, applications,
  policies and templates.
- Policies are Nix modules that can be bound at user, group, device or
  application level. Precedence must be explicit and testable.
- Endpoint configuration includes the NixOS system, hardware data and
  device-bound policies.
- Server configuration uses the same model with a server-oriented default
  structure, including container support.
- Deployments are reproducible from a commit. The controller records what was
  built and what was pushed.

### Network model

- Endpoints connect out to the controller over the VPN.
- The controller deploys over the VPN address, so an endpoint does not need a
  public address.
- The design prefers a closed-port setup with no port forwarding. If the VPN
  technology requires a forwarded port, the requirement is documented per
  deployment.
- Internal DNS resolves controller services for enrolled endpoints.
- The reverse proxy terminates TLS with a controller-generated CA that enrolled
  endpoints trust.
- Firewall defaults deny inbound traffic except the documented service ports.

### Authentication and audit

- Accounts, groups and roles are declared in Nix and deployed with the
  controller configuration. There is no SSO, LDAP or Kerberos dependency.
- Roles for administrator, operator, auditor and read-only users.
- Every write, build, deployment and login is audited.
- Secrets for VPN, cache signing and deployment are stored outside the Nix
  store and never written to the managed repository.
- Recovery access is documented and tested.

### Documentation

- The network diagram and the ports that must be open.
- Enrollment: how an endpoint joins, trusts the CA and receives its first
  configuration.
- Deployment: build, push, rollback and failed-deploy handling.
- Backup and restore, including a full controller restore.
- Certificate, VPN and cache-key rotation.
- Capacity and sizing for a small fleet.
- The security model and the trust boundary between controller and endpoints.

### Tests

- The server flake evaluates and builds.
- Each service starts in a VM and answers on its expected address.
- An endpoint enrolls, pulls a configuration and receives a deployment.
- A failed deployment leaves the endpoint on its previous generation.
- The cache serves a path that Hydra built.
- Backup and restore round-trip a home directory.
- A controller restart preserves the deployment queue and audit log.
- The generated install ISO targets the correct controller address.

## Limits and non-goals

- Not a drop-in Entra ID or Intune replacement.
- No cloud service. The controller is self-hosted.
- No Windows endpoint management.
- No guarantee that every network permits a port-free VPN.
- No multi-controller high availability in the first scope.
- No automatic policy authoring by AI and no automatic remediation.
- No telemetry sent outside the fleet.
- No built-in support team or certification program.

## Open questions

- VPN technology: WireGuard with NAT traversal, Tailscale-style coordination,
  or another overlay.
- How controller accounts and roles are declared, reviewed and rotated in Nix.
- Secrets backend: sops, age, Vault or a controller-owned store.
- Single multi-system flake versus one repository per endpoint.
- Backup transport and snapshot mechanism.
- Whether the Git host is mandatory or optional.
- How a controller upgrade is rolled out without losing enrolled endpoints.

## Sources

- `nixos-configs/notes/ojuny pro - early draft.md`, especially the web panel and
  central services section.
- `nixos-configs/notes/Ojuny Deployment Methods.md`.
- `nix-cache-discovery/README.md`.
- `ojuny-docs/02-project-architecture.md`, `08-scope-and-non-goals.md`.
