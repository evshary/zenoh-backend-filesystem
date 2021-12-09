<img src="http://zenoh.io/img/zenoh-dragon-small.png" width="150">

[![CI](https://github.com/eclipse-zenoh/zenoh-backend-filesystem/workflows/CI/badge.svg)](https://github.com/eclipse-zenoh/zenoh-backend-filesystem/actions?query=workflow%3A%22CI%22)
[![Gitter](https://badges.gitter.im/atolab/zenoh.svg)](https://gitter.im/atolab/zenoh?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)
[![License](https://img.shields.io/badge/License-EPL%202.0-blue)](https://choosealicense.com/licenses/epl-2.0/)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

# File system backend for Eclipse zenoh

In zenoh a backend is a storage technology (such as DBMS, time-series database, file system...) alowing to store the
keys/values publications made via zenoh and return them on queries.
See the [zenoh documentation](http://zenoh.io/docs/manual/backends/) for more details.

This backend relies on the host's file system to implement the storages.
Its library name (without OS specific prefix and extension) that zenoh will rely on to find it and load it is **`zbackend_fs`**.


:point_right: **Download stable versions:** https://download.eclipse.org/zenoh/zenoh-backend-filesystem/

:point_right: **Build "master" branch:** see [below](#How-to-build-it)

-------------------------------
## :warning: Documentation for previous 0.5 versions:
The following documentation related to the version currently in development in "master" branch: 0.6.x.

For previous versions see the README and code of the corresponding tagged version:
 - [0.5.0-beta.9](https://github.com/eclipse-zenoh/zenoh-backend-filesystem/tree/0.5.0-beta.9#readme)
 - [0.5.0-beta.8](https://github.com/eclipse-zenoh/zenoh-backend-filesystem/tree/0.5.0-beta.8#readme)

-------------------------------
## **Examples of usage**

Prerequisites:
 - You have a zenoh router (`zenohd`) installed, and the `zbackend_fs` library file is available in `~/.zenoh/lib`.
 - Declare the `ZBACKEND_FS_ROOT` environment variable to the directory where you want the files to be stored (or exposed from).
   If you don't declare it, the `~/.zenoh/zbackend_fs` directory will be used.

You can setup storages either at zenoh router startup via a configuration file, either at runtime via the zenoh admin space, using for instance the REST API.
### **Setup via a JSON configuration file**

  - Create a `zenoh.json` configuration file containing:
    ```json5
    {
      plugins: {
        // configuration of "storages" plugin:
        storages: {
          backends: {
            // configuration of a "fs" backend (the "zbackend_fs" library will be loaded at startup)
            fs: {
              storages: {
                // configuration of a "demo" storage using the "fs" backend
                demo: {
                  // the key expression this storage will subscribes to
                  key_expr: "/demo/example/**",
                  // this prefix will be stripped from the received key when converting to file path
                  strip_prefix: "/demo/example",
                  // the key/values will be stored as files within this directory (relative to ${ZBACKEND_FS_ROOT})
                  dir: "example"
    } } } } } } }
    ```
  - Run the zenoh router with:  
    `zenohd -c zenoh.json`

### **Setup at runtime via `curl` commands on the admin space**

  - Run the zenoh router without any specific configuration, but loading the storages plugin:  
    `zenohd -P storages`
  - Add the "fs" backend (the "zbackend_fs" library will be loaded):  
    `curl -X PUT -H 'content-type:application/json' -d '{}' http://localhost:8000/@/router/local/config/plugins/storages/backends/fs`
  - Add the "demo" storage using the "fs" backend:  
    `curl -X PUT -H 'content-type:application/json' -d '{"key_expr":"/demo/example/**","strip_prefix":"/demo/example","dir":"example"}' http://localhost:8000/@/router/local/config/plugins/storages/backends/fs/storages/demo`

### **Tests using the REST API**

Using `curl` to publish and query keys/values, you can:
```bash
# Put values that will be stored under ${ZBACKEND_FS_ROOT}/example
curl -X PUT -d "TEST-1" http://localhost:8000/demo/example/test-1
curl -X PUT -d "B" http://localhost:8000/demo/example/a/b

# Retrive the values
curl http://localhost:8000/demo/example/**
```

<!-- TODO: after release of eclipse/zenoh:0.6.0 update wrt. conf file and uncomment this:

### **Usage with `eclipse/zenoh` Docker image**
Alternatively, you can test the zenoh router in a Docker container:
 - Download the [docker-compose.yml](https://github.com/eclipse-zenoh/zenoh-backend-filesystem/blob/master/docker-compose.yml) file
 - In the same directory, create the `./zenoh_docker/lib` sub-directories and place the `libzbackend_fs.so` library
   for `x86_64-unknown-linux-musl` target within.
 - Also create a `./zenoh_filesystem/test` directory that will be used for the storage.
 - Start the containers running
   ```bash
   docker-compose up -d
   ```
 - Run the `curl` commands above, and explore the resulting file in `./zenoh_filesystem/test`
-->

-------------------------------
## **Properties for Backend creation**

- **`"lib"`** (optional) : the path to the backend library file. If not speficied, the Backend identifier in admin space must be `fs` (i.e. zenoh will automatically search for a library named `zbackend_fs`).

-------------------------------
## **Properties for Storage creation**

- **`"key_expr"`** (**required**) : the Storage's [Key Expression](../abstractions#key-expression)

- **`"strip_prefix"`** (**required**) : a prefix of the `"key_expr"` that will be stripped from each key to store.  
  _Example: with `"key_expr"="/demo/example/**"` and `"strip_prefix"="/demo/example/"` the value with key `"/demo/example/foo/bar"` will be stored as file: `"foo/bar"`. But replying to a get on `"/demo/**"`, the file path `"foo/bar"` will be transformed back to the original key (`"/demo/example/foo/bar"`)._

- **`"dir"`** (**required**) : The directory that will be used to store

- **`"read_only"`** (optional) : the storage will only answer to GET queries. It will not accept any PUT or DELETE message, and won't write any file. Not set by default. *(the value doesn't matter, only the property existence is checked)*

- **`"on_closure"`** (optional) : the strategy to use when the Storage is removed. There are 2 options:
  - *unset*: the storage's directory remains untouched (this is the default behaviour)
  - `"delete_all"`: the storage's directory is deleted with all its content.

- **`"follow_links"`** (optional) : If set to `true` the storage will follow the symbolic links. The default value is `false`.

- **`"keep_mime_types"`** (optional) : When replying to a GET query with a file for which the zenoh encoding is not known, the storage guess its mime-type according to the file extension. If the mime-type doesn't correspond to a supported zenoh encoding, this option will drive the returned value:
   - if `true` (default value): a [Custom value](https://docs.rs/zenoh/latest/zenoh/enum.Value.html#variant.Custom)
     is returned with the description set to the mime-type.
   - if `false`: a [Raw value](https://docs.rs/zenoh/latest/zenoh/enum.Value.html#variant.Raw) with
     APP_OCTET_STREAM encoding is returned.

-------------------------------
## **Behaviour of the backend**

### Mapping to file system
Each **storage** will map to a directory with path: `${ZBACKEND_FS_ROOT}/<dir>`, where:
  * `${ZBACKEND_FS_ROOT}` is an environment variable that could be specified before zenoh router startup.
     If this variable is not specified `${ZENOH_HOME}/zbackend_fs` will be used
     (where the default value of `${ZENOH_HOME}` is `~/.zenoh`).
  * `<dir>` is the `"dir"` property specified at storage creation.
Each zenoh **key/value** put into the storage will map to a file within the storage's directory where:
  * the file path will be `${ZBACKEND_FS_ROOT}/<dir>/<relative_zenoh_key>`, where `<relative_zenoh_key>`
    will be the zenoh key, stripped from the `"strip_prefix"` property specified at storage creation.
  * the content of the file will be the value written as a RawValue. I.e. the same bytes buffer that has been
    transported by zenoh. For UTF-8 compatible formats (StringUTF8, JSon, Integer, Float...) it means the file
    will be readable as a text format.
  * the encoding and the timestamp of the key/value will be stored in a RocksDB database stored in the storage directory.

### Behaviour on deletion
On deletion of a key, the corresponding file is removed. An entry with deletion timestamp is inserted in the
RocksDB database (to avoid re-insertion of points with an older timestamp in case of un-ordered messages).  
At regular interval, a task cleans-up the RocksDB database from entries with old timestamps that don't have a
corresponding existing file.

### Behaviour on GET
On GET operations, the storage searches for matching and existing files, and return their raw content as a reply.
For each, the encoding and timestamp are retrieved from the RocksDB database. But if no entry is found in the
database for a file (e.g. for files created without zenoh), the encoding is deduced from the file's extension
(using [mime_guess](https://crates.io/crates/mime_guess)), and the timestamp is deduced from the file's
modification time.


-------------------------------
## How to build it

At first, install [Cargo and Rust](https://doc.rust-lang.org/cargo/getting-started/installation.html). 

:warning: **WARNING** :warning: : As Rust doesn't have a stable ABI, the backend library should be
built with the exact same Rust version than `zenohd`. Otherwise, incompatibilities in memory mapping
of shared types between `zenohd` and the library can lead to a `"SIGSEV"` crash.

To know the Rust version you're `zenohd` has been built with, use the `--version` option.  
Example:
```bash
$ zenohd --version
The zenoh router v0.6.0-dev-24-g1f20c86 built with rustc 1.57.0 (f1edd0429 2021-11-29)
```
Here, `zenohd` has been built with the rustc version `1.57.0`.  
Install and use this toolchain with the following command:

```bash
$ rustup default 1.57.0
```

And then build the backend with:

```bash
$ cargo build --release --all-targets
```
