---
title: "MIT 6.S191: Deep Sequence Modeling - From RNNs to Transformers"
aliases: ["Deep Sequence Modeling", "MIT 6.S191 Lecture 2", "RNNs and Transformers"]
tags: ["deep-learning", "sequence-modeling", "rnn", "lstm", "attention", "transformer", "nlp", "mit"]
date: 2025-05-07
description: "Comprehensive guide to sequence modeling in deep learning, covering RNNs, LSTMs, attention mechanisms, and transformers for processing sequential data like text and time series."
series: "MIT 6.S191 Deep Learning"
series_order: 2
---

# MIT 6.S191: Deep Sequence Modeling

*From Recurrent Neural Networks to Transformers - Understanding Sequential Data Processing*

## Introduction: The Sequential Data Challenge

While feedforward networks excel at processing fixed-size inputs independently, the real world is fundamentally sequential. Language, speech, music, financial time series, sensor data, and video all contain temporal dependencies where context and order matter critically.

Consider these examples:
- **Language**: "The bank was steep" vs "The bank was closed" - identical words, different meanings based on context
- **Music**: A sequence of notes creates melody; order determines harmonic progression
- **Finance**: Stock prices depend on historical trends and temporal patterns
- **Video**: Object motion requires understanding across multiple frames

The core limitation of standard feedforward networks is their inability to maintain **memory** - information from previous inputs that influences current processing.

## Recurrent Neural Networks: Adding Memory to Neural Networks

### The Fundamental Innovation

Recurrent Neural Networks (RNNs) introduce the concept of **internal state** - a memory mechanism that allows networks to maintain information across time steps. This transforms neural networks from stateless functions to stateful processors.

**Key Innovation**: Instead of processing inputs independently, RNNs maintain a hidden state that captures information from all previous inputs in the sequence.

### Mathematical Foundation

The RNN computation involves two key equations:

**Hidden State Update**:
```
h_t = tanh(W_hh × h_{t-1} + W_xh × x_t + b_h)
```

**Output Generation**:
```
ŷ_t = W_hy × h_t + b_y
```

Where:
- `h_t`: hidden state at time t
- `x_t`: input at time t
- `ŷ_t`: output at time t
- `W_hh`: hidden-to-hidden weights
- `W_xh`: input-to-hidden weights  
- `W_hy`: hidden-to-output weights
- `b_h`, `b_y`: bias terms

### Parameter Sharing: The Key to Scalability

A crucial insight in RNN design is **parameter sharing** - the same weight matrices are used at every time step. This provides several advantages:

1. **Variable Length Processing**: Handle sequences of any length with fixed parameters
2. **Computational Efficiency**: Fewer parameters to learn
3. **Translation Invariance**: Patterns learned at one time step apply to all time steps
4. **Generalization**: Better performance on unseen sequence lengths

### TensorFlow Implementation

```python
import tensorflow as tf

# Simple RNN layer
rnn_layer = tf.keras.layers.SimpleRNN(
    units=128,           # Hidden state dimensionality
    return_sequences=True,  # Return full sequence or just final output
    return_state=True    # Also return final hidden state
)

# Using RNN in a model
model = tf.keras.Sequential([
    tf.keras.layers.Embedding(vocab_size, embedding_dim),
    tf.keras.layers.SimpleRNN(128, return_sequences=True),
    tf.keras.layers.Dense(vocab_size, activation='softmax')
])

# Custom RNN cell implementation
class CustomRNNCell(tf.keras.layers.Layer):
    def __init__(self, units):
        super(CustomRNNCell, self).__init__()
        self.units = units
        self.state_size = units
        
    def build(self, input_shape):
        # Weight matrices
        self.W_xh = self.add_weight(
            shape=(input_shape[-1], self.units),
            initializer='glorot_uniform'
        )
        self.W_hh = self.add_weight(
            shape=(self.units, self.units),
            initializer='orthogonal'
        )
        self.b_h = self.add_weight(
            shape=(self.units,),
            initializer='zeros'
        )
    
    def call(self, inputs, states):
        prev_h = states[0]
        # RNN computation
        h = tf.nn.tanh(tf.matmul(inputs, self.W_xh) + 
                       tf.matmul(prev_h, self.W_hh) + self.b_h)
        return h, [h]  # output, new_states
```

