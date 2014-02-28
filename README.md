// vim: set noexpandtab:
# libcouchbase - The C Couchbase Client Library
	
The Couchbase C client library provides an interface to the Couchbase Cluster
from a C application.  It is used by many of our client libraries as lower
layer which performs network I/O and protocol handling. It may also be used
as a standalone client.

The library is known to function on Linux, Mac OS X and Windows.  Other
platforms may work but are not routinely tested.

The library tries to be conforming to the _C89_ standard ("ANSI C") and as such
should build with any C compiler.

## Basic Usage and APIs
In the basic usage of the client library, your application will establish a
client instance and connect to it. Once the instance has been connected you may
start peforming operations

> ### Error Handling
> In the initial examples error handling will be treated by a simple `die()`
  function which will print the error message and exit. In real applications
  there are various types of errors, some more severe than others, and should
  be handled by the application's own logic where applicable. Error handling
  will be discussed in later sections.

### Initialization and Connection

To create an instance, an `lcb_create_st` structure must first be initalized
with the proper fields:

```C
#include <libcouchbase/couchbase.h>
static void die(const char *msg, lcb_error_t err)
{
	fprintf(stderr,
			"%s: 0x%x, %s\n",
			msg,
			err,
			lcb_strerror(NULL, err));
	exit(EXIT_FAILURE);
}

void create_instance(void) {
	lcb_t instance;
	lcb_error_t err;

	struct lcb_create_st crparams = { 0 };

	/** Initialize the host to one of the cluster nodes */
	crparams.v.v0.host = "1.2.3.4";

	err = lcb_create(&instance, &crparams);
	if (err != LCB_SUCCESS) {
		die("Couldn't create instance", err);
	}
}
```

Once your instance has been created it must first be connected. Since at the
core the library is _asynchronous_ the connection involves _scheduling_ for it,
and then _waiting_ for it to complete.

```C
err = lcb_connect(instance);
if (err != LCB_SUCCESS) {
	die("Couldn't schedule connection", err);
}

lcb_wait(instance);
/** Instance is now connected! */
```

The `host` field in the parameter structure may be one of the following

1. A simple hostname or IP, e.g. `foo.com`
2. A semicolon-delimited list of hosts to try, e.g. `foo.com;bar.com`
3. A host-port pair (of which multiple entries may be passed), thus
   `foo.com:9000;bar.com:800`. Note that passing the port should only be needed
   if the cluster is not listening on the default port of `8091`. This may happen
   if using a customized cluster (for example the `cluster_run` script packaged
   with the server source code) or if there is an HTTP proxy layer between the
   client and server which listens on an alternate port.

Note that errors may take place during the initial connection attempt as well.
We'll examine how to check for errors like this later on.

#### Notable Parameters

In a production deployment, an application will typically be using a non-default
bucket with a password. The following fields are available in the
`v0`, `v1`, and `v2` creation options.

* `user` - The username for the bucket. At the time of writing this should 
   either be empty _or_ set to the name of the bucket itself. Do _not_ use
   `Administrator` as the administrative account may not be used to access
   the data buckets
* `bucket` - the bucket to connect to
* `passwd` - the SASL password for the bucket. As with `user`, do _not_ use
  the administrative password. This password is the _SASL Bucket_ password.

### Performing Key-Value operations
Key value operations in lcb are composed of three stages

1. Schedule the operation
2. Begin I/O on the operation
3. Handle the callback which contains the operation's result


#### Creating the operation request
```C
/**
 * Operations are passed to libcouchbase as an array of pointers
 * to operation structures
 */

lcb_error_t err;
lcb_store_cmd_t cmd = { 0 }, *cmdlist = &cmd;

/** Buffer and size to use for the key */
cmd.v.v0.key = "Hello";
cmd.v.v0.nkey = strlen((const char *)cmd.v.v0.key);

/** Buffer and size to use for the value */
cmd.v.v0.bytes = "World!";
cmd.v.v0.nbytes = 6;

/**
 * Since store_cmd covers different mutation types, we must explicitly state
 * which one we want here
 */
cmd.v.v0.operation = LCB_SET;

err = lcb_store(instance, NULL, 1, &cmdlist);
if (err != LCB_SUCCESS) {
	die("Couldn't schedule storage operation", err);
}

/** Perform I/O on the operation */
lcb_wait(instance);
```

> ##### Operation Structures
> The full API of the operation structures may be found in
  `<libcouchbase/arguments.h>`


#### Setting Up Callbacks for Operations
To actually handle the response, you need to set up a callback for the
operation.  Note that callbacks are set per-instance (not per-operation),
however a different callback must be provided for each distinct operation type.
You typically should do this right after the instance has been created:

```C
#include <inttypes.h> /** For PRIX64 */

static void store_handler(lcb_t instance,
						  const void *cookie,
						  lcb_storage_t op,
						  lcb_error_t err,
						  const lcb_store_resp_t *response)
{
	if (err != LCB_SUCCESS) {
		die("Error during store", err);
	}
	printf("Stored key %.*s with CAS=0x" PRIX64 "\n",
		   resp->v.v0.nkey, resp->v.v0.key, resp->v.v0.cas);
}

void create_instance(void)
{
	/** .. previous code … */
	lcb_set_store_callback(instance, store_handler);
}
```

> ##### Callback Signatures
> The signatures for the callbacks as well as the API functions which can set
  them are found in `<libcouchbase/callbacks.h>`


## Concepts and Detailed Overview

### Synchronous and Asynchronous Interfaces

The library is asynchronous at its core. In addition to the fact that it must
operate in asynchronous environments, it must also operate with and utilize
multiple sockets simultaneously.

Almost all operations within the library are comprised of the following stages

1. Define the request structure itself; this means providing any input for the
   specified operation
   
2. Associate a user defined context with the request. This user defined context
   (which is known in the library as a `cookie`) can be used to associate a
   specific response received in a callback with a specific request. This is
   effectively an opaque void pointer that is never dereferenced or freed by the
   library.
   
3. Schedule the operation along with its `cookie` in the API. This will
   place the request inside the library's internal queue.
   
4. Wait for the operation to complete. The completion of the operation is
   signalled by the receipt of the callback. The callback is always invoked
   (either explicitly or implicitly) with the `cookie` associated with the
   request for the response passed to it. Waiting for the operation to
   complete may be done with a call to `lcb_wait()`.
   

Note that the `lcb_wait()` is intended primarily for synchronous environments
where there is no explicit _event loop_ to which the calling code implicitly
returns control to; thus `lcb_wait` drives the event loop until the operation
is complete. For asynchronous environments, the API calls _schedule_
asynchronous I/O operations and those operations will be performed as soon as
the event loop regains control, and thus there is no need to call `lcb_wait()`

### Error Handling
Most operations return an `lcb_error_t` status code. A successful operation is
defined by the return code of `LCB_SUCCESS`. A full listing of error codes may
be found inside the `<libcouchbase/error.h>` header.

In order to handle the errors properly, the application and developer must
understand what the errors mean and whether they indicate a retriable or fatal
error.

Examples of _transient_ errors include timeout errors or temporary
failures such as `LCB_ETIMEDOUT` (took too long to get a reply),
or `LCB_ETMPFAIL` (server was too busy). Examples of _fatal_ errors include
`LCB_AUTH_ERROR` (authentication failed) or `LCB_BUCKET_ENOENT`
(bucket does not exist).

The distinguishing factor between a fatal and transient error is that a fatal
error would require external intervention (possibly reconfiguring the client
or cluster manually) whereas a transient error does not typically require
intervention as it may be caused by temporary load issues. However depending
on environmental factors an excessive amount of transient errors received over
a prolonged duration of time may also indicate a need for external intervention.

In the examples above, an `LCB_ETIMEDOUT` error indicates a degree of load
on either the server or the network. The load would typically be temporary -
perhaps it is caused by an unusual spike in traffic on an application server
and the `LCB_ETIMEDOUT` errors will likely disappear once the load returns to
normal. On the other hand, a wrong password or bucket name (`LCB_AUTH_ERROR`
and `LCB_BUCKET_ENOENT`) are typically not load related and are more likely
caused by a misconfigured client (bucket name or password was spelled wrong)
or an administrative issue (bucket password was suddenly changed, or bucket
was deleted).

