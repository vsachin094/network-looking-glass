# Network Looking Glass

[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Python](https://img.shields.io/badge/python-3.6%2B-blue.svg)](requirements.txt)
[![Flask](https://img.shields.io/badge/Flask-1.0.2-black.svg)](requirements.txt)
[![Paramiko](https://img.shields.io/badge/SSH-Paramiko-blue.svg)](requirements.txt)

A web based **looking glass** for network troubleshooting — pick a network, pick a
router, run a command, and watch the output stream back in your browser.

---

## What is a looking glass?

A looking glass is a server that lets network operators run diagnostic
commands (BGP lookups, ping, traceroute) against routers from a remote
location. Instead of logging into every device by hand, you open one page and
query any router in the inventory.

This project is a self contained looking glass built with
[Flask](https://flask.palletsprojects.com/) and
[Paramiko](https://www.paramiko.org/). It talks to the routers over SSH and
streams the output back while the command is still running.

## Features

- **Routers grouped by ASN** — the inventory is split per autonomous system.
- **Ready made operations** — `bgp`, `ping` and `traceroute` are translated
  automatically to the correct syntax for the platform of the router.
- **Custom commands** — run any command you like through the same interface.
- **Live output** — the response pane fills up while the command runs, you do
  not have to wait for the router to finish.
- **Multi platform** — command templates for IOS-XE, IOS-XR, NX-OS and JunOS.
- **Plain REST API** — every action is available as JSON, so the looking glass
  can be scripted or embedded in your own tools.
- **Zero database** — the router inventory is a single Python file.

## Requirements

- Python 3.6 or newer
- The packages listed in [`requirements.txt`](requirements.txt)
- SSH access to the routers you want to query

## Quick start

```sh
# 1. Get the code
git clone https://github.com/vsachin094/network-looking-glass.git
cd network-looking-glass

# 2. Create a virtual environment
python3 -m venv venv
source venv/bin/activate

# 3. Install the dependencies
pip install -r requirements.txt

# 4. Put your credentials in .env   (see "Configuration" below)

# 5. Describe your routers in routers.py   (see "Configuration" below)

# 6. Start it
python app.py
```

Then open **<http://localhost:5000/>**.

> The repository ships with the free **Cisco DevNet sandboxes** in the sample
> inventory, so you can try the interface before pointing it at your own gear.
> Those sandboxes are shared by thousands of people, so timeouts and
> authentication errors are normal there.

## Configuration

### 1. Credentials — `.env`

`routers.py` reads the SSH credentials from the environment. The `.env` file in
the project root is loaded automatically by `python-dotenv`, so you never have
to touch the code.

```sh
# .env
USERNXE=username
PASSWXE=password
USERNXR=username
PASSWXR=password
USERNNX=username
PASSWNX=password
```

| Variable  | Used for                       |
| --------- | ------------------------------ |
| `USERN`   | user name of the jump server   |
| `PASSW`   | password of the jump server    |
| `USERNXE` | user name of the IOS-XE router |
| `PASSWXE` | password of the IOS-XE router  |
| `USERNXR` | user name of the IOS-XR router |
| `PASSWXR` | password of the IOS-XR router  |
| `USERNNX` | user name of the NX-OS router  |
| `PASSWNX` | password of the NX-OS router   |

### 2. Router inventory — `routers.py`

Every router is one dictionary in the `routers_list`:

```python
routers_list = [
    dict(
        address=("sandbox-iosxe-recomm-1.cisco.com", 22),
        usern=os.getenv("USERNXE"),
        passw=os.getenv("PASSWXE"),
        type="IOS-XE",
        asn="65534",
        location="Test1",
        jumpserver=jumpserver1,
    ),
]
```

| Key          | Meaning                                                                |
| ------------ | ---------------------------------------------------------------------- |
| `address`    | tuple of `(hostname or IP, SSH port)`                                  |
| `usern`      | SSH user name, normally read from the environment                       |
| `passw`      | SSH password, normally read from the environment                        |
| `type`       | platform — `IOS-XE`, `IOS-XR`, `NXOS` or `JunOS` — picks the command syntax |
| `asn`        | autonomous system number, used to group the routers in the UI          |
| `location`   | free text label describing the site                                     |
| `jumpserver` | a jump server dictionary, or `False` when it is not needed              |

### 3. The network dropdown — `templates/index.html`

The list of ASNs shown in the web interface is a plain HTML list. Add one line
per network:

```html
<option value="65535">My network - AS65535</option>
```

## Usage

1. Choose the **network** — the router list is fetched from `GET /routers/<asn>`.
2. Choose the **router**.
3. Choose an operation:
   - `bgp`, `ping` or `traceroute` need an **IP / prefix** such as `8.8.8.8` or
     `10.0.0.0/24`;
   - **Custom Command** hides the IP field and lets you type any command.
4. Press **Go !** and watch the output arrive.

## API reference

### `GET /`

Renders the web interface.

### `GET /routers/<asn>`

Lists the routers of one ASN.

```sh
curl http://localhost:5000/routers/65534
```

```json
[
  {
    "name": "sandbox-iosxe-recomm-1.cisco.com",
    "router": "sandbox-iosxe-recomm-1.cisco.com",
    "type": "IOS-XE",
    "cmds": "test cmd"
  }
]
```

### `POST /lg`

Runs a command on a router and streams the output back as plain text.

**Body**

| Field       | Required                              | Description                                    |
| ----------- | ------------------------------------- | ---------------------------------------------- |
| `router`    | yes                                   | host name of the router, as listed by `/routers` |
| `cmd`       | yes                                   | `bgp`, `ping`, `traceroute` or `custom`        |
| `ipprefix`  | for every operation except `custom`   | IP address or CIDR prefix                      |
| `customCmd` | for `custom`                          | the command to run                             |

**Example**

```sh
curl -N -X POST http://localhost:5000/lg \
  -H 'Content-Type: application/json' \
  -d '{"router": "sandbox-iosxe-recomm-1.cisco.com", "cmd": "ping", "ipprefix": "8.8.8.8"}'
```

| Status | Meaning                              |
| ------ | ------------------------------------ |
| `200`  | the output of the command (streamed) |
| `404`  | the router is unknown                |

### `POST /cmds`

Returns a static test payload. It is a placeholder for a future command
catalogue and is not used by the web interface.

## Command reference

This is how the operations are translated per platform — the source of truth is
[`helpers.py`](helpers.py).

| Operation    | IOS-XE / IOS-XR                              | NX-OS                         | JunOS                                                   |
| ------------ | -------------------------------------------- | ----------------------------- | ------------------------------------------------------- |
| `bgp`        | `show bgp ipv4 unicast {ipprefix}`           | `show bgp ipv4 unicast detail` | `show route protocol bgp {ipprefix} table inet.0 detail` |
| `ping`       | `ping {ipprefix} count 5`                    | `ping {ipprefix} count 5`     | `ping {ipprefix} count 5`                               |
| `traceroute` | `traceroute {ipprefix} timeout 1 probe 2`    | `traceroute {ipprefix} timeout 1 probe 2` | `traceroute {ipprefix} wait 1`           |
| `custom`     | the command you type, executed as is         | the command you type, executed as is | the command you type, executed as is  |

## Project structure

```
network-looking-glass/
├── app.py                 # Flask application and the HTTP routes
├── helpers.py             # command templates per platform
├── routers.py             # router inventory and credentials
├── ssh_client.py          # SSH client, streams the output back
├── test_script.py         # example: run commands on many devices at once
├── requirements.txt       # Python dependencies
├── .env                   # credentials for the routers
├── templates/
│   └── index.html         # the web interface
└── static/
    ├── css/material-kit.min.css
    └── js/lg.js           # browser side logic
```

## Troubleshooting

| Symptom                                | What to check                                                                 |
| -------------------------------------- | ----------------------------------------------------------------------------- |
| `Router not found`                     | the host name in the request has to match `address[0]` of the inventory entry   |
| `401` / `Permission denied` from SSH    | the credentials in `.env` and the user name in `routers.py`                    |
| The page loads but the router list is empty | the ASN in the dropdown has to match the `asn` field of a router            |
| The command produces no output         | the router may need a `vty` / `exec` privilege level, or an `enable` password |
| The request hangs                      | the SSH connection has no timeout yet — see [Security notes](#security-notes)   |

## Security notes

This project was written for a **lab environment**. Before you expose it to
anything else, keep the following in mind:

- **There is no authentication.** Anyone who can reach the web interface can
  run commands on every router in the inventory. Keep it on a trusted network,
  put a reverse proxy with authentication in front of it, or tunnel it over
  SSH.
- **The server runs in debug mode and listens on all interfaces**
  (`app.run(debug=True, host="0.0.0.0")`). That is convenient while
  developing and unsafe in production — use a WSGI server such as `gunicorn`
  and turn the debug mode off.
- **Custom commands are executed verbatim** on the router. A custom command can
  change the configuration of a device, so control who is allowed to use the
  application.
- **The `.env` file is part of the repository.** It contains the public DevNet
  sandbox credentials. If you point the inventory at your own gear, replace
  them and keep the result out of your commits.
- **The pinned dependencies are old** (Flask 1.0.2, Paramiko 2.4.1). They work,
  but they no longer get security fixes. Upgrade them before you rely on this
  for anything that matters.

## Known limitations

- The jump host code in `ssh_client.py` is present but commented out, so it
  cannot be used yet.
- The NX-OS `bgp` template ignores the prefix entered in the interface.
- `helpers.get_vars` assumes the requested command exists for the platform of
  the router and raises an exception when it does not.
- The web interface polls the response every 500 ms instead of using a real
  streaming transport.
- The network/ASN dropdown in `templates/index.html` is maintained by hand.

## Contributing

Contributions are welcome — bug reports, new platform templates or a better
interface are all useful.

1. Fork the repository and create a branch for your change.
2. Keep credentials out of your commits.
3. Make sure the application still starts with `python app.py` and describe how
   you tested your change in the pull request.

Have a look at the [open issues](https://github.com/vsachin094/network-looking-glass/issues)
or open a new one to discuss an idea first.

## License

Released under the [MIT License](LICENSE).

## Acknowledgements

- The Cisco DevNet team for the free sandboxes used in the sample inventory.
- Everyone who reported bugs and sent ideas for the project.
