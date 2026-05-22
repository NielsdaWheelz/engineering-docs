# Cloud Machine Service

## Scope

Tenant machine RPCs live under `Cloud.Tenant.Machine.*`; the web mirror is
`Cloud.Web.Tenant.Machine.*` and additionally exposes machine listing.

This service owns tenant-visible cloud machines, their generic backing records,
and dispatch to provider-specific machine backing drivers. Workspace-local
machine rows, filesystem RPCs, command execution after machine creation, and
billing are outside this module.

## Public Contract

Common behavior:

- Tenant RPCs use `CurrentTenant`.
- Machine-handle RPCs accept `CloudMachineHandle`, unseal it to
  `CloudMachineId`, and verify current-tenant ownership.
- Ownership misses are `MachineNotFound`.
- Mutating RPCs accept `ReplayKey`.

Common shapes:

```ts
type MachineSpec = {
  provider: "digitalocean";
  zone: string;
  type: string;
  os: string;
};

type MachineDetail = Record<string, unknown>;
type MachineRuntimeInfo = Record<string, unknown>;
type MachinePowerState = "Running" | "Stopped";

type MachineCatalogEntry = {
  spec: MachineSpec;
  detail: MachineDetail;
  pricing: { monthlyPrice: MachineMonthlyPrice };
};

type MachineRef = {
  machineHandle: CloudMachineHandle;
  detail: MachineDetail;
};

type MachineSummary = {
  machineHandle: CloudMachineHandle;
  spec: MachineSpec;
  detail: MachineDetail;
};
```

Web classes `TenantMachineRef`, `TenantMachineSummary`, and
`TenantMachinePowerStateInfo` use the same field shapes.

Except for web-only `ListRpc`, each row exists under both
`Cloud.Tenant.Machine.*` and `Cloud.Web.Tenant.Machine.*`.

| RPC | Input | Output | Errors | Behavior |
| --- | --- | --- | --- | --- |
| `ListCatalogRpc` | `{}` | `MachineCatalogEntry[]` | `never` | Return public `spec`, `detail`, `pricing`; hide `inventory` and `backing`. |
| `CreateRpc` | `spec`, `startBashCommand`, `replayKey` | `MachineRef` | `MachineCatalogEntryNotFound` | Validate catalog entry, claim or provision backing, start command, attach tenant, return handle/detail. |
| `DeleteRpc` | `machineHandle`, `replayKey` | `void` | `MachineNotFound` | Remove tenant visibility, generic rows, and provider backing. |
| `StartRpc` | `machineHandle`, `replayKey` | `void` | `MachineNotFound`; `MachineAlreadyRunning` | Dispatch backing to running state. |
| `StopRpc` | `machineHandle`, `replayKey` | `void` | `MachineNotFound`; `MachineAlreadyStopped` | Dispatch backing to stopped state. |
| `GetRpc` | `machineHandle` | `MachineSummary` | `MachineNotFound` | Return handle, spec, and detail from the generic backing record. |
| `Cloud.Web.Tenant.Machine.ListRpc` | `{}` | `TenantMachineSummary[]` | `never` | Web-only; return current-tenant summaries ordered by `cloud_machine.created_at` descending. |
| `GetPowerStateRpc` | `machineHandle` | `{ powerState: MachinePowerState }` | `MachineNotFound` | Driver reads provider state and maps it to `Running` or `Stopped`. |
| `GetRuntimeInfoRpc` | `machineHandle` | `MachineRuntimeInfo` | `MachineNotFound` | Driver returns provider-specific runtime info as JSON. |

## Catalog

The public catalog is derived from private machine definitions:

```ts
type MachineDefinition = {
  spec: MachineSpec;
  detail: MachineDetail;
  pricing: { monthlyPrice: MachineMonthlyPrice };
  inventory: { warmPoolSize: number };
  backing: MachineBackingDefinition;
};
```

All current definitions use `provider = "digitalocean"`, `zone = "sfo3"`,
`os = "debian-13"`, `driverKey = "DigitalOceanDroplet"`,
`region = "sfo3"`, `image = "debian-13-x64"`, and `warmPoolSize = 0`.

| type | memoryGb | diskGb | monthlyPrice | DigitalOcean size |
| --- | ---: | ---: | ---: | --- |
| `basic-512mb` | 0.5 | 10 | 4 | `s-1vcpu-512mb-10gb` |
| `basic-1gb` | 1 | 25 | 6 | `s-1vcpu-1gb` |
| `basic-2gb` | 2 | 50 | 12 | `s-1vcpu-2gb` |

