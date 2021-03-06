# Nydusd

`nydusd` running as daemon to expose a FUSE mountpoint or a Virtio-FS mountpoint inside guest for containers to access

## Run Nydusd Daemon

``` shell
# Prepare nydusd configuration
cat /path/to/config-localfs.json
{
  "device": {
    "backend": {
      "type": "localfs",
      "config": {
        "dir": "/path/to/blobs",
      }
    }
  }
}
```

### Run With FUSE

``` shell
sudo target-fusedev/debug/nydusd \
  --config /path/to/config-localfs.json \
  --mountpoint /path/to/mnt \
  --bootstrap /path/to/bootstrap \
  --log-level info
```

### Run With Virtio-FS

``` shell
sudo target-virtiofsd/debug/nydusd \
  --config /path/to/config-localfs.json \
  --sock /path/to/vhost-user-fs.sock \
  --bootstrap /path/to/bootstrap \
  --log-level info
```

To start a qemu process, run something like:

``` shell
./qemu-system-x86_64 -M pc -cpu host --enable-kvm -smp 2 \
        -m 2G,maxmem=16G -object memory-backend-file,id=mem,size=2G,mem-path=/dev/shm,share=on -numa node,memdev=mem \
        -chardev socket,id=char0,path=/path/to/vhost-user-fs.sock \
        -device vhost-user-fs-pci,chardev=char0,tag=nydus,queue-size=1024,indirect_desc=false,event_idx=false \
        -serial mon:stdio -vga none -nographic -curses -kernel ./kernel \
        -append 'console=ttyS0 root=/dev/vda1 virtio_fs.dyndbg="+pfl" fuse.dyndbg="+pfl"' \
        -device virtio-net-pci,netdev=net0,mac=AE:AD:BE:EF:6C:FB -netdev type=user,id=net0 \
        -qmp unix:/path/to/qmp.sock,server,nowait \
        -drive if=virtio,file=./bionic-server-cloudimg-amd64.img
```

Then we can mount nydus virtio-fs inside the guest with:

``` shell
mount -t virtio_fs none /mnt -o tag=nydus,default_permissions,allow_other,rootmode=040000,user_id=0,group_id=0,nodev
```

Or simply below if you are running newer guest kernel:

``` shell
mount -t virtiofs nydus /mnt
```

### Nydus Configuration

#### Common Fields In Config

``` JSON
{
  "device": {
    "backend": {
      // localfs | oss | registry
      "type": "...",
      "config": {
        ...
        // Access remote storage backend via P2P proxy, for example Dragonfly client address
        "proxy": "http://p2p-proxy:65001",
        // Fallback to remote storage backend if P2P proxy ping failed
        "proxy_fallback": true,
        // Endpoint of P2P proxy health check
        "proxy_ping_url": "http://p2p-proxy:40901/server/ping",
        // Interval of P2P proxy checking, in seconds
        "proxy_check_interval": 5,
        // Drop the read request once http request timeout, in seconds
        "timeout": 5,
        // Drop the read request once http connection timeout, in seconds
        "connect_timeout": 5,
        // Retry count when read request failed
        "retry_limit": 0
      }
    },
    "cache": {
      // Blobcache: enable local fs cache
      // Dummycache: disable cache, access remote storage backend directly
      "type": "blobcache",
      // Enable cache compression
      "compressed": true,
      "config": {
        // Directory of cache files, only for blobcache
        "work_dir": "/cache"
      }
    },
    // direct | cached
    "mode": "direct",
    // Validate inode tree digest and chunk digest on demand
    "digest_validate": false,
    // Enable file IO metric
    "iostats_files": true,
    // Enable support of fs extended attributes
    "enable_xattr": false,
    "fs_prefetch": {
      // Enable blob prefetch
      "enable": false,
      // Prefetch thread count
      "threads_count": 10,
      // Maximal read size per prefetch request, for example 128kb
      "merging_size": 131072
    }
  },
  ...
}
```

#### Use Different Storage Backends

##### Localfs Backend

``` JSON
{
  "device": {
    "backend": {
      "type": "localfs",
      "config": {
        // The directory included all blob files declared in bootstrap
        "dir": "/path/to/blobs/",
        // Record read access log, prefetch data on next time
        "readahead": true,
        // Duration of recording access log
        "readahead_sec": 10
      }
    },
    ...
  },
  ...
}
```

##### OSS backend with blobcache

``` JSON
{
  "device": {
    "backend": {
      "type": "oss",
      "config": {
        "endpoint": "region.aliyuncs.com",
        "access_key_id": "",
        "access_key_secret": "",
        "bucket_name": ""
      }
    },
    ...
  },
  ...
}
```

##### Registry backend

``` JSON
{
  "device": {
    "backend": {
      "type": "registry",
      "config": {
        "scheme": "https",
        "host": "my-registry:5000",
        "repo": "test/repo",
        // Base64(username:password)
        "auth": "<base64_encoded_auth>",
        "blob_url_scheme": "http"
      }
    },
    ...
  },
  ...
}
```

### Mount Bootstrap Via API

To mount a bootstrap via api, first launch nydusd without a bootstrap:

``` shell
sudo target-fusedev/debug/nydusd \
  --apisock /path/to/api.sock \
  --config /path/to/config.json \
  --mountpoint /path/to/mountpoint
```

Then use curl to call the mount api:

``` shell
curl --unix-socket api.sock \
     -X PUT "http://localhost/api/v1/mount" -H "accept: */*" \
     -H "Content-Type: application/json" \
     -d "{\"source\":\"<path-to-bootstrap>\",\"fstype\":\"rafs\",\"mountpoint\":\"/path/to/mnt\","config\":\"/path/to/config.json\"}"
```

### Multiple Pseudo Mounts

One single nydusd can have multiple pseudo mounts within a mountpoint.

To achieve that, you can trigger backend fs (e.g., rafs) mount through the HTTP interfaces using curl command.

When starting nydusd without the --bootstrap option, there will be no backend file system in a nydus mountpoint. You can use curl command to mount multiple backend fs at different sub-directories.

#### Example

Given that your mountpoint is `/mnt` which can be a directory in local host or inside guest.

When you have two pseudo mounts which are named "pseudo_1" and "pseudo_2" identified in http request body.

pseudo_1 and pseudo_2 correspond to bootstrap respectively.

``` shell
tree -L 1 mnt
mnt
├── pseudo_1
└── pseudo_2
```