The `lcb_errflags_t` enumeration defines a set of flags which are associated
with each error code. These flags define the _type_ of error e.g.
`LCB_ERRTYPE_INPUT` if this is a result of a malformed parameter passed to the
library, `LCB_ERRTYPE_DATAOP` if this is an error code received from the server
if it was unable to satisfy certain data constraints (for example, a missing
key or a CAS mismatch) and so on.

The `LCB_EIF<TYPE>` where `<TYPE>` represents one of the `errflags_t` flags may
be used to check if an error is of a specific type.


```C
static void get_callback(
	lcb_t instance,
	const void *cookie,
	lcb_error_t err,
	const lcb_get_resp_t *resp)
{
	if (err == LCB_SUCCESS) {
		printf("Successfuly retrieved key!\n");
	} else if (LCB_EIFDATA(err)) {
		switch (err) {
			case LCB_KEY_ENOENT:
				printf("Key not found!\n");
				break;
			default:
				printf("Received other unhandled data error\n");
				break;
		}
	} else if (LCB_EIFTMP(err)) {
		printf("Transient error received. May retry\n");
	}
}
```

#### Success and Failure
Success and failure depend on the context. A successful return code for one of
the data operation APIs (for example `lcb_store`) does not mean the operation
itself was succeeded and the key was successfuly stored. Rather it means the
key was successfuly placed inside the library's internal queue. The actual
error code is delivered within the response itself.

#### Errors Received in Scheduling
Errors may be received when scheduling operations inside the library. If a
scheduling API returns anything but `LCB_SUCCESS` then it implies the operation
itself failed as a whole and _no callback will be delivered for that
operation_. The library may also _mask_ errors during the scheduling phase and
deliver them asynchronously to the user via a callback if e.g. implementation
constraints do not easily allow the immediate returning of an error code.

Conversely, if a scheduling API returns an `LCB_SUCCESS` then the callback
_will always be invoked_.

### Modifying Settings

Various settings may be retrieved and tuned via the `lcb_cntl` interfaces. This
interface allows to modify various timeout, buffer size, and behavior settings.

The `lcb_cntl` is modelled after the `fcntl` and `ioctl` interface in that it
is passed a `mode` and `operation` constant, and an operation-specific pointer
to serve as the operand.

To set the "operation timeout" for example, you may perform the following

```C
lcb_error_t err;
lcb_uint32_t tmo;

/** Make the timeout 5 seconds */
tmo = 5000000;
err = lcb_cntl(instance, LCB_CNTL_SET, LCB_CNTL_OP_TIMEOUT, &tmo);
```

The first argument to `lcb_cntl` is the instance which represents the current
handle on the cluster. The second is the mode of the operation which can either
be `LCB_CNTL_SET` to modify a setting or `LCB_CNTL_GET` to retrieve the
setting.  The third argument is the actual setting to operate on - which in
this case is `LCB_CNTL_OP_TIMEOUT`. Finally the last argument is the operand
for the operation.  The type and format of this argument is specific to the
mode and operation specified.

Typically for the `LCB_CNTL_GET` mode, data will be copied from the internal
setting into the argument provided, and in the `LCB_CNTL_SET` mode, data will
be copied from the operand into the internal setting. Note that not all
operations support all modes and that some operations may not be available in
some instance/cluster versions.

The operation will return an `LCB_SUCCESS` if successeful. If the operation is
not supported in the current library version, `LCB_NOT_SUPPORTED` will be
retuned. Note that specific handlers may return specific error codes as well.

The full listing of all the opcodes and the specific arguments they accept can
be found inside the `<libcouchbase/cntl.h>` header.

In order to retain backwards and forwards compatibility with older/newer
versions it is recommended to reference the actual numeric opcode instead of
the constant name itself.  This way if the application is compiled against an
older library version it will result in a runtime error rather than preventing
compilation (unless of course you want this).

### Logging

You may use the library's logging API to forward messages to your logging
framework of choice. Additionally you may also enable logging to the console's
standard error by setting the `LCB_LOGLEVEL` environment variable.

Setting the logging level via the environment allows applications linked against
the library which do not offer native support for logging to still employ the
use of the diagnostics provided by the library; thus you may do something like

```
$ LCB_LOGLEVEL=5 ./my_app
```

and logging information will be displayed.

The value of the `LCB_LOGLEVEL` is an integer from 1 and higher. The higher
the value the more verbose the details.

To set up your own logger, you must first define a logging callback to be
invoked whenever the library emits a logging message

```C
static void logger(
	lcb_logprocs *procs,
	const char *module,
	int severity,
	const char *srcfile,
	int srcline,
	const char *fmt,
	va_list ap)
{
	char buf[4096];
	vsprintf(buf, ap, buf);
	dispatch_to_my_logging(buf);
}

static void apply_logging(lcb_t instance)
{
	lcb_logprocs procs = { 0 };
	procs.v.v0.callback = logger;
	lcb_cntl(instance, LCB_CNTL_SET, LCB_CNTL_LOGGER, &procs);
}
```

Note that the `lcb_logprocs` pointer must point to valid memory and must not be
freed by the user once passing it to the library until the instance associated
with it is destroyed.


Additional diagnostic information is also provided by the _error callback_.
The error callback is considered to be a legacy interface and should generally
not be used. The error callback however does allow programmatic capture of some
errors - something which is not possible with the logging interface (easily).

Specifically the error callback will receive error information when a bootstrap
or configuration update has failed.

```C
static void error_callback(lcb_t instance, lcb_error_t err, const char *msg)
{
	/** … */
}

lcb_set_error_callback(instance, error_callback);
```

### Timeouts

As a general prelude to timeouts, timeouts implemented in the library are
implemented with the aim of preventing a single operation hanging or waiting
indefinitely for an I/O operation to complete. Thus while typically timeouts
may be exact, if they are not they will tend to fire sooner rather than
later.

On the other hand, note that internally timeouts are implemented within an
event loop (as opposed to a kernel-based signal timer). This means

* The timer can only fire when and if the event loop is active
* The timer may end up being delayed if it ends up being scheduled
  _after_ other operations which themselves perform blocking timeouts or long
  busy loops. (Note that libcouchbase does not do this, but your own code may).

As an I/O library, libcouchbase has quite a few timeout settings. The main
timeout setting is known as the _operation timeout_ and indicates the amount of
time to wait from when the operation was scheduled. If a response was not
received before this time period the library will fail the operation and invoke
the callback - at which point the error will be set to `LCB_ETIMEDOUT`. The
operation to control this via `lcb_cntl` is `LCB_CNTL_OP_TIMEOUT`.

In addition to the operation timeout, there is also the bootstrap timeout which
controls the amount of time the library will wait for the initial bootstrap.
The bootstrap timeout is comprised of two timeout values. One value controls
the overall amount of time the library should wait until a configuration has
been received, while the other controls the amount of time per-node that the
library will wait for until the next node is retried.

The first timeout represents the absolute timeout and is accessed via the
`LCB_CNTL_CONFIGURATION_TIMEOUT`. As mentioned before this value places the
absolute limit on the amount of time the library will try to wait until the
client has been bootstrapped before delivering the error. Note that this timeout
only makes sense during the initial connection and has no effect threafter.

An additional timeout setting is the per-node bootstrap timeout. When fetching
the cluster configuration there are often multiple nodes which may function
as configuration sources. Configurations are initially retrieved once during
initialization but may also be fetched later on typically when an error condition
is detected by the library or when the cluster topology changes. In such
situations the behavior is to start fetching the configuration from each
node in the list, in sequence. If the retrieval of the config from the first
node fails with a timeout (i.e. the node is unresponsive) the library will
proceeed to fetch from the next node and so on until all nodes are exhausted.
This setting, may be accessed via the `LCB_CNTL_CONFIG_NODE_TIMEOUT` operation.

### Cluster Map and Bootstrapping

At the core of the library is the ability to discover the toplogy of the
cluster.  This topology is known as a _cluster map_ or a _vBucket map_, or
simply a _config_. The config provides the client with information about which
_nodes_ are located in the cluster, and which _vBuckets_ belong to which nodes.

Each time the client is given a request for an operation, it will query the
config to determine the appropriate node the request should be sent to, and
then place the serialized request into that node's socket buffer.

During cluster topology changes, the client may receive or request
configuration updates multiple times. These updates typically reflect a vBucket
being moved to another node.  During these changes it will also happen that the
client may forward data to the wrong server and can internally retry
operations several times.

