+++
date = '2025-02-08'
draft = false
title = 'Ansible Tech - Configuring GNOME Shell'
+++

# Intro
This article depicts practical ways of using Ansible to configure GNOME Shell.
Most examples were taken from my [personal repository](https://github.com/allpower2slaves/ansible-setup-public/) that I use on my desktop systems and laptops.

Using tasks from this article implies you know how GNOME stores it's
configuration under the hood, if not, I suggest you learn about `dconf` and
`gsettings`. There are some links in [Suggested links](#suggested-links) section.

This article might be useful for:
+ people who like to set everything up with Ansible
+ engineers that are managing multiple GNOME-based workstations
+ novices trying to learn Ansible

I think the only thing this article lacks is a sane method to install GNOME
Shell extensions☹️. This article will be updated whenever I manage to hack
something together 👽.

# Ansible tasks

Most tasks require
[community.general.dconf module](https://docs.ansible.com/ansible/latest/collections/community/general/dconf_module.html).
In the examples below, `community.general.dconf` module would be invoked with
`dconf`.

The original tasks that I use can be found
[here](https://github.com/allpower2slaves/ansible-setup-public/blob/master/roles/fedora-personal/tasks/gnome-setup.yaml).


## dconf data types
There are four data types that you will encounter in this article:
+ integers
+ booleans
+ strings
+ string arrays

The first two types (integers and booleans) require no wrapping, strings should
be wrapped in single quotes and double quotes `"'like this'"`. String arrays
are be handled either as strings or as Ansible(YAML) lists depending on the
context.

## Basic GNOME configuration methods

### Single option
```
- name: GNOME dconf setup -- locale
  dconf:
    key: "/system/locale/region"
    value: "'en_GB.UTF-8'"
    state: present
```
In this very basic example, I set the my system locale to be `en_GB.UTF-8`.

### Multiple key-value pairs (One application)
```
- name: GNOME dconf setup -- nautilus
  dconf:
    key: "/org/gnome/nautilus/preferences/{{ item.key }}"
    value: '{{ item.value }}'
    state: present
  loop: "{{ dconf_loop | dict2items }}"
  vars:
    dconf_loop:
      default-folder-viewer: "'icon-view'"  # strings should be wrapped in "'
      default-sort-in-reverse-order: true # booleans should be wrapped in ' or without any wrapping
      default-sort-order: "'mtime'"
      migrated-gtk-settings: 'true'
      search-filter-time-type: "'last_modified'"
```
The provided task will configure multiple Nautlus file manager. Notice how i wrapped single string and boolean values.

The grouping of these keys makes most sense -- `dconf dump /` will list them
like this:
```
[org/gnome/nautilus/preferences]
default-folder-viewer='icon-view'
default-sort-in-reverse-order=true
default-sort-order='mtime'
migrated-gtk-settings=true
search-filter-time-type='last_modified'
```

### Manipulating string arrays as strings (easy)
```
- name: GNOME dconf setup -- input-sources
  dconf:
    key: "/org/gnome/desktop/input-sources/{{ item.key }}"
    value: '{{ item.value | string }}'  # so string here was needed after all....
    state: present
  loop: "{{ dconf_loop | dict2items }}"
  vars:
    dconf_loop:
      mru-sources: "[('xkb', 'us'), ('xkb', 'ru')]"
      sources: "[('xkb', 'us'), ('xkb', 'ru')]"
      xkb-options: "['caps:none']"
```
Notice how string arrays are wrapped as strings. Obviously, you can not add or
remove new entries to these arrays. There is a [more flexible and sophisticated
method](#hard-lists) to manipulate string arrays in dconf database. Keep in
mind that the `input-sources` family of entries are stored as "tuples of 2
strings", but in this example they are handled as strings -- there should be no
problem with syntax.

### Multiple independent key-value pairs
```
- name: GNOME dconf setup -- oneliners # for oneliners
  dconf:
    key: "{{ item.key }}"
    value: '{{ item.value }}'
    state: present
  loop:
    - { key: '/org/gnome/desktop/interface/clock-format', value: "'24h'" }         # strings should be covered in ""
    - { key: '/org/gnome/desktop/peripherals/mouse/accel-profile', value: "'flat'" } # booleans should be covered in ''
    - { key: '/org/gnome/desktop/privacy/report-technical-problems', value: false }
    - { key: '/org/gnome/desktop/search-providers/disabled', value: "['org.mozilla.firefox.desktop', 'org.gnome.Software.desktop']" }
    - { key: '/org/gnome/desktop/wm/preferences/resize-with-right-button', value: true }
    - { key: '/org/gnome/mutter/attach-modal-dialogs', value: false}
    - { key: '/org/gnome/shell/extensions/caffeine/show-notifications', value: false }
    - { key: '/org/gnome/system/location/enabled', value: false }
    - { key: '/org/gnome/tweaks/show-extensions-notice', value: false }
    - { key: '/org/gnome/settings-daemon/plugins/power/sleep-inactive-ac-type', value: "'nothing'"} 
```
This would be useful for setting multiple independent key-value pairs -- as you
can see in the provided example, unlike with Nautilus setup, options here are
in different categories. It would be very tedious otherwise to create
individual tasks for each option.

### Resetting options to their default state {#reset-options}
```
- name: GNOME dconf setup -- reset options
  dconf:
    key: "{{ item }}"
    state: absent
  loop:
    - '/org/gnome/desktop/wm/preferences/focus-mode'
    - '/org/gnome/shell/app-switcher/current-workspace-only'
    - '/org/gnome/mutter/experimental-features'
```
To reset an option (dconf key), just use `state: absent`. To configure
system-level defaults, see [Setting system-level defaults](#dconf-defaults) section.


## Configuring custom shortcuts
In this section, there will be a group of tasks that are responsible with
configuring custom shortcuts in GNOME. I highly recommend grouping them with
blocks or even separate tasklists (imported files with tasks). Please, do not
blindly copy and paste the tasks for reasons you will understand later 😃.

### Initialization
```
- name: GNOME dconf setup -- shortcuts init # if youre reading this -- have a nice day :)
  dconf:
    key: '/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings'
    value: "['/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom0/', '/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom1/', '/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom2/']"
    state: present
```
This task is very important -- notice the strings ranging from `custom0` to
`custom2` -- these are basically entries that enable the shortuts.

I honestly did not go into much detail about how this works internally -- all I
am going to provide are methods of how can you make it work 😃.

### The shortcuts themselves
```
- name: GNOME dconf setup -- shortcuts -- nautilus
  dconf:
    key: '/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom0/{{ item.key }}'
    value: '{{ item.value }}'
    state: present
  loop: "{{ dconf_loop | dict2items }}"
  vars:
    dconf_loop:
      binding: "'<Super>e'"
      command: "'nautilus -w'" # `-w` for new window
      name: "'nautilus'"

- name: GNOME dconf setup -- shortcuts -- ghostty
  dconf:
    key: '/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom1/{{ item.key }}'
    value: '{{ item.value }}'
    state: present
  loop: "{{ dconf_loop | dict2items }}"
  vars:
    dconf_loop:
      binding: "'<Control><Alt>t'"
      command: "'ghostty'"
      name: "'ghostty'"

- name: GNOME dconf setup -- shortcuts -- ptyxis
  dconf:
    key: '/org/gnome/settings-daemon/plugins/media-keys/custom-keybindings/custom2/{{ item.key }}'
    value: '{{ item.value }}'
    state: present
  loop: "{{ dconf_loop | dict2items }}"
  vars:
    dconf_loop:
      binding: "'<Control><Alt>y'"
      command: "'ptyxis --new-window'"
      name: "'ptyxis'"
```
After initializing the dconf database keys responsible for shortcuts, there are
the shortuts themselves -- notice the `key` values in each task.

## Special example -- my personal ptyxis config
```
- block: # ptyxis setup
  - name: GNOME dconf setup -- ptyxis -- add profile
    dconf:
      key: '/org/gnome/Ptyxis/Profiles/replaceme/{{ item.key }}'
      value: '{{ item.value }}'
    loop: "{{ dconf_loop | dict2items }}"
    vars:
      dconf_loop:
        bold-is-bright: false
        custom-command: "'fish'"
        label: "'second-term'"
        login-shell: false
        palette: "'solarized'"
        use-custom-command: true

  - name: GNOME dconf setup -- ptyxis -- global config
    dconf:
      key: '/org/gnome/Ptyxis/{{ item.key }}'
      value: '{{ item.value }}'
    loop: "{{ dconf_loop | dict2items }}"
    vars:
      dconf_loop:
        default-columns: 60
        default-profile-uuid: "'replaceme'"
        default-rows: 20
        font-name: "'Spleen 16x32 Bold 12'"
        interface-style: "'dark'"
        profile-uuids: "['notarealuuid']"
        restore-session: false
        restore-window-size: false
        use-system-font: false
        #window-size: "(uint32 60, uint32 20)"

# END of BLOCK
```
Here is my ptyxis terminal emulator config in asnible. Notice how I replaced
UUID and profile name to `replaceme` just for this example, so it might not
work without tweaking them.

## Advanced GNOME setup

### GNOME extensions management (enable/disable extensions)
This is my personal masterpiece -- the most hacky task I have ever made. I
spent around 6 hours total making these ones and the tasks found in the next section.
Notice the long `shell` module command entry -- that's how I adapted basic
shell invokations into something that will properly report changes in Ansible.
I think the `echo 'ok'` part can be removed, but I did not bother to try removing it.

I made this task in two variants -- chose the one you like the most.

**Variant A** (the one I use):
```
- name: manage GNOME extensions # ternary version
  shell:
    cmd: >
      gnome-extensions list --{{ item.enabled | default(true) | ternary('enabled', 'disabled', 'enabled') }} | grep -qxe {{ item.name }} && echo 'ok'
      ||
      (gnome-extensions {{ item.enabled | default(true) | ternary('enable', 'disable', 'enable') }} {{ item.name }} && echo 'changed')
  register: extensions_enable
  changed_when: "'changed' in extensions_enable.stdout"
  loop:
    - { name: "quick-lang-switch@ankostis.gmail.com", enabled: true }
    - { name: "caffeine@patapon.info", enabled: false }
    - { name: "ProxySwitcher@flannaghan.com" } # will be enabled!!!
```
I have made this with `ternary` and `default` filters to emulate `systemd`
module behaivour -- there is an `enabled` key that takes boolean values. If the
value is undefined, the task will enable that extension.

**Variant B**:
```
- name: manage GNOME extensions # version with state setting...
  shell:
    cmd: >
      gnome-extensions list --{{ item.state | default('enabled') }}| grep -qxe {{ item.name }} && echo 'ok'
      ||
      (gnome-extensions {{ item.state[0:-1] | default('enable') }} {{ item.name }} && echo 'changed')
  register: extensions_enable
  changed_when: "'changed' in extensions_enable.stdout"
  loop:
    - { name: "quick-lang-switch@ankostis.gmail.com", state: "enabled" }
    - { name: "caffeine@patapon.info", state: "disabled" }
    - { name: "ProxySwitcher@flannaghan.com" }  # will be enabled
```
Actually, this one came first. You have to set the `state` key to be either
`"enabled"` or `"disabled"`. As with the other variant, unset `state` key
will result in the extension being enabled.

I think it is pretty obvious why I changing the string values to boolean ones 😁.

The reasons for choosing `gnome-extensions` CLI tool over `dconf` module will
become obvious after reviewing the next section👽👽👽.

### Proper (and complex) way to manipulate string arrays {#hard-lists}
```
- block: # GNOME extension enabling
  - name: collect information about enabled GNOME extensions
    command:  # dconf module is bad for this, i tried :)
      cmd: gsettings get org.gnome.shell enabled-extensions
    changed_when: false
    register: gsettings_parse_extensions

  - name: collect information about disabled GNOME extensions
    command: # what
      cmd: gsettings get org.gnome.shell disabled-extensions
    changed_when: false
    register: gsettings_parse_extensions_disabled

  - name: add GNOME extensions to enabled list  # might report as `changed` first time executing -- it will sort them
    dconf:
      key: "/org/gnome/shell/enabled-extensions" 
      value: "{{ gsettings_parse_extensions.stdout[1:-1] | replace(\"'\", '') | split(', ') | union(gnome_enable_extensions) | sort }}"

  - name: remove GNOME extensions from disable overrides # hacky solution but whatever
    when: "'@as []' not in (gsettings_parse_extensions_disabled.stdout | string )"
    dconf:
      key: "/org/gnome/shell/disabled-extensions"
      value: "'{{ gsettings_parse_extensions_disabled.stdout[1:-1] | replace(\"'\", '') | split(', ') | difference(gnome_enable_extensions) | sort | string }}'"

  vars: # block variables
    gnome_enable_extensions: # list of GNOME extensions to enable
      - caffeine@patapon.info
      - quick-lang-switch@ankostis.gmail.com
```

This is what I have spent most time hacking together in this 6 hour timeframe
(all thanks to GNOME having a separate string array just for override-disabling
extension😁) I just mentioned. Eventually I discarded this. Noneless, this is how
I would dynamically add or remove entries into string arrays.

When using string arrays sourced from a string -- that being
`gsettings_parse_extensions_disabled.stdout[1:-1]` (The `[1:-1]` will strip the
square brackets), I also stripped `'` symbols -- otherwise they will be treated
as part of the strings and look like `[ "'this'", 'and not this']` after the
final conversion.

All this string manipulation can be skipped -- if you run 
```
- set_fact:
    ggs: "{{ gsettings_parse_extensions.stdout }}"
```
`ggs` would be a valid list, not a string you then brutally cut and strip for
it to be transformed back into a list.


Notice how I use `sort` at the end of the array manipulation -- that way
`difference` or `union` filters wont reorder the Ansible (yaml? jinja2?) lists
and report `changed` even when no new array entries were added.

What this block lacks is a task to resolve conflicts -- what if I add the same
extension in the "enabled" and "disabled" lists? One might hack a few tasks
that will deal with that. Noneless, while making this pack of tasks I think I
learned a thing or two👽. If not for the `gnome-extensions` tool, I think I
would have been using this solution.

## Setting system-level defaults {#dconf-defaults}
```
- name: change GNOME system-level default settings
  copy:
    src: gnome-system.dconf
    dest: /etc/dconf/db/local.d/09-gnome-example.dconf
    mode: '0644'
    backup: no
    # maybe a gnome tag here....
  register: dconf_update # this might be a notifies
  notify: reboot # because of GDM

- name: update dconf
  command:
    cmd: dconf update
  when: dconf_update.changed
```

And the contents of `gnome-system.dconf` would resemble `dconf dump /`
ini-formatted output:
```
[org/gnome/software]
download-updates=false
first-run=false

[org/gnome/settings-daemon/plugins/power]
sleep-inactive-ac-type='nothing'
```

If you know a thing or two about dconf (as you should), there is a basic understanding how to set GNOME defaults. This is a simple example of how one can set GNOME Shell defaults.  The `dest:` value was made with Fedora Workstation in mind, I dont know how would it work on other GNU/Linux distributions.

It would be even better if `update dconf` task would be a
handler.

If you wish to apply the new defaults, [reset](#reset-options) the values on the user side. Beware that some options are
automatically sourced and **set** by GNOME Shell -- like
`sleep-inactive-ac-type`, so running tasks that reset them will result in
Ansible reporting "changed" after a few subsequent playbook executions.

# Suggested links {#suggested-links}
+ [https://wiki.gnome.org/Projects/dconf](https://wiki.gnome.org/Projects/dconf)
+ [RHEL 8 Docs](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/8/html/using_the_desktop_environment_in_rhel_8/configuring-gnome-at-low-level_using-the-desktop-environment-in-rhel-8#monitoring_key_changes)
+ [my old gnome dconf config](https://github.com/allpower2slaves/dotfiles/blob/master/gnome-setup.dconf)
