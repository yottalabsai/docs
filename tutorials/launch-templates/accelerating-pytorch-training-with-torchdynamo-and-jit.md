---
hidden: true
---

# Accelerating PyTorch Training with TorchDynamo and JIT

#### :checkered\_flag:What is TorchDynamo?

TorchDynamo is a **dynamic optimization engine** introduced in PyTorch 2.9.0. Think of it as a smart assistant that observes your code at runtime and transforms it into an optimized computation graph.

**What makes it special?**

* It's dynamic - handles Python control flow (if statements, loops, etc.)
* Works great with RNNs, Transformers, and other dynamic models
* No need to change your code structure, it just accelerates what you already have

#### :checkered\_flag:What is the JIT Compiler?

The JIT (Just-In-Time) compiler is like adding a turbo boost to your Python code. It converts Python code into more efficient C++ code, dramatically improving performance.

Using `torch.jit.script` or `torch.jit.trace`, your model transforms into TorchScript, which runs significantly faster.

***

### Step 1: Environment Setup

First, let's get our environment ready. Make sure you're using PyTorch 2.9.0.

```python
# Install PyTorch 2.9.0 using the magic command
%pip install torch==2.9.0 torchvision torchaudio --quiet
```

**Note:** If you're using a GPU, verify your CUDA version is compatible. Check with `nvidia-smi` in your terminal.

***

### Step 2: Import Required Libraries

Let's prepare our toolkit with all the necessary imports.

```python
# Import PyTorch and related libraries
import torch
import torch.nn as nn
import torch.optim as optim
import torch.nn.functional as F
from torch.utils.data import DataLoader
from torchvision import datasets, transforms

# Utility imports
import time
import torch._dynamo as dynamo

print("All libraries imported successfully")
print(f"PyTorch version: {torch.__version__}")
print(f"CUDA available: {torch.cuda.is_available()}")
if torch.cuda.is_available():
    print(f"GPU: {torch.cuda.get_device_name(0)}")
```

***

### Step 3: Define the Neural Network

We'll use a simple but effective fully-connected neural network for our experiments. It's straightforward yet demonstrates the optimization techniques well.

```python
# Define a simple feedforward neural network
class SimpleNN(nn.Module):
    def __init__(self):
        super(SimpleNN, self).__init__()
        # Input layer to hidden layer: 784 pixels -> 128 neurons
        self.fc1 = nn.Linear(28 * 28, 128)
        # Hidden layer to output layer: 128 neurons -> 10 classes
        self.fc2 = nn.Linear(128, 10)

    def forward(self, x):
        # Flatten the image into a 1D vector
        x = x.view(-1, 28 * 28)
        # ReLU activation (classic choice)
        x = F.relu(self.fc1(x))
        # Output layer
        x = self.fc2(x)
        # Log-Softmax for classification
        return F.log_softmax(x, dim=1)

print("Neural network defined")
print("Architecture: 784 -> 128 -> 10")
```

***

### Step 4: Prepare the MNIST Dataset

We'll use the classic MNIST handwritten digits dataset - the "Hello World" of deep learning.

```python
# Data preprocessing: convert to tensor and normalize
transform = transforms.Compose([
    transforms.ToTensor(),
    transforms.Normalize((0.5,), (0.5,))  # Normalize to [-1, 1]
])

print("Downloading MNIST dataset...")
print("(First run will download, subsequent runs use local cache)\n")

# Training set
train_dataset = datasets.MNIST(
    './data', 
    train=True, 
    download=True, 
    transform=transform
)
train_loader = DataLoader(
    train_dataset, 
    batch_size=64, 
    shuffle=True
)

# Test set
test_dataset = datasets.MNIST(
    './data', 
    train=False, 
    download=True, 
    transform=transform
)
test_loader = DataLoader(
    test_dataset, 
    batch_size=64, 
    shuffle=False
)

print(f"Dataset ready")
print(f"Training samples: {len(train_dataset)}")
print(f"Test samples: {len(test_dataset)}")
print(f"Batch size: 64")
```

***

### Step 5: Baseline Test - Original Training Speed

Let's establish our baseline by measuring training speed without any optimizations. This gives us a reference point.

