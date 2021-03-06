This topic describes how to build a CPI.

## Distribution {: #distribution }

CPIs are distributed as regular releases, typically with a release job called `cpi` and a few packages that provide compilation/runtime environment for that job if necessary (e.g. [bosh-aws-cpi-release](https://github.com/cloudfoundry-incubator/bosh-aws-cpi-release) includes Ruby and [bosh-warden-cpi-release](https://github.com/cppforlife/bosh-warden-cpi-release) includes golang). To qualify to be a CPI release, it must include a release job that has `bin/cpi` executable.

Both `bosh create-env` command and the Director expect to be configured with a CPI release to function properly. In the case of `bosh create-env` command, specified CPI release is unpacked and installed on the machine running the command. For the Director, CPI release job is colocated on the same VM, so that the director release job can access it.

CPIs written in Ruby:

- [AWS CPI release](https://github.com/cloudfoundry-incubator/bosh-aws-cpi-release)
- [Azure CPI release](https://github.com/cloudfoundry-incubator/bosh-azure-cpi-release)
- [OpenStack CPI release](https://github.com/cloudfoundry-incubator/bosh-openstack-cpi-release)
- [vSphere CPI release](https://github.com/cloudfoundry-incubator/bosh-vsphere-cpi-release)
- [vCloud CPI release](https://github.com/cloudfoundry-incubator/bosh-vcloud-cpi-release)

CPIs written in golang:

- [Warden CPI release](https://github.com/cppforlife/bosh-warden-cpi-release)

---
## RPC {: #rpc }

Since CPI is just an executable, following takes place for each CPI method call:

1. Callee starts a new CPI executable OS process (shells out)
1. Callee sends a [single JSON request](#request) over STDIN
   - CPI optionally starts logging debugging information to STDERR
1. CPI responds with a [single JSON response](#response) over STDOUT
1. CPI executable exits
1. Callee parses and interprets JSON response ignoring process exit code

As a reference the Director's implementation is in [bosh_cpi's external\_cpi.rb](https://github.com/cloudfoundry/bosh/blob/master/src/bosh-director/lib/cloud/external_cpi.rb) and the BOSH CLI implements it in [cpi\_cmd\_runner.go](https://github.com/cloudfoundry/bosh-cli/blob/master/cloud/cpi_cmd_runner.go).

### Request {: #request }

* **method** [String]: Name of the CPI method. Example: `create_vm`.
* **arguments** [Array]: Array of arguments that are specific to the CPI method.
* **context** [Hash]: Additional information provided as a context of this execution. Specified for backwards compatibility and should be ignored.

Example:

```yaml
{
	"method": "delete_disk",
	"arguments": ["vol-1b7fb8fd"],
	"context": { "director_uuid":"fefb87c8-38d1-46a5-4552-9749d6b1195c" }
}
```

### Response {: #response }

* **result** [Null or simple values]: Single return value. It must be null if `error` is returned.
* **error** [Null or hash]: Occurred error. It must be null if `result` is returned.
	* **type** [String]: Type of the error.
    * **message** [String]: Description of the error.
    * **ok\_to\_retry** [Boolean]: Indicates whether callee should try calling the method again without changing any of the arguments.
* **log** [String]: Additional information that may be useful for auditing, debugging and understanding what actions CPI took while executing a method. Typically includes info and debug logs, error backtraces.

Example successful response from `create_vm` CPI call:

```yaml
{
	"result": "i-384959",
	"error": null,
	"log": ""
}
```

Example error response from `create_vm` CPI call:

```yaml
{
	"result": null,
	"error": {
		"type": "Bosh::Clouds::CloudError",
		"message": "Flavor `m1.2xlarge' not found",
		"ok_to_retry": false
	},
	"log": "Rescued error: 'Flavor `m1.2xlarge' not found'. Backtrace: ~/.bosh_init/ins..."
}
```

BOSH team maintains several CPIs written in Ruby. We currently take advantage of the `bosh_cpi` gem which provides `Bosh::Cpi::Cli` class that deserialized requests and serializes responses. If you are planning to write a CPI in Ruby, we recommend to take advantage of this gem.

---
## Testing {: #testing }

There are two test suites each CPI is expected to pass before it's considered to be production-ready:

- its own [CPI Lifecycle Tests](https://github.com/cloudfoundry/bosh/blob/master/docs/running_tests.md#cpi-lifecycle-tests) which should provide integration level coverage for each CPI method
- shared [BOSH Acceptance Tests (BATS)](https://github.com/cloudfoundry/bosh/blob/master/docs/running_tests.md#bosh-acceptance-tests-bats) (provided by the BOSH team) which verify high level Director behavior with the CPI activated

---
## Concurrency {: #concurrency }

The CPI is expected to handle multiple method calls concurrently (and in parallel) with a promise that arguments represent different IaaS resources. For example, multiple `create_vm` CPI method calls may be issued that all use the same stemcell cloud ID; however, `attach_disk` CPI method will never be called with the same VM cloud ID concurrently.

!!! note
    Since each CPI method call is a separate OS process, simple locking techniques (Ruby's <code>Mutex.new</code> for example) will not work.

---
## Rate Limiting {: #rate-limiting }

Most CPIs have to deal with IaaS APIs that rate limit (e.g. OpenStack, AWS APIs). Currently it's the responsibility of the CPI to handle rate-limiting errors, properly catch them, wait and retry actions that were interrupted. Given that there is no timeout around how long a CPI method can run, it's all right to wait as long as necessary to resume making API calls. Though it's suggested to log such information.

---
## Debugging {: #debugging }

It usually useful to get a detailed log of CPI requests and responses from the callee. To get a full debug log from `bosh create-env` command set `BOSH_LOG_LEVEL=debug` environment variable.

When working with the Director you can find similar debug logs via `bosh task X --debug` command.
