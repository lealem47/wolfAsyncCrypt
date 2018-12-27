# wolfSSL / wolfCrypt Asynchronous Support

This repository contains the async.c and async.h files required for using Asynchronous Cryptography with the wolfSSL library.

* The async.c file goes into `./wolfcrypt/src/`.
* The async.h file goes into `./wolfssl/wolfcrypt/`.

This feature is enabled using:
`./configure --enable-asynccrypt` or `#define WOLFSSL_ASYNC_CRYPT`.

The async crypt simulator is enabled by default if the hardware does not support async crypto or it can be manually enabled using `#define WOLFSSL_ASYNC_CRYPT_TEST`.

## Design

Each crypto algorithm has its own `WC_ASYNC_DEV` structure, which contains a `WOLF_EVENT`, local crypto context and local hardware context.

For SSL/TLS the `WOLF_EVENT` context is the `WOLFSSL*` and the type is `WOLF_EVENT_TYPE_ASYNC_WOLFSSL`. For wolfCrypt operations the `WOLF_EVENT` context is the `WC_ASYNC_DEV*` and the type is `WOLF_EVENT_TYPE_ASYNC_WOLFCRYPT`. 

A generic event system has been created using a `WOLF_EVENT` structure when `HAVE_WOLF_EVENT` is defined. The event structure resides in the `WC_ASYNC_DEV`.

The asynchronous crypto system is modeled after epoll. The implementation uses `wolfSSL_AsyncPoll` or `wolfSSL_CTX_AsyncPoll` to check if any async operations are complete.


## Hardware

Supported hardware:

* Intel QuickAssist with QAT 1.6 or QAT 1.7 driver. See README.md in `wolfcrypt/src/port/intel/README.md`.
* Cavium Nitrox III and V. See README.md in `wolfcrypt/src/port/cavium/README.md`.

## API's

### ```wolfSSL_AsyncPoll```
```
int wolfSSL_AsyncPoll(WOLFSSL* ssl, WOLF_EVENT_FLAG flags);
```

Polls the provided WOLFSSL object's reference to the WOLFSSL_CTX's event queue to see if any operations outstanding for the WOLFSSL object are done. Return the completed event count on success.

### ```wolfSSL_CTX_AsyncPoll```
```
int wolfSSL_CTX_AsyncPoll(WOLFSSL_CTX* ctx, WOLF_EVENT** events, int maxEvents, WOLF_EVENT_FLAG flags, int* eventCount)
```

Polls the provided WOLFSSL_CTX context event queue to see if any pending events are done. If the `events` argument is provided then a pointer to the `WOLF_EVENT` will be returned up to `maxEvents`. If `eventCount` is provided then the number of events populated will be returned. The `flags` allows for `WOLF_POLL_FLAG_CHECK_HW` to indicate if hardware should be polled again or just return more events.

### ```wolfAsync_DevOpen```
```
int wolfAsync_DevOpen(int *devId);
```

Open the async device and returns an `int` device id for it. 

### ```wolfAsync_DevOpenThread```
```
int wolfAsync_DevOpenThread(int *devId, void* threadId);
```
Opens the async device for a specific thread. A crypto instance is assigned and thread affinity set.

### ```wolfAsync_DevClose```
```
void wolfAsync_DevClose(int *devId)
```

Closes the async device.

### ```wolfAsync_DevCtxInit```
```
int wolfAsync_DevCtxInit(WC_ASYNC_DEV* asyncDev, word32 marker, void* heap, int devId);
```

Initialize the device context and open the device hardware using the provided `WC_ASYNC_DEV ` pointer, marker and device id (from wolfAsync_DevOpen).

### ```wolfAsync_DevCtxFree```
```
void wolfAsync_DevCtxFree(WC_ASYNC_DEV* asyncDev);
```

Closes and free's the device context.


### ```wolfAsync_EventInit```
```
int wolfAsync_EventInit(WOLF_EVENT* event, enum WOLF_EVENT_TYPE type, void* context, word32 flags);
```

Initialize an event structure with provided type and context. Sets the pending flag and the status code to `WC_PENDING_E`. Current flag options are `WC_ASYNC_FLAG_NONE` and `WC_ASYNC_FLAG_CALL_AGAIN` (indicates crypto needs called again after WC_PENDING_E).

### ```wolfAsync_EventWait ```
```
int wolfAsync_EventWait(WOLF_EVENT* event);
```

Waits for the provided event to complete.

### ```wolfAsync_EventPoll```
```
int wolfAsync_EventPoll(WOLF_EVENT* event, WOLF_EVENT_FLAG event_flags);
```

Polls the provided event to determine if its done.

### ```wolfAsync_EventPop ```