```python
# Initialize model, loss function, and optimizer
model = SimpleNN().cuda() if torch.cuda.is_available() else SimpleNN()
criterion = nn.CrossEntropyLoss()
optimizer = optim.Adam(model.parameters(), lr=0.001)

print("Model loaded to:", "GPU" if torch.cuda.is_available() else "CPU")

# Define training function
def train(model, train_loader, optimizer, criterion):
    model.train()
    for batch_idx, (data, target) in enumerate(train_loader):
        # Move data to GPU (if available)
        if torch.cuda.is_available():
            data, target = data.cuda(), target.cuda()
        
        # Zero gradients
        optimizer.zero_grad()
        # Forward pass
        output = model(data)
        # Calculate loss
        loss = criterion(output, target)
        # Backward pass
        loss.backward()
        # Update parameters
        optimizer.step()

print("\nStarting baseline test (no optimization)...")
print("Training for 5 epochs, please wait...\n")

# Record start time
start_time = time.time()

for epoch in range(5):
    train(model, train_loader, optimizer, criterion)
    print(f"Epoch {epoch+1}/5 completed")

# Record end time
end_time = time.time()
baseline_time = end_time - start_time

print(f"\nBaseline training time: {baseline_time:.2f} seconds")
print("This is our starting speed - let's see how much we can improve")
```

***

### Step 6: JIT Optimization - First Wave of Acceleration

Now, let's apply JIT compiler optimization to convert our model to TorchScript for better performance.

```python
print("Applying JIT optimization...")

# Reinitialize model (for fair comparison)
model = SimpleNN().cuda() if torch.cuda.is_available() else SimpleNN()
optimizer = optim.Adam(model.parameters(), lr=0.001)

# Key step: compile the model using JIT
model_jit = torch.jit.script(model)
print("Model converted to TorchScript")

print("\nStarting JIT-optimized training...")
print("Training for 5 epochs...\n")

# Record start time
start_time = time.time()

for epoch in range(5):
    train(model_jit, train_loader, optimizer, criterion)
    print(f"Epoch {epoch+1}/5 completed (JIT optimized)")

# Record end time
end_time = time.time()
jit_time = end_time - start_time

print(f"\nJIT-optimized training time: {jit_time:.2f} seconds")
print(f"Improvement over baseline: {((baseline_time - jit_time) / baseline_time * 100):.1f}%")
print("Notice the speedup? That's JIT at work")
```

***

### Step 7: TorchDynamo Optimization - Ultimate Acceleration

Let's try TorchDynamo - PyTorch 2.x's killer feature for dynamic optimization.

```python
print("Enabling TorchDynamo optimization...")

# Reinitialize model
model = SimpleNN().cuda() if torch.cuda.is_available() else SimpleNN()
optimizer = optim.Adam(model.parameters(), lr=0.001)

# Enable TorchDynamo with the inductor backend (strongest optimization)
dynamo.config.verbose = False  # Keep output clean

# Use compile for optimization (PyTorch 2.0+ API)
model_dynamo = torch.compile(model, backend="inductor")
print("TorchDynamo optimization enabled")

print("\nStarting TorchDynamo-optimized training...")
print("Training for 5 epochs...\n")
print("Note: First epoch may be slower (compilation overhead), then it gets fast")

# Record start time
start_time = time.time()

for epoch in range(5):
    train(model_dynamo, train_loader, optimizer, criterion)
    print(f"Epoch {epoch+1}/5 completed (TorchDynamo optimized)")

# Record end time
end_time = time.time()
dynamo_time = end_time - start_time

print(f"\nTorchDynamo-optimized training time: {dynamo_time:.2f} seconds")
print(f"Improvement over baseline: {((baseline_time - dynamo_time) / baseline_time * 100):.1f}%")
print(f"Improvement over JIT: {((jit_time - dynamo_time) / jit_time * 100):.1f}%")
print("This is the power of dynamic optimization")
```

***

### Step 8: Performance Comparison - Let the Data Speak

Let's visualize the optimization results with a clear comparison chart.