For servers before 2.5 the only way to retrieve the cluster configuration was
via the HTTP REST API endpoint. The client would make a request to
`/pools/default/bucketStreaming/$bucket` and receive a sequence of
configuration information encoded as a JSON payload. (`$bucket` here means the
name of the bucket).

With server 2.5 the cluster has added support for bootstrap over the
`memcached` protocol which allows us to not require a dedicated socket for
retrieving the operation. From an API perspective this is mostly transparent,
but will be detailed later on.

If the client fails to bootstrap during the initial connect phase then the
client is considered to be unusable and should be reconstructed with the proper
list of hosts.  The client will attempt to retrieve the configuration from each
host and will fail when either all hosts have been exhausted or when the
bootstrap timeout has been reached.

#### Configuration Sources

The cluster configuration may be obtained either the Memcached protocol (in
what is called **C**luster **C**onfiguration **C**arrier **P**ublication, or
_CCCP_) as well as via the legacy mode which utilizes a connection to the REST
API.

Note that `CCCP` is available only in cluster versions 2.5 and higher and
LCB versions 2.3 and higher.

The default behavior of the library is to first attempt bootstrap over `CCCP`
and then fallback to HTTP if the former fails. `CCCP` bootstrap is preferrable
as it does not require a dedicated "Configuration Socket", does not require the
latency and overhead of the HTTP protocol, and is more efficient to the internals
of the cluster as well.

It is recommended to explicitly enable or disable one of the configuration
sources. The only reason why the library attempts _CCCP_ and then HTTP is to
support legacy environments where a < 2.5 cluster is available, and/or a mixed
version cluster where some nodes may support _CCCP_ and some may not.

To explicitly define the set of configuration modes to use specify this inside
the `lcb_create_st` structure

The following snippet disables HTTP bootstrapping:

```C
struct lcb_create_st crparams = { 0 };

lcb_config_transport transports[] = {
	LCB_CONFIG_TRANSPORT_CCCP,
	LCB_CONFIG_TRANSPORT_LIST_END
};

/** Requires version 2 */
cparams.version = 2;
crparams.v.v2.transports = transports;
crparams.v.v2.host = "foo.com;bar.org;baz.net";

lcb_t instance;
lcb_create(&instance, &crparams);

```

The `transports` field accepts an array of "enabled transports" followed by the
end-of-list element (`LCB_CONFIG_TRANSPORT_LIST_END`). If this parameter is
specified then the library will _only_ use the transports in the list which are
specified.

> The enabled transports will remain valid for the duration of the instance. This
> means that it will be used for the initial bootstrap as well as any subsequent
> configuration updates.

> #### Memcached Buckets
> Memcached buckets do **not** support _CCCP_. You must ensure that HTTP
> is enabled for them (which is the default)

#### Configuration Updates

During the lifetime of a client, its cluster configuration may need to change
or be retrieved. Reasons for this include

* The cluster pushing a new configuration to the client
* Nodes being added, removed, or failed over to/from the cluster
* Client proactively fetching configuration due to an error


These changes are collectively known as _topology changes_ or _configuration
changes_. Note that as mentioned in the list, fetching a configuration may
not _always_ be the result of an actual toplogy change. For example if a certain
node becomes unreachable to the client then the client will attempt to fetch a
new configuration under the assumption that the unresponsive node has potentially
been ejected from the cluster.

For proactive configuration fetches (such as in response to an error) there is
effectively a responsiveness-for-efficiency trade off. If the client does not
check for updated configurations often enough then it risks failing operations
which could have succeeded if it knew about the updated topology, while if the
client fetches the configuration too quickly it can cause a lot of traffic and
increase load on the server (this especially holds true with HTTP). Additionally
some server versions allocate quite a few resources for each HTTP connection and
thus maintaining idle HTTP connections may also be more expensive than
re-establishing them on-demand.

Rather than guess your hardware and infrastructure requirements, there are
several settings in the library to help you tune it to your needs.

##### Error Threshholds

Each time a network-like error is encountered in the library (i.e. a timeout 
or connection error), an internal error counter is incremented. When this
counter's value reaches a specified value the library fetches a new configuration
and resets this counter to zero.

Additionally, the intervals between successive increments are timed, and if
a call is made to increment the counter and the time between the last call
exceeds a certain threshold then the configuration is refetched again as well.

Both these behaviors can be controlled via `lcb_cntl()`. To modify the error
counter threshold, use the `LCB_CNTL_CONFERRTHRESH` setting. To modify the
error delay threshold, use the `LCB_CNTL_CONFDELAY_THRESH` setting.

#### Source-Specific settings

Settings may be supplied to each specific source as well to tune it.

##### _CCCP_-Specific settings

_CCCP_ relies on being able to connect to a memcached instance listening
on a predefined port. Usually this is `11210`, but may be different in some
environments.

You may use the `mchosts` field in the `v2` sub-structure of `lcb_create_st`
to specify `host:port` pairs for memcached nodes (that is, which memcached ports
are used by Couchbase). If specified these override the `host` field in the same
structure.

##### HTTP-Specific Settings

HTTP-bootstrap used to follow a model of maintaining a persistent connection
to the cluster and relying on the cluster to _push_ updates to it. As part of
the migration to _CCCP_, the internal behavior regarding HTTP has changed as
well, having the library utilize it as a pull-based configuration source.
Nevertheless it can still function slightly as a _push_ source.

Whenever the client has finished retrieving the configuration from the HTTP
interface, rather than closing the socket immediately it will let the socket
idle for a small amount of time. The goal of this idle period is to allow any
subsequent pushed configurations from the server to arrive without having to
re-open the socket.

To control this idle timeout, use the `LCB_CNTL_HTCONFIG_IDLE_TIMEOUT` operation
for `lcb_cntl`

#### File-Based Configuration Cache

The library allows to "cache" the configuration settings in a file and then pass
this file to create another instance which will be configured without having to
perform network I/O to retrieve the initial map.

The goal of this feature is to limit the number of initial connections for
retrieving the cluster map, as well as to reduce latency for performing simple
operations. Such a usage is ideal for short lived applications which are spawned
as simple scripts (e.g. PHP scripts).

To use the configuration cache you must use a special form of initialization:

```C
lcb_error_t err;
lcb_t instance;
struct lcb_cached_config_st cacheinfo;

memset(&cacheinfo, 0, sizeof(cacheinfo));
cacheinfo.cachefile = "/tmp/lcb_cache";
/** Access the normal lcb_create_st structure via the 'createopt' field */
cacheinfo.createopt.v.v0.host = "foo.com";

err = lcb_create_compat(LCB_CACHED_CONFIG, &cacheinfo, &instance, NULL);
if (err != LCB_SUCCESS) {
	die("Couldn't create cached info", err)
}
```

If the cache file does not exist, the client will bootstrap from the network
and write the information to the file once it has received the map itself.
You may check to see if the client has loaded a configuration from the cache
(and does not need to fetch the config from the network) by using `lcb_cntl()`

```C
int is_loaded = 0;
lcb_cntl(instance, LCB_CNTL_GET, LCB_CNTL_CONFIG_CACHE_LOADED, &is_loaded);
if (is_loaded) {
	printf("Configuration cache file loaded OK\n");
} else {
	printf("Configuration cache likely does not exist yet\n");
}
```

> _Memcached_ buckets are not supported with the configuration cache

### Operation Scheduling

The core goal of the library is of course to allow you to perform data operations
against the cluster. Data operations are submitted to one of the API calls which
will then convert your inputs into packets which are scheduled over the network.
For these simple data commands you may provide one or more request structures and
schedule them. If the scheduling is successful then you are guaranteed to receive
a number of callbacks exactly equal to the number of requests scheduled.

Crucial to working effectively with callbacks and data operations are the
following:

1. You will typically need to pass your own `cookie` pointer to the API call. The
   cookie is your only way to associate the rest of the application code.
2. If you have multiple keys to deal with it is significantly more efficient to
   schedule them in bulk by allocating an array of commands an an array of
   pointers to those commands.
   

The following example shows how to store a key in the cluster, retrieve it,
reverse its value, and store it again

