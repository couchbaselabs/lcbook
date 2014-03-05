<!-- vim: set noexpandtab: --!>
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

_Note that the `error\_callback` and `configuration\_callback`
handlers are not specific to the initial connection phase and may be invoked
any time the library receives a new configuration or experiences errors
in the configuration; so be sure to set an internal flag checking whether this
is the first time either of the callbacks were invoked if you are relying on 
this for initial connection notifications_.

### Writing your own I/O Backend

You may also author your own IO backend or I/O plugin. You can choose between
the simpler event-based interface or the more complex buffer-based interface.

Which interface you employ depends on the environments you are working with.
Typically most libraries will employ the event-based interface which follows
the normal `select()` or `poll()` model where a socket is polled for read
or write ability and is then dispatched to a handler which performs the actual
I/O operations.

Some event systems (notably _UV_, Windows' IOCP and others) use a buffer-based
completion model wherein sockets are submitted operations to perform on a
buffer (particularly read in the buffer and write from it).

This section assumes you have some understanding of asynchronous I/O.

The IOPS interface is defined in `<libcouchbase/types.h>`

#### Event Model

In the event model you are presented with opaque 'event' structures which
contain information about:

1. What events to watch for
2. What to do when said events take place

A sample event structure may look like this:

```C
struct my_event {
	int sockfd; /** Socket to poll on */
	short events; /** Mask of events to watch */
	/** callback to invoke when ready */
	void (*callback)(int sockfd, short events, void *arg);
	/** Argument to pass to callback */
	void *cbarg;}
```

In concept this is similar to the `pollfd` structure (see `poll(2)`).

The lifecycle for events takes place as follows:

1. An event structure's memory is first allocated. This is done by a call
   to `create_event()`.

2. The structure is paired with a socket, event, callback and data. When
   the event is ready, the callback is invoked with those parameters. This
   is done via the `update_event()` call. Note the `void *` pointer in the
   callback signature. This is how the library is able to associate a given
   socket/event with an internal structure of its own.

3. The polling for events is stopped for the specified socket via the
   `delete_event()` call. This undoes anything performed by the `update_event`
   call

4. When the event structure is no longer needed, it is destroyed (and its
   memory resources freed) via `destroy_event`.


Note that the most common calls will be to `update_event` and `delete_event`
and thus plugins should optimize for these.

The events may be a mask of `LCB_READ_EVENT` and `LCB_WRITE_EVENT`; these
have the same semantics as `POLLIN` and `POLLOUT` respectively. If an error
takes place on a socket, the `LCB_ERROR_EVENT` flag may be _output_ as well.

When an event is ready, the callback is to be invoked specifying the mask
of the events which _actually took place_, for example if `update_event()`
was called with `LCB_READ_EVENT|LCB_WRITE_EVENT` but the socket was only
available for writing, then the callback is only invoked with
`LCB_WRITE_EVENT`.

Note that the library will call `update_event()` whenever it wishes to
receive an event. The registration should be considered cleared (i.e
as if `delete_event` is called) right before the callback is to be
invoked, thus e.g.

```C
void send_events(my_event *event, short events)
{
	delete_event(my_event);
	my_event->allback(my_event->sockfd, events, my_event->arg);
}
```

##### I/O Operations

The IOPS structure for the 'event' model contains function pointers for
basic I/O operations like `send`, `recv`, etc.

The semantics of these operations are identical to those functions of the
corresponding name within the Berkeley Socket API. Some of the naming might
be a bit different, for example `sendv` does not exist in the BSD socket APIs
and is typically implemented as `sendmsg` (Portable) or `writev` (POSIX only).

These operations are also passed the Plugin Instance itself as their first
argument. If your plugin just proxies to OS calls then you may ignore it in
those functions.

##### Timer Operations

Timer operations determine that a callback will be invoked after a specific
amount of time has elapsed. The interval itself is measured in microseconds.

The signature of the timer callback is the same as the event callback, except
that the socket and event arguments are ignored. Timers are considered
non-repeating and thus once a timer callback has been delivered it should not
be delivered again until `update_timer` has been called.