## Backing Contract

A machine is the tenant-visible object. A backing is the provider resource behind
it.

```ts
type MachineBackingDefinition = {
  driverKey: MachineBackingDriverKey;
  config: unknown;
};

type MachineBackingRef = {
  driverKey: MachineBackingDriverKey;
  driverRef: unknown;
};
```

- `MachineBackingDefinition` is private catalog configuration.
- `MachineBackingRef` is a private persisted identity pointer.
- `config` and `driverRef` decode through the selected driver's schemas.
- `MachineBackingRef` contains only `{ driverKey, driverRef }`; durable provider
  metadata belongs in provider-owned tables.
- No database foreign key exists from JSON `backing_ref.driverRef` into
  provider-owned tables.

Each installed backing driver provides:

```ts
interface MachineBackingDriver<Config, DriverRef> {
  provisionMachineBacking(options: {
    config: Config;
    bootstrapScript: string;
  }): ReplayableMultiMutation<{ driverRef: DriverRef }, never, never>;

  deleteMachineBacking(
    driverRef: DriverRef,
  ): ReplayableMultiMutation<void, MachineBackingAlreadyDeleted, never>;

  startMachineBacking(
    driverRef: DriverRef,
  ): ReplayableSingleMutation<
    void,
    MachineNotFound | MachineAlreadyRunning,
    never
  >;

  stopMachineBacking(
    driverRef: DriverRef,
  ): ReplayableSingleMutation<
    void,
    MachineNotFound | MachineAlreadyStopped,
    never
  >;

  getMachinePowerState(
    driverRef: DriverRef,
  ): ReplayableQuery<MachinePowerState, MachineNotFound, never>;

  getMachineRuntimeInfo(
    driverRef: DriverRef,
  ): ReplayableQuery<MachineRuntimeInfo, MachineNotFound, never>;
}
```

`MachineBackings` dispatches by `driverKey`. The only installed driver is
`DigitalOceanDroplet`.

Driver errors:

```text
deleteMachineBacking -> MachineBackingAlreadyDeleted
startMachineBacking  -> MachineNotFound | MachineAlreadyRunning
stopMachineBacking   -> MachineNotFound | MachineAlreadyStopped
read operations      -> MachineNotFound
```

## Database Model

```ts
type cloud_machine_backing = {
  id: CloudMachineBackingId;
  spec: Json<MachineSpec>;
  detail: Json<MachineDetail>;
  backing_state: "Provisioning" | "Ready";
  cloud_executor_worker_token: WorkerToken;
  backing_ref: Json<MachineBackingRef> | null;
};

type cloud_machine = {
  id: CloudMachineId;
  machine_backing_id: CloudMachineBackingId;
};

type cloud_tenant_machine = {
  id: CloudTenantMachineId;
  machine_id: CloudMachineId;
  tenant_id: CloudTenantId;
};
```

Constraints used by the service:

- At most one `cloud_machine` may point at a backing.
- At most one tenant ownership row may point at a machine.
- Backing worker tokens are unique.
- A tenant-visible machine must have non-null `backing_ref`.
- A Ready backing claimed for a machine must have non-null `backing_ref` and no
  existing `cloud_machine` row.
- Foreign keys link `cloud_machine.machine_backing_id` to
  `cloud_machine_backing`, `cloud_tenant_machine.machine_id` to `cloud_machine`,
  `cloud_tenant_machine.tenant_id` to `cloud_tenant`, and DigitalOcean
  droplet-name short keys to `short_key`.

## Lifecycle

Create:

```text
tenant request
  -> validate spec against catalog
  -> claim oldest matching unclaimed Ready backing with non-null backing_ref
  -> otherwise reserve Provisioning backing, provision provider resource, wait
     for executor worker, set backing_state = Ready, store backing_ref
  -> insert cloud_machine
  -> start startBashCommand through executor
  -> insert cloud_tenant_machine
  -> return machineHandle/detail
```

Create edges:

- Finalizing a backing requires an existing Provisioning backing with no
  `cloud_machine` row and no existing `backing_ref`.
- Tenant attachment is one-time.
- `cloud_machine` is inserted before `startBashCommand`; `cloud_tenant_machine`
  is inserted only after the command is accepted by the executor.