```C
/** vim: set noexpandtab: */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <libcouchbase/couchbase.h>

static void die(const char *msg, lcb_error_t err)
{
	fprintf(stderr, "Got error %s. Code=0x%x, %s\n",
			msg, err, lcb_strerror(NULL, err));
	exit(EXIT_FAILURE);
}

static void get_callback(
	lcb_t instance,
	const void *cookie,
	lcb_error_t err,
	const lcb_get_resp_t *resp
);

static const char *key = "My Key";
static const char *value = "My Value";

int main(void)
{
	lcb_t instance;
	lcb_error_t err;
	lcb_store_cmd_t s_cmd = { 0 }, *s_cmdlist = &s_cmd;
	lcb_get_cmd_t g_cmd = { 0 }, *g_cmdlist = &g_cmd;
	char scratch[4096];
	
	err = lcb_create(&instance, NULL);
	if (err != LCB_SUCCESS) {
		die("Couldn't create instance", err);
	}
	
	lcb_set_get_callback(instance, get_callback);
	err = lcb_connect(instance);
	if (err != LCB_SUCCESS) {
		die("Couldn't schedule connection", err);
	}
	lcb_wait(instance);
	
	s_cmd.v.v0.key = key;
	s_cmd.v.v0.nkey = strlen(key);
	s_cmd.v.v0.bytes = value;
	s_cmd.v.v0.nbytes = strlen(value);
	s_cmd.v.v0.operation = LCB_SET;
	
	err = lcb_store(instance, NULL, 1,
			(const lcb_store_cmd_t * const *)&s_cmdlist);

	if (err != LCB_SUCCESS) {
		die("Couldn't schedule store operation!", err);
	}
	lcb_wait(instance);
	
	g_cmd.v.v0.key = s_cmd.v.v0.key;
	g_cmd.v.v0.nkey = s_cmd.v.v0.nkey;
	err = lcb_get(instance, scratch, 1,
			(const lcb_get_cmd_t * const *)&g_cmdlist);
	if (err != LCB_SUCCESS) {
		die("Couldn't schedule get operation!", err);
	}
	lcb_wait(instance);
	
	/**
	 * Inside the get callback we've copied over the value data from the
	 * callback into the scratch buffer which we passed as a cookie
	 */
	{
		char reversed[4096];
		int ii;
		int curlen = strlen(scratch);
		curlen--;
		for (ii = curlen; ii >= 0; ii--) {
			reversed[curlen-ii] = scratch[ii];
		}
		s_cmd.v.v0.bytes = reversed;
		s_cmd.v.v0.operation = LCB_APPEND;
		/** Since it's reversed there's no need to re-set the size again */
		err = lcb_store(instance, NULL, 1,
				(const lcb_store_cmd_t * const *)&s_cmdlist);
		if (err != LCB_SUCCESS) {
			die("Couldn't schedule second APPEND", err);
		}
		lcb_wait(instance);
	}
	/** Now get the key back again */
	err = lcb_get(instance, scratch, 1,
			(const lcb_get_cmd_t * const *)&g_cmdlist);
	if (err != LCB_SUCCESS) {
		die("Could not schedule final get operation", err);
	}
	lcb_wait(instance);
	printf("Buffer in server is now %s\n", scratch);
	lcb_destroy(instance);
}

static void get_callback(
	lcb_t instance,
	const void *cookie,
	lcb_error_t err,
	const lcb_get_resp_t *resp)
{
	char *target = (char *)cookie;
	if (err != LCB_SUCCESS) {
		die("Got error inside get callback", err);
	}
	memcpy(target, resp->v.v0.bytes, resp->v.v0.nbytes);
	target[resp->v.v0.nbytes] = '\0';
}
```

Compile as

```
$ cc -o example example.c -lcouchbase
$ ./example
Buffer in server is now My ValueeulaV yM
```

The above example program does several things in sequence. First it creates an
instance. Note that the `lcb_create_st` structure is not passed. In lieu of this
structure it is assumed you are connecting to the `default` bucket and the host
is `localhost`.

After instantiation, the initial connection is performed via a call to 
`lcb_connect()`. To actually apply the connection, `lcb_wait()` is called.

The program will then try to store a key, retrieve its value, append the
reversed form of the value to the existing value, and finally retrieve the
value again.

Command structures are not owned by the library when passed to them, thus the
memory pointed to by the command structure, the command list, and the key/value
data need only remain valid for the duration of the scheduling API (i.e.
`lcb_store()`, `lcb_get()`). Once the operation completes any relevant data
will have already been copied out of the buffers and they may be freed or reused
for other operations.

Callbacks for operations return the data inside a _response_ structure. The
pointers received within the callback (except for the cookie itself) are
valid only within the duration of the callback itself. If you wish to use
the data outside the callback then you must copy the data elsewhere as
demonstrated in the example where the cookie is a `char[4096]` which is the
target for `memcpy()`.

#### Scheduling/Batching Operations in Bulk
Operations may be scheduled in bulk. Bulk operations will optimize and decrease
network usage by packing many commands into a single buffer. Typically bulked
operations will not actually save raw transfer sizes but will substantially
decrease TCP overhead.

Currently there are two ways to schedule operations in bulk. The first way
involves creating a list of commands and passing it to the library in a single
call and the second involves calling the scheduling API with a single command
and finally calling `lcb_wait()` when all commands have been scheduled.

Scheduling with a command list:

```C
#define NUM_COMMANDS 10
int ii;
lcb_store_cmd_t cmds[NUM_COMMANDS];
lcb_store_cmd_t const* cmdlist[NUM_COMMANDS];
lcb_error_t err;

for (ii = 0; ii < NUM_COMMANDS; ii++) {
	char *kbuf;
	char *vbuf;
	
	kbuf = malloc(128);
	vbuf = malloc(128);
	sprintf(kbuf, "Key_%d", ii);
	sprintf(vbuf, "Value_%d", ii);
	cmds[ii].v.v0.key = kbuf;
	cmds[ii].v.v0.nkey = strlen(kbuf);
	cmds[ii].v.v0.bytes = vbuf;
	cmds[ii].v.v0.nbytes = strlen(vbuf);
	cmds[ii].v.v0.operation = LCB_SET;
	cmdlist[ii] = &cmds[ii];
}

err = lcb_store(instance, NULL, NUM_COMMANDS, cmdlist);
for (ii = 0; ii < NUM_COMMANDS; ii++) {
	free(cmds[ii].v.v0.key);
	free(cmds[ii].v.v0.bytes);
}
lcb_wait(instance);
```

Note that two arrays are required here. One array is required to hold the
command structures themselves, and the other array is required to hold the
_pointer list_ for the command structures.

Note that in this mode if the scheduling function returns with an error
then no callback will be invoked. If it returns with `LCB_SUCCESS` then exactly
ten callbacks will be invoked.

An alternative approach is scheduling within a loop like so:

```C
#define NUM_COMMANDS 10
int ii;
lcb_error_t err;
for (ii = 0; ii < NUM_COMMANDS; ii++) {
	char kbuf[128];
	char vbuf[128];
	lcb_store_cmd_t cmd = { 0 }, *cmdp = &cmd;
	
	sprintf(kbuf, "Key_%d", ii);
	sprintf(vbuf, "Value_%d", ii);
	cmd.v.v0.key = kbuf;
	cmd.v.v0.nkey = strlen(kbuf);
	cmd.v.v0.bytes = vbuf;
	cmd.v.v0.nbytes = strlen(vbuf);
	cmd.v.v0.operation = LCB_SET;
	err = lcb_store(instance, NULL, 1, (const lcb_store_cmd_t * const *)&cmdp);
}
lcb_wait(instance);
```

Here the operations are implicitly scheduled in bulk, relying on the fact that
no actual network I/O is performed before `lcb_wait()` is called.

Both methods have their own benefits and drawbacks. The first method simplifies
error handling by allowing a single error code to dictate the success or failure
of scheduling all the operations, while the second method must figure out what
to do if one of the commands failed to schedule.

However the first method requires much more memory allocations than the
latter as it requires all key/value data to remain valid until all them can
be passed to the library. Additionally it requires that a dedicated pointer
array exist as well.

How you batch operations and how many operations you batch will depend on your
use requirements and network speed. Typically TCP stacks will only be able to
buffer so much data (usually several hundred kilobytes) before completely
filling up the kernel's write buffer. Data batched beyond that will just slow
down the library causing it to allocate and buffer more data than needed.

**It is strongly recommended you use the batched operation mode if you have
multiple items to send/receive**

#### Key and Value data format and limits
There are no limitations on the format of the key and value data (other than
the key must not be empty). It is _recommended_ that the key be a _UTF-8_
compatible sequence without any whitespace.

