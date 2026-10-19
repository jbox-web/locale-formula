# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
bundle install
bin/kitchen list                                 # 3 instances, one suite
bin/kitchen test default-debian-13-3008-2        # single instance
bin/kitchen verify default-debian-11-3006-27     # re-run InSpec only
pre-commit run --all-files                       # all lint hooks, as the CI does
```

Instances: `default-debian-13-3008-2`, `default-debian-13-3006-27`,
`default-debian-11-3006-27`.

## Architecture — how this fork departs from the template

- **One state file.** All logic lives in `locale/init.sls`; there is no
  `package/` / `config/` / `service/` split, no `libtofs.jinja`, no managed
  templates. The only states are `locale` and the test-only `locale._mapdata`.
- **IDs in `init.sls`**, each rendered only when its pillar key is set:
  - `locale_pkgs` installs `locale.pkgs` (`locales` on Debian);
  - `locale_present_<name>` (`locale.present`), one per `locale:present`
    entry; the ID is the entry with `.` and spaces replaced by `_`;
  - `dbus_for_locale` installs and, when `dbus.run_service`, runs `dbus`,
    both `require_in` `locale_default`: on Debian 11 Salt sets the default
    through `localectl`, which needs D-Bus (on Debian 13 it uses
    `update-locale` instead);
  - `locale_default` (`locale.system`) requires
    `locale_present_<locale:default:requires>`. `requires` must therefore be
    spelled exactly like a `locale:present` entry, or the highstate fails on a
    missing requisite;
  - `locale_conf` writes `locale.config` from `contents_pillar: locale:conf`.
- **Two spellings of one locale.** `present` and `default:requires` take the
  `/etc/locale.gen` form `"en_US.UTF-8 UTF-8"`; `default:name` takes
  `en_US.UTF-8` only. A trailing ` UTF-8` in `name` breaks Salt on Debian
  (`LookupError: unknown encoding: utf_8_utf_8`, see `pillar.example`).
- **Map files are no-ops beyond `osfamilymap.yaml`.** The Debian values
  (`pkgs: [locales]`, `config: /etc/default/locale`) come from
  `osfamilymap.yaml`; `dbus` from `defaults.yaml`. `osfingermap.yaml` lists
  only Debian 8–10 as empty dicts, so Debian 11/13 fall through to the family
  values. `osarchmap.yaml` sets an `arch` key nothing reads.
- **Pillar keys**: `locale:present`, `locale:default:{name,requires}`,
  `locale:conf`, plus `pkgs`, `config`, `dbus:{pkg,service,run_service}`
  (overridable directly or under `locale:lookup`).

## Tests

- One suite, `default`, with pillar `test/salt/pillar/default.sls`
  (`en_US` and `de_DE`, default `en_US.UTF-8`, no `conf`).
  `test/salt/pillar/redhat.sls` is referenced by no suite.
- Controls: `package_spec.rb` (`locales` installed) and `config_spec.rb`
  (both locales in `/etc/locale.gen` on Debian, `locale -a` elsewhere). The
  locale literals in the control must follow the test pillar. Nothing asserts
  the system default or `locale_conf`.
- `locale._mapdata` runs in `state_top`, but this fork has **no**
  `test/integration/default/files/_mapdata/` reference dumps and no control
  comparing them: the merged map is dumped, not asserted.
