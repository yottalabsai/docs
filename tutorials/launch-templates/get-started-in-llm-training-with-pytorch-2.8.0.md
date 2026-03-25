# Get Started in LLM Training with Pytorch 2.8.0

:shushing\_face:**Before we get started:**

From the jupyter notebook, type:

```
!python
```

then enter the following code:

```
import torch
x = torch.rand(5, 3)
print(x)
```

The output should be something similar to:

```
tensor([[0.3380, 0.3845, 0.3217],
        [0.8337, 0.9050, 0.2650],
        [0.2979, 0.7141, 0.9069],
        [0.1449, 0.1132, 0.1375],
        [0.4675, 0.3947, 0.1426]])
```

### **Let’s Get Started!**

This tutorial is going to guide you through a complete MNIST handwritten digit recognition project using PyTorch, from data loading to model training and testing:hugging:

***

### 1. Environment Setup

First, ensure you have the necessary Python packages installed:

```python
%pip install torch torchvision matplotlib numpy
```

### 2. Import Libraries

```python
import torch
import torch.nn as nn
import torch.optim as optim
import torchvision
import torchvision.transforms as transforms
import matplotlib.pyplot as plt
import numpy as np
```

### 3. Data Loading and Preprocessing

The MNIST dataset contains 60,000 training images and 10,000 test images. Each image is a 28x28 pixel grayscale handwritten digit (0-9).

So, for any starters, we should first clarify the concept of "training set" and "test set". Usually, we divide a classification dataset into 2 basic subparts: a training set to help our machine learn the hidden patterns, and a test set to examine if it actually "learns" instead of simply memorizing the data- for which the LLM researchers have a fancy name called "overfitting".

In our MNIST project:

🔧**Training set**: 60,000 handwritten digit images - the model learns from these

📝**Test set**: 10,000 handwritten digit images - the model has never seen these before

The golden rule: **Never let your model peek at the test set during training!** Otherwise, you're essentially letting a student see the exam questions while studying - the grades won't reflect true understanding.

The MNIST dataset contains 60,000 training images and 10,000 test images. Each image is a 28x28 pixel grayscale handwritten digit (0-9).

![img](../../.gitbook/assets/img)

```python
# Set batch size
BATCH_SIZE = 32
​
# Data transformation: convert images to tensors
transform = transforms.Compose([
    transforms.ToTensor()  # Convert PIL image or numpy array to tensor and normalize to [0,1]
])
```

**What's happening here?** The `transforms.ToTensor()` does two things:

1️⃣Translates the image from a PIL Image or NumPy array into a PyTorch tensor (the format-or "language"- PyTorch understands).

2️⃣Automatically scales pixel values from \[0, 255] to \[0, 1] - this normalization helps the model train more effectively

```python
# Download and load training dataset
trainset = torchvision.datasets.MNIST(
    root='./data',           # Data storage path
    train=True,              # Load training set
    download=True,           # Download if data doesn't exist
    transform=transform      # Apply the transformation defined above
)
​
trainloader = torch.utils.data.DataLoader(
    trainset, 
    batch_size=BATCH_SIZE,   # 32 samples per batch
    shuffle=True,            # Shuffle the data
    num_workers=2            # Use 2 subprocesses for data loading
)
​
# Download and load test dataset
testset = torchvision.datasets.MNIST(
    root='./data', 
    train=False,             # Load test set
    download=True, 
    transform=transform
)
​
testloader = torch.utils.data.DataLoader(
    testset, 
    batch_size=BATCH_SIZE,
    shuffle=False,           # No need to shuffle test set
    num_workers=2
)
​
print(f"Training set size: {len(trainset)}")
print(f"Test set size: {len(testset)}")
```

✔️**Why do we use DataLoader?**