Currently key sizes are limited to 150 bytes and value sizes are limited to
20 MB. Consult the server manual for your server version to get more exact
information on these constraints.

You may make use of the `flags` field which is provided as input to the
`lcb_store_cmd_t` structure (for storing data ) and provided as output to the
`lcb_get_resp_t` structure (when it is fetched) to provide extra hints and
metadata about the key-value pair.

> ##### Flags Usage
> Traditionally (and currently) the flags are used to indicate the format of the
> data. Currently there is no standard for the flags format, but several clients
> rely on client-specific flags values to determine the value of the data.
> Before you use this field, check to see if you are operating with any other
> client/SDK and ensure that your usage of the flag field will not conflict
> with it. Additionally note that clients may extend their usage of flags
> as well.


#### Simple vs. Compound/Complex Commands
This terminology refers not to the data operation of the command but rather
the callback semantics. Most commands are _Simple_; that is you receive a single
callback for a single request. Other commands are however more complex and
invoke a callback multiple times per request. These commands are finished with
a so-called _NULL-Callback_ which indicates that no more callbacks will be
invoked for the command.

Read the callback documentation carefully for each of the APIs to see how
to handle the received callbacks.


### Views Queries
In addition to the _memcached_ protocol, this library also provides the ability
to issue HTTP queries. HTTP queries may be issued to port _8092_ and require
a path.

Note that currently the library does not include utilities for constructing
the view queries themselves. Specifically the library provides these features

* Selecting a proper node for a query
* Utilizing the asynchronous I/O layer as the rest of the library
* Receiving the response either as a series of callbacks for each received
  chunk, or all at once when the entire response has been buffered.
  
To create an HTTP request you will need to create an `lcb_http_cmd_t`. In
addition to the command structure you will also be given an 
`lcb_http_request_t` which may be used to cancel the request later on.

```C
lcb_http_cmd_t htcmd = { 0 };
lcb_http_request_t htreq;
lcb_error_t err;

const char *path = "beer-sample/_design/beer/_view/brewery_beers";

htcmd.v.v0.path = path;
htcmd.v.v0.npath = strlen(path);
htcmd.v.v0.method = LCB_HTTP_METHOD_GET;

err = lcb_make_http_request(
	instance,
	cookie,
	LCB_HTTP_TYPE_VIEWS,
	&htcmd,
	&htreq);
	
if (err != LCB_SUCCESS) {
	die("Couldn't schedule HTTP request", err);
}

```

And the callbacks:

```C
static void
http_complete(
	lcb_http_request_t req,
	lcb_t instance,
	const void *cookie,
	lcb_error_t err,
	const lcb_http_resp_t *resp)
{
	const char **curhdr;
	if (err != LCB_SUCCESS) {
		die("Got fatal error for request", err);
	}
	
	if (resp->v.v0.status < 200 || resp->v.v0.status > 299) {
		fprintf(stderr, "Got non-success HTTP status: %d\n",
				resp->v.v0.status);
	}
	
	for (curhdr = resp->v.v0.headers; *curhdr; curhdr += 2) {
		printf("Got HTTP Header %s: %s\n", curhdr[0], curhdr[1])
	}
	if (resp->v.v0.nbytes) {
		fprintf(stderr, "Got %.*s for response body",
				(int)resp->v.v0.nbytes, resp->v.v0.bytes);
	}
}
```

The `http_complete` callback is invoked when the HTTP response is
complete. This either happens if an error has been detected or if the
server has sent all the data for the response. You can see the callback
via the `lcb_set_http_complete_callback` (see 
`<libcouchbase/callbacks.h>` for the API).

Note that the callback should be ready to detect two error conditions.
The first is an actual error in the response delivery, for example a
network or timeout error. The second condition is a negative status
reply by the HTTP server itself.

Also provided with the response is a list of headers inside a `char**`
array. The char array is structure in the form of 
`{ "header", "value"}` and so on, so iterate over the array incrementing
the current element by two each time.

#### Error Handling

HTTP Operations require error handling to ensure the scheduling of the
request does not return an `LCB_EINVAL`. This may happen for example if
the path string is empty or contains illegal characters. Other error codes
may be returned if there are no current nodes available for a view request.

Various HTTP codes may be returned if the target path does not exist or
the URL and/or body is not recognized on the server.

#### Uploading Data

Data may be uploaded to the server using the `LCB_HTTP_METHOD_POST` and
`LCB_HTTP_METHOD_PUT` request methods. The method appropriate depends on
the specific REST API being used (for example, creating a new design document
will require a `PUT` whereas sending extended view query parameters should be
done via a `POST`).

The server may also require that you provide a `Content-Type` header for your
data. Typically this should be `application/json`.

#### Redirect Handling

By default the library will attempt to retry redirect responses (HTTP `3xx`
codes) and forward them to the new server. The number of maximum redirects
the library will try to follow before returning an error may be specified
via the `LCB_CNTL_MAX_REDIRECTS` setting via `lcb_cntl()`. Redirects will
typically take place when a server is being ejected from the cluster as
a notification to the client that it should use a different node.

#### HTTP Streaming

Instead of receiving the entire response in one gulp, you can use the
streaming (i.e. `chunked`) mode inside the request, thus:

```C
lcb_http_cmd_t cmd;
/** ... */
cmd.v.v0.chunked = 1;
/** ... */
```

> Note that `chunked` does not imply the HTTP-level
> `Transfer-Encoding: chunked` directive of the same name.

With this option you will receive chunks of data as they arrive inside
a `data_callback`. The prototype of the data callback is the same as
the completion callback, except that the `body` and `nbody` fields will
contain the number of bytes in the current chunk rather than for the
entire response.

When the response is complete, the completion callback will be invoked
and the `body` and `nbody` fields will be empty.

#### Using Alternate HTTP Implementations

If you wish to use a different HTTP library you can still benefit from the
library's automatic handling of cluster updates. You may select a node
from the list of cluster nodes returned via the `lcb_get_server_list()`
function.

#### HTTP Timeouts

You may adjust the timeout for HTTP operations by modifying the
`LCB_CNTL_VIEW_TIMEOUT` setting in `lcb_cntl`.

### Durability and Persistence Requirements

Utilizing the `OBSERVE` command (explained later) the client may
determine the replication and persistence status of a given item across
the cluster. For clients who which to ensure their application does not
proceed until a given item has been persisted/replicated to a specific
number of nodes, they may use the `lcb_durability_poll()` API call to
poll the servers until either an interval is reached or the items have
been properly persisted/replicated to their respective targets.

```C

static void
durability_callback(
	lcb_t instance,
	const void *cookie,
	lcb_error_t err,
	const lcb_durability_resp_t *res)
{
	if (err != LCB_SUCCESS) {
		die("Response was not successful for durability", err);
	}
	printf("Exists on master?\n", res->v.v0.exists_master);
	printf("Item persisted to %d nodes\n", res->v.v0.npersisted);
	printf("Item replicated to %d nodes\n", res->v.v0.nreplicated);
	printf("Needed %d responses to poll\n", res->v.v0.nresponses);
	printf("Conformance to requirements with err=0%x", res->v.v0.err);
	printf("CAS of item (as exists on master): " PRIX64, resp->v.v0.cas);
}

static void
storage_callback(
	lcb_t instance,
	const void *cookie,
	lcb_storage_t op,
	lcb_error_t err,
	const lcb_store_resp_t *resp)
{
	lcb_durability_opts_t options;
	lcb_durability_cmd_t cmd = { 0 }, *cmdp = &cmd;
	
	if (err != LCB_SUCCESS) {
		die("Got error in store callback", err);
	}

	cmd.v.v0.key = resp->v.v0.key;
	cmd.v.v0.nkey = resp->v.v0.nkey;
	cmd.v.v0.cas = resp->v.v0.cas;
	
	options.persist_to = -1;
	options.replicate_to = -1;
	options.cap_max = 1;
	
	lcb_durability_poll(instance, cookie, &options, 1, &cmdp);
}	
```

The API for the structures may be found in `<libcouchbase/durability.h>`.