## Language Representation for Neural Networks

### The Embedding Challenge

Neural networks process numerical vectors, but language consists of discrete symbols. Converting words to numbers requires careful consideration of semantic relationships.

### Word Embeddings: From Symbols to Vectors

**One-Hot Encoding** (Traditional Approach):
- Each word represented by a sparse vector
- Dimension equals vocabulary size
- Single '1' at word's index, zeros elsewhere
- **Limitations**: High dimensional, no semantic similarity

**Learned Embeddings** (Modern Approach):
- Dense, low-dimensional vectors (typically 50-300 dimensions)
- Learned during training to capture semantic relationships
- Similar words have similar vector representations
- Enables arithmetic operations: "king" - "man" + "woman" ≈ "queen"

```python
# Embedding layer in TensorFlow
embedding_layer = tf.keras.layers.Embedding(
    input_dim=vocab_size,    # Vocabulary size
    output_dim=embedding_dim,  # Embedding dimension
    input_length=sequence_length  # Fixed sequence length (optional)
)

# Usage in model
model = tf.keras.Sequential([
    tf.keras.layers.Embedding(10000, 128, input_length=50),
    tf.keras.layers.LSTM(64),
    tf.keras.layers.Dense(1, activation='sigmoid')
])
```

## Training RNNs: Backpropagation Through Time

### Unrolling the Recurrence

Training RNNs requires "unrolling" the recurrent computation across time, creating a deep feedforward network where each layer corresponds to a time step.

**Forward Pass Process**:
1. Initialize hidden state h₀
2. For each time step t:
   - Compute hidden state h_t
   - Generate output ŷ_t
   - Calculate loss L_t
3. Sum losses across all time steps

**Backward Pass Process**:
1. Compute gradients for each time step
2. Backpropagate errors through time
3. Accumulate gradients for shared parameters
4. Update weights using accumulated gradients

```python
# Training loop with gradient tape
optimizer = tf.keras.optimizers.Adam(learning_rate=0.001)

for batch in dataset:
    with tf.GradientTape() as tape:
        # Forward pass
        predictions = model(batch['inputs'])
        loss = loss_function(batch['targets'], predictions)
    
    # Backward pass
    gradients = tape.gradient(loss, model.trainable_variables)
    optimizer.apply_gradients(zip(gradients, model.trainable_variables))
```

### The Vanishing Gradient Problem

Standard RNNs suffer from the **vanishing gradient problem** when processing long sequences:

**Root Cause**: Gradients decay exponentially as they propagate backward through time
**Mathematical Explanation**: Gradients are repeatedly multiplied by the weight matrix W_hh
**Consequence**: Network cannot learn long-term dependencies effectively

**Gradient Decay**:
```
∂L/∂W ∝ (W_hh)^t × ∂L/∂h_t
```

When eigenvalues of W_hh are less than 1, gradients vanish exponentially with sequence length.

## Long Short-Term Memory Networks (LSTMs)

### Architectural Innovation

LSTMs solve the vanishing gradient problem through a sophisticated **gating mechanism** that controls information flow.

**Core Components**:
1. **Cell State (C_t)**: Long-term memory pathway
2. **Hidden State (h_t)**: Short-term memory and output
3. **Forget Gate**: Decides what information to discard
4. **Input Gate**: Controls new information storage  
5. **Output Gate**: Manages information retrieval

### LSTM Mathematical Formulation

**Forget Gate**:
```
f_t = σ(W_f × [h_{t-1}, x_t] + b_f)
```

**Input Gate**:
```
i_t = σ(W_i × [h_{t-1}, x_t] + b_i)
C̃_t = tanh(W_C × [h_{t-1}, x_t] + b_C)
```

**Cell State Update**:
```
C_t = f_t ∗ C_{t-1} + i_t ∗ C̃_t
```

**Output Gate**:
```
o_t = σ(W_o × [h_{t-1}, x_t] + b_o)
h_t = o_t ∗ tanh(C_t)
```

### TensorFlow LSTM Implementation