```
int wolfAsync_EventPop(WOLF_EVENT* event, enum WOLF_EVENT_TYPE event_type);
```

This will check the event to see if the event type matches and the event is complete. If it is then the async return code is returned. If not then `WC_NOT_PENDING_E` is returned.


### ```wolfAsync_EventQueuePush```
```
int wolfAsync_EventQueuePush(WOLF_EVENT_QUEUE* queue, WOLF_EVENT* event);
```

Pushes an event to the provided event queue and assigns the provided event.

### ```wolfAsync_EventQueuePoll```
```
int wolfAsync_EventQueuePoll(WOLF_EVENT_QUEUE* queue, void* context_filter,
    WOLF_EVENT** events, int maxEvents, WOLF_EVENT_FLAG event_flags, int* eventCount);
```

Polls all events in the provided event queue. Optionally filters by context. Will return pointers to the done events.

### ```wc_AsyncHandle```
```
int wc_AsyncHandle(WC_ASYNC_DEV* asyncDev, WOLF_EVENT_QUEUE* queue, word32 flags);
```

This will push the event inside asyncDev into the provided queue.

### ```wc_AsyncWait```    
```    
int wc_AsyncWait(int ret, WC_ASYNC_DEV* asyncDev, word32 flags);
```

This will wait until the provided asyncDev is done (or error).

### ```wolfAsync_HardwareStart```
```
int wolfAsync_HardwareStart(void);
```

If using multiple threads this allows a way to start the hardware before using `wolfAsync_DevOpen` to ensure the memory system is setup. Ensure that `wolfAsync_HardwareStop` is called on exit. Internally there is a start/stop counter, so this can be called multiple times, but stop must also be called the same number of times to shutdown the hardware.

### ```wolfAsync_HardwareStop```
```
void wolfAsync_HardwareStop(void);
```

Stops hardware if internal `--start_count == 0`.

## Examples

### TLS Server Example

```
#ifdef WOLFSSL_ASYNC_CRYPT
    static int devId = INVALID_DEVID;

    ret = wolfAsync_DevOpen(&devId);
    if (ret != 0) {
        err_sys("Async device open failed");
    }
    wolfSSL_CTX_UseAsync(ctx, devId);
#endif /* WOLFSSL_ASYNC_CRYPT */

	err = 0;
	do {
	#ifdef WOLFSSL_ASYNC_CRYPT
	    if (err == WC_PENDING_E) {
	       ret = wolfSSL_AsyncPoll(ssl);
	       if (ret < 0) { break; } else if (ret == 0) { continue; }
	    }
	#endif
	
	    ret = wolfSSL_accept(ssl);
	    if (ret != SSL_SUCCESS) {
	        err = wolfSSL_get_error(ssl, 0);
	    }
	} while (ret != SSL_SUCCESS && err == WC_PENDING_E);
    
#ifdef WOLFSSL_ASYNC_CRYPT
    wolfAsync_DevClose(&devId);
#endif
```

### wolfCrypt RSA Example

```
#ifdef WOLFSSL_ASYNC_CRYPT
    static int devId = INVALID_DEVID;

    ret = wolfAsync_DevOpen(&devId);
    if (ret != 0) {
        err_sys("Async device open failed");
    }
#endif /* WOLFSSL_ASYNC_CRYPT */

	RsaKey key;
	ret = wc_InitRsaKey_ex(&key, HEAP_HINT, devId);
	
	ret = wc_RsaPrivateKeyDecode(tmp, &idx, &key, (word32)bytes);
	
	do {
#if defined(WOLFSSL_ASYNC_CRYPT)
        ret = wc_AsyncWait(ret, &key.asyncDev, WC_ASYNC_FLAG_CALL_AGAIN);
#endif
        if (ret >= 0) {
            ret = wc_RsaPublicEncrypt(in, inLen, out, outSz, &key, &rng);
        }
    } while (ret == WC_PENDING_E);
    if (ret < 0) {
        err_sys("RsaPublicEncrypt operation failed");
    }
    
#ifdef WOLFSSL_ASYNC_CRYPT
    wolfAsync_DevClose(&devId);
#endif
```

## Build Options

1. Async mult-threading can be disabled by defining `WC_NO_ASYNC_THREADING`. 
2. Software benchmarks can be disabled by defining `NO_SW_BENCH`.
3. The `WC_ASYNC_THRESH_NONE` define can be used to disable the cipher thresholds, which are tunable values to determine at what size hardware should be used vs. software.
4. Use `WOLFSSL_DEBUG_MEMORY` and `WOLFSSL_TRACK_MEMORY` to help debug memory issues. QAT also supports `WOLFSSL_DEBUG_MEMORY_PRINT`.


## References

### TLS Client/Server Async Example

