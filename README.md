# README

An Ansible module to interact with Erlang nodes via Erlang RPC.

## Overview

If your architecture includes one or more Erlang nodes and you use
Ansible to orchestrate them, you may find this Ansible module helpful.

## Mininmum Requirements

- Ansible 2.0.0.2
- Erlang/OTP 17.0

## Installation

Clone the ansible-nodetool repository:

git clone https://github.com/robertoaloi/ansible-nodetool.git /path/to/ansible-nodetool

Then, you need to tell Ansible where to find the new module.
You can do this by
appending the repository path to the `library` value in your
`~/.ansible.cfg` file. You can find the `library` value under the
`defaults` group. If you do not have a `defaults` group in your
`~/.ansible.cfg` file (or if you do not have a `~/.ansible.cfg` file)
add one. You can find more information about configuring Ansible
[here](http://docs.ansible.com/ansible/intro_configuration.html).

    [defaults]
    library = /path/to/ansible-nodetool

Alternatively, you can specify the `-M` option when invoking a
playbook. Example:

    ansible-playbook -M /path/to/ansible-nodetool my_playbook.yml

Or when running an [ad-hoc
command](http://docs.ansible.com/ansible/intro_adhoc.html). Example:

    ansible -m nodetool \
            -M /path/to/ansible-nodetool \
            -a 'node=alice@localhost cookie=secret action=ping' \
            localhost

## Parameters

     node:
         description: The remote Erlang node
         required:    true
     action:
         description: The action to be performed
         choices:     [getpid, ping, stop, restart, reboot, eval]
         required:    true
     nametype:
         description: Nametype to be used
         choices:     [longnames, shortnames]
         required:    false
         default:     shortnames
     cookie:
         description: Erlang Cookie to be used for the connection
         required:    false
     timeout:
         description: Timeout (in ms) for the actions
         required:    false
         default:     60000

## Usage

The ansible-nodetool module is typically used from an Ansible
playbook.

To try things out, start a sample Erlang node named
'alice' and with a 'secret' cookie:

    erl -sname alice@localhost -setcookie secret

You can now ping the node using the following playbook:

    ---

    - hosts: localhost
      tasks:
      - name: "Ping the 'alice' Erlang node"
        nodetool:
          action: ping
          cookie: secret
          node:   alice@{{ inventory_hostname_short }}

Example:

    ansible-playbook ping.yml

    PLAY
    ***************************************************************************

    TASK [setup]
    *******************************************************************
    ok: [localhost]

    TASK [Ping the 'alice' Erlang node]
    ********************************************
    changed: [localhost]

    PLAY RECAP
    *********************************************************************
    localhost                  : ok=2    changed=1    unreachable=0
    failed=0

But the ansible-nodetool module is not only about pinging nodes.
You can also evaluate custom Erlang expressions on a remote Erlang
node using the 'eval' action.

The following playbook gets the list of running applications in an
Erlang node, registers the result into an Ansible variable and prints
the result:

    ---

    - hosts: localhost
      tasks:
      - name: "Return a list of running applications"
        nodetool:
          action:  eval
          command: application:which_applications()
          cookie:  secret
          node:    alice@{{ inventory_hostname_short }}
        register: applications
      - debug:
          msg: "{{ applications.stdout_lines }}"

Let's see it in action:

    $ ansible-playbook applications.yml

    PLAY
    ***************************************************************************

    TASK [setup]
    *******************************************************************
    ok: [localhost]

    TASK [Return a list of running applications]
    ***********************************
    changed: [localhost]

    TASK [debug]
    *******************************************************************
    ok: [localhost] => {
      "msg": [
        "[{sasl,\"SASL  CXC 138 11\",\"2.4.1\"},",
        " {stdlib,\"ERTS  CXC 138 10\",\"2.4\"},",
        " {kernel,\"ERTS  CXC 138 10\",\"3.2.0.1\"}]"
      ]
    }

    PLAY RECAP
    *********************************************************************
    localhost                  : ok=3    changed=1    unreachable=0
    failed=0

Another available action is `getpid`, which returns the process
identifier of the current Erlang emulator:

    ---

    - hosts: localhost
      tasks:
      - name: "Return the process identifier of the current Erlang emulator"
        nodetool:
          action: getpid
          cookie: secret
          node:   alice@{{ inventory_hostname_short }}
        register: pid
      - debug:
          msg: "{{ pid.stdout }}"

There are also other available actions, such as `stop`, `restart` or
`reboot` to control a remote Erlang node.

## Result

As part of the Ansible result, the ansible-nodetool provides a JSON structure
which contains the following fields:

 FIELD        | DESCRIPTION
--------------|-----------------------------------------------------------------
rc            | The Erlang RPC _return code_ (0 => success, 1 => failure)
stdout        | The return value of the Erlang RPC
stdout_lines  | The return value of the Erlang RPC call, split in lines
remote_output | The stdout on the remote Erlang node, captured via [group leader](http://erlang.org/doc/man/erlang.html#group_leader-0)

I have seen many people trying to assert the return value of the
Erlang RPC in complicated ways. Remember that, if you want to verify
that the return value of a given Erlang expression corresponds to a
specific value you can simply use the [pattern matching
operator](http://erlang.org/doc/reference_manual/patterns.html) for
this. No need to find complicated Ansible-based solutions!

Example:

    ---

    - hosts: localhost
      tasks:
      - name: "Ensure that the reverse of 'foo' is 'oof'"
        nodetool:
          action:  eval
          command: "\"oof\"=lists:reverse(\"foo\")"
          cookie:  secret
          node:    alice@{{ inventory_hostname_short }}

## Idempotence

Idempotence is a crucial principle in Ansible. Given the nature of the
Ansible nodetool module, which allow the operator to perform custom
Erlang expressions via the `eval` action, it is left to the user,
which should try not to violate it as much as possible.

As an example, to start an Erlang application, the following playbook
would *not* be idempotent:

    ---

    # THIS PLAYBOOK IS NOT IDEMPOTENT!!!
    # RUNNING IT TWICE WOULD CAUSE A FAILURE!!!
    - hosts: localhost
      tasks:
      - name: "Start the SASL application"
        nodetool:
          action:  eval
          command: ok=application:start(sasl).
          cookie:  secret
          node:    alice@{{ inventory_hostname_short }}

The following is a much better implementation:

    ---

    - hosts: localhost
      tasks:
      - name: "Start the SASL application"
        nodetool:
          action:  eval
          command: ok=application:ensure_started(sasl)
          cookie:  secret
          node:    alice@{{ inventory_hostname_short }}

## Credits

The work is based on the `nodetool` script from the Yaws project:

https://github.com/klacke/yaws

## Contributors

A big thanks to:

- Amir Moulavi
- Fabricio Leotti

For their precious contributions.