```python
# LSTM layer
lstm_layer = tf.keras.layers.LSTM(
    units=128,
    return_sequences=True,
    dropout=0.2,
    recurrent_dropout=0.2
)

# Bidirectional LSTM for capturing both forward and backward dependencies
bidirectional_lstm = tf.keras.layers.Bidirectional(
    tf.keras.layers.LSTM(64, return_sequences=True)
)

# Complete model for sequence-to-sequence tasks
model = tf.keras.Sequential([
    tf.keras.layers.Embedding(vocab_size, 256),
    tf.keras.layers.LSTM(512, return_sequences=True, dropout=0.3),
    tf.keras.layers.LSTM(512, dropout=0.3),
    tf.keras.layers.Dense(vocab_size, activation='softmax')
])
```

## Attention Mechanisms: The Breakthrough Innovation

### Limitations of RNN-based Architectures

Even LSTMs face fundamental constraints:
1. **Sequential Processing**: Cannot parallelize across time steps
2. **Information Bottleneck**: Final hidden state must encode entire sequence
3. **Long-range Dependencies**: Still difficult for very long sequences
4. **Computational Efficiency**: Training is inherently slow

### The Attention Revolution

**Core Insight**: Instead of compressing entire sequences into fixed-size representations, allow the model to **attend** to different parts of the input as needed.

**Key Innovations**:
1. **Selective Focus**: Attend to relevant information dynamically
2. **Parallelization**: Process all positions simultaneously
3. **Long-range Dependencies**: Direct connections between distant positions
4. **Interpretability**: Attention weights provide insight into model decisions

### Self-Attention Mechanism

Self-attention allows each position to attend to all positions in the sequence, including itself.

**Mathematical Framework**:
```
Attention(Q, K, V) = softmax(QK^T / √d_k)V
```

Where:
- **Q (Query)**: What information are we looking for?
- **K (Key)**: What information is available?
- **V (Value)**: The actual information content
- **d_k**: Scaling factor (key dimension)

**Multi-Head Attention**:
```python
class MultiHeadAttention(tf.keras.layers.Layer):
    def __init__(self, d_model, num_heads):
        super(MultiHeadAttention, self).__init__()
        self.num_heads = num_heads
        self.d_model = d_model
        self.depth = d_model // self.num_heads
        
        self.wq = tf.keras.layers.Dense(d_model)
        self.wk = tf.keras.layers.Dense(d_model)
        self.wv = tf.keras.layers.Dense(d_model)
        self.dense = tf.keras.layers.Dense(d_model)
    
    def split_heads(self, x, batch_size):
        x = tf.reshape(x, (batch_size, -1, self.num_heads, self.depth))
        return tf.transpose(x, perm=[0, 2, 1, 3])
    
    def call(self, v, k, q, mask=None):
        batch_size = tf.shape(q)[0]
        
        # Generate Q, K, V
        q = self.wq(q)
        k = self.wk(k)
        v = self.wv(v)
        
        # Split into multiple heads
        q = self.split_heads(q, batch_size)
        k = self.split_heads(k, batch_size)
        v = self.split_heads(v, batch_size)
        
        # Scaled dot-product attention
        scaled_attention = self.scaled_dot_product_attention(q, k, v, mask)
        
        # Concatenate heads
        scaled_attention = tf.transpose(scaled_attention, perm=[0, 2, 1, 3])
        concat_attention = tf.reshape(scaled_attention, 
                                    (batch_size, -1, self.d_model))
        
        # Final linear transformation
        output = self.dense(concat_attention)
        return output
```

## The Transformer Architecture

### Complete Architecture Overview

Transformers combine self-attention with feedforward networks and normalization to create powerful sequence models.

**Core Components**:
1. **Multi-Head Self-Attention**: Parallel attention mechanisms
2. **Position Encoding**: Inject sequence order information
3. **Layer Normalization**: Stabilize training
4. **Residual Connections**: Enable deep architectures
5. **Feedforward Networks**: Non-linear transformations

### Positional Encoding

Since attention mechanisms have no inherent notion of order, transformers inject positional information:

```python
def positional_encoding(position, d_model):
    angle_rads = np.arange(position)[:, np.newaxis] / np.power(10000, 
        (2 * (np.arange(d_model)[np.newaxis, :] // 2)) / np.float32(d_model))
    
    # Apply sin to even indices
    angle_rads[:, 0::2] = np.sin(angle_rads[:, 0::2])
    # Apply cos to odd indices  
    angle_rads[:, 1::2] = np.cos(angle_rads[:, 1::2])
    
    pos_encoding = angle_rads[np.newaxis, ...]
    return tf.cast(pos_encoding, dtype=tf.float32)
```

