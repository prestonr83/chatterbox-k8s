# chatterbox-k8s

[Chatterbox TTS](https://github.com/resemble-ai/chatterbox) on k3s, using the GPU on `k3s-server-01` (RTX 3080).
It runs [devnen/Chatterbox-TTS-Server](https://github.com/devnen/Chatterbox-TTS-Server), which adds a web UI with
voice cloning, an OpenAI-compatible `/v1/audio/speech` endpoint for Home Assistant, and all three models
(Original, Turbo, Multilingual) that you can switch between from the UI.

GitHub Actions builds the image. Nothing is built on the node.

```
.github/workflows/build-image.yml   builds upstream's NVIDIA Dockerfile and pushes it to ghcr.io
UPSTREAM_REF                        the upstream branch/tag/commit to build
k8s/                                namespace, PVCs, Deployment, Service, optional IngressRoute
```

## Pipeline

- **Triggers:** a push to `main` that changes `UPSTREAM_REF` or the workflow, or a manual **Actions → Build image →
  Run workflow** (where you can optionally type an upstream ref).
- **Pushes** `ghcr.io/<owner>/chatterbox-tts-server:latest` and `:<upstream short sha>`.
  The run summary prints the exact tags.
- **To upgrade upstream:** put a newer tag or commit in `UPSTREAM_REF` and push.

## Deploy

This assumes your cluster already runs GPU workloads: the `nvidia` RuntimeClass exists and the node advertises
`nvidia.com/gpu`. If your GPU pods use `nvidia` as the default containerd runtime instead, delete the
`runtimeClassName` line in `k8s/chatterbox.yaml`.

```
kubectl apply -k k8s/
kubectl -n tts logs -f deploy/chatterbox        # the first start downloads model weights, which takes a few minutes
kubectl -n tts exec deploy/chatterbox -- nvidia-smi   # should show a python3 process
```

**Pin the version:** set `newTag` in `k8s/kustomization.yaml` to the sha tag from the run summary, then
`kubectl apply -k k8s/`. That way a rebuild of `latest` never changes what's running without you knowing.

## Web UI

```
kubectl -n tts port-forward svc/chatterbox 8004:8004
```
Then open http://localhost:8004 (API docs are at `/docs`). For `https://tts.rdz.sh`, uncomment `ingressroute.yaml` in
the kustomization after creating the DNS record and the `tts-rdz-sh-tls` secret.

The server has **no authentication**. Keep it LAN-only.

## Voice cloning

1. Record 10–20 s of clean speech from one speaker (WAV or MP3, no music or noise). Clips over 30 s are rejected.
2. In the UI, choose **Voice Cloning** mode, upload the clip and generate. Set a seed for consistent results.
3. Tuning: `exaggeration` (0.5 is neutral; higher is more dramatic) and `cfg_weight` (~0.3 for fast or expressive speakers).
   Turbo understands tags such as `[laugh]` and `[chuckle]`.
4. To use a cloned voice from Home Assistant, it must appear in `GET /v1/audio/voices`. If it doesn't,
   upload it as a predefined voice instead (`/upload_predefined_voice`).

Only clone voices you have permission to use.

Uploads, settings (`config.yaml`) and model weights live on PVCs, so they survive restarts and image upgrades.

## Home Assistant

1. HACS → **OpenAI TTS** (sfortis/openai_tts) → restart HA.
2. Add it with the **Chatterbox** preset or a custom endpoint at `http://chatterbox.tts.svc.cluster.local:8004`, with no API key.
3. Pick a voice from `/v1/audio/voices`, then choose this TTS entity for your assistant under Settings → Voice assistants.

Turbo has the lowest latency on the 3080 and suits voice-assistant replies best.