The basic use model is to schedule a durability poll right after the item
has been stored (for example, within a successful store callback).
The `lcb_durability_opts_t` structure defines the parameters
for the persistence and replication requirements. You can define these to
contain the minimum number of nodes you wish the item to be persisted
and replicated to. Note that the maximum number of nodes you may persist
to is the the least of the number of nodes in the cluster, or the number
of replicas plus one node. The maximum number of nodes you may replicate to
is the least of the number of replicas or the number of nodes in the cluster
minus one (to exclude the master).

You can also set the `cap_max` field to true and then set the `persist_to` and
`replicate_to` fields to a high number (as they are unsigned values, you can
simply use `-1`) and the library will auto-adjust for the desired number of
replicas which are available at the time of the request.

To actually specify the keys to poll for, use the `lcb_durability_cmd_t`
structure, employing one such structure per key (as with the other commands).

You should also set the CAS of the structure (if possible) so that you can
safeguard against concurrent modifications of the same item.

Note that the command is performed via polling multiple nodes repeatedly.
Depending on application, network, and server load this may have negative
consequences on performance. To adjust the polling preferences set the
`interval` field in the options structure. This will specify the time to
wait between successive polls of all the nodes in the event where the first
poll did not persist or replicate to the number of required nodes. A higher
interval means less network load (but a potentially longer time to receive
a response) whereas a lower interval means a quicker response time at the
expense of more calls to the network.

Additionally you may batch multiple items into a single durability polling
operation. At the expense of the added latency in waiting for all the responses
from the first (store) operation to arrive you can gain more efficiency at the
polling end by batching as many keys as possible into a single `OBSERVE`
request.

#### Timeouts

Since persisting a node across multiple servers may take longer than simply
storing the item to the master node's cache, durability operations have
their own timeout setting. You may modify the timeout inside the options
structure. If a specific timeout is not provided it will default to the
value of the `LCB_CNTL_DURABILITY_TIMEOUT` `lcb_cntl()` setting.

In addition to the durability timeout, the default durability poll interval
may also be adjusted using the `LCB_CNTL_DURABILITY_INTERVAL` setting.

### `OBSERVE` Сommand

The `OBSERVE` command may be used to retrieve some information about a given
item's replication and persistence status on a variety of nodes.

The following shows a sample usage of `lcb_observe()`

```C
static void
observe_callback(
	lcb_t instance,
	const void *cookie,
	lcb_error_t err,
	const lcb_observe_resp_t *resp)
{
	
	if (err != LCB_SUCCESS) {
		die("Observe failed", err);
	}
	
	if (!resp->v.v0.nkey) {
		/**
		 * We get an indeterminate number of callbacks. The final
		 * callback will always contain 0 for nkey and/or NULL for key
		 */
		printf("Got final null callback for %p.\n", cookie);
	}
	
	if (resp->v.v0.from_master) {
		printf("Got reply from master\n");
	} else {
		printf("Got reply from replica\n");
	}
	
	printf("Got raw status code 0x%x\n", resp->v.v0.status);
	printf("Got raw CAS 0x" PRIX64 "\n", resp->v.v0.cas);
	
	/** Since LCB_OBSERVE_FOUND is 0x00 we can't do a bitwise comparison */
	
	if (resp->v.v0.status == LCB_OBSERVE_FOUND) {
		printf("Key exists in node's cache (but not persisted)\n");
	}
	
	if (resp->v.v0.status & LCB_OBSEVE_PERSISTED) {
		printf("Key is persisted on disk\n");
	}
	
	if (resp->v.v0.status & LCB_OBSERVE_NOT_FOUND) {
		printf("Key does not exist in cache\n");
		
		/** 
		 * An item can still be schedule for deletion from disk, but actually
		 * exist on the disk while being logically purged from cache
		 */
		if (resp->v.v0.status & LCB_OBSERVE_PERSISTED) {
			printf("But key is still awaiting deletion from disk\n");
		}
	}
}

void
schedule_observe()
{
	lcb_observe_cmd_t cmd = { 0 }, *cmdp = &cmd;
	lcb_error_t err;
	
	cmd.v.v0.key = "foo";
	cmd.v.v0.nkey = 3;
	err = lcb_observe(instance, NULL, 1, &cmdp);
	lcb_wait(instance);
}
```

The response callback for the `lcb_observe()` call deserves most detail. A
callback (set via `lcb_set_observe_callback()`) will be invoked for each
node in the cluster which is either a replica or a master for the key. Once
all the callbacks have been received the library will send a so-called
'null callback' which contains the cookie, but with the response fields all
set to empty/NULL values.

The response structure itself will contain a `from_master` field indicating
if this reply was received from a master or replica node.
The response will also contain the CAS of the current item as it exists on
the given node (if the item exists) and a status code containing various bits
set depending on whether the item has been persisted, replicated, both, or
neither.

As of version 2.3, the `lcb_observe()` call may also be used to determine
if the item exists on master without requiring to perform
any specific kind of mutation or retrieval on the item. This is done by
setting the `version` field of the command structure to 1 and setting
the `options` field to `LCB_OBSERVE_MASTER_ONLY` like so:

```C
void schedule_observe()
{
	lcb_observe_cmd_t cmd, *cmdp = &cmd;
	lcb_error_t err;
	
	memset(&cmd, 0, sizeof(cmd));
	cmd.version = 1;
	cmd.v.v1.key = "foo";
	cmd.v.v1.nkey = 3;
	cmd.v.v1.options = LCB_OBSERVE_MASTER_ONLY;
	err = lcb_observe(instance, NULL, 1, &cmdp);
	if (err != LCB_SUCCESS) {
		die("Couldn't schedule observe request", err);
	}
	lcb_wait(instance);	
}

Note that using the `LCB_OBSERVE_MASTER` option will **only contact the master**
node, and will not contact any replica.
and as such will only send out a single packet to the cluster.
```

Note that you will still receive two callbacks; one which contains the reply
from the master, and one which will signal the final _null callback_.

### Reading Replies from Replicas

While couchbase is a multi-master multi-replica cluster, only a single master
can exist for a single vbucket at a given time. If the master goes down then
master access for the keys it hosted become unavailable. While replicas
get promoted to masters during a failover, for some time sensitive data and use
cases it may not be reasonable to wait until a node is manually or
automatically failed over.

You may read data from a replica by using the `lcb_get_replica()` call.
This functions like a normal `lcb_get()` except it reads data from the
replicas. Note that the data from a replica is not considered to be
authoritative as it may be older than data on the master. If the master
comes back online it may potentially be overwritten by a newer copy.

To invoke `lcb_get_replica` you will need an `lcb_get_replica_cmd_t`. The
command structure contains the normal `key` and `nkey` fields, but adds
two additional fields

* `strategy`. This field indicates how many replicas will be contacted. The
  options are `LCB_REPLICA_SELECT` to contact a specific replica,
  `LCB_REPLICA_FIRST` to contact each replica once until a successful reply
  is recieved, and `LCB_REPLICA_ALL` to fetch a response from each replica
  and return all the responses from the user
* `index`. Only valid with `LCB_REPLICA_SELECT` this selects the replica to
  contact.
  
In `LCB_REPLICA_ALL` you will receive **multiple** `get` callbacks for the
same key. You will not receive a final _null_ callback for the item and must
thus use a counter to determine how many times the callback was invoked. This
can be found by using the value returned from `lcb_get_num_replicas()` which
will return the number of replicas the bucket is configured with.

### Thread Safety

While the library contains no global functions or variables, the library itself
is not thread safe. In other words if using the library from multiple threads,
a lock must be held around _each_ library API call (including `lcb_wait()`).

Note that with some creativity it is possible to get the library to function
efficiently in a multi threaded environment using only a single client
instance, however it would involve tight integration and wrapping around
several APIs and changing the operation semantics a bit.

See <https://github.com/couchbaselabs/lcbmt> for an experiment with such an
endeavor.

## I/O Integration

libcouchbase is an asynchronous I/O client and can integrate with numerous
existing event frameworks. The library gives several levels of control over
its I/O integration. You can

* Use a predefined backend with normal synchronous operation
* Use a predefined backend with asynchronous operation
* Implement an adaptor between your custom I/O system and libcouchbase

### Using a compiled I/O backend

> ###### Availability of Backends
> Note that the availability of a backend will depend on whether the supporting
> library is available and installed on the current platform. Not all backends
> are available for all platforms.

You may use a predefined event loop backend either by forcing it via an
environment variable or by specifically placing it inside the constructor
options.

To do this via an environment variable you can use the 
`LIBCOUCHBASE_EVENT_PLUGIN_NAME`, thus e.g.