```python
import matplotlib.pyplot as plt
import numpy as np

print("Generating performance comparison chart...\n")

# Prepare data
methods = ['Baseline', 'JIT Optimization', 'TorchDynamo Optimization']
times = [baseline_time, jit_time, dynamo_time]
colors = ['#ff6b6b', '#4ecdc4', '#45b7d1']

# Create bar chart
plt.figure(figsize=(10, 6))
bars = plt.bar(methods, times, color=colors, alpha=0.8, edgecolor='black', linewidth=1.5)

# Add value labels on bars
for bar, t in zip(bars, times):
    height = bar.get_height()
    plt.text(bar.get_x() + bar.get_width()/2., height,
             f'{t:.2f}s',
             ha='center', va='bottom', fontsize=12, fontweight='bold')

# Beautify the chart
plt.ylabel('Training Time (seconds)', fontsize=12, fontweight='bold')
plt.title('PyTorch Training Optimization Results - 5 Epochs', fontsize=14, fontweight='bold')
plt.grid(axis='y', alpha=0.3, linestyle='--')
plt.tight_layout()

# Display chart
plt.show()

# Print detailed comparison
print("=" * 60)
print("Performance Comparison Report")
print("=" * 60)
print(f"Baseline:              {baseline_time:.2f} seconds  [reference]")
print(f"JIT Optimization:      {jit_time:.2f} seconds  [speedup: {((baseline_time - jit_time) / baseline_time * 100):.1f}%]")
print(f"TorchDynamo:           {dynamo_time:.2f} seconds  [speedup: {((baseline_time - dynamo_time) / baseline_time * 100):.1f}%]")
print("=" * 60)

# Calculate best method
fastest = min(times)
fastest_method = methods[times.index(fastest)]
speedup = baseline_time / fastest

print(f"\nWinner: {fastest_method}")
print(f"Total speedup: {speedup:.2f}x")
print(f"Time saved: {baseline_time - fastest:.2f} seconds")
print(f"\nIf you were training 100 epochs...")
print(f"   Time saved: {(baseline_time - fastest) * 20:.2f} seconds ≈ {(baseline_time - fastest) * 20 / 60:.1f} minutes")
```

***

### Step 9: Verify Model Accuracy

Optimization is important, but accuracy matters too. Let's test our model's performance.

```python
def test(model, test_loader):
    model.eval()
    correct = 0
    total = 0
    
    with torch.no_grad():
        for data, target in test_loader:
            if torch.cuda.is_available():
                data, target = data.cuda(), target.cuda()
            
            output = model(data)
            pred = output.argmax(dim=1, keepdim=True)
            correct += pred.eq(target.view_as(pred)).sum().item()
            total += target.size(0)
    
    accuracy = 100. * correct / total
    return accuracy

print("Testing model accuracy...\n")

# Test the optimized model
accuracy = test(model_dynamo, test_loader)

print(f"Test accuracy: {accuracy:.2f}%")
print(f"Correct predictions: {int(accuracy * len(test_dataset) / 100)}/{len(test_dataset)}")

if accuracy > 95:
    print("Excellent - accuracy above 95%")
elif accuracy > 90:
    print("Good - accuracy above 90%")
else:
    print("Room for improvement - try training more epochs")
```

***

### Step 10: Practical Tips and Best Practices

Let me share some insights from real-world projects.

```python
print("PyTorch Optimization Tips")
print("=" * 60)
print()
print("1. Choosing the Right Optimization:")
print("   • Small models → JIT is sufficient")
print("   • Large/complex models → TorchDynamo is stronger")
print("   • Production → combine both for best results")
print()
print("2. Important Notes:")
print("   • TorchDynamo first run includes compilation (slower)")
print("   • Ensure PyTorch version >= 2.0")
print("   • GPU training shows more dramatic improvements")
print()
print("3. Advanced Optimizations:")
print("   • Use mixed precision training (torch.cuda.amp)")
print("   • Enable cudnn.benchmark (torch.backends.cudnn.benchmark = True)")
print("   • Set appropriate num_workers for data loading")
print()
print("4. Debugging Tips:")
print("   • If issues arise, disable optimization first")
print("   • Use torch._dynamo.explain() to see optimization details")
print("   • Check torch._dynamo.config settings")
print("=" * 60)
```
