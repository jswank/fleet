# Load test of osquery queries in macOS and Windows

Following are the steps to load test osquery on macOS and Windows.
The purpose is to know the impact of Fleet provided queries on real devices.

> The scripts that process osquery logs were written and tested on macOS.

> At the time of writing, the changes that add watchdog logging needed for this script are
> merged but not released yet (https://github.com/osquery/osquery/pull/8070).
> You will have to download and extract the osqueryd executable from the PR: https://github.com/osquery/osquery/suites/14033523376/artifacts/783724086

## Requirements

- Install gnuplot and ripgrep, e.g. on macOS:
```sh
brew install gnuplot ripgrep
```

## Architecture

We are going to use the [fleetd tables](../../../orbit/cmd/fleetd_tables/README.md) as an extension so that it is also monitored by the watchdog.

```mermaid
graph LR;
    subgraph Device
        osquery_watchdog[osqueryd\nwatchdog process];
        osqueryd_worker[osqueryd\nworker process];
        fleetd_tables[fleetd_tables\nextension process];

        osquery_watchdog -- monitors\nCPU and memory --> osqueryd_worker;
        osquery_watchdog -- monitors\nCPU and memory --> fleetd_tables;
    end
```

## macOS

### Build fleetd_tables extension

```sh
make fleetd-tables-darwin-universal
sudo cp fleetd_tables_darwin_universal.ext /usr/local/osquery_extensions/fleetd_tables.ext
echo "/usr/local/osquery_extensions/fleetd_tables.ext" > /tmp/extensions.load
```

### Run osquery

> The following assumes a Fleet server instance running and listening at `localhost:8080`.

```sh
mkdir -p /Users/luk/osqueryd/osquery_log
```

```sh
sudo ENROLL_SECRET=<...> ./osquery/osqueryd \
    --verbose=true \
    --tls_dump=true \
    --pidfile=/Users/luk/osqueryd/osquery.pid \
    --database_path=/Users/luk/osqueryd/osquery.db \
    --logger_path=/Users/luk/osqueryd/osquery_log \
    --host_identifier=instance \
    # /Users/luk/fleetdm/git/fleet is the location of the Fleet mono repository.
    --tls_server_certs=/Users/luk/fleetdm/git/fleet/tools/osquery/fleet.crt \
    --enroll_secret_env=ENROLL_SECRET \
    --tls_hostname=localhost:8080 \
    --enroll_tls_endpoint=/api/v1/osquery/enroll \
    --config_plugin=tls \
    --config_tls_endpoint=/api/v1/osquery/config \
    --config_refresh=60 \
    --disable_distributed=false \
    --distributed_plugin=tls \
    --distributed_tls_max_attempts=10 \
    --distributed_tls_read_endpoint=/api/v1/osquery/distributed/read \
    --distributed_tls_write_endpoint=/api/v1/osquery/distributed/write \
    --logger_plugin=tls,filesystem \
    --logger_tls_endpoint=/api/v1/osquery/log \
    --disable_carver=false \
    --carver_disable_function=false \
    --carver_start_endpoint=/api/v1/osquery/carve/begin \
    --carver_continue_endpoint=/api/v1/osquery/carve/block \
    --carver_block_size=2000000 \
    --extensions_autoload=/tmp/extensions.load \
    --allow_unsafe \
    --enable_watchdog_debug \
    --distributed_denylist_duration 0 \
    --enable_extensions_watchdog 2>&1 | tee /tmp/osqueryd.log
```

## Windows

### Build fleetd_tables extension

In a macOS device run:
```sh
make fleetd-tables-windows
```
Choose path to store the extension in the Windows device, on this guide we will use `C:\Program Files\Fleetd\`.

- Place the generated `fleetd_tables_windows.exe` on the chosen location, on this guide it would be `C:\Program Files\Fleetd\fleetd_tables_windows.exe`.
- Create a text file `C:\Program Files\Fleetd\extensions.load` with the following line in it: `C:\Program Files\Fleetd\fleetd_tables_windows.exe` (the location of the extension).

### Run osquery

> The following assumes a Fleet server instance running and listening at `localhost:8080`.

Create the following directories:
```sh
mkdir C:\Users\Lucas Rodriguez\Downloads\osqueryd\local
mkdir C:\Users\Lucas Rodriguez\Downloads\osqueryd\local\osqueryd_log
```

Copy the Fleet test certificate (`./tools/osquery/fleet.crt`) into a known path:
```sh
C:\Users\Lucas Rodriguez\Downloads\osqueryd\local\fleet.crt
```

```sh
set ENROLL_SECRET=<...>

osqueryd.exe --verbose=true --tls_dump=true --pidfile="C:\Users\Lucas Rodriguez\Downloads\osqueryd\local\osquery.pid" --database_path="C:\Users\Lucas Rodriguez\Downloads\osqueryd\local\osquery.db" --logger_path="C:\Users\Lucas Rodriguez\Downloads\osqueryd\local\osqueryd_log" --host_identifier=instance --tls_server_certs="C:\Users\Lucas Rodriguez\Downloads\osqueryd\local\fleet.crt" --enroll_secret_env=ENROLL_SECRET --tls_hostname=host.docker.internal:8080 --enroll_tls_endpoint=/api/v1/osquery/enroll --config_plugin=tls --config_tls_endpoint=/api/v1/osquery/config --config_refresh=60 --disable_distributed=false --distributed_plugin=tls --distributed_tls_max_attempts=10 --distributed_tls_read_endpoint=/api/v1/osquery/distributed/read --distributed_tls_write_endpoint=/api/v1/osquery/distributed/write --logger_plugin=tls --logger_tls_endpoint=/api/v1/osquery/log --disable_carver=false --carver_disable_function=false --carver_start_endpoint=/api/v1/osquery/carve/begin --carver_continue_endpoint=/api/v1/osquery/carve/block --carver_block_size=2000000 --extensions_autoload="C:\Program Files\Fleetd\extensions.load" --allow_unsafe --enable_watchdog_debug --distributed_denylist_duration 0 --enable_extensions_watchdog > osqueryd.log 2>&1 
```

## Log analysis

The following (macOS) commands and scripts can be used to analyze the load in the device (as monitored by the watchdog process).

### Watchdog process kills

Run the following commands to check if watchdog trigger a worker kill:
```sh
rg "utilization limit" /tmp/osqueryd.log
rg "Memory limit" /tmp/osqueryd.log
```
If the above commands return no output then the load on the device was below the limits configured by osquery.

### Render CPU and memory usage

The following script renders the CPU and memory utilization throughout the load test:

On macOS (while osqueryd is running):
```sh
./tools/loadtest/osquery/gnuplot_osqueryd_cpu_memory.sh
```

For Windows, first, locate the `osqueryd.log` generated by `osqueryd.exe` and place it in the macOS host in `/tmp/osqueryd.log`.
Then, grab the osquery worker pid and run the following:
```sh
OSQUERYD_PID=7732 ./tools/loadtest/osquery/gnuplot_osqueryd_cpu_memory.sh
```

> The horizontal red line is the configured CPU usage limit (hardcoded to `1200ms` in the `gnuplot_osqueryd_cpu_memory.sh`)
