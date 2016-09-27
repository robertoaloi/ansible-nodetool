# README

An Ansible module to interact with Erlang nodes.

## Available options

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

## Credits

The work is based on the `nodetool` script from the Yaws project:

https://github.com/klacke/yaws

## Contributors

A big thanks to:

- Amir Moulavi
- Fabricio Leotti

For their precious contributions.
