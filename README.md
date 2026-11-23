# AI Image Generator (Stable Diffusion): Open OnDemand App

An [Open OnDemand](https://openondemand.org/) Batch Connect app for the Tufts HPC
cluster (`pax`). It runs the
[AUTOMATIC1111 Stable Diffusion Web UI](https://github.com/AUTOMATIC1111/stable-diffusion-webui)
from a Singularity container on a GPU compute node. Users reach it from the
OnDemand dashboard under **Interactive Apps → AI**.

## How it works

1. The user fills out the form (`form.yml.erb`, `form.js`) and submits it.
2. `submit.yml.erb` turns the form values into a Slurm job: cores, memory,
   partition, GPU type (`--gres`/`--constraint`) and, if requested, `--qos=interactive`.
3. On the compute node:
   - `template/before.sh.erb` picks a free port and generates a random password.
   - `template/script.sh.erb` loads `singularity/4.3.4` and creates the output
     directory layout. It also writes a `ui-config.json` that sends generated
     images into the user's output directory, and launches the Web UI from
     `/cluster/tufts/apps/container/ngc/images/stable-diffusion-webui_<version>.sif`.
     It then waits up to 20 minutes for the server to answer before it declares the
     session ready.
   - `template/after.sh.erb` waits briefly so users don't connect before the
     server is up.
4. `view.html.erb` renders the session card with a **Connect to Diffusion**
   button and the login credentials. The username is `$USER` and the password
   is the one generated for that session.

## Form fields

| Field | Description |
|---|---|
| `bc_num_hours`, `num_cores`, `num_memory`, `partition`, `gpu_type` | Resource fields. They come from the shared partial `/etc/ood/config/apps/dashboard/batch_connect/partials/ai_tools.yml`. |
| `version` | Container version (currently `1.10.1`). |
| `outdir` | Where outputs, caches, repositories, temp files and config states are written. |
| `modelroot` | Root model directory. Defaults to the shared collection at `/cluster/tufts/hpc/tools/stable-diffusion-webui/models`. It should contain the standard subfolders (`Stable-diffusion/`, `ControlNet/`, `Lora/`, `ESRGAN/`, …). |
| `ta_qos` | "Request priority start": submits with `--qos=interactive`. |
| `bc_email_on_started` | Email the user when the session starts. |

The disclaimer block at the top of the form is read from
`/etc/ood/config/apps/dashboard/batch_connect/partials/ai_disclaimer.md` when
that file exists.

### Interactive QOS caps (`form.js`)

When **Request priority start** is checked, `form.js` caps the form to the
interactive QOS limits and shows a notice:

- walltime: 4 hours
- CPU cores: 16
- memory: 64 GB
- GPUs: 1

When the box is unchecked, the partition-based limits from the resource-discovery
handlers come back. To change these limits, edit `QOS_LIMITS` and the warning
text in `form.js`.

## Output directory layout

Inside the selected `outdir`:

```
outdir/
├── outputs/
│   ├── txt2img-images/   txt2img-grids/
│   ├── img2img-images/   img2img-grids/
│   ├── extras-images/    init-images/
├── log/images/
├── cache/                         # HF / torch caches
├── repositories/
├── tmp/
├── config_states/
├── controlnet-annotator-downloads/
└── ui-config.json                 # regenerated on every launch
```

## Repository layout

| File | Purpose |
|---|---|
| `manifest.yml` | App name, category and description shown in the dashboard |
| `form.yml.erb` | Form definition |
| `form.js` | Client-side interactive QOS caps |
| `submit.yml.erb` | Slurm submission options |
| `template/before.sh.erb` | Picks the port and password |
| `template/script.sh.erb` | Main job script: sets up directories and launches the Web UI |
| `template/after.sh.erb` | Waits after launch |
| `template/config.json` | Reference Web UI settings file inherited from upstream (the job script writes its own `ui-config.json`) |
| `view.html.erb` | Session card with the connect button |
| `icon.png` | Dashboard icon |

## Development

To test changes, deploy the repo to your OnDemand sandbox apps directory
(`~/ondemand/dev/<app>`), open **Develop → My Sandbox Apps** in the dashboard,
and launch a session. Job output (`output.log`) is in the session's staged
directory under `~/ondemand/data/sys/dashboard/batch_connect/`.

## License

MIT. See [LICENSE.txt](LICENSE.txt). Originally based on work by the Ohio
Supercomputer Center.