- Create waits for `startBashCommand` acceptance, not command completion.
- Concurrent Ready-backing claims can race at the
  `cloud_machine.machine_backing_id` uniqueness guard.

Delete:

```text
tenant request
  -> unseal handle
  -> delete matching cloud_tenant_machine row, or MachineNotFound
  -> read backing_ref
  -> delete cloud_machine
  -> delete cloud_machine_backing
  -> driver deletes provider resource
```

Delete removes Codapt visibility before provider teardown: tenant row, generic
machine row, generic backing row, provider-owned driver row, then provider
resource.

Start/stop:

```text
handle -> tenant ownership -> backing_ref -> driverKey dispatch
  -> driver checks provider state
  -> already desired: MachineAlreadyRunning/MachineAlreadyStopped
  -> otherwise reconcile provider resource to desired power state
```

Power/runtime reads:

```text
handle -> tenant ownership -> backing_ref -> driverKey dispatch
  -> driver reads provider state/info
```

If a driver reports `MachineNotFound` after a backing-ref check, start/stop
recheck the machine/backing record and reads recheck the tenant machine record.
If the recheck no longer proceeds, return `MachineNotFound`.

Warm pool:

```text
maintenance tick
  -> select MachineDefinitions with warmPoolSize > 0
  -> submit Cloud.MachineBacking.EnsureWarmPool
  -> count unclaimed Provisioning or Ready backings for the spec
  -> reserve/provision only the deficit
  -> mark new backings Ready without creating cloud_machine rows
```

All current `warmPoolSize` values are `0`. Warm-pool capacity counts unbound
Provisioning or Ready backings by spec and does not require non-null
`backing_ref`; Create only claims Ready backings with non-null `backing_ref`.

Durable operation names:

```text
Cloud.Tenant.Machine.Create
Cloud.Tenant.Machine.Delete
Cloud.Tenant.Machine.Start
Cloud.Tenant.Machine.Stop
Cloud.MachineBacking.EnsureWarmPool
```

## DigitalOceanDroplet Driver

Driver key:

```ts
"DigitalOceanDroplet"
```

Shapes:

```ts
type DigitalOceanDropletBackingConfig = {
  region: string;
  size: string;
  image: string;
};

type DigitalOceanDropletBackingRef = {
  digitalOceanDropletBackingId: CloudDigitalOceanDropletBackingId;
};

type cloud_digitalocean_droplet_backing = {
  id: CloudDigitalOceanDropletBackingId;
  droplet_id: ColumnType<string, number, number>; // DB bigint
  droplet_name_short_key_id: ShortKeyId;
};

type DigitalOceanDropletRuntimeInfo = {
  network: {
    public_ipv4: string | null;
  };
};
```

Driver table constraints:

- `droplet_id` is unique.
- `droplet_name_short_key_id` is unique.
- `droplet_id` must decode to a positive safe integer before driver use.

Behavior:

- Provision reserves a hostname-compatible droplet name, creates a unique
  `provisioning:<slug>` tag, creates a droplet with `managed` and provisioning
  tags, and waits for `active` before persisting the driver ref.
- Provision rejects a fresh provisioning tag that already finds a droplet,
  multiple droplets for one provisioning tag, returned droplet-name mismatch,
  and just-created droplet 404 during activation polling.
- Start maps existing `active` to `MachineAlreadyRunning`; otherwise it powers
  on and reconciles until `active`.
- Stop maps existing `off` to `MachineAlreadyStopped`; otherwise it powers off
  and reconciles until `off`.
- Delete removes the driver row before deleting the droplet. Missing driver row
  is `MachineBackingAlreadyDeleted`; already-gone droplet after a driver row
  existed is unrecoverable.
- Power state maps DigitalOcean `active` to `Running` and `off` to `Stopped`;
  persisted refs must not point at `new` or `archive` droplets.
- Runtime info returns `network.public_ipv4` or `null`, does not require the
  droplet to be running, maps DigitalOcean 404 to `MachineNotFound`, and retries
  transient API errors.
- Create/delete/start/stop use operation-specific reconcile schedules. 404
  handling differs by operation.

Tag meanings:

- `managed`: broad Codapt ownership marker.
- `provisioning:<slug>`: create-reconciliation identity used to find the droplet
  after an uncertain create outcome.