We have a full TLS client/server async examples here:

* [https://github.com/wolfSSL/wolfssl-examples/blob/master/tls/server-tls-epoll-perf.c](https://github.com/wolfSSL/wolfssl-examples/blob/master/tls/server-tls-epoll-perf.c)

* [https://github.com/wolfSSL/wolfssl-examples/blob/master/tls/client-tls-perf.c](https://github.com/wolfSSL/wolfssl-examples/blob/master/tls/client-tls-perf.c)

#### Usage

```
git clone git@github.com:wolfSSL/wolfssl-examples.git
cd wolfssl-examples
cd tls
make
sudo ./server-tls-epoll-perf
sudo ./client-tls-perf
```

```
Waiting for a connection...
SSL cipher suite is TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
wolfSSL Client Benchmark 16384 bytes
	Num Conns         :       100
	Total             :   777.080 ms
	Total Avg         :     7.771 ms
	t/s               :   128.687
	Accept            :   590.556 ms
	Accept Avg        :     5.906 ms
	Total Read bytes  :   1638400 bytes
	Total Write bytes :   1638400 bytes
	Read              :    73.360 ms (   21.299 MBps)
	Write             :    74.535 ms (   20.963 MBps)
```

## Change Log

### wolfSSL Async Release v3.15.7 (12/27/2018)

* Fixes for various analysis warnings (https://github.com/wolfSSL/wolfssl/pull/2003).
* Added QAT v1.7 driver support.
* Added QAT SHA-3 support.
* Added QAT RSA Key Generation support.
* Added support for new usdm memory driver.
* Added support for detecting QAT version and features.
* Added `QAT_ENABLE_RNG` option to disable QAT TRNG/DRBG.
* Added alternate hashing method to cache all updates (avoids using partial updates).

### wolfSSL Async Release v3.15.5 (11/09/2018)

* Fixes for various analysis warnings (https://github.com/wolfSSL/wolfssl/pull/1918).
* Fix for QAT possible double free case where `ctx->symCtx` is not trapped.
* Improved QAT debug messages when using `QAT_DEBUG`.
* Fix for QAT RNG to allow zero length. This resolves PSS case where `wc_RNG_GenerateBlock` is called for saltLen == 0.


### wolfSSL Async Release v3.15.3 (06/20/2018)

* Fixes for fsantize tests with Cavium Nitrox V.
* Removed typedef for `CspHandle`, since its already defined.
* Fixes for a couple of fsanitize warnings.
* Fix for possible leak with large request to `IntelQaDrbg`.

### wolfSSL Async Release v3.14.4 (04/13/2018)

* Added Nitrox V ECC.
* Added Nitrox V SHA-224 and SHA-3
* Added Nitrox V AES GCM
* Added Nitrox III SHA2 384/512 support for HMAC.
* Added error code handling for signature check failure.
* Added error translate for `ERR_PKCS_DECRYPT_INCORRECT`
* Added useful `WOLFSSL_NITROX_DEBUG` and show count for pending checks.
* Cleanup of Nitrox symmetric processing to use single while loops.
* Cleanup to only include some headers in cavium_nitrox.c port.
* Fixes for building against Nitrox III and V SDK.
* Updates to README.md with required CFLAGS/LDFLAGS when building without ./configure.
* Fix for Intel QuickAssist HMAC to use software for unsupported hash algorithms.


### wolfSSL Async Release v3.12.2 (10/22/2017)

* Fix for HMAC QAT when block size aligned. The QAT HMAC final without any buffers will fail incorrectly (bug in QAT 1.6).
* Nitrox fix for rename of `ContextType` to `context_type_t`. Updates to Nitrox README.md.
* Workaround for `USE_QAE_THREAD_LS` issue with realloc from a different thread.
* Fix for hashing to allow zero length. This resolves issue with new empty hash tests.
* Fix bug with blocking async where operation was being free'd before completion. Set freeFunc prior to performing operation and check ret code in poll.
* Fix leak with cipher symmetric context close.
* Fix QAT_DEBUG partialState offset.
* Fixes for symmetric context caching.
* Refactored async event initialization so its done prior to making possible async calls.
* Fix to resolve issue with QAT callbacks and multi-threading. 
* The cleanup is now handled in polling function and the event is only marked done from the polling thread that matches the originating thread.
* Fix possible mem leak with multiple threads `g_qatEcdhY` and `g_qatEcdhCofactor1`.
* Fix the block polling to use `ret` instead of `status`.
* Change order of `IntelQaDevClear` and setting `event->ret`.
* Fixes to better handle threading with async.
* Refactor of async event state.
* Refactor to initialize event prior to operation (in case it finishes before adding to queue).
* Fixes issues with AES GCM decrypt that can corrupt up to authTag bytes at end of output buffer provided.
* Optimize the Hmac struct to replace keyRaw with ipad.
* Enhancement to allow re-use of the symmetric context for ciphers.
* Fixes for QuickAssist (QAT) multi-threading. Fix to not set return code until after callback cleanup.
* Disable thread binding to specific CPU by default (enabled now with `WC_ASYNC_THREAD_BIND`).
* Added optional define `QAT_USE_POLLING_CHECK ` to have only one thread polling at a time (not required and doesn't improve performance).
* Reduced default QAT_MAX_PENDING for benchmark to 15 (120/num_threads).
* Fix for IntelQaDrbg to handle buffer over 0xFFFF in length.
* Added working DRBG and TRNG implementations for QAT.
* Fix to set callback status after ret and output have been set. Cleanup of the symmetric context.
* Updates to support refactored dynamic types.
* Fix for QAT symmetric to allow NULL authTag.
* Fix GCC 7 build warning with braces.
* Cleanup formatting.

### wolfSSL Async Release v3.11.0 (05/05/2017)

* Fixes for Cavium Nitrox III/V.
	- Fix with possible crash when using a request Id that is already complete, due to partial submissions not marking event done.
	- Improvements to max buffer lengths.
	- Fixes to handle various return code patterns with CNN55XX-SDK.
	- All Nitrox V tests and benchmarks pass. Bench: RSA 2048-bit public 336,674 ops/sec and private (CRT) 66,524 ops/sec.

* Intel QuickAssist support and various async fixes/improvements:
    - Added support for Intel QuickAssist v1.6 driver with QuickAssist 8950 hardware
	- Added QAE memory option to use static memory list instead of dynamic list using `USE_QAE_STATIC_MEM`.
	- Added tracking of deallocs and made the values signed long.
	- Improved code for wolf header check and expanded to 16-byte alignment for performance improvement with TLS.
	- Added ability to override limit dev access parameters and all configurable QAT fields.
	- Added async simulator tests for DH, DES3 CBC and AES CBC/GCM.
	- Rename AsyncCryptDev to WC_ASYNC_DEV.
	- Refactor to move WOLF_EVENT into WC_ASYNC_DEV.
	- Refactor the async struct/enum names to use WC_ naming.
	- Refactor of the async event->context to use WOLF_EVENT_TYPE_ASYNC_WOLFSSL or WOLF_EVENT_TYPE_ASYNC_WOLFCRYPT to indicate the type of context pointer.
	- Added flag to WOLF_EVENT which is used to determine if the async complete should call into operation again or goto next `WC_ASYNC_FLAG_CALL_AGAIN`.
	- Cleanup of the "wolfAsync_DevCtxInit" calls to make sure asyncDev is always cleared if invalid device id is used.
	- Eliminated WOLFSSL_ASYNC_CRYPT_STATE.
	- Removed async event type WOLF_EVENT_TYPE_ASYNC_ANY.
	- Enable the random extra delay option by default for simulator as it helps catch bugs.
	- Cleanup for async free to also check marker.
	- Refactor of the async wait and handle to reduce duplicate code.
	- Added async simulator test for RSA make key.
	- Added WC_ASYNC_THRESH_NONE to allow bypass of threshold for testing
	- Added static numbers for the async sim test types, for easier debugging of the “testDev->type” value.
	- Populate heap hint into asyncDev struct.
	- Enhancement to cache the asyncDev to improve poll performance.
	- Added async threading helpers and new wolfAsync_DevOpenThread.
	- Added WC_NO_ASYNC_THREADING to prevent async threading.
	- Added new API “wc_AsyncGetNumberOfCpus” for getting number of CPU’s.
	- Added new “wc_AsyncThreadYield” API.
	- Added WOLF_ASYNC_MAX_THREADS.
	- Added new API for wolfAsync_DevCopy.
	- Fix to make sure an async init failure sets the deviceId to INVALID_DEVID.
	- Fix for building with async threading support on Mac.
	- Fix for using simulator so it supports multiple threads.

* Moved Intel QuickAssist and Cavium Nitrox III/V code into async repo.
* Added new WC_ASYNC_NO_* options to allow disabling of individual async algorithms.
	- New defines are: WC_ASYNC_NO_CRYPT, WC_ASYNC_NO_PKI and WC_ASYNC_NO_HASH.
	- Additionally each algorithm has a WC_ASYNC_NO_[ALGO] define.


### wolfSSL Async Release v3.9.8 (07/25/2016)

* Asynchronous wolfCrypt and Cavium Nitrox V support. 

### wolfSSL Async Release v3.9.0 (03/04/2016)

* Initial version with async simulator and README.md.


## Support

For questions email wolfSSL support at support@wolfssl.com