```
$ LIBCOUCHBASE_EVENT_PLUGIN_NAME=select ./foo
```

**Note that the environment variable takes precedence over any setting
defined in code**

You can set a backend via the `lcb_create_st` structure as well:

```C
lcb_io_opt_t io;
lcb_error_t err;
struct lcb_create_io_ops_st io_cropts;
lcb_t instance;
struct lcb_create_st cropts;

memset(&io_cropts, 0, sizeof(io_cropts));
io_cropts.version = 0;
io_cropts.type = LCB_IO_OPS_LIBEV;
err = lcb_create_io_ops(&io, &io_cropts);
if (err != LCB_SUCCESS) {
	die("Couldn't create IO structure", err);
}
memset(&cropts, 0, sizeof(cropts));
cropts.v.v0.io = io;
/** Initialize more fields here */
err = lcb_create(&instance, &cropts);

if (err != LCB_SUCCESS) {
	die("Failed to initialize instance!\n");
}
```

Be sure to check the error return value from `lcb_create_io_ops`. Specifically
if a dependency or plugin is not found, an `LCB_DLOPEN_FAILED` or
`LCB_DLSYM_FAILED` will be returned.

You can also set `LIBCOUCHBASE_DLOPEN_DEBUG` in the environment which will
print details on what the library is doing when searching for the plugin.

#### Backend Selection/Search order

Selecting the backend is done with this algorithm

1. If a backend is specified in the `LIBCOUCHBASE_EVENT_PLUGIN_NAME` it is used
   and anything else ignored. If the plugin in the environment cannot be found
   the `LCB_BAD_ENVIRONMENT` error will be returned
2. If a backend is specified explicitly in the `lcb_create_io_ops_st` structure
   then it is used
3. Otherwise, the backend selected will be the platform default. The platform
   default is determined by the availability of dependent libraries at the time
   _libcouchbase was compiled_. For *nix-based systems this goes in order of
   preference as:
     * libevent
     * libev
     * libuv
     * select

### Getting Backend Information

You may query libcouchbase to determine information about I/O backend
selection. Specifically you may get information about:

* The platform default backend
* Any backend specified by the environment variable
* The current backend object being used by the library

The following snippet of code displays information on the backend which
the library shall select:

```
struct lcb_create_io_ops_st io_cropts = { 0 };
struct lcb_cntl_iops_info_st io_info = { 0 };
io_info.v.v0.options = &io_cropts;

lcb_error_t err;

err = lcb_cntl(NULL, LCB_CNTL_GET, LCB_CNTL_IOPS_DEFAULT_TYPES, &io_info);
if (err != LCB_SUCCESS) {
	die("Couldn't get I/O defaults", err);
}
printf("Default IO For platform is %d\n", io_info.v.v0.os_default);
printf("Effective IO (after options and environment) is %d\n",
		io_info.v.v0.effective);

```

The `os_default` shows the platform default which is determined when
libcouchbase was compiled. The `effective` shows the effective backend
after examining any overrides in the environment variable and the creation
structure.

Note that the creation structure is used only for informational purposes
and is _not_ modified (and is declared in the info structure as `const`).

To get the actual IO pointer table, use the `LCB_CNTL_IOPS` on the instance
itself.

### Using libcouchbase In Async Environments

#### Configuring Backends for Asynchronous Operation

While most of the bundled backends in libcouchbase are _capable_ of
asynchronous operation, by default they are configured to be used
synchronously. Some backends may require specific steps to make them suitable
for async usage.

Most backends require you pass them a specific "loop" structure. You may do
this via the `lcb_create_io_ops_st` structure's `cookie` field.

Each built-in backend contains its own header file which describes
the type of argument it accepts for its `cookie` parameter.

Additionally, do not forget to free the IO structure (i.e. the
`lcb_io_opt_t`) - as it will **not** be freed when the library itself is
destroyed - this is because unless the IO structure was created by the library
it may be possible for several `lcb_t` structures to share the same 
`lcb_io_opt_t` object.

##### Configuring the _libevent_ backend for Async Operation

The _libevent_ backend works with both >= 1.4 and 2.x versions. To use
the backend in async mode, you will need to pass it an `event_base` structure.

```C
#include <event2/event.h> /** For libevent 2.x. 1.x has a different header */
struct lcb_create_io_opt_t io_cropts = { 0 };
struct event_base *evbase = event_base_new();
io_cropts.v.v0.type = LCB_IO_OPS_LIBEVENT;
io_cropts.v.v0.cookie = evbase;

```

The plugin will _not_ free the `event_base` structure, so be sure to free it
yourself once it's not needed (and after the plugin itself is destroyed).

##### Configuring the _libev_ backend for Async Operation

The _libev_ backend works with both 3.x and 4.x versions of `libev`. The usage
pattern does not differ much from _libevent_.

The default backend behavior is _not_ to use the default `ev_loop` object
but to allocate a new one (in case there is one already in use). To use
your own `ev_loop` structure (or use the default one):

```C
#include <ev.h>
struct lcb_create_io_opt_t io_cropts = { 0 };
ev_loop *loop;

loop = ev_default_loop(0);
io_cropts.v.v0.type = LCB_IO_OPS_LIBEV;
io_cropts.v.v0.cookie = loop;

```

##### Configuring the _libuv_ backend for Async Operation

