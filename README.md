# SD AutoScoring — tool + hosted firmware (GitHub Pages)

The whole tool is one file (`index.html`, esptool-js baked in). Firmware builds
live in `firmware/` and are listed in `firmware/firmware.json`; the flasher shows
them in a dropdown, so the operator just picks a build and flashes — master or node,
same tool.

```
/
├── index.html                 the tool (served at the Pages root)
├── firmware/
│   ├── firmware.json          the build list (edit this)
│   ├── master/                master .bin files
│   └── node/                  node .bin files
└── README.md
```

## 1. Host it on GitHub Pages

Push this folder to a repo, then **Settings → Pages → Build and deployment →
Source: Deploy from a branch → `main` / `/ (root)`**. It goes live at
`https://<you>.github.io/<repo>/`. No build step or Action needed — it's static.

Open it in **Chrome or Edge on desktop** (Web Serial needs a secure context, which
Pages provides over HTTPS).

> **Privacy:** a public Pages site serves `firmware/` to anyone with the URL. If the
> binaries are proprietary, host this on a private/internal HTTPS server instead —
> the tool behaves the same.

## 2. Add / update a firmware

1. Drop the `.bin` file(s) into `firmware/master/` or `firmware/node/`.
2. Add (or edit) an entry in `firmware/firmware.json` and bump `version`.
3. Commit & push. In the tool, open **Flash** and hit **↻** to reload the list.

### `firmware.json` schema

```json
{
  "firmwares": [
    {
      "name": "AutoScoring Master",     // shown in the dropdown
      "target": "master",               // label only (master | node)
      "version": "1.0",                 // shown as "v1.0"
      "chip": "ESP32-S3",               // checked against the connected board
      "files": [
        { "path": "master/bootloader.bin",      "offset": "0x0"    },
        { "path": "master/partition-table.bin", "offset": "0x8000" },
        { "path": "master/AutoscoreMaster.bin", "offset": "0x10000" }
      ]
    },
    {
      "name": "Scoring Node", "target": "node", "version": "1.0", "chip": "ESP32-C3",
      "files": [ { "path": "node/node-merged.bin", "offset": "0x0" } ]
    }
  ]
}
```

- **Two layouts per entry:** the three build files at `0x0 / 0x8000 / 0x10000`, **or**
  a single **merged** image at `0x0` (`idf.py merge-bin`). The entry's `files` list
  decides — the tool just writes each file at its offset.
- **`chip`** is matched against the connected board, so a master build can't be
  flashed onto a node (and vice-versa) when their chips differ. Set it correctly.
- `path` is relative to the `firmware/` folder.

### Update check

On connect, the tool reads the device's firmware version (master via
`GET_FW_VERSION`, node via the `fw=` field in `GET STATUS`) and compares it to the
`version` of the matching `target` entry in `firmware.json`. If they differ, it
shows an "update available" prompt and pre-selects that build in the Flash sheet.

So the `version` you put in `firmware.json` must match what the device reports
(the master reports its `PROJECT_VER`, e.g. `"1.0"`; set the node entry's version
to match whatever the node firmware puts in `fw=`). Comparison ignores a leading
`v` and case.

## 3. Flash a board

1. Open the tool → **Flash** (top of the Lane screen).
2. Pick the firmware from the dropdown.
3. Plug the board in over USB, press **Flash**, choose the COM port, wait for green.

The **WCH / CH343 reset fix** is on by default; **Erase whole chip first** is for a
fresh board (wipes NVS/stored config).

---

Notes: Chrome / Edge on desktop only (Web Serial). Node provisioning and scoring are
unchanged — this only adds hosted-firmware flashing. The master speaks the binary
HostLink protocol; nodes still use their text provisioning protocol.