Instead of feeding all 60,000 images at once (which would overwhelm your computer's memory), we use batches!A batch is just a small group of images processed together. Here we use batches of 32 images - think of it as studying 32 flashcards at a time instead of trying to memorize all 60,000 at once.

❓Also, think about this question: Why shuffle=True for training but False for testing?

> Shuffling the training data prevents the model from learning the order of examples rather than the actual patterns. For testing, order doesn't matter since we're just evaluating - no learning happens.

### 4. Data Exploration

Before training, it's important to understand the structure and content of the data. As the old saying goes: "garbage in, garbage out" - understanding your data is crucial for successful machine learning.

```python
# Visualize some training samples
def show_sample_images():
    # Get one batch of data
    dataiter = iter(trainloader)
    images, labels = next(dataiter)
    
    # Create figure
    fig, axes = plt.subplots(2, 5, figsize=(12, 6))
    axes = axes.ravel()
    
    # Display first 10 images
    for i in range(10):
        axes[i].imshow(images[i].squeeze(), cmap='gray')
        axes[i].set_title(f'Label: {labels[i].item()}')
        axes[i].axis('off')
    
    plt.tight_layout()
    plt.show()
​
show_sample_images()
```

This visualization helps you catch potential issues early - are the images rotated correctly? Are they actually digits? Is the quality good enough?

```python
# Check data dimensions
dataiter = iter(trainloader)
images, labels = next(dataiter)
​
print(f"Image batch dimensions: {images.shape}")  # [batch_size, channels, height, width]
print(f"Label batch dimensions: {labels.shape}")  # [batch_size]
```

**Expected Output:**

```python
Image batch dimensions: torch.Size([32, 1, 28, 28])
Label batch dimensions: torch.Size([32])
```

Wow, wait!

Are these numbers and brackets making your head spin?

Here is explanation:

1️⃣`32`: Batch size - we're processing 32 images at once

2️⃣`1`: Number of channels - grayscale images have 1 channel (RGB images would have 3)

3️⃣`28, 28`: Height and width of each image in pixels

4️⃣Labels are just a 1D array of 32 numbers, each indicating which digit (0-9) the corresponding image represents

### 5. Build CNN Model

Now comes the fun part - building our neural network! We'll use a Convolutional Neural Network (CNN), which is particularly good at recognizing visual patterns.

**Why CNN for images?** Traditional neural networks treat each pixel independently, but CNNs are smart enough to recognize that nearby pixels form patterns (like edges, curves, and eventually whole digits). It's like how you recognize a face by seeing eyes, nose, and mouth in specific spatial arrangements, not just as random dots.

python

```python
class SimpleCNN(nn.Module):
    def __init__(self):
        super(SimpleCNN, self).__init__()
        
        # Convolutional layer: input 1 channel, output 32 channels, kernel size 3x3
        self.conv = nn.Conv2d(in_channels=1, out_channels=32, kernel_size=3)
        
        # First fully connected layer
        # Input dimension: 26*26*32 (feature map size after convolution)
        # Output dimension: 128
        self.d1 = nn.Linear(26 * 26 * 32, 128)
        
        # Second fully connected layer
        # Input dimension: 128
        # Output dimension: 10 (corresponding to 10 digit classes: 0-9)
        self.d2 = nn.Linear(128, 10)
    
    def forward(self, x):
        # Convolutional layer + ReLU activation
        x = self.conv(x)
        x = torch.relu(x)
        
        # Flatten feature maps: from [batch, 32, 26, 26] to [batch, 26*26*32]
        x = x.view(-1, 26 * 26 * 32)
        
        # First fully connected layer + ReLU
        x = self.d1(x)
        x = torch.relu(x)
        
        # Second fully connected layer
        x = self.d2(x)
        
        # Softmax to get probability distribution
        x = torch.softmax(x, dim=1)
        
        return x
​
# Create model instance and move to device
model = SimpleCNN().to(device)
print(model)
```

**Let's break down what each component does:**

1. **Convolutional Layer (`self.conv`)**: This is like a pattern detector. It slides a 3x3 window across the image, looking for 32 different patterns (edges, curves, corners, etc.). Each of these 32 "filters" learns to detect a different feature.
2. **ReLU Activation**: Stands for "Rectified Linear Unit" - it's a simple function that helps the network learn non-linear patterns. Without it, the network could only learn straight-line relationships, which isn't useful for complex images.
3. **Flatten (`view`)**: After convolution, we have a 3D structure (32 feature maps of size 26x26). We need to flatten this into a 1D array before feeding it to regular neural network layers.
4. **Fully Connected Layers (`self.d1`, `self.d2`)**: These layers combine all the patterns detected by the convolutional layer to make the final decision about which digit it is.
5. **Softmax**: Converts the final layer's outputs into probabilities that sum to 1. For example: \[0.1, 0.05, 0.7, 0.05, ...] means 70% confidence it's a "2", 10% confidence it's a "0", etc.

**Where does 26×26 come from?**

The convolutional layer shrinks the image slightly. Here's the formula:

```python
H_out = (H_in - kernel_size) / stride + 1
```

In our case:

* Input: 28 × 28
* Kernel: 3 × 3
* Stride (default): 1
* Padding (default): 0

Calculation: `H_out = (28 - 3) / 1 + 1 = 26`

So each of our 32 feature maps is 26 × 26 pixels, giving us 26 × 26 × 32 = 21,632 features to feed into the first fully connected layer.

```python
# Test model forward pass
dataiter = iter(trainloader)
images, labels = next(dataiter)
​
print(f"Input batch size: {images.shape}")
​
# Move data to device and pass through model
images = images.to(device)
output = model(images)
​
print(f"Output dimensions: {output.shape}")  # Should be [32, 10]
```

**Expected output:** `torch.Size([32, 10])` - for each of the 32 images in the batch, we get 10 probabilities (one for each digit 0-9).

### 7. Train the Model

Training is where the magic happens - this is where the model actually learns from the data!

```python
# Define loss function
criterion = nn.CrossEntropyLoss()
​
# Define optimizer (Adam)
optimizer = optim.Adam(model.parameters(), lr=0.001)
​
# Function to calculate accuracy
def get_accuracy(logit, target, batch_size):
    """Calculate accuracy for the current batch"""
    corrects = (torch.max(logit, 1)[1].view(target.size()).data == target.data).sum()
    accuracy = 100.0 * corrects / batch_size
    return accuracy.item()
```

**What are these components?**

* **Loss Function (CrossEntropyLoss)**: Measures how wrong the model's predictions are. Lower loss = better predictions. It's like a grading system that tells the model how badly it messed up.
* **Optimizer (Adam)**: Decides how to adjust the model's weights to reduce the loss. Think of it as a GPS that guides the model toward better performance. Adam is popular because it adapts the learning speed automatically - taking bigger steps when far from the goal, smaller steps when getting close.
* **Learning Rate (lr=0.001)**: Controls how big each adjustment step is. Too large, and the model overshoots the optimal solution; too small, and training takes forever.

```python
# Training loop
def train_model(epochs=5):
    for epoch in range(epochs):
        train_running_loss = 0.0
        train_acc = 0.0
        
        # Set model to training mode
        model.train()
        
        # Iterate through training data
        for i, (images, labels) in enumerate(trainloader):
            # Move data to device
            images = images.to(device)
            labels = labels.to(device)
            
            # Forward pass
            outputs = model(images)
            loss = criterion(outputs, labels)
            
            # Backward pass and optimization
            optimizer.zero_grad()  # Zero the gradients
            loss.backward()        # Backward propagation
            optimizer.step()       # Update parameters
            
            # Accumulate loss and accuracy
            train_running_loss += loss.item()
            train_acc += get_accuracy(outputs, labels, BATCH_SIZE)
        
        # Calculate averages
        avg_loss = train_running_loss / len(trainloader)
        avg_acc = train_acc / len(trainloader)
        
        print(f'Epoch: {epoch} | Loss: {avg_loss:.4f} | Train Accuracy: {avg_acc:.2f}%')
​
# Start training
train_model(epochs=5)
```

**Understanding the training loop:**

An **epoch** is one complete pass through the entire training dataset. We typically need multiple epochs because the model learns gradually - like reading a textbook multiple times to fully understand it.

For each batch of images:

1. **Forward Pass**: Feed images through the model to get predictions
2. **Calculate Loss**: Compare predictions to true labels to see how wrong we are
3. **Backward Pass**: Calculate how each weight contributed to the error (using calculus!)
4. **Update Weights**: Adjust weights to reduce the error

**Why `optimizer.zero_grad()`?** PyTorch accumulates gradients by default. If we don't reset them to zero, gradients from previous batches would interfere with the current batch - like trying to navigate with directions from your last trip still on the GPS.

**Expected Output:**

```python
Epoch: 0 | Loss: 0.2145 | Train Accuracy: 93.47%
Epoch: 1 | Loss: 0.0812 | Train Accuracy: 97.53%
Epoch: 2 | Loss: 0.0565 | Train Accuracy: 98.24%
Epoch: 3 | Loss: 0.0436 | Train Accuracy: 98.66%
Epoch: 4 | Loss: 0.0355 | Train Accuracy: 98.91%
```

Notice how the loss decreases and accuracy increases with each epoch - the model is learning! The improvements get smaller over time because the easy patterns are learned first.

### 8. Test the Model

Now for the moment of truth - let's see how well our model performs on data it has never seen before!

```python
def test_model():
    # Set model to evaluation mode
    model.eval()
    
    test_acc = 0.0
    
    # Don't calculate gradients to save memory and computation
    with torch.no_grad():
        for i, (images, labels) in enumerate(testloader):
            # Move data to device
            images = images.to(device)
            labels = labels.to(device)
            
            # Forward pass
            outputs = model(images)
            
            # Calculate accuracy
            test_acc += get_accuracy(outputs, labels, BATCH_SIZE)
    
    # Calculate average accuracy
    avg_test_acc = test_acc / len(testloader)
    print(f'Test Accuracy: {avg_test_acc:.2f}%')
​
# Test the model
test_model()
```

**Key differences from training:**

* **`model.eval()`**: Tells the model we're evaluating, not training. Some layers (like Dropout, which we don't have here) behave differently during evaluation.
* **`torch.no_grad()`**: Disables gradient calculation. Since we're not updating weights during testing, we don't need gradients - this saves memory and speeds things up.
* **No backward pass**: We only do forward propagation to get predictions. No learning happens here!

**Expected Output:**

```python
Test Accuracy: 98.45%
```

**How to interpret the results:**

* **98.45% test accuracy**: Our model correctly identifies about 98 out of every 100 handwritten digits it's never seen before. Pretty impressive!
* **Compare with training accuracy (98.91%)**: The test accuracy is slightly lower than training accuracy, which is normal and expected. A small gap like this indicates healthy generalization.
*   Red flag scenarios

    :

    * Training: 99%, Test: 65% → **Severe overfitting** (the model memorized instead of learned)
    * Training: 65%, Test: 64% → **Underfitting** (the model didn't learn enough)
    * Training: 98%, Test: 98.5% → **Suspicious** (test shouldn't be better; might indicate data leakage)

Happy learning! 🎉
