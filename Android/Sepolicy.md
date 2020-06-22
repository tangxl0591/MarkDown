

# Sepolicy测试说明

### 1.HIDL Service 

**file.te**

```bash
type nls_hal_scancamera, domain;
type nls_hal_scancamera_exec, exec_type, file_type, vendor_file_type;
```

**hal_scancamera.te**

```bash
# HwBinder IPC from clients into server, and callbacks
binder_call(hal_scancamera_client, hal_scancamera_server)
binder_call(hal_scancamera_server, hal_scancamera_client)

# give permission for hal client
allow hal_scancamera_client nls_hal_scancamera_hwservice :hwservice_manager find;
```

**attributes**

```bash
attribute hal_scancamera;
attribute hal_scancamera_client;
attribute hal_scancamera_server;
```

**hwservice.te**

```bash
type nls_hal_scancamera_hwservice, hwservice_manager_type;
```

**file_contexts**

```bash
/(system\/vendor|vendor)/bin/hw/vendor\.mediatek\.hardware\.scancamera@1\.0-service u:object_r:nls_hal_scancamera_exec:s0
```

**hwservice_contexts**

```bash
vendor.mediatek.hardware.scancamera::IScanCamera	u:object_r:nls_hal_scancamera_hwservice:s0
```

**system_server.te**

```bas
allow system_server nls_hal_scancamera_hwservice:hwservice_manager find;
allow system_server nls_hal_scancamera:binder { call transfer };
```

**nls_hal_scancamera.te**

```bash
# Setup for domain transition
init_daemon_domain(nls_hal_scancamera)

# Allow to use HWBinder IPC
hwbinder_use(nls_hal_scancamera);

# Allow a set of permissions required for a domain to be a server which provides a HAL implementation over HWBinder.
hal_server_domain(nls_hal_scancamera, hal_scancamera)

# add/find permission rule to hwservicemanager
add_hwservice(hal_scancamera_server, nls_hal_scancamera_hwservice)

allow nls_hal_scancamera system_server:binder { call transfer };
```

### 2.VNDK Native Service 

**nlsserver.te**

```bash
type nlsserver, domain;
type nlsserver_exec, exec_type, file_type, vendor_file_type;
type nlsserver_socket, file_type;

init_daemon_domain(nlsserver)
vndbinder_use(nlsserver)
hwbinder_use(nlsserver)

allow nlsserver nlsserver_vndservice:service_manager add;
```

**vndservice_contexts**

```bash
NewlandService          u:object_r:nlsserver_vndservice:s0
```

**vndservice.te**

```bash
type nlsserver_vndservice,             vndservice_manager_type;
```

**file_contexts**

```bash
/vendor/bin/nlsserver 		u:object_r:nlsserver_exec:s0
```

**system_server.te**

```bash
binder_call(system_server, nlsserver)
```

