# CI Tests

For CI, OVN-Kubernetes runs the
[Kubernetes E2E tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-testing/e2e-tests.md)
and some [locally defined](https://github.com/ovn-kubernetes/ovn-kubernetes/tree/master/test/e2e) tests. 
[GitHub Actions](https://help.github.com/en/actions)
are used to run a subset of the Kubernetes E2E tests on each pull request. The
local workflow that controls the test run is located in
[ovn-kubernetes/.github/workflows/test.yml](https://github.com/ovn-kubernetes/ovn-kubernetes/blob/master/.github/workflows/test.yml).

The following tasks are performed by that workflow:

- Lint and verify generated code/mocks
- Build OVN-Kubernetes images
- Check out the Kubernetes source tree and compile some dependencies
- Install KIND
- Run an explicit matrix of end-to-end tests using KIND

Documentation is not built by `test.yml`. That workflow ignores `docs/**`, `*.md`, and `mkdocs.yml`, so docs-only pull requests never start `ovn-ci`.

The scheduled run (`cron: '0 */12 * * *'`, twice daily) uses the **same** `e2e` `include:` list as pull requests. It is not a larger “full” matrix.

The following sections should help you understand (and if needed modify) the set of tests that run and how to run these
tests locally.

## CI fails: what do I do?

Some tests are known to be flaky, see [`kind/ci-flake` issues.](https://github.com/ovn-kubernetes/ovn-kubernetes/issues?q=is%3Aissue%20state%3Aopen%20label%3Akind%2Fci-flake)
At the end of your failed test run, you will see something like:

```
Summarizing 1 Failure:
  [FAIL] e2e egress firewall policy validation with external containers [It] Should validate the egress firewall policy functionality for allowed IP
  /home/runner/work/ovn-kubernetes/ovn-kubernetes/test/e2e/egress_firewall.go:130
```
then search for "e2e egress firewall policy validation" in the open issues. 

If you find an issue that matches your failure, update the issue with your job link. 
If the issue doesn't exist, it either means the failure is introduced in your PR or it is a new flake. 
Try to run the same test locally multiple times, and if doesn't fail, report a new flake.
Reporting a new flake is fairly straightforward, but you can use already open issues as an example.
It may also be useful sometimes to search through the closed issues to see if the flake was reported
previously and (not really) fixed, then reopening it with the new job failure.

Only after following these steps, comment **exactly** `/retest-failed` on the PR (the entire comment body must be that string, nothing else). The retest action ignores comments that only *contain* the command among other text. A rocket reaction is added after a successful trigger; `/retest` reruns completed workflows, `/cancel` cancels in-progress ones.

Put flake issue links in an earlier comment or in the PR description. Do not mix them into the `/retest-failed` comment.

## Understanding the CI Test Suite

The tests fall into Kubernetes E2E **shards** (`shard-%` in
[ovn-kubernetes/test/Makefile](https://github.com/ovn-kubernetes/ovn-kubernetes/blob/master/test/Makefile))
and **locally defined** suites invoked via `make control-plane` (often with `WHAT=`).
GitHub `ovn-ci` also has dedicated e2e targets such as `multi-homing`, `node-ip-mac-migration`,
`external-gateway`, `network-segmentation`, `bgp`, `evpn`, `serial`, and `tools`. Those are not extra
shard names; they select `make` targets or a focused `WHAT` in `test.yml`.

### Shard tests

The shard tests are broken into a set of shards, which is just a grouping of tests,
and each shard is run in a separate job in parallel. Shards execute the `shard-%` target in 
[ovn-kubernetes/test/Makefile](https://github.com/ovn-kubernetes/ovn-kubernetes/blob/master/test/Makefile).
The set of shards may change in the future. Below is an example of the shards at time of this writing:

- shard-network
  - All E2E tests that match `[sig-network]`
- shard-conformance
  - All E2E tests that match `[Conformance]|[sig-network]`
- shard-test
  - Single E2E test that matches the name of the test specified with a regex. 
  - When selecting the `shard-test` target, you focus on a specific test by appending `WHAT=<test name>` to the make command.
  - Examples are in the [Local Testing Guide](../developer-guide/local_testing_guide.md).

Shards use the [E2E framework](https://kubernetes.io/blog/2019/03/22/kubernetes-end-to-end-testing-for-everyone/). By
selecting a specific shard, you modify ginkgo's `--focus` parameter.

The regex expression for determining which E2E test is run in which shard, as
well as the list of skipped tests is defined in
[ovn-kubernetes/test/scripts/e2e-kind.sh](https://github.com/ovn-kubernetes/ovn-kubernetes/blob/master/test/scripts/e2e-kind.sh).

### Control-plane tests

In addition to the `shard-%` tests, there is also a `control-plane` target in 
[ovn-kubernetes/test/Makefile](https://github.com/ovn-kubernetes/ovn-kubernetes/blob/master/test/Makefile).
Below is a description of this target:

- control-plane
  - Locally defined tests, with skip/focus from `e2e-cp.sh`.
  - Focus with `WHAT=<test name>`. Examples are in the [Local Testing Guide](../developer-guide/local_testing_guide.md).

All local tests are run by `make control-plane` when `WHAT` is unset, except suites `e2e-cp.sh` skips unless requested
(node IP/MAC migration, and other focused suites). GitHub jobs often pass `WHAT` so a lane only runs one of those
groups. The skip/focus logic lives in
[ovn-kubernetes/test/scripts/e2e-cp.sh](https://github.com/ovn-kubernetes/ovn-kubernetes/blob/master/test/scripts/e2e-cp.sh)
and the tests live in
[ovn-kubernetes/test/e2e/](https://github.com/ovn-kubernetes/ovn-kubernetes/tree/master/test/e2e).

#### Node IP migration tests

The node IP migration tests are part of the control-plane tests but due to their impact they cannot be run concurrently
with other tests and they are disabled when running `make control-plane`.
Instead, they must explicitly be requested with `make -C test control-plane WHAT="Node IP and MAC address migration"`. That `WHAT` value must match the skip exception in `test/scripts/e2e-cp.sh`; a shorter name still skips the suite.

### Github CI integration through Github Actions Matrix

Pull-request and scheduled `e2e` jobs do **not** take a cartesian product of every gateway mode, IP family, SNAT
setting, and bridge count and then `exclude:` combinations. `test.yml` lists each lane under `strategy.matrix.include`.

Lanes vary some of:

* Target (`shard-conformance`, `control-plane`, `no-uplink`, `multi-homing`, `bgp`, and others listed in the `include` comments)
* Local vs shared gateway mode. See [Architecture](../design/architecture.md).
* IPv4, IPv6, or dual stack
* `noSnatGW` vs `snatGW`
* One bridge vs two (`1br` / `2br`)
* Extra flags (interconnect, route advertisements, network segmentation, DNS name resolver, image family, and others)

Read the current `include:` list in
[ovn-kubernetes/.github/workflows/test.yml](https://github.com/ovn-kubernetes/ovn-kubernetes/blob/master/.github/workflows/test.yml)
before adding a lane. Other workflows (`kind-dpu-offload.yml`, `performance-test.yml`) are separate from `ovn-ci`.

# Conformance Tests

Network Policy v2 conformance is `make -C test conformance`, which runs `TestNetworkPolicyV2Conformance`
(`test/conformance/network_policy_v2_test.go`) against the
[network-policy-api](https://github.com/kubernetes-sigs/network-policy-api/tree/master/conformance)
suite. Changes to those tests go upstream first, then into this repo via a version bump.

In `ovn-ci`, `make conformance` runs after some shard-style e2e targets when `ipfamily` is not `ipv6`
(see the e2e job in `test.yml`).

# Documentation Build Check

MkDocs is built by [`.github/workflows/docs.yml`](https://github.com/ovn-kubernetes/ovn-kubernetes/blob/master/.github/workflows/docs.yml)
(`Test Docs Build`), not by `test.yml`. The job runs `mkdocs build --strict` and uploads the `test-mkdocs-site` artifact
from the step **Upload Artifact (Test)**. Download and unzip that artifact to preview what would be published to
[ovn-kubernetes.io](https://ovn-kubernetes.io/). Deployment of the versioned site is handled separately by
`docs-versioning.yml`.
