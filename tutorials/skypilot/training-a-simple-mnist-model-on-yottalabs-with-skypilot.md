# Training a simple MNIST model on YottaLabs with SkyPilot

In this guide, we will train a small neural network on the MNIST handwritten digit dataset. The model will take in images of digits from 0 to 9 and learn to classify them. The point is still to keep the workflow approachable, but this time the job is genuinely learning from real data instead of just importing torch and other libraries.

#### Step 1 — Create working directory and training script

```bash
mkdir ./test
```

{% hint style="info" %}
Use your own workspace and change `train_mnist.yaml` accordingly.
{% endhint %}

Start by creating a file named `train_mnist.py`. This script downloads MNIST, builds a small classifier, trains for a few epochs, evaluates on the test set, and saves a checkpoint at the end.

```python
import os
import torch
import torch.nn as nn
import torch.optim as optim
from torchvision import datasets, transforms
from torch.utils.data import DataLoader

device = "cuda" if torch.cuda.is_available() else "cpu"
print(f"Using device: {device}")

batch_size = 128
epochs = 3
learning_rate = 1e-3

transform = transforms.Compose([
    transforms.ToTensor(),
    transforms.Normalize((0.1307,), (0.3081,))
])

train_dataset = datasets.MNIST(
    root="./data",
    train=True,
    download=True,
    transform=transform,
)
test_dataset = datasets.MNIST(
    root="./data",
    train=False,
    download=True,
    transform=transform,
)

train_loader = DataLoader(train_dataset, batch_size=batch_size, shuffle=True, num_workers=2)
test_loader = DataLoader(test_dataset, batch_size=batch_size, shuffle=False, num_workers=2)

model = nn.Sequential(
    nn.Flatten(),
    nn.Linear(28 * 28, 256),
    nn.ReLU(),
    nn.Linear(256, 128),
    nn.ReLU(),
    nn.Linear(128, 10),
).to(device)

criterion = nn.CrossEntropyLoss()
optimizer = optim.Adam(model.parameters(), lr=learning_rate)

for epoch in range(epochs):
    model.train()
    running_loss = 0.0

    for images, labels in train_loader:
        images = images.to(device)
        labels = labels.to(device)

        optimizer.zero_grad()
        outputs = model(images)
        loss = criterion(outputs, labels)
        loss.backward()
        optimizer.step()

        running_loss += loss.item() * images.size(0)

    avg_loss = running_loss / len(train_loader.dataset)

    model.eval()
    correct = 0
    total = 0

    with torch.no_grad():
        for images, labels in test_loader:
            images = images.to(device)
            labels = labels.to(device)

            outputs = model(images)
            predictions = outputs.argmax(dim=1)

            total += labels.size(0)
            correct += (predictions == labels).sum().item()

    accuracy = 100.0 * correct / total
    print(f"Epoch {epoch + 1}/{epochs} - loss: {avg_loss:.4f} - test accuracy: {accuracy:.2f}%")

os.makedirs("outputs", exist_ok=True)
torch.save(model.state_dict(), "outputs/mnist_model.pt")
print("Saved checkpoint to outputs/mnist_model.pt")
```

This script is still intentionally simple, but now it is doing something meaningful. It downloads a real dataset, trains on labeled examples, evaluates on held-out test data, and saves the trained weights to disk. Because MNIST is small, it should finish quickly while still producing a recognizable training curve.

#### Step 2 — Define the SkyPilot task

Next, create a file named `train_mnist.yaml`. Since your cluster reports `RTX5090` as available, we should request that explicitly instead of using a hardcoded accelerator from another environment.

```yaml
workdir: ./test

resources:
  accelerators: RTX5090:1

setup: |
  pip install torch torchvision

run: |
  python train_mnist.py
```

This YAML keeps the structure familiar. We ask for one `RTX5090`, install the libraries we need, and then run the training script. The nice thing here is that the workflow has not changed at all. Only the workload became more realistic.

#### Step 3 — Launch the training job

Now launch the job with a cluster name of your choice.

```bash
sky launch -c mnist-demo train_mnist.yaml
```

SkyPilot will provision the node, install PyTorch and torchvision, download the dataset, and run the training script. On the first run, the dataset download may add a little extra time, which is expected.

Once the job is active, stream the logs so you can watch the training progress.

```bash
sky logs mnist-demo
```

You should see output that looks something like this:

<figure><img src="../../.gitbook/assets/image (11).png" alt="" width="563"><figcaption></figcaption></figure>

The exact numbers will vary, but the important pattern is that the loss should go down and the test accuracy should rise to a high value.&#x20;

#### Step 4 — Understand what this job is doing

At a practical level, this job is doing what most people mean when they say “training a model.” It is reading a real dataset of images, converting those images into tensors, passing them through a neural network, measuring the prediction error, computing gradients, and updating the model so it improves over time.

MNIST is simple enough that even a small fully connected network can learn it well. That makes it a good teaching example because the code remains easy to read while the model still reaches strong test accuracy. In other words, it feels like a real training run without introducing unnecessary complexity too early.

#### Step 5 — Save and inspect the result

The script saves the trained model to:

```
outputs/mnist_model.pt
```

That gives the tutorial a more complete ending. Instead of only printing metrics, the job also produces an artifact that you could later reuse for evaluation, inference, or checkpoint handling in a larger workflow.

If you later want to extend this, a natural next step would be to save logs and outputs to a persistent storage location, but for now the saved checkpoint is enough to prove the end-to-end path is working.

#### Step 6 — Clean up the cluster

When the run is complete, shut down the resources:

```bash
sky down mnist-demo
```

This keeps the environment clean and avoids leaving the training cluster running unnecessarily.
