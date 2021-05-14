# ansible-role-tpotce

This role is an ansible implementation of the [t-pot](https://github.com/dtag-dev-sec/tpotce) installer script. It is limited to a sensor only configuration.

## Role Variables

### `honeypot_list`

Use this variable to specify which t-pot honeypots you want deployed to the system. The default settings specify the following honeypots:

  - `ciscoasa`
  - `adbhoney`
  - `conpot`
  - `cowrie`
  - `dionaea`
  - `honeypy`
  - `glutton`
  - `heralding`
  - `mailoney`
  - `medpot`
  - `p0f`
  - `rdpy`
  - `suricata`
  - `tanner`
  - `fatt`

You may choose from any of the following:

  - `ciscoasa`
  - `adbhoney`
  - `conpot`
  - `cowrie`
  - `dicompot`
  - `dionaea`
  - `elasticpot`
  - `honeypy`
  - `honeysap`
  - `ipphoney`
  - `glutton`
  - `honeytrap`
  - `heralding`
  - `mailoney`
  - `medpot`
  - `p0f`
  - `rdpy`
  - `suricata`
  - `tanner`
  - `fatt`
  - `citrixhoneypot`

The `glutton` and `honeytrap` honeypots _can't_ be used on the same system.

### `logrotate_days`

Use this variable to specify the number of days that logs are retained on the sensor. The default value is 30 days. Depending on the disk size of the honeypot and activity it receives, this value may require adjustment to prevent exhausting disk space.

## Filebeat

If you would like to make use of filebeat to send your logs to logstash you need to run the [honeynet/ansible-role-tpotce-filebeat](https://github.com/honeynet/ansible-role-tpotce-filebeat) role in addition to this one. You will need to add `filebeat` to the `honeypot_list` var.

### `filebeat_version`

Use this variable to specify what version of filebeat you would like to use. The following are supported.

    - 7.12.0
    - 7.11.2
    - 7.11.1
    - 7.11.0
    - 7.10.1
    - 7.10.0
    - 7.9.3
    - 7.9.2
    - 7.9.1
    - 7.9.0
    - 7.8.1
    - 7.8.0
    - 7.7.1
    - 7.7.0
    - 7.6.2
    - 7.6.1
    - 7.5.2
    - 7.4.2
    - 7.3.2
    - 7.2.1
    - 7.1.1
    - 7.0.1

## Example Playbook

```
- hosts: all
  become: true
  roles:
    - ansible-role-tpotce
```

## License

The role is licensed under GPLv3


