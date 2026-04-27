---
description: Run your first GPU job
---

# Getting started with YottaLabs on SkyPilot

The easiest way to understand a new compute platform is to run something real on it. Once you see a workload go from your local machine to a remote GPU and back with results, the whole system starts to make sense.&#x20;

#### Step 1 — Install SkyPilot

We’ll start by installing SkyPilot, which provides the CLI used to launch and manage workloads. This is the main interface you’ll interact with throughout the tutorial.

```bash
pip install skypilot
```

Once the installation completes, it’s worth quickly confirming that everything is available:

```bash
sky --help
```

If you see a list of commands, you’re ready to move forward.

***

#### Step 2 — Configure your YottaLabs credentials

To configure Yotta access: Go to the [Yotta console](https://yottalabs.ai), retrieve your **Org ID** from`Launch a Console -> Settings -> Organization` and generate an **API Key** from `Launch a Console -> Settings -> Access` .Then, create the credentials file:

```bash
mkdir -p ~/.yotta
vim ~/.yotta/credentials
```

Add the following content:

```
orgId=<your_org_id>
apikey=<your_api_key>
```

***

#### Step 3 — Verify your setup

At this point, it’s a good idea to check that everything is correctly configured before launching a job. SkyPilot provides a built-in verification command for this purpose.

```bash
sky check
```

This command inspects your environment and reports which providers are available. You should see YottaLabs listed as ready. If something is missing, the output will usually include a helpful hint about what needs to be fixed.

***

#### Step 4 — Define your first workload

Now we’ll create a minimal workload. Instead of writing a full script, we’ll use a YAML file that describes the resources, setup steps, and the command to run.

Create a file named `hello_gpu.yaml`:

```yaml
resources:
  accelerators: A100:1

setup: |
  pip install torch

run: |
  python -c "import torch; print(torch.cuda.is_available())"
```

This configuration does three things in sequence. It requests a machine with one A100 GPU, installs PyTorch, and then runs a short Python command that checks whether the GPU is accessible. If everything is working correctly, the output will be `True`.

***

#### Step 5 — Launch the job

With the workload defined, we can now launch it. This is where SkyPilot starts interacting with YottaLabs behind the scenes.

```bash
sky launch -c test-yl hello_gpu.yaml
```

When you run this command, SkyPilot provisions a machine on YottaLabs, connects to it, executes the setup commands, and finally runs your task. The first run may take a little longer because it includes environment setup, but the process is fully automated.

As the job runs, you don’t need to manually connect to anything. SkyPilot manages the entire lifecycle for you.

You'll see something like:

<figure><img src="../../.gitbook/assets/image (6) (2).png" alt=""><figcaption></figcaption></figure>

***

#### Step 6 — Inspect the logs

Once the job is running, you can follow its progress by streaming the logs:

```bash
sky logs test-yl
```

This gives you visibility into each stage, including dependency installation and the final execution. When the job completes, you should see the output from your Python command confirming whether CUDA is available.

Watching the logs is especially helpful when you start running more complex workloads, as it lets you debug issues without leaving the CLI.

***

#### Step 7 — Clean up resources

After the job finishes, it’s important to shut down the resources you created. This ensures you’re not leaving anything running unintentionally.

```bash
sky down test-yl
```

This command tears down the cluster and releases the GPU instance back to YottaLabs.

***

#### Wrapping up

At this point, you’ve completed a full end-to-end workflow. You defined a workload, ran it on a remote GPU through YottaLabs, monitored its execution, and cleaned up the resources afterward. Even though the example was simple, the same pattern applies to much more complex tasks.

In the next tutorial, we’ll build directly on this foundation. Instead of running on a single GPU type, we’ll take a similar workload and run it across different hardware backends, showing how you can move between accelerators without rewriting your code.