[libuv](https://github.com/joyent/libuv) is a cross platform event loop
which uses a completion model interface. At the time of writing binary
distributions do not exist for most platforms and typically a users of
libuv will need to compile the library themselves.

As such, the binary distribution of libcouchbase does not contain a
_libuv_ binary plugin. However the entire plugin code itself may be
"included" into your own code, as the plugin is installed along with
the libcouchbase headers and you may use it like so:

```C
#include <libcouchbase/plugins/io/libuv/plugin-libuv.c>
```

Compile this file as `plugin-libuv.o` and ensure the `LCBUV_EMBEDDED_SOURCE`
flag is set on your compiler, for example:

```
$ cc -c -o uvapp.o -DLCBUV_EMBEDDED_SOURCE
$ cc -c -o plugin-libuv.o -DLCBUV_EMBEDDED_SOURCE
$ cc -o uvapp uvapp.o plugin-libuv.o -lcouchbase -luv
```

The `LCBUV_EMBEDDED_SOURCE` primarily determines whether symbols are to be
imported (for Windows this means using the `__declspec(dllimport)`) from an
external shared object, or whether the symbols are present in the current
binary. Setting the `LCBUV_EMBEDDED_SOURCE` macro will remove those
import and export decorators.

To actually use the plugin you should initialize a
```C
lcbuv_options_t uv_options;
uv_options.version = 0;
uv_options.v.v0.loop = uv_default_loop();
uv_options.v.v0.startsop_noop = 1;
```

The `startstop_noop` flag is used to determine whether the loop is self
contained within the library or not. Because of how _libuv_ handles
resources, streams which are logically closed by libcouchbase may not
have all their resources destroyed until a callback is delivered. If
this flag is not set then `lcb_destroy()` will run the UV event loop
until all pending items have received their close callback.

#### Connecting and Executing Operations in Async Mode

In all the examples above we demonstrated operations via the _schedule-wait_
sequence, where the completion of an operation could be determined once
`lcb_wait()` returns. However `lcb_wait()` is a blocking operation and cannot
be used in asynchronous environments.

Async operation follows the same pattern as sync operations, but requires a
different approach. Most of these modifications do not affect how libcouchbase
behaves but rather how to effectively program with libcouchbase in async.

The first notable difference is the handling of the `cookie` parameter.
While in sychronous mode it was safe to pass a stack-allocated cookie
it is not safe to do so in `async` mode. Specifically with async mode the
data callback will never complete when the scheduler's stack is in scope
and thus once the callback is invoked, trying to dereference the pointer
to stack data will likely result in stack corruption.

Thus it is recommended you use only dynamically allocated memory (i.e.
`malloc`) for your cookie data. As a consequence you must also ensure that
your cookie data correctly determines the number of callbacks it shall receive
before it can be freed. In other words, if you are using your cookie for more
than a single operation it would be wise to keep it reference counted.

The following comprehensive example details several key aspects in handling
operations within async mode. It may be more robust in error handling, but
does demonstrate the proper sequence of events and resource management
when running in async.

The example requires `libevent2`. It contains a function called
`write_keys_to_file` which opens a file, fetches several keys from the cluster
and then writes their values to the file. The function is asynchronous
which means that the function will only _schedule_ the operations and not
actually wait for execution. A callback is passed to this function to
notify the caller when all the keys have been retrieved and written to the
file.

The file also contains specific details on handling the library instance
itself which we'll go into later.

```C
/** vim: set noexpandtab: */
/**
 * Compile as:
 * cc -o cbfiles cbfiles.c -levent -lcouchbase
 */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <sys/time.h>

#include <libcouchbase/couchbase.h>
#include <event2/event.h>

struct app_context_st;
struct app_context_st {
	lcb_t instance;
	lcb_io_opt_t io;
	int connected;
	struct event_base *evbase;
	void (*on_connected)(struct app_context_st *, int);
	void (*on_cleaned)(struct app_context_st *);
	struct event *async_timer;
};

struct my_cookie_st {
	unsigned remaining;
	FILE *fp;
	char *prefix;
	void (*callback)(struct app_context_st *, void *);
	void *arg;
};

static void
configuration_callback(lcb_t instance, lcb_configuration_t config)
{
	struct app_context_st *ctx;
	ctx = (void *)lcb_get_cookie(instance);
	if (ctx->connected) {
		return;
	}
	ctx->connected = 1;
	ctx->on_connected(ctx, 1);
}

static void
error_callback(lcb_t instance, lcb_error_t err, const char *msg)
{
	struct app_context_st *ctx = (void *)lcb_get_cookie(instance);
	if (ctx->connected) {
		return;
	}
	ctx->on_connected(ctx, 0);
}

static void
get_callback(
	lcb_t instance,
	const void *cookie,
	lcb_error_t err,
	const lcb_get_resp_t *resp)
{
	struct my_cookie_st *info = (void *)cookie;
	fprintf(info->fp, "%s:%.*s ",
			info->prefix, (int)resp->v.v0.nkey, resp->v.v0.key);

	if (err != LCB_SUCCESS) {
		fprintf(info->fp, "<NOT_FOUND>");
	} else {
		fprintf(info->fp, "%.*s", (int)resp->v.v0.nbytes, resp->v.v0.bytes);
	}

	fprintf(info->fp, "\n");

	if (! --info->remaining) {
		struct app_context_st *ctx = (void *)lcb_get_cookie(instance);
		info->callback(ctx, info->arg);
		fclose(info->fp);
		free(info->prefix);
		free(info);
	}
}

void
app_context_init(struct app_context_st *ctx,
				void (*callback)(struct app_context_st *, int))
{	
	struct lcb_create_io_ops_st io_cropts = { 0 };
	struct lcb_create_st cropts = { 0 };
	lcb_error_t err;

	ctx->evbase = event_base_new();
	ctx->on_connected = callback;
	ctx->instance = NULL;
	ctx->connected = 0;

	io_cropts.v.v0.type = LCB_IO_OPS_LIBEVENT;
	io_cropts.v.v0.cookie = ctx->evbase;
	err = lcb_create_io_ops(&ctx->io, &io_cropts);
	if (err != LCB_SUCCESS) {
		fprintf(stderr, "couldn't create io structure\n");
		exit(EXIT_FAILURE);
	}

	cropts.v.v0.io = ctx->io;
	err = lcb_create(&ctx->instance, &cropts);
	if (err != LCB_SUCCESS) {
		fprintf(stderr, "couldn't create instance!\n");
		exit(EXIT_FAILURE);
	}

	lcb_set_cookie(ctx->instance, ctx);
	lcb_set_get_callback(ctx->instance, get_callback);
	lcb_set_error_callback(ctx->instance, error_callback);
	lcb_set_configuration_callback(ctx->instance, configuration_callback);

	err = lcb_connect(ctx->instance);
	if (err != LCB_SUCCESS) {
		fprintf(stderr, "Couldn't schedule connection!\n");
		exit(EXIT_FAILURE);
	}
}


static void
inner_clean(int sockdummy, short events, void *arg)
{
	struct app_context_st *ctx = arg;
	evtimer_del(ctx->async_timer);
	event_free(ctx->async_timer);
	lcb_destroy(ctx->instance);
	lcb_destroy_io_ops(ctx->io);
	ctx->on_cleaned(ctx);
}

void
app_context_clean(
	struct app_context_st *ctx,
	void (*callback)(struct app_context_st *))
{
	struct timeval tv =  { 0, 0 };
	ctx->async_timer = evtimer_new(ctx->evbase, inner_clean, ctx);
	ctx->on_cleaned = callback;
	evtimer_add(ctx->async_timer, &tv);
}

void
write_keys_to_file(
	struct app_context_st *ctx,
	const char *path,
	const char *prefix,
	const char * const * keys,
	void (*callback)(struct app_context_st *, void*),
	void *arg)
{
	struct my_cookie_st *info;
	const char * const *curkey;
	lcb_t instance = ctx->instance;

	info = calloc(1, sizeof(*info));
	info->prefix = malloc(strlen(prefix) + 1);
	strcpy(info->prefix, prefix);
	info->fp = fopen(path, "a");
	info->callback = callback;
	info->arg = arg;

	for (curkey = keys; *curkey; curkey++) {
		lcb_error_t err;
		lcb_get_cmd_t cmd = { 0 }, *cmdp = &cmd;

		cmd.v.v0.key = *curkey;
		cmd.v.v0.nkey = strlen(*curkey);
		err = lcb_get(instance, info, 1, (const lcb_get_cmd_t * const *)&cmdp);
		if (err != LCB_SUCCESS) {
			fprintf(info->fp, "%s:%s <ERROR>\n", prefix, *curkey);
		} else{
			++info->remaining;
		}
	}
}

static void
do_exit(struct app_context_st *ctx)
{
	event_base_loopexit(ctx->evbase, NULL);
}

static int keysets_remaining;

static void
done_callback(struct app_context_st *ctx, void *arg)
{
	char *keyset = arg;
	printf("All keys for keyset %s done!\n", keyset);
	if (!--keysets_remaining) {
		app_context_clean(ctx, do_exit);
	}
}

static void
instance_connected(struct app_context_st *ctx, int ok)
{
	const char *keyset1[] = { "foo1", "bar1", "baz1" , NULL };
	const char *keyset2[] = { "foo2", "bar2", "baz2", NULL };
	if (!ok) {
		fprintf(stderr, "Connection Failed!\n");
		exit(EXIT_FAILURE);
	}

	write_keys_to_file(ctx, "/tmp/keys_1", "first", keyset1, done_callback, "k1");
	write_keys_to_file(ctx, "/tmp/keys_2", "second", keyset2, done_callback, "k2");
	keysets_remaining = 2;
}



int
main(void)
{
	struct app_context_st *ctx = calloc(1, sizeof(*ctx));
	app_context_init(ctx, instance_connected);
	event_base_loop(ctx->evbase, 0);
	event_base_free(ctx->evbase);
	free(ctx);
	exit(EXIT_SUCCESS);
}
```

#### Async Connections

Connecting an instance in async mode follows the same normal creation
parameters described above. However instead of calling `lcb_wait` you should
instead return control to the event loop (this is usually implicit in exiting
your function as you function is likely a callback (or a callee thereof) 
itself).

Before you begin scheduling operations you must ensure the instance has already
been connected and configured. Otherwise you will receive 
`LCB_NO_MATCHIN_SERVER` and/or `LCB_CLIENT_ETMPFAIL` error codes as the library
does not know where to send commands to.

Initial configuration notifications are done via the the _configuration
callback_ which is set via `lcb_set_configuration_callback`. The configuration
callback is invoked whenever the client receives a new configuration and is
passed the instance and a status depending on whether the configuration was
the first one we received (`LCB_CONFIGURATION_NEW`), or it was a modified
version (`LCB_CONFIGURATION_CHANGED`) or was identical to the one the library
was already using (`LCB_CONFIGURATION_UNCHANGED`). In any event receiving such
a callback signals to the user that the instance contains a configuration
and may start being used for data operations.

If an error occurs during configuration it will be relayed via the
`error_callback`.

_Note that the `error_callback` and `configuration_callback`
handlers are not specific to the initial connection phase and may be invoked
any time the library receives a new configuration or experiences errors
in the configuration; so be sure to set an internal flag checking whether this
is the first time either of the callbacks were invoked if you are relying on 
this for initial connection notifications_.