### Complete Transformer Implementation

```python
class TransformerBlock(tf.keras.layers.Layer):
    def __init__(self, d_model, num_heads, dff, rate=0.1):
        super(TransformerBlock, self).__init__()
        self.att = MultiHeadAttention(d_model, num_heads)
        self.ffn = tf.keras.Sequential([
            tf.keras.layers.Dense(dff, activation='relu'),
            tf.keras.layers.Dense(d_model)
        ])
        self.layernorm1 = tf.keras.layers.LayerNormalization(epsilon=1e-6)
        self.layernorm2 = tf.keras.layers.LayerNormalization(epsilon=1e-6)
        self.dropout1 = tf.keras.layers.Dropout(rate)
        self.dropout2 = tf.keras.layers.Dropout(rate)
    
    def call(self, x, training, mask):
        # Multi-head attention + residual connection + layer norm
        attn_output = self.att(x, x, x, mask)
        attn_output = self.dropout1(attn_output, training=training)
        out1 = self.layernorm1(x + attn_output)
        
        # Feedforward + residual connection + layer norm
        ffn_output = self.ffn(out1)
        ffn_output = self.dropout2(ffn_output, training=training)
        out2 = self.layernorm2(out1 + ffn_output)
        
        return out2

# Complete Transformer model
class Transformer(tf.keras.Model):
    def __init__(self, num_layers, d_model, num_heads, dff, 
                 input_vocab_size, maximum_position_encoding, rate=0.1):
        super(Transformer, self).__init__()
        self.d_model = d_model
        self.embedding = tf.keras.layers.Embedding(input_vocab_size, d_model)
        self.pos_encoding = positional_encoding(maximum_position_encoding, d_model)
        
        self.enc_layers = [TransformerBlock(d_model, num_heads, dff, rate) 
                          for _ in range(num_layers)]
        self.dropout = tf.keras.layers.Dropout(rate)
    
    def call(self, x, training, mask):
        seq_len = tf.shape(x)[1]
        
        # Embedding + positional encoding
        x = self.embedding(x)
        x *= tf.math.sqrt(tf.cast(self.d_model, tf.float32))
        x += self.pos_encoding[:, :seq_len, :]
        
        x = self.dropout(x, training=training)
        
        # Pass through encoder layers
        for i in range(len(self.enc_layers)):
            x = self.enc_layers[i](x, training, mask)
        
        return x
```

## Practical Applications and Examples

### Language Modeling Example

```python
# Prepare text data
def create_sequences(text, seq_length):
    sequences = []
    for i in range(len(text) - seq_length):
        sequences.append(text[i:i + seq_length + 1])
    return sequences

# Build dataset
def build_dataset(sequences, tokenizer):
    dataset = tf.data.Dataset.from_generator(
        lambda: sequences,
        output_signature=tf.TensorSpec(shape=(None,), dtype=tf.int32)
    )
    
    dataset = dataset.map(lambda x: (x[:-1], x[1:]))  # Input/target pairs
    dataset = dataset.batch(32).prefetch(tf.data.AUTOTUNE)
    return dataset

# Training configuration
model = Transformer(
    num_layers=6,
    d_model=512,
    num_heads=8,
    dff=2048,
    input_vocab_size=vocab_size,
    maximum_position_encoding=1000
)

optimizer = tf.keras.optimizers.Adam(
    learning_rate=CustomSchedule(d_model),
    beta_1=0.9,
    beta_2=0.98,
    epsilon=1e-9
)
```

### Sequence Classification Example

```python
# Sentiment analysis with LSTM
def build_sentiment_model(vocab_size, max_length):
    model = tf.keras.Sequential([
        tf.keras.layers.Embedding(vocab_size, 128, input_length=max_length),
        tf.keras.layers.LSTM(64, dropout=0.3, recurrent_dropout=0.3),
        tf.keras.layers.Dense(32, activation='relu'),
        tf.keras.layers.Dropout(0.5),
        tf.keras.layers.Dense(1, activation='sigmoid')
    ])
    
    model.compile(
        optimizer='adam',
        loss='binary_crossentropy',
        metrics=['accuracy']
    )
    return model
```

