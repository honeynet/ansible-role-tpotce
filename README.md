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
  - `dionaea`
  - `honeypy`
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

The `glutton` and `honeytrap` honeypots _can't_ be used on the same system.

### `logrotate_days`

Use this variable to specify the number of days that logs are retained on the sensor. The default value is 30 days. Depending on the disk size of the honeypot and activity it receives, this value may require adjustment to prevent exhausting disk space.

## Example Playbook

```
- hosts: all
  become: true
  roles:
    - ansible-role-tpotce
```

## License

The role is licensed under GPLv3


