# promtail

[![Build Status](https://drone.osshelp.io/api/badges/ansible/promtail/status.svg)](https://drone.osshelp.io/ansible/promtail)

Role for Promtail installation and configuration.

## Usage (example)

```yaml
    - role: promtail
      promtail_labels_env: production
      promtail_collect_system: true
      promtail_collect_nginx: true
      promtail_clients:
        - url: https://example.domain.tld/loki/api/v1/push
        - url: https://with-tenant-and-auth.domain.tld/loki/api/v1/push
          tenant_id: 11111111-2222-3333-4444-555555555555
          basic_auth:
            username: someuser
            password: SomeStrongPassword111
```

## Available parameters

### Main

| Param | Default | Description |
| -------- | -------- | -------- |
| `promtail_setup` | `full` | Setup mode. See [OSSHelp KB article](https://oss.help/kb4895) |
| `promtail_version` | `2.6.1` | Version to install. Select from [available releases](https://github.com/grafana/loki/releases). |
| `promtail_clients` | `- url: 'http://localhost:3100/loki/api/v1/push'` | List of clients. |
| `promtail_additional_groups` | `[adm]` | List of groups to add the promtail user to (for logs access). By default Promtail is running under root for now, so no need in changing this param anyhow. |
| `promtail_main_cfg_template` | `promtail-main-cfg.j2` | j2-tempalte to use for main configuration file generation. |
| `promtail_labels_env` | `production` | Value to use in `environment` label in generated jobs. |
| `promtail_labels_instance` | `"{{ inventory_hostname }}"` | Value to use in `instance` label in generated jobs. |

### Jobs enabling

Can be done with following switches:

| Param | Default | Description |
| -------- | -------- | -------- |
| `promtail_relay_enabled` | `false` | Job for receiving logs from other Promtail instances. For example, from lxc-containers. |
| `promtail_collect_system` | `true` | Collecting logs from `/var/log/*log`. |
| `promtail_collect_nginx` | `false` | Collecting nginx access logs from `/var/log/nginx/*access.log`.  |
| `promtail_collect_audit` | `false` | Collecting audit logs from `/var/log/audit/*.log`. |
| `promtail_collect_journal` | `false` | Collecting messages from journald. |
| `promtail_collect_docker` | `false` | Collecting messages from dockerd. |

### Generating custom jobs

Using it will not override any job enablers above, so make sure that `promtail_collect_<job>` for the job you're going to override is pinned to `false` or you'll get duplicates later. You can describe custom jobs with `promtail_scrape_configs` list, for example:

```yaml
      promtail_scrape_configs:
        - job_name: system
          static_configs:
            - targets:
                - localhost
              labels:
                job: varlogs
                env: production
                instance: "{{ ansible_host }}"
                source: "{{ inventory_hostname }}"
                __path__: /var/log/*log
```

You can also use your own j2-template to generate custom config, see `promtail_main_cfg_template` param.

### Rate limiting

Can be configured with `promtail_limits_config` list, params:

| Param | Default | Description |
| -------- | -------- | -------- |
| `readline_rate_enabled` | `true`|  When `true`, enforces rate limiting. |
| `readline_rate_drop` | `true` | When `true`, exceeding the rate limit causes Promtail to discard log lines, rather than sending them to Loki. When `false`, exceeding the rate limit causes Promtail to temporarily hold off on sending the log lines and retry later. |
| `readline_rate` | `100`| The rate limit in log lines per second that Promtail may push to Loki. |
| `readline_burst` | `1000`| The cap in the quantity of burst lines that Promtail may push to Loki. |

Usage example:

```yaml
      promtail_limits_config:
        readline_rate_enabled: true
        readline_rate_drop: false
        readline_rate: 35
        readline_burst: 100
```

#### Per job limiting

You can override limits above for supported jobs with these variables:

| Param | Description |
| -------- | -------- |
| `promtail_limits_system` | Limits for system job. |
| `promtail_limits_nginx` | Limits for nginx job. |
| `promtail_limits_audit` | Limits for audit job. |
| `promtail_limits_journal` | Limits for journal job. |
| `promtail_limits_docker` | Limits for docker job. |

For every job you can control rate, burst and drop mode, e.g.:

```yaml
      promtail_limits_nginx:
        rate: 25
      promtail_limits_system:
        rate: 10
        burst: 25
        drop: false
```

`drop` and `burst` keys are optional - if omitted, they will be inherited from global `promtail_limits_config.readline_rate_drop` and `promtail_limits_config.readline_burst`.

### Custom pipeline stages (per job)

You can add custom elements to pipeline_stages section. It can be done with these lists:

| Param | Description |
| -------- | -------- |
| `promtail_system_custom_stages` | Custom pipeline stages for system job. |
| `promtail_nginx_custom_stages` | Custom pipeline stages for nginx job. |
| `promtail_audit_custom_stages` | Custom pipeline stages for audit job. |
| `promtail_journal_custom_stages` | Custom pipeline stages for journal job. |
| `promtail_docker_custom_stages` | Custom pipeline stages for docker job. |

These custom pipeline stages will be added after per job limits, if they have been applied.

An example with the drop stage for journald job (value matches the source):

```yaml
      promtail_journal_custom_stages:
          - drop:
              source: syslog_identifier
              value: systemd
              drop_counter_reason: "Custom: testing"
```

An example with the drop stage for the system job (regex matches the source):

```yaml
      promtail_system_custom_stages:
          - drop:
              selector: '{filename="/var/log/syslog"}'
              expression: ".*.(smtp|SMTP|postfix|DKIM|dovecot|unexpected RCODE REFUSED).*"
              drop_counter_reason: "Custom: lines from mail."
```

If you won't set drop_counter_reason it can provoke Log_entries_drop_ENV_XXpct alerts. To avoid it - start the reason with "Custom:" (e.g. "Custom: some reason here"), alert-rules will ignore drop-counters with such values as reason. If you want to learn more about drop stages, check [this](https://grafana.com/docs/loki/latest/clients/promtail/stages/drop/).

### Resource usage

Could be controlled via `promtail_limits`.

| Param | Default | Description |
| -------- | -------- | -------- |
| `promtail_service_limits.memory` | `1G` | Specify the absolute limit on memory usage. Takes a memory size in bytes. If the value is suffixed with K, M, G or T, the specified memory size is parsed as Kilobytes, Megabytes, Gigabytes, or Terabytes (with the base 1024), respectively. Alternatively, a percentage value may be specified, which is taken relative to the installed physical memory on the system. |

## FAQ

### Promtail refuses to start and no errors in logs

Try launching it manually:

```plain
~# promtail -config.file /etc/promtail/config.yml
Unable to parse config: /etc/promtail/config.yml: yaml: unmarshal errors:
  line 31: field docker_sd_configs not found in type scrapeconfig.plain
```

If you are getting such errors try updating the [version](https://github.com/grafana/loki/releases) (some params are supported only by newest releases).

## Useful links

- [Official documentation](https://grafana.com/docs/loki/latest/clients/promtail/)
- [Available releases](https://github.com/grafana/loki/releases)

## TODO

- Reduce priviledges, working under root for now to collect all possible logs
- Proper jobs (remove placeholders) generation after moving to newer jinja versions or adding do-extension
- Path customization for built-in jobs?

## License

GPL3

## Author

OSSHelp Team, see <https://oss.help>