## Comparing Architectures: RNNs vs LSTMs vs Transformers

| Aspect | RNNs | LSTMs | Transformers |
|--------|------|-------|--------------|
| **Long-term Dependencies** | Poor | Good | Excellent |
| **Training Parallelization** | Sequential | Sequential | Parallel |
| **Memory Efficiency** | Low | Medium | High (with optimizations) |
| **Computational Speed** | Slow | Medium | Fast (parallel) |
| **Interpretability** | Limited | Limited | High (attention weights) |
| **Parameter Efficiency** | High | Medium | Lower |

## Modern Applications and Impact

### Language Models and AI Applications

**Pre-trained Language Models**:
- **BERT**: Bidirectional understanding for many NLP tasks
- **GPT Series**: Autoregressive generation and instruction following
- **T5**: Text-to-text unified framework
- **Modern LLMs**: ChatGPT, Claude, Gemini built on transformer architectures

**Real-world Applications**:
- Machine translation (Google Translate, DeepL)
- Question answering systems
- Code generation (GitHub Copilot, CodeT5)
- Content creation and summarization
- Conversational AI assistants

### Beyond Natural Language Processing

**Computer Vision**: Vision Transformers (ViTs) for image classification
**Protein Folding**: AlphaFold's attention mechanisms
**Reinforcement Learning**: Decision transformers for sequential decision making
**Time Series Analysis**: Financial forecasting, sensor data processing
**Speech Recognition**: End-to-end speech-to-text systems

## Key Insights and Future Directions

### Fundamental Principles
1. **Memory is Essential**: Sequential data requires maintaining context across time
2. **Attention > Recurrence**: Direct connections outperform sequential processing
3. **Parallelization Enables Scale**: Training speed determines practical applicability
4. **Architecture Matters**: Design choices significantly impact performance

### Current Challenges and Research Directions

**Efficiency Improvements**:
- Linear attention mechanisms
- Sparse attention patterns
- Memory-efficient architectures
- Hardware-optimized implementations

**Long Context Understanding**:
- Extended context windows (100K+ tokens)
- Hierarchical attention mechanisms
- Memory-augmented transformers
- Retrieval-augmented generation

**Multimodal Integration**:
- Vision-language models
- Audio-text processing
- Cross-modal attention mechanisms
- Unified multimodal architectures

## Summary and Next Steps

Sequence modeling represents one of the most impactful areas in modern deep learning. The evolution from RNNs to LSTMs to Transformers demonstrates the field's rapid progress in solving fundamental computational challenges.

**Key Takeaways**:
1. **RNNs** introduced the concept of memory in neural networks
2. **LSTMs** solved the vanishing gradient problem through gating mechanisms
3. **Attention** eliminated the need for sequential processing
4. **Transformers** became the foundation for modern AI systems

The principles learned in sequence modeling extend far beyond natural language processing, influencing computer vision, reinforcement learning, and scientific computing.

### Continue Your Deep Learning Journey

1. **[[MIT-6S191-Convolutional-Neural-Networks|Convolutional Neural Networks]]**: Specialized architectures for spatial data
2. **[[MIT-6S191-Deep-Generative-Modeling|Deep Generative Modeling]]**: Creating new data with neural networks
3. **[[MIT-6S191-Deep-Reinforcement-Learning|Deep Reinforcement Learning]]**: Learning through interaction
4. **[[Technical/Programming/NVIDIA-CUDA-From-History-to-AI-Revolution|NVIDIA CUDA: From History to AI Revolution]]**: Optimizing sequence model training

## Related Resources

- **[[Learning/Huberman/dopamine-motivation-systems|Dopamine and Motivation Systems]]**: Biological inspiration for sequential processing
- **[[Technical/Programming/NVIDIA-CUDA-From-History-to-AI-Revolution|NVIDIA CUDA: From History to AI Revolution]]**: The infrastructure enabling modern sequence models

---

*This article is part of the MIT 6.S191 Deep Learning Series. Practice implementing these concepts with the course labs at [introtodeeplearning.com](http://introtodeeplearning.com).